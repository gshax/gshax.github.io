# voice service, app_dsp, and the ARM-side codec stack

## overview

getting audio between the CSS DSP and linux userspace requires three cooperating
subsystems:

1. **DUA** (digital unit allocation) — allocates DSP units, sets pipeline modes,
   creates audio routing connections, configures TDM. runs on CSS, controlled
   via AF_COMA socket to the "dua" service.

2. **voice service** — manages encoder/decoder CFIFOs that carry codec frames
   between the CSS and linux. runs on CSS, exposed to linux via `/dev/voiceN`
   character devices backed by the `coma-voice.ko` kernel driver.

3. **BGSC / app_dsp** — the ARM-side codec processing system. ALL codec
   encode/decode (G.711, G.722, G.726, G.729, iLBC, Opus, AMR-WB, and even
   L16 linear PCM) runs on the ARM processor via background scheduler threads.
   the CSS firmware is compiled with `-DDSPonARMonly=1`.

in the stock firmware:
- `app_dsp` handles #1 (DUA init only) and #3 (BGSC)
- `gs_ata` (via `libcordless.so`) handles the rest of #1 (unit allocation,
  connections, TDM) and all of #2 (voice sessions)
- both processes connect to the "dua" COMA service concurrently

## /dev/voiceN character devices

### kernel driver: coma-voice.c

16 devices (`/dev/voice0` through `/dev/voice15`), one per voice session.
minor number = session_id. dynamically allocated major number.

the kernel config `CONFIG_CORDLESS_USER_SPACE_FIFOS` is **NOT** set on the
ht818, so the chardev read/write/poll interface is available (not mmap).

### file operations

```c
.open    = voice_open      // GET_SESSION + SET_SESSION_FIFOS to CSS
.release = voice_release   // STOP_SESSION + FREE_SESSION to CSS
.read    = voice_read      // blocks on enc_fifo, returns one packet
.write   = voice_write     // pushes one packet into dec_fifo
.poll    = voice_poll       // POLLIN when enc_fifo not empty
.ioctl   = voice_ioctl     // SETCODEC, STOP, FLUSH, etc.
```

### session lifecycle

1. `open("/dev/voiceN", O_RDWR)` — kernel sends:
   - `CMSG_VOICE_REQUEST_GET_SESSION` (type 0) → CSS allocates session slot
   - `CMSG_VOICE_REQUEST_SET_SESSION_FIFOS` (type 2) → CSS maps enc/dec CFIFO
     physical addresses (20KB each, DMA-coherent)

2. `ioctl(fd, VOICE_IOCSETCODEC, &rtp_session_config)` — kernel sends:
   - `CMSG_VOICE_REQUEST_START_SESSION` (type 4) with the full 836-byte config
   - CSS starts codec pipeline, eventually replies via completion

3. `read(fd, buf, len)` — blocks until enc_fifo has data, copies one packet.
   returns `-EBUSY` if kernel-mode socket is active.

4. `write(fd, buf, len)` — pushes packet into dec_fifo.
   returns `-EBUSY` if kernel-mode socket is active.

5. `ioctl(fd, VOICE_IOCSTOP_SESSION)` + `close(fd)` — kernel sends STOP + FREE.

**important**: do NOT call `VOICE_IOCKERNELMODE` — that hands a UDP socket to a
kernel thread for RTP send/recv, which makes read/write return EBUSY.

### CFIFO packet format

each packet in the enc/dec FIFO is prefixed with a 12-byte header:

```c
struct rtp_packet_header {   // 12 bytes on ARM32
    uint32_t packetType;     // 1 = CFIFO_RTP_PACKET, 2 = CFIFO_RTCP_PACKET
    uint32_t ttl;
    uint32_t receiptTime;    // milliseconds
};
// followed by raw RTP packet (header + codec payload)
```

for G.711u at 20ms ptime: 12 (cfifo hdr) + 12 (RTP hdr) + 160 (ulaw) = 184 bytes.
for L16 at 20ms: 12 + 12 + 320 (16-bit PCM) = 344 bytes.

when reading via chardev, you get the full cfifo data including the header.
when writing via chardev, you must include the header yourself.

### ioctl numbers

| ioctl | value | purpose |
|-------|-------|---------|
| VOICE_IOCSETCODEC | `_IOW('V', 0, ...)` | configure codec + start session |
| VOICE_IOCFLUSH | `_IO('V', 1)` | flush encoder FIFO |
| VOICE_IOCKERNELMODE | `_IOW('V', 2, ...)` | enable kernel UDP (DON'T USE) |
| VOICE_IOCSTOP_SESSION | `_IO('V', 8)` | stop active session |
| VOICE_IOCUPDATE_SESSION | `_IOW('V', 6, ...)` | update codec mid-call |
| VOICE_IOCGET_FIFO_INFO | `_IOR('V', 10, ...)` | query FIFO layout |
| VOICE_IOCGET_LAST_CSS_ERROR | `_IOR('V', 11, ...)` | last CSS error code |

### rtp_session_config struct (836 bytes, packed)

the big config struct passed to VOICE_IOCSETCODEC. key fields:

```
opts               — session option flags (RTP_OPT_*)
audio_mode         — 1=send, 2=recv, 3=active, 4=inactive
media_loop_level   — 0=none, 1=IP loop, 2=DSP loop, 3=RTP loop
lib_rtp_mode       — 0=user, 1=kernel, 2=app_stack, 3=app
codec.tx_pt        — transmit payload type (0=G711u, 8=G711a, 9=G722...)
codec.rx_list[]    — up to 6 accepted receive codecs
codec.duration     — ptime in ms (typically 20)
codec.CodecStr     — codec name ("pcmu/8000", "G722/8000", "L16/16000"...)
codec.opts         — codec-specific options (PLC, VAD, etc.)
voip_line_id       — links to SPVOIPNDA instance index
session_id         — voice session ID (= /dev/voiceN minor number)
jib_config         — jitter buffer params (if RTP_OPT_USE_JIB set)
srtp_config        — SRTP keys (if encryption enabled)
```

full struct definition in `libcomatose/include/comatose/voice_defs.h`.

## CSS voice service internals

### service registration

registered at CSS boot: `voice_init()` @ 0x02271e9c calls
`coma_register(6, "voice", ...)`. service ID = 6.

### message dispatcher

`voice_process_message()` @ 0x0229ffbc dispatches on message type:

| type | handler |
|------|---------|
| 0 GET_SESSION | `voice_get_session()` — allocates session slot |
| 2 SET_SESSION_FIFOS | `voice_set_session_fifos()` — maps CFIFO addresses |
| 4 START_SESSION | `voice_start_session()` @ 0x022a0cc4 |
| 6 STOP_SESSION | `voice_request_stop_session()` |
| 8 FREE_SESSION | `voice_free_session()` |
| 10 SEND_DTMF | `voice_send_dtmf()` |
| 19 UPDATE_SESSION | `voice_update_session()` |

### session state machine

each session is 0x14 bytes. state at offset +0xc:
- 1 = FREE (initial)
- 2 = ALLOCATED (after get_session + set_fifos)
- 3 = T38 active
- 4 = STOPPING
- 5 = RTP active

sub-type at offset +0xd:
- 1 = RTP
- 3 = T38
- 5 = AEC mode 6
- 6 = AEC mode 7

### voice_start_session dispatch

`voice_start_session()` @ 0x022a0cc4 routes based on config:

- if `opts & 0x80000000` (T38 flag) → `voice_start_t38()`
- if `lib_rtp_mode == 6` → `voice_start_aec()` (AEC mode 5)
- if `lib_rtp_mode == 7` → `voice_start_aec()` (AEC mode 6)
- **otherwise** → `voice_start_rtp()` (the normal audio path)

### voice_start_rtp flow

`voice_start_rtp()` @ 0x022a0a58:
1. validates session_id < 16, state == 0x02 (allocated), FIFOs set
2. fills `RTP_DATA` struct from the config (codec type, CodecStr, capabilities,
   FIFO addresses, voip_line_id, session_id, full 0x344 bytes of session config)
3. calls `p_rtpapp_Start()`

### p_rtpapp_Start flow

`p_rtpapp_Start()` @ 0x02299358 has three paths:

**path A: audio_mode == 4 (INACTIVE)**
- inits session, sets state to `ST_RTP_START_INACTIVE`
- calls `p_rtpapp_SetSesionStatus()` which sends the reply immediately
- no codec allocation — session is "started" but silent
- returns 0

**path B: media_loop_level == 1 or 4 (loopback)**
- inits session, sets state to ST_RTP_STARTING → ST_RTP_RUN
- calls `p_rtpapp_SetSesionStatus()` which sends the reply
- no codec allocation — packets loop back at IP/RTP level
- returns session_id

**path C: normal (the one that times out without BGSC)**
- inits session
- sets state to `ST_RTP_STARTING`
- calls `p_rtpapp_unit_mode_set(session_id)` to set SPVOIPNDA UMT mode via
  internal DUA `p_dua_UnitSetReq`
- returns — but **does NOT send the reply yet!**
- the reply is sent later via async callback chain:
  1. DUA completes UMT mode set → fires callback
  2. callback invokes `p_rtpapp_start_enc_dec()` @ 0x0229caa0
  3. which calls `p_rtpapp_ENCDECstart()` @ 0x02296930
  4. ENCDECstart allocates AUC encoder + decoder channels via
     `p_da_AUCChannelAllocEnc/Dec` and starts them with
     `p_da_AUCChannelPairStart`
  5. AUC pair start writes BGSC entries to shared memory for ARM processing
  6. when ARM acknowledges → `p_rtpapp_SetSesionStatus()` sends the reply

**this is why START_SESSION times out without app_dsp: step 6 never happens
because no ARM-side BGSC threads are processing the codec entries.**

### p_rtpapp_unit_mode_set codec → VOIP mode mapping

`p_rtpapp_unit_mode_set()` @ 0x0229cc98 looks up the AUC codec type from the
MTD codec table, then maps it to a SPVOIPNDA UMT mode:

| AUC codec range | VoIPUnitMode | meaning |
|-----------------|-------------|---------|
| default (G.711, G.726, etc.) | 1 | narrowband 20ms |
| 0x51 (G.722 WB variant) | 2 | narrowband 40ms |
| 0x50, 0x52-0x55 (Opus) | 3 | narrowband 60ms |
| 0x2f (G.722), 0x38 (G.729), 0x3d (G.723) | 4 | wideband 20ms |
| 0x3e-0x4f, 0x61-0x62 (AMR-WB) | 5 | wideband 40ms |

then sends `p_dua_UnitSetReq(uid, -2, UMT_EXEC_GEN, mode)` — an internal
CSS DUA command to set the SPVOIPNDA unit mode.

### p_rtpapp_ENCDECstart — no L16 bypass

`p_rtpapp_ENCDECstart()` @ 0x02296930 handles ALL codecs uniformly:
- looks up encode/decode AUC codec types from MTD table
- for dynamic PT codecs (>= 96), maps codec strings to AUC types:
  - `"pcmu/8000"` → AUC 0x1e
  - `"pcma/8000"` → AUC 0x1d
  - `"G729/8000"` → AUC 0x34
  - `"G722/8000"` → AUC 0x2f
- L16 only gets a small special case: disables PLC (packet loss concealment)
- **ALL codecs** including L16 go through:
  `p_da_AUCChannelAllocDec()` → `p_da_AUCChannelAllocEnc()` →
  `p_da_AUCChannelPairStart()` → BGSC shared memory entries
- there is NO passthrough/bypass path for any codec

## app_dsp architecture

### what app_dsp does

`app_dsp` is a userspace daemon that runs alongside gs_ata. **it is NOT a
regular COMA API consumer** — it is an outsourced internal subsystem of the CSS
that runs BG processing levels 1-3 on the ARM core. the CSS runs level 0
internally. see [bgsc_shared_memory.md](bgsc_shared_memory.md) for the shared
memory protocol details.

its role is to:
1. initialize the shared memory pool and DUA transport
2. register codec modules into element descriptors (dfl_module_startup)
3. run the BGSC dispatch loop (FTAB processing, ring buffer messaging)

**it does NOT**:
- open `/dev/voiceN` devices
- allocate DUA units (FXS, VOIP)
- create connections or configure TDM
- manage voice sessions

those are all handled by gs_ata via libcordless.so.

### initialization sequence (main @ 0x00014378)

1. **signal handlers** — SIGINT, SIGHUP, SIGTERM
2. **shared memory init** (0x00018a18) — opens `/dev/sharedmem`, maps 0x80000
   bytes, zeroes it, writes pool header (size at +4, granularity 0x30 at +8)
3. **DUA init** (0x000155d0 = `duasync_init`) — connects AF_COMA to "dua",
   starts recv_thread for DUA response handling
4. **BGSC table setup** (0x0001571c) — stores callback function pointers
   (interrupt lock/unlock mutex hooks) into DSPAPI config
5. **DSPAPI init** (0x00015744) — sends DUA `InitReq` with 5 params:
   `[5, 0xffffffff, 0xffffffff, 0xdeadbeef, shared_mem_ptr]`,
   then sends `ApplInit`. equivalent to our `dua_init_hw()` + `dua_appl_init()`
6. **BGSC callback registration** — stores int_disable (mutex lock) and
   int_enable (mutex unlock) callbacks for the BGSC dispatch loop
7. **DSP app startup** (0x0007e790 = `p_dsp_app_startup`) — **critical**:
   - checks DSP SW version (0x1140)
   - stores shared mem pool base into BGSC subsystem
   - allocates 0x79c bytes from shared memory for ARM-side codec state
   - initializes 4 per-BG-level structures
   - calls codec module startup (0x00075810) which initializes ALL codec
     modules: G1D, G1E, G2D, G2E, G3D, G3E, G6D, G6E, G9D, G9E,
     ILBCD, ILBCE, OPUSD, OPUSE, PLC, QFR, QFT, SSX, TPD, TPE, TSC,
     WAD, WAE, AMRWD, AMRWE
   - marks each BG level as initialized
   - writes version tag at shared_mem+0x14
8. **scheduler priority** — sets SCHED_FIFO at configured priority
9. **spawn 3 BG level threads** at priorities max, max-1, max-2:
   - thread 0 (BG level 1): periodic loop calling `FUN_0007e968(1)`
   - thread 1 (BG level 2): periodic loop calling `FUN_0007e968(2)`
   - thread 2 (BG level 3): periodic loop calling `FUN_0007e968(3)`
10. **wait for shutdown** — blocks until CSS signals completion

### BGSC dispatch loop (FUN_0007e968)

the per-level background processing function. each call:
- multi-phase startup state machine (states 2→3→4→5→0x200→1)
- in steady state: calls `FUN_000885b0` which iterates the BGSC table —
  a linked list of DSP module entries in shared memory, each with status
  bits and function pointers
- dispatches to codec encode/decode functions based on status bits
- BG level 0 also sends periodic heartbeat messages to the DSP message queue

### build flags (from compiler command string in _css.elf)

key defines that affect the architecture:
```
-DDSPonARMonly=1              — ALL codec processing on ARM, not CSS
-DSPLIT_NDA_UNITS_ENABLE=16   — 16 split NDA (codec) units
-DMAXNUM_AUC_CHANNELS=400     — max audio codec channels
-DNUM_VOICE_CHANNELS=16       — 16 voice sessions
-DRTP_MAX_SESSIONS=16
-DDA_MAX_DFC=9                — 8 FXS + 1 extra
-DDA_MAX_TOG=51               — 51 tone generators
-DCODEC_TO_DSP_LINE=2
-DNDA_CODEC_ENABLE=1
-DDAAFEATURES=(DAAF_AUC_A_G711|DAAF_AUC_A_G722|DAAF_AUC_A_G726|
               DAAF_AUC_A_G723|DAAF_AUC_A_G729|DAAF_AUC_A_AMRWB|
               DAAF_AUC_A_ILBC|DAAF_JIB|DAAF_AUC_A_L16|DAAF_AUC_A_OPUS)
```

### implications for libcomatose

to use the `/dev/voiceN` path for audio, the ARM-side BGSC system MUST be
running. options:

1. **run stock app_dsp alongside libcomatose tools** — app_dsp does DUA init +
   BGSC; our tools skip DUA init and only do unit allocation, connections,
   TDM, and voice sessions. this is how the stock system works (app_dsp +
   gs_ata as separate processes both connecting to "dua").

2. **implement minimal BGSC replacement** — replicate just enough of the codec
   init and BG thread processing for the codecs we need (e.g., G.711 is a
   simple lookup table, L16 is a passthrough). requires RE of the ARM-side
   app_dsp binary and understanding the BGSC shared memory layout.

3. **bypass the voice service entirely** — access DSP FIFOs directly through
   shared memory to get raw PCM. avoids the codec stack entirely but requires
   understanding the FIFO memory layout and dealing with real-time constraints.

option 1 is the fastest path. option 2 is the cleanest for an open source
replacement. option 3 is the most independent but most complex.

**current approach (libcomatose):** we are pursuing option 2 — a minimal BGSC
replacement. our implementation handles DUA init, unit allocation, TDM, and
voice sessions. the BGSC module (`bgsc.c`) handles ring buffer messaging,
element table discovery, and control word dispatch. voice sessions reach
ST_RTP_RUN and produce frames, but the frames contain silence because element
descriptor registration (`dfl_module_startup` equivalent) is incomplete.

see [bgsc_shared_memory.md](bgsc_shared_memory.md) for the dispatch protocol,
[codec_table_investigation.md](codec_table_investigation.md) for what's missing
from element registration, and [module_readiness_cascade.md](module_readiness_cascade.md)
for the cascade that activates audio routing.
