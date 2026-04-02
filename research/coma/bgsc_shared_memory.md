# BGSC shared memory protocol

**context:** this document covers the shared memory layout used by the BGSC
(background scheduler) DSP framework. the shared memory is the primary
interface between the CSS and ARM processors for codec processing. for the
overall architecture, see [voice_service_and_app_dsp.md](voice_service_and_app_dsp.md).
for the ring buffer message protocol, see [ring_buffer_protocol.md](ring_buffer_protocol.md).
for what needs to be written to element descriptors, see
[codec_table_investigation.md](codec_table_investigation.md).

**note:** this document contains both well-established findings and evolving
investigation notes. sections marked "confirmed" have been validated on hardware.
the later sections (dispatch protocol, status) reflect iterative progress and
may contain superseded information.

## shared memory layout (from live dumps + static analysis)

the shared memory region is mapped via `/dev/sharedmem` (ioctl 0xc0045302).
size: 0x100000 (1MB). accessible only via mmap (no read/write on chardev).

### pool header (offset 0x00 - 0x2f)

```
+0x00: uint32  (unused, zero)
+0x04: uint32  pool_size        (0x80000 = 512KB, written by dua_init_hw)
+0x08: uint32  alloc_watermark  (advances as CSS + app_dsp allocate)
+0x0c: uint32  (zero)
+0x10: uint32  dsp_version_1    (0x11400154 — contains DSP version 0x1140)
+0x14: uint32  dsp_version_2    (same — written by app_dsp after codec init)
+0x18: uint32  (0xb850)
+0x1c: uint32  alloc_granularity (0x30, written by dua_init_hw at +8... 
                                  but later the CSS moves the alloc ptr forward)
+0x20: ...
+0x28: uint32  saved_alloc_ptr  (where app_dsp found the alloc ptr)
+0x2c: uint32  codec_state_ptr  (start of app_dsp's codec state allocation)
+0x30: uint32  session_active   (0 = idle, 1 = voice session active) ← CHANGES
```

note: the pool header offsets are approximate. the CSS advances +0x08 (alloc
watermark) during DUA init as it allocates internal structures. by the time
app_dsp reads it, the watermark is already at ~0x1ae54.

### app_dsp allocations (from p_dsp_app_startup @ 0x0007e7c4)

app_dsp allocates two regions from the pool:

1. **ring buffer header** — 0x10 bytes, at current alloc_watermark
2. **codec state area** — 0x79c bytes (1948 bytes), immediately after

in our live dump, the codec state starts at offset 0xbe14.

### codec state area (0xbe14 - 0xc5b0 in our dump)

when a voice session is active with L16/16000:
- **0xbe14 - 0xc094**: filled with 16-bit signed PCM samples (the audio buffer!)
  - 640 bytes of audio = 320 samples = 20ms at 16kHz ← one L16 frame
  - data is little-endian signed 16-bit, centered around zero
  - ~4 frames visible (640 * 4 ≈ 2560, matches the area that fills up)
- **0xc094 onwards**: remains zero

when idle: entire region is zero.

## control word changes during voice session start

diffing shared memory before/during an active L16 session reveals exactly
which words the CSS flips:

| offset | before | during | interpretation |
|--------|--------|--------|---------------|
| 0x0030 | 00 | 01 | global "session active" flag |
| 0x0534 | 00 | 01 | per-unit control word (enc/dec active?) |
| 0x0538 | 00 | 01 | per-unit control word |
| 0x0778 | 00 | 01 | FTAB control word (codec activated) |
| 0x0b54 | 00 | 01 | per-unit control word |
| 0x0b58 | 00 | 01 | per-unit control word |
| 0x0b64 | 01 | 04 | state: IDLE(1) → RUNNING(4)? |
| 0x0b80 | 00 | 01 | FTAB control word (codec activated) |

### incrementing counters (active during session)

region 0xb710-0xb840 contains values that change between dumps:
```
before: 38 16 a9 00  →  during: fe 53 bb 00   (at +0xb718)
before: 50 15 a9 00  →  during: 16 53 bb 00   (at +0xb748)
before: 72 14 a9 00  →  during: 36 52 bb 00   (at +0xb778)
...pattern continues every 0x30 bytes
```
these are 32-bit counters incrementing at roughly the same rate — likely
per-channel sample counters or timestamp values updated by the BGSC threads.

### function pointer update

offset 0x0778 shows a CSS function pointer change during session:
```
before: 94 1e d7 b6  14 1e d7 b6
during: 8c 20 d7 b6  54 1f d7 b6
```
this is in an FTAB-like region — the CSS swapped the codec function pointers
to point to the L16 encoder/decoder functions.

## FTAB entry format (from FUN_000885b0 decompilation)

each FTAB entry is 16 bytes (4 x uint32):

```c
struct ftab_entry {
    uint32_t *control_word;  // pointer to control word in shared memory
    void     *param1;        // I/O buffer 1 (input or output)
    void     *param2;        // I/O buffer 2
    void     *calc_func;     // codec processing function (ARM address space)
};
```

the list is terminated by a NULL control_word pointer.

### control word bit meanings (from dispatch loop)

```
bit 0 (0x01):  ACTIVE — codec function is called with full vtable dispatch
bit 1 (0x02):  REPEAT — stay on current entry, don't advance to next
bit 2 (0x04):  (part of "running" check with bit 1)
bit 3 (0x08):  POST_CALC_1 — call handler at calc_func-0x0c after main calc
bit 4 (0x10):  POST_CALC_2 — call handler at calc_func-0x08 after main calc
bit 5 (0x20):  PREPARE_RUN — I-switch activation pending (BG level startup)
bit 7 (0x80):  ALT_PREPARE — alternate activation for codec re-start
bits 8-9 (0x300): MODE_FLAGS — affects which prepare bit is used
bit 9 (0x200): used in I-switch setup when codec not yet active
```

after dispatch, control word is set to `1` if bits 1 or 2 were set, else `0`.

### dispatch logic pseudocode

```c
void dfl_process_flow(ftab_entry *list) {
    ftab_entry *cur = list;
    while (cur->control_word != NULL) {
        uint32_t cw = *cur->control_word;
        if (cw & 1) {
            // ACTIVE: full dispatch
            cur->calc_func(cw, cur->param1, cur->param2);
        } else if ((cw & 0x1f) == 0) {
            // inactive, no pending bits: call alternate entry
            (cur->calc_func - 4)();
        } else {
            // pending: call pre-calc
            (cur->calc_func - 4)(cur->control_word, cur->param1, cur->param2);
            if (cw & 8) (cur->calc_func - 0x0c)();  // post-calc 1
            if (cw & 0x10) (cur->calc_func - 0x08)();  // post-calc 2
        }
        // advance or repeat
        if (cw & 2) stay on current entry;
        else advance to next entry;
        // update control word
        *cur->control_word = (cw & 6) ? 1 : 0;
    }
}
```

## dfl_calc state machine (BG level processing, FUN_0007e968)

each BG level has a DFL_PROCESSING struct (0x1c bytes):

```c
struct dfl_processing {
    void    *flow_table;      // +0x00: pointer to flow table in shared memory
    void    *flow_table_copy; // +0x04: same pointer (used for reset)
    int      level_index;     // +0x08: BG level number (0-3)
    int      state;           // +0x0c: state machine state
    int      field_10;        // +0x10: -1 initially
    int      field_14;        // +0x14: 0 initially
    uint32_t tick_count;      // +0x18: from flow_table + 0x18
};
```

state machine progression: `2 → 3 → 4 → 5 → 0x200 → 1 (IDLE)`

- states 2-3: simple transitions
- state 4 (level 0) / state 3 (levels 1-3): I-switch setup — iterates FTAB
  entries and sets PREPARE_RUN bits (0x20 or 0x80) based on the I-switch
  bitmap in the flow table
- state 5/0x200: transitions to IDLE, sends ARM→CSS message `[0, 1, level, 3]`
  via ring buffer
- state 1 (IDLE): dispatches FTAB entries via dfl_process_flow

for BG level 0 only: periodic heartbeat every N ticks via
`FUN_000841dc(0, 4, 0, NULL)` — sends a tick message to CSS.

## ring buffers (CSS↔ARM messaging)

two ring buffers in shared memory, each with header:
```c
struct ring_buf {
    uint32_t *base;
    uint32_t *end;
    uint32_t *write_ptr;
    uint32_t *read_ptr;
};
```

message format: `[instance_id, (type << 13) | num_params, param0, ...]`

### CSS→ARM messages
written by CSS via `adsp_msg_put_short/long`, read by ARM in dfl_calc.
triggers state transitions and codec activation.

### ARM→CSS messages  
written by ARM via `FUN_0008438c` (short) / `FUN_000841dc` (long).
signals codec completion and heartbeat ticks.

known messages:
- `[0, 1, level, 3]` — "BG level ready" (sent after state machine completes)
- `[0, 4, 0, NULL]` — heartbeat tick (BG level 0 periodic)
- `[0, 3, 2, &data]` — cleanup/flush (sent when field_14 != 0)

## codec function registration

during `p_dsp_app_startup` → `FUN_00075810`, app_dsp writes codec function
pointers to shared memory tables. the CSS reads these when creating FTAB
entries for a codec channel. each codec module (G1E, G1D, L16_enc, L16_dec,
etc.) has a startup function that registers its calc function address.

for our BGSC replacement, we need to register our own L16 calc function at
the same shared memory location so the CSS uses our address in FTAB entries.

**key unknown**: the exact shared memory offsets where codec function pointers
are registered. needs more RE of FUN_00075810 and FUN_000844f8/FUN_00084508.

## critical finding: FTAB lives in ARM process memory, NOT shared memory

a comprehensive search of shared memory during an active L16 voice session found
**zero** app_dsp function pointers (0x10000-0xae000 range) in shared memory.
the changes in the 0xbe14+ region during a session are audio PCM samples, not
pointers.

this means: **the flow tables (FTAB entries with calc_func pointers) are in
app_dsp's process-local memory (heap/BSS), not in the CSS shared memory pool.**

the CSS communicates with the ARM BGSC only via:
- **I-switch bitmaps** in shared memory (control word bits)
- **ring buffer messages** (CSS→ARM and ARM→CSS)
- **shared audio buffers** (PCM samples at 0xbe14+)

the CSS does NOT read or write function pointers. it signals "start codec N"
via I-switch bits. the ARM side looks up the codec function from its own local
tables and processes the audio.

### confidence summary (as of 2026-04-02)

**confirmed on hardware:**
1. shared memory pool header layout (+0x04 size, +0x08 watermark, +0x14 version)
2. app_dsp allocates 0x10 (ring buf hdr) + 0x79c (codec state) from the pool
3. audio samples appear at the codec state offset during active sessions
4. control words in shared memory flip specific bits when sessions start/stop
5. the CSS sends "start" signals via I-switch bitmaps, not function pointers
6. ring buffer message format: [instance, (type<<13)|nparams, params...]
7. BG level architecture: CSS runs level 0, ARM runs levels 1-3

**confirmed via static analysis:**
8. the FTAB dispatch loop code (fully decoded from disassembly, see below)
9. WAE_startup is trivial — L16 has no complex init
10. there is no L16-specific codec module — L16 is handled by WAE/WAD
11. the CSS has identical dfl_process_flow code (see [css_level0_dispatch.md](css_level0_dispatch.md))

**partially understood (active investigation):**
- how `dfl_module_startup` populates element descriptors — see
  [codec_table_investigation.md](codec_table_investigation.md)
- whether a CSS→ARM ring buffer exists alongside the COMA socket events —
  see [ring_buffer_protocol.md](ring_buffer_protocol.md)
- whether the module readiness cascade fully completes —
  see [css_arm_events.md](css_arm_events.md)

**not yet investigated:**
- how the CSS tells the ARM WHICH codec to use (codec type → FTAB entry mapping)
- the exact tick rates for BG level threads
- what the WAD/WAE calc functions actually do with their I/O buffers

## FTAB structure (from process memory dump)

the FTAB lives at 0xbee7c in app_dsp's data+bss segment (NOT in shared memory).
320 entries, NULL-terminated. organized as 16 groups of 20 entries each (one
group per SPVOIPNDA unit).

each group of 20 entries:
- entry[0]: WAD/L16 codec (calc_func = pre_dsp_wad @ 0x89ed8 — a NO-OP!)
- entries[1-19]: signal processing helpers (from dfl_asm2c_shell vtables)

the helper functions are in a branch table at 0x7f19c (section `dfl_asm2c_shell`).
each "function" is a 16-byte vtable of 4 ARM branch instructions:
```
+0x00: b <codec_calc>      (main processing)
+0x04: b <codec_startup>    (initialization)
+0x08: b <codec_reset>      (reset handler)
+0x0c: b <codec_off>        (cleanup/shutdown)
```

the dispatch loop uses negative offsets from the calc_func pointer to access
different handlers — this matches the decompiled FUN_000885b0 behavior.

### L16 codec entry: literally a no-op

the FTAB entry for L16/WAD points to 0x89ed8 which disassembles to:
```
89ed8: e12fff1e   bx lr    ; return immediately
```

this confirms: **for L16, the ARM codec function does nothing.** the CSS
signal processing handles L16 data movement internally. the ARM just needs to
maintain the BGSC infrastructure (state machine, control word processing,
heartbeat) — the L16 "codec" is a pure no-op.

### helper function types (12 unique, by trampoline target)

| address | count | trampoline to | likely purpose |
|---------|-------|---------------|---------------|
| 0x7f2d8 | 48x | 0x879a8 | most common helper |
| 0x7f1c8 | 32x | G1D_calc(0x7f9ec) | G.711 decoder |
| 0x7f1e8 | 32x | G1E_startup area | G.711 encoder |
| 0x7f228 | 32x | various | codec helper |
| 0x7f248 | 32x | various | codec helper |
| 0x7f2b8 | 32x | various | signal routing |
| 0x7f2a8 | 16x | 0x8850c | signal output |
| ... | ... | ... | ... |

## element table discovery (runtime)

the element descriptor table is ALWAYS at **shm+0xb854** regardless of boot.
368 entries, each a userspace virtual address (our mmap base + offset) pointing
to a DSP element descriptor in shared memory. the first word of each descriptor
is the control word.

the within-group offsets and group stride vary between boots (depend on CSS
allocation order during DUA init), so control word locations MUST be read from
this table at runtime.

verified: the table uses OUR process's mmap base address (set during DUA InitReq).

## CSS text trace (invaluable for debugging)

enable via css_shell: `text set 1 255 1` (module 1 = rtp, level 255, print=yes).
shows detailed flow through voice_start_rtp, p_rtpapp_Start, ENCDECstart, etc.

key findings from trace during SETCODEC:
- CSS recognizes L16/16000, codec type 0x61
- AUC channels allocated successfully (enc=5, dec=4)
- state: ST_RTP_FREE → ST_RTP_STARTING → ST_RTP_DUA_CB_START_WAIT → ST_RTP_STARTING
- p_rtpapp_ENCDECstart runs successfully, allocates JIB instance
- THEN: silence for 3 seconds → timeout
- the CSS is stuck AFTER ENCDECstart, at the AUCChannelPairStart step

## dispatch protocol — what we know works and what's missing

### what works:
- ring buffer: CSS reads our "level ready" messages (read ptr advances)
- element table: 368 control words read correctly at runtime from shm+0xb854
- control word dispatch: our loop catches CSS writes (0x14, 0x12, 0x1)
- clearing control words per protocol: `(val & 6) ? 1 : 0`

### what's missing:
- clearing control words alone is NOT sufficient
- the stock dispatch calls actual functions when processing entries:
  - for bits 1-4 set: calls `calc_func - 4` (main processing) with params
  - for bit 3: calls `calc_func - 0xc` (post-calc 1)
  - for bit 4: calls `calc_func - 0x10` (post-calc 2)
- these functions update DSP element state beyond just the control word
- the CSS checks element state (not just control word) to detect completion
- the dsp_sko function at +0x08 reads `element+4` and does computation
- writing `1` to `element+4` alone doesn't help either

## status as of 2026-04-01 (session 2: BGSC rewrite)

### completed RE work

the dispatch function (dfl_process_flow_asm at 0x885b0) has been **fully
decoded** from disassembly. register usage:

```
stmdb sp!, {r0-r12,lr}       ; save all, r0 (ftab ptr) at [sp+0]
loop_top:
  ldr r4, [sp, #0]            ; r4 = current ftab position
  adr lr, loop_top             ; LR = loop top (for tail calls)
  ldmia r4!, {r0,r1,r2,r3}    ; load FTAB entry, r4 advances
  cmp r0, #0                   ; null terminator?
  ldr r5, [r0], #4             ; r5 = *cw, r0 = cw+4
  str r4, [sp, #0]             ; save advanced position
  
  tst r5, #1
  bxne r3                      ; ACTIVE: tail-call calc_func (+0x0c)
  sub r0, r0, #4               ; restore r0 = cw
  sub r6, r3, #4               ; r6 = calc_func-4 (+0x08)
  tst r5, #0x1f
  bxeq r6                      ; INACTIVE: tail-call calc_func-4 (Z flag set)
  
  ; PENDING: call with params, then post-calc
  stmdb sp!, {r0,r3,r4,r5}
  blx r6                        ; calc_func-4(cw, param1, param2)
  ldmia sp!, {r0,r3,r4,r5}
  tst r5, #8  → calc_func-0x0c  ; POST_CALC_1
  tst r5, #0x10 → calc_func-0x08 ; POST_CALC_2 (startup)
  *cw = (val & 6) ? 1 : 0
  b loop_top
```

the FTAB entry structure (confirmed from live process memory dump):
```c
struct ftab_entry {  /* 16 bytes */
    uint32_t *cw_ptr;    /* pointer to control word in shared memory */
    void     *param1;    /* I/O buffer or BG level struct addr */
    void     *param2;    /* same */
    void     *calc_func; /* vtable +0x0c entry */
};
```

320 entries = 16 groups × 20 per SPVOIPNDA unit. entries within each group:
- entry[0]: codec (dsp_sko for L16: calc_func = 0x89ed8, bx lr noop)
- entries[1-6]: helpers (unique shm control words)
- entry[7]: buffer helper (params = BG level struct addr, unique cw)
- entries[8-13]: DUPLICATES of entries 1-6 (same cw pointers!)
- entry[14]: duplicate of entry 1
- entries[15-19]: helpers with different trampolines than 1-6

live FTAB dump confirmed: only entries 0 and 7 have non-zero param1/param2.

### L16 calc functions (all confirmed via decompilation)

**dsp_sko vtable at 0x89ecc:**
- +0x00 (init): branch to init handler
- +0x04 (startup): bx lr (noop, FUN_00089ed0)
- +0x08 (calc): dsp_sko_calc at 0x89edc — conditional accumulator
  reads elem+4 (NOT +8 as previously assumed!), accumulates r6 += elem+4 << 4
  uses EQ/NE flags from dispatch to handle INACTIVE vs PENDING paths
  for dsp_sko, element+4 is 0 for inactive entries → noop
- +0x0c (active): bx lr (noop, FTAB calc_func points here)

**helper trampoline vtables at 0x7f19c (16 bytes each, 4 branches):**
- +0x00: b <main_calc>
- +0x04: b <startup>      ← called as POST_CALC_2 when bit 4 set
- +0x08: b <pending_calc>  ← called for INACTIVE and PENDING paths
- +0x0c: b <active_handler> ← FTAB calc_func points here

active handlers for L16 (called every dispatch cycle when cw==1):
- entry 6 (encoder, 0x87B1C): `b __aeabi_memclr` with r1=param1=0 → noop
- entry 7 (buffer, 0x87970): `strb r1,[r0,#0x18]; bx lr` → 1 byte write
- entry 19 (decoder, 0x7f2d4→0x8857c): checks elem+8 bit 0, sends type=1 ack

pending calc functions (called during activation when bits 1-4 set):
- entry 6 (encoder, 0x87620): `ldmia sp!,{r4,r5}; b FUN_00086f50`
  CORRUPTS dispatch stack (pops 2 of 4 saved regs), then does large encoder init.
  FUN_00086f50 writes ~30 uint16 fields to encoder element in shared memory.
- entry 7 (buffer, 0x871C0): `str r1,[r0,#4]; bx lr` → writes param1 to elem+4
- entry 19 (decoder, 0x879A0): `str r1,[r0,#0x20]; bx lr` → writes param1 to elem+0x20

POST_CALC_2 startup functions (called when bit 4 set during activation):
- all three (0x88514, 0x88548, 0x884c8): same pattern — check elem+8 bit 0,
  send type=1 [elem+8_addr, (1<<13)|0], clear elem+8. during initial activation
  elem+8 is 0, so no message is sent. buffer startup also writes 0xff/0xfe
  to elem+0x1e/+0x1f.

### what works (current BGSC rewrite)
- SETCODEC succeeds (requires type=0 completion events during activation)
- frame production (requires periodic type=3 decoder events every ~10ms)
- encoder element initialized via encoder_init() replicating FUN_00086f50
- ring buffer healthy (CSS reads our messages)
- voice_tap receives 652-byte L16 packets

### what doesn't work
- audio content is silence (TDM audio not reaching encoder input)
- elem[21]+8 (frame counter) stays at 0 — CSS doesn't use counter handshake
- elem[22]+4 and +8 are identical pointers (stock differs by ~0x140)

### element state comparison (stock vs ours, active session)

| elem | field | stock | ours | notes |
|------|-------|-------|------|-------|
| 0 | +4 | 0x0e | 0x00 | CSS writes in stock after successful start |
| 6 | +4 (u32) | 0x1 | 0x10001 | init writes u16(1) at +4 AND +6; stock clears +6 later |
| 6 | +8 (u32) | 0x1 | 0x100000 | init writes u16(0) at +8, u16(0x10) at +a; CSS rewrites |
| 21 | +4 | 0x1 | 0x0 | CSS writes in stock |
| 21 | +8 | counter | 0x0 | frame counter never starts in our system |
| 22 | +4 vs +8 | differ by ~0x140 | identical | dual-buffer not differentiated |

### corrected false assumptions from previous session

1. **"stock doesn't send type=0 during activation"** — PARTIALLY WRONG.
   static analysis of dsp_sko shows no ring buffer messages, but the enc_pending_calc
   corrupts the dispatch stack causing unpredictable post-calc handler behavior.
   empirically, type=0 events ARE required to unblock SETCODEC.

2. **"type=3 decoder events are spurious"** — WRONG. they are required to
   kick-start frame production. without them, CSS never begins producing frames.
   the type=1 decoder ack (from FUN_0008857c) handles per-frame handshake AFTER
   production starts, but type=3 starts the production.

3. **"accumulator reads elem+8"** — WRONG. dsp_sko_calc reads elem+4 (not +8).
   the accumulation is `r6 += elem[+4] << 4`. for inactive entries elem+4 is 0,
   making it a noop. the accumulator is stored at [sp+0] which overlaps the
   FTAB pointer — this works because the accumulation adds 0.

4. **"CSS→ARM communication is via ring buffer"** — PARTIALLY CLARIFIED.
   CSS→ARM communication uses at least TWO channels: (a) DUA async callbacks
   (cmd=0x7f) on the COMA socket — confirmed for cascade events 0xf2 and 0x0e
   (see [css_arm_events.md](css_arm_events.md)), and (b) control word bits in
   shared memory. stock app_dsp also reads a CSS→ARM ring buffer in `dfl_calc`
   — whether this exists as a separate channel or is the same as (a) is still
   unclear and needs investigation.

### what to focus on next

the audio silence is the remaining blocker. possible causes:
1. CSS DSP pipeline audio routing not configured — check DUA setup vs stock
2. missing element state that configures signal routing (e[0]+4=0x0e etc.)
3. BG level state machine (I-switch setup) might configure routing on first session
4. elem[22] dual-buffer issue — +4/+8 should differ but don't
5. run stock app_dsp with our DUA init (--no-bgsc) to isolate BGSC vs DUA issue
behavior.
