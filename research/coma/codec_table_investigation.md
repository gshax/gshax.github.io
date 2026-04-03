# codec table investigation (2026-04-01)

**context:** this is part of the audio silence investigation. the silence has
two potential contributing factors: (1) element descriptor registration (this
doc) and (2) the module readiness cascade completion
([module_readiness_cascade.md](module_readiness_cascade.md)). these are not
independent — the cascade may depend on descriptor data, and RouteCODEC may
depend on both. see [css_level0_dispatch.md](css_level0_dispatch.md) for how
the CSS uses descriptor data during audio routing.

## the root cause of audio silence

**updated 2026-04-02:** verified on hardware that TDM hardware registers and
ISR counters are identical between stock and ours. the cascade and RouteCODEC
also work identically (DRT=0x01 in both cases). the silence is now confirmed
to be **purely a CSS level 0 dispatch issue**: either element descriptors are
missing data that the CSS calc functions need for signal routing, or our ARM
dispatch is interfering with CSS level 0 dispatch on shared control words.

`dfl_module_startup` (FUN_00075810, 36KB, 21 callees) writes extensively to
element descriptors in shared memory during stock app_dsp's startup. these
writes register codec modules by populating fields in each element descriptor
beyond the control word. without this data, the CSS level 0 dispatch's signal
routing functions (SSW, SSR, SU2) don't know how to route TDM audio to the
encoder input buffer.

**additionally**, our ARM dispatch currently iterates ALL elements including
CSS-level-0 ones. stock app_dsp only dispatches ARM-level elements (levels 1-3).
this means our dispatch may race with CSS level 0 dispatch on shared control
words, corrupting the CSS state machine and preventing proper audio routing.
both issues likely need to be fixed.

**evidence:** diffing idle shared memory between stock app_dsp and our BGSC
shows **2137 lines** of stock-specific non-zero data. the diff is saved on
the device at `/root/shm_diff.txt`. stock dumps at `/root/shm_baseline.bin`
(idle) and `/root/shm_active.bin` (L16 session with audio).

## architecture: how the codec table works

the element descriptors in shared memory serve dual purpose:
1. **control words** (first uint32): CSS↔ARM protocol flags for dispatch
2. **codec registration data** (remaining fields): module configuration that
   the CSS reads to know which codecs are available and how to route audio

stock `app_dsp` "publishes" ARM-side function pointers into the element
descriptors. the CSS stores these as opaque tokens. during dispatch, the CSS
passes them back to the ARM, which interprets them as function pointers and
jumps to them. this is an optimization — the same dispatch mechanism would
work if the codec ran on the CSS itself.

**key insight:** the CSS never dereferences these pointers. it uses them as
opaque handles. for L16 (where all codec functions are noops), we don't need
real function pointers — we just need the CSS to see a properly populated
descriptor structure.

## what we know about the element descriptor layout

from stock shm dumps, each element descriptor extends well beyond the control
word. for group 0's elem[0] (shm+0x30), the stock data spans ~0x70 bytes:

```
+0x00: 01 00 00 00  control word (ACTIVE)
+0x04: 0e 00 00 00  codec config (0x0e — set by dfl_module_startup)
+0x08: 00 00 00 00
+0x0c: 00 00 00 00
+0x10: 02 00 00 00  (stock=2, ours=1 — BG level count or module flag)
+0x14: [shm ptr]    CSS-allocated (varies per boot)
+0x18: [shm ptr]    CSS-allocated
+0x1c: [process addr 0x125060]  ← dfl_module_startup writes
+0x20: [shm ptr]    CSS-allocated
+0x24: [process addr 0x0c3ff8]  ← dfl_module_startup writes (BG level struct area)
+0x28: 00 00 f0 00  config value
+0x2c-0x90: mix of zeros, config values, process addresses
```

the process-local addresses (0x000cxxxx, 0x0011xxxx) are from stock app_dsp's
BSS/data segments. they appear at regular positions within each element's
extended descriptor.

## element descriptor data pattern (per group)

the 24 elements per group have different descriptor sizes and contents.
examining stock baseline (group 0):

- **elem[0] (codec, shm+0x30):** ~0x60 bytes of data. has process addrs at
  +0x1c, +0x24, +0x78. has config values at +0x28 (0xf00000), +0x68 (0xa0).
  key: +0x04 = 0x0e, +0x10 = 0x02.

- **elem[1] (helper, shm+0x38):** shorter descriptor. has +0x08=1 initially.

- **elem[2-5] (helpers):** similar to elem[1], varying config values.
  elem[2] at shm+0xa8 has extended data starting around +0x10.

- **elem[6] (encoder, shm+0x534):** large descriptor (~0xc0 bytes).
  contains the encoder init values (0x7fa1, 5000, 6000, etc.) at offsets
  +0x30 onwards (NOT at +0x04 as we incorrectly assumed — there's a ~0x2c
  byte header of pointers and config before the codec init fields).
  stock idle: +0x10 = [04 00 04 00], many process addresses.

- **elem[7-15]:** CSS-internal elements, not in FTAB. smaller descriptors.

- **elem[16-20] (helpers):** similar structure to elem[2-5].

- **elem[21] (decoder, shm+0xb54):** moderate descriptor.
  stock idle: +0x10 = [01 00 01 00].

- **elem[22] (buffer, shm+0xb80):** has pointer fields at +4/+8/+c/+10.
  during active session, +4 shifts by 0x140 from +8 (dual-buffer setup).

- **elem[23]:** CSS-internal, cw=1 at idle.

## what the encoder init writes (FUN_00086f50 field offset correction)

our encoder_init was writing to WRONG OFFSETS. the function receives cw_ptr
but the codec init fields (0x7fa1, 5000, etc.) appear at elem+0x42, +0x58,
etc. in stock — offset by ~0x2c from what the decompilation says.

this may be because:
1. FUN_00086f50 receives a pointer offset from cw_ptr (not cw_ptr itself)
2. the enc_pending_calc's stack manipulation modifies r0 before the call
3. the decompilation is correct but the CSS overwrites the init values

stock active encoder (shm+0x534) shows the init values at these offsets:
```
+0x42: a1 7f = 0x7fa1 (threshold)
+0x44: 28 00 = 0x28 (sVar1 << 4 with sVar1=2.5? unclear)
+0x46: 40 01 = 0x140
+0x48: 40 01 = 0x140
+0x4a: 28 00 = 0x28
+0x4c: b4 00 = 0xb4
+0x50: 14 00 = 0x14
+0x54: 28 00 = 0x28
+0x58: 88 13 = 0x1388 = 5000
+0x5a: 88 13 = 0x1388 = 5000
```

these are at +0x2c MORE than the decompilation offsets (+0x16, +0x18, +0x2c).
the base for FUN_00086f50's writes is likely cw_ptr + 0x2c, not cw_ptr.

## FTAB calc_func pointer layout (CORRECTED)

**critical correction:** for helper trampolines, calc_func in the FTAB points
to trampoline **+0x00**, NOT +0x0c like dsp_sko.

dsp_sko: calc_func = vtable+0x0c (the noop at the end)
helpers: calc_func = vtable+0x00 (the init/main entry at the start)

this means dispatch paths for helpers are:
- ACTIVE: calls trampoline+0x00 → function epilogues (pop regs, return)
- PENDING: calls trampoline-0x04 → PREVIOUS trampoline's +0x0c
- POST_CALC_1: calls trampoline-0x0c → prev trampoline's +0x04
- POST_CALC_2: calls trampoline-0x08 → prev trampoline's +0x08

the helper ACTIVE handlers are all function epilogues:
- encoder (0x87950): ldmia sp!, {r4-r10}; bx lr
- buffer (0x874A4): mov r0,r7; ldmia sp!, {r4-r11,pc}
- decoder (0x87A28): str r1,[r4,#0x6c]; ldmia sp!, {r4-r6,pc}

## what we know works (empirically confirmed)

1. **type=0 completion events** `[elem+4_addr, 0]` during PENDING activation
   → required for SETCODEC to succeed (CSS waits for ring buffer ack)

2. **type=3 decoder events** `[elem+4_addr, (3<<13)|1, 0]` every ~10ms
   → required for CSS to produce frames (kicks frame production loop)

3. **level-ready** `[0, (1<<13)|1, level]` for levels 0-3 at startup
   → required for CSS to accept the ARM as a BG executor

4. **heartbeat** `[0, (4<<13)|0]` every ~80ms from level 1
   → required for ongoing CSS↔ARM health check

5. **control word protocol** `*cw = (val & 6) ? 1 : 0` for PENDING entries
   → required for CSS state machine progression

6. **elem[0]+4 = 0x0e** → set by dfl_module_startup, purpose unclear but
   stock always has it

## what we know we DON'T know

1. **the complete codec table structure** — we know there's ~2137 lines of
   data that dfl_module_startup writes, but we don't know which fields the
   CSS actually reads vs which are ARM-only bookkeeping

2. **the minimum viable codec table** — for L16, many of the 21 module
   registrations are irrelevant (G.711, G.729, etc.). which modules must
   be registered for the CSS to route audio?

3. **which process-local addresses the CSS uses as opaque tokens** — the
   stock data has ~50 unique process addresses scattered through element
   descriptors. we need to know which ones the CSS passes back during
   dispatch (these need to be valid for our process)

4. **the exact encoder init base offset** — is it cw_ptr + 0x2c or
   something else? the decompilation says cw_ptr, stock data says +0x2c

5. **whether the CSS validates pointer values** — can we use sentinel
   values like 0xDEAD0001 or does the CSS check for valid ARM addresses?

6. **the BG level state machine's role in audio routing** — the I-switch
   setup (states 2→3→4→5→0x200→1) might configure signal routing that
   our static dispatch doesn't do

## CSS parameter architecture (decoded from CSS firmware with DWARF symbols)

the CSS accesses element descriptor parameters through a 2-level ITAB:

```
instance handle format:
  bits 0-9:   entry index (0-1023)
  bit 11:     validity flag (0x800) — all callers check this
  bits 12-14: BG level number (0-7)

lookup chain:
  dfl_get_itab_entry(instance):
    level = (instance >> 12) & 7
    index = instance & 0x3ff
    return level_table[level][index]  → descriptor base in shared memory

  dfl_get_param_addr(instance, offset):
    base = dfl_get_itab_entry(instance)
    return base + (offset & 0xfff)   (lower 12 bits = byte offset)

  offset encoding:
    bits 0-11:  byte offset within descriptor
    bit 12:     uint16 access
    bit 13:     uint32 access
    bit 14:     uint32 extended access
    bit 15:     sign-extend from int16
```

key CSS functions using this system:
- `dfl_set_I_switch(inst, val)` → writes to descriptor+0 (the control word)
- `WRITE_DSP_PARAM(inst, offs, val)` → writes to descriptor+offs
- `dfl_get_param_addr(inst, offs)` → resolves to memory address

the ITAB is populated by the CSS during DUA init. the entries point to element
descriptors in shared memory. `dfl_module_startup` (ARM-side) then writes codec
configuration VALUES into these descriptors at specific offsets.

CSS functions that access descriptors during L16 session:
- `p_auc_set_hw_rate` → WRITE_DSP_PARAM(inst, pcod->coder_offs, rate)
- `p_da_SwitchInstance` → dfl_set_I_switch(inst, 0x14)
- `p_auc_start_chan` → reads AUC channel struct fields, calls CB_start callback
- frame production → reads descriptor params for audio routing config

## plan: implement our own codec table (dfl_module_startup equivalent)

the "codec table" is not a separate structure — it IS the element descriptors
in shared memory, populated at specific offsets by dfl_module_startup. the CSS
reads these via the (instance, offset) parameter system during frame production
to configure audio routing.

### approach

1. **map stock descriptor layout:** take ONE group (24 elements) from the stock
   baseline dump. for each element, list every non-zero field with its offset
   and classify: CSS-written (shm pointer, varies per boot), ARM-written (process
   addr or constant, written by dfl_module_startup), or shared (both sides access).

2. **identify the CSS-read offsets:** cross-reference with CSS firmware functions
   that call WRITE_DSP_PARAM/READ_DSP_PARAM. look at p_auc_set_hw_rate's
   coder_offs, and frame production path param reads.

3. **build our module_startup:** write a function that populates each element
   descriptor at the required offsets. for fields that had process-local
   addresses in stock, use our own function table (noop entries for L16,
   sentinel values for unsupported codecs).

4. **use real function pointer table:** allocate a small dispatch table in our
   BSS. for L16 codec entries, put a noop function address. for unsupported
   codecs, put an address that logs "unsupported codec dispatched" and returns.
   the CSS will pass these back to us during dispatch and we'll call them.

### data sources

- `/root/shm_baseline.bin` — stock idle (all module registrations, no session)
- `/root/shm_active.bin` — stock active L16 session
- `/root/shm_bgsc_idle.bin` — our BGSC idle (missing registrations)
- `/root/shm_diff.txt` — full diff between stock and our idle states

## element descriptor layout (from shm_decode of stock baseline)

full decoded output: `research/claudes_notes/stock_baseline_decoded.txt`

### common element header (offsets +0x10 to +0x20)

nearly every element (2-21) shares this header structure:
```
+0x10: uint32 group_type
         0x00040004 for elem[2-9]  (BG level 0 elements?)
         0x00010001 for elem[13-21] (BG level 1 elements?)
+0x14: [CSS shm ptr] — CSS-allocated, points to per-element CSS data
+0x18: uint32 0x100 (constant, possibly buffer size or config)
+0x1c: [CSS shm ptr] — always shm+0xb84 (level 0) or shm+0x778 (level 1)
         these might be flow table or group descriptor pointers
+0x20: [ARM addr] — nearly universal, stock uses 0x000c3ff8 (BG level struct)
         THIS IS THE KEY REGISTRATION FIELD: tells CSS the ARM module is present
```

### element categories

| elem | role | size | ARM addrs | notes |
|------|------|------|-----------|-------|
| 0 | codec config | 8B | 0 | +4=0x0e, +10=0x02. group "header" element |
| 1 | helper | 112B | 3 | has pointers at +0x14, +0x40 |
| 2 | G.711 dec? | 284B | 19 | HUGE — full codec with func tables at 0x0011aaxx |
| 3 | passthrough? | 476B | 2 | large but sparse, only +0x20 and +0x30 ARM |
| 4 | signal proc | 352B | 14 | codec params + func tables |
| 5 | output | 52B | 4 | small, has 0x0011aa00 |
| 6 | encoder | 232B | 24 | HUGE — full encoder with init data, func tables |
| 7 | config | 8B | 0 | just +4=0x0c |
| 8-9 | helpers | 56-80B | 5 each | moderate |
| 10 | config | 8B | 0 | just +4=0x0c |
| 11 | helper | 56B | 4 | similar to 8 |
| 12 | buffer mgr | 180B | 10 | has audio buffer ptrs (shm+0xbe14!) |
| 13 | helper | 56B | 6 | has 0x0011f548 |
| 14 | helper | 100B | 12 | codec params + func tables |
| 15 | jitter buf? | 116B | 6 | has 16000, 32000, 320 (sample rates/sizes) |
| 16 | helper | 92B | 8 | codec params |
| 17 | passthrough? | 312B | 2 | large but sparse |
| 18 | helper | 56B | 5 | similar to 11 |
| 19 | signal proc | 160B | 9 | codec params + func tables |
| 20 | output | 56B | 4 | small |
| 21 | decoder | 44B | 2 | small! only +0x20 ARM addr + config at +0x28 |
| 22 | buffer | 28B | 0 | **ZERO ARM addrs** — entirely CSS-managed! |
| 23 | codec config | 128B | 3 | same pattern as elem[0] — group 1's header |

### key ARM process addresses (from stock app_dsp BSS)

| address | freq | likely purpose |
|---------|------|---------------|
| 0x000c3ff8 | 12x | BG level struct area (universal registration field) |
| 0x000c4098 | 6x | per-group config (group 0) |
| 0x000c4138 | 7x | per-group config (group 0 encoder) |
| 0x000c4a98 | 5x | per-group config (different module) |
| 0x000cd088 | 3x | per-group config (yet another module) |
| 0x0011aa00 | 6x | codec function table (WAD/L16?) |
| 0x0011aa7c | 5x | codec function table |
| 0x0011f548 | 7x | codec function table (another set) |
| 0x00125060 | 1x | module data (elem[1] only) |

### minimum viable registration hypothesis

for L16, we might only need:
1. elem[0]+4 = 0x0e (already doing this)
2. common header at +0x10 to +0x20 for each element
3. the ARM addr at +0x20 (a valid address in our process, e.g. our BG ctx)
4. possibly the codec function tables at 0x0011aaxx offsets (but for L16 these
   are noops, so maybe sentinel values work)

### key unknowns remaining

- does the CSS validate ARM addresses at +0x20 or just check non-zero?
- are the 0x0011aaxx function table entries used during L16 frame production?
- is the group_type at +0x10 (0x00040004 vs 0x00010001) significant?
- does elem[12]'s audio buffer pointer setup happen automatically once
  registration is done?

## hybrid approach notes (attempted, partially working)

tried: kill -9 stock app_dsp, run our BGSC on its shared memory.
result: element table pointers need rebasing (mmap base differs between
processes). rebasing logic implemented but CSS panics — likely because:
1. our ring buffer reinit overwrites stock's ring buffer state
2. level-ready messages confuse the CSS (stock already sent them)
3. process-local function pointers in element descriptors are now invalid

the hybrid approach could work with more care (skip ring buffer reinit,
skip level-ready, rebase ALL process addresses in element descriptors).
but implementing our own codec table is cleaner and more maintainable.
