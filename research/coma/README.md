# claude's notes

reverse engineering notes on the CSS coprocessor, DUA subsystem, BGSC codec
framework, and voice service. generated through static analysis of `_css.elf`
(with DWARF symbols), `app_dsp`, `libcordless.so`, GPL kernel source, and
extensive hardware testing on a netbooted alpine environment.

**status as of 2026-04-02:** bidirectional audio works through the CSS DSP
pipeline (confirmed 2026-03-31). voice sessions reach ST_RTP_RUN and produce
frames, but the frames contain silence — TDM audio is not reaching the encoder.
the remaining blocker is the BGSC↔CSS interaction: specifically, element
descriptor registration and the module readiness cascade that triggers audio
routing activation.

## system architecture (how everything fits together)

```
layer 4: consumer apps (gs_ata)
  │ uses: libcordless DUA API, /dev/voiceN, /dev/fxsN
  │ role: SIP signaling, RTP networking, call routing
  │
layer 3: voice service + TAPI
  │ voice: kernel coma-voice.c ↔ CSS voice_process_message
  │ tapi: kernel drv_tapi.ko ↔ SiLabs SLIC chips (SPI)
  │ role: session lifecycle, encoder/decoder CFIFOs, SLIC hardware
  │
layer 2: DUA + BGSC (the "DSP framework")
  │ DUA:  AF_COMA "dua" socket ↔ CSS dua_process
  │ BGSC: shared memory + ring buffers ↔ CSS dfl_process_flow
  │ role: unit allocation, DSP pipeline config, audio routing, codec exec
  │ NOTE: DUA and BGSC share element descriptors in shared memory.
  │       DUA creates the skeleton (units, connections, ITAB).
  │       BGSC fills in codec registration data.
  │       CSS level 0 dispatch reads both to route audio.
  │
layer 1: CSS firmware + TDM hardware
  role: TDM bus, signal blocks, audio routing tables, real-time pump
```

the critical insight is that `app_dsp` is NOT a regular COMA API consumer —
it's an outsourced internal subsystem of the CSS. the CSS firmware is compiled
with `-DDSPonARMonly=1`. BG level 0 runs on the CSS; levels 1-3 are dispatched
to the ARM (app_dsp) via the FTAB + ring buffer protocol. see
[voice_service_and_app_dsp.md](voice_service_and_app_dsp.md) and
[bgsc_shared_memory.md](bgsc_shared_memory.md).

## files — foundational (stable, well-validated)

- [css_architecture_overview.md](css_architecture_overview.md) — dual-processor
  model, COMA message bus, memory layout, CSS service table, ghidra setup
- [dua_protocol.md](dua_protocol.md) — DUA message format, **complete** command
  table (all IDs mapped), DUASM state machine enum, response format (sync + async)
- [dua_units_and_elements.md](dua_units_and_elements.md) — unit types
  (SPVOIPNDA, TRACE, FXS), UID encoding, pin instances, DSP elements, connections
- [dua_params_and_events.md](dua_params_and_events.md) — **complete** DUA_PARAM
  table (43 params), DUAEV events (129), DUA error codes
- [audio_data_path.md](audio_data_path.md) — full audio path from SLIC through
  TDM, DSP FIFOs, DUA routing, voice service, to linux
- [key_functions_reference.md](key_functions_reference.md) — address table for
  ~100 CSS firmware functions
- [tapi_slic_control.md](tapi_slic_control.md) — TAPI ioctl interface, kernel
  module stack, SiLabs ProSLIC API, minimal init sequence
- [umt_modes.md](umt_modes.md) — UMT bytecode format, all SPVOIPNDA + FXS modes
  decoded, frame size analysis

## files — solved investigations (historical, conclusions are sound)

- [tdm_grant_investigation.md](tdm_grant_investigation.md) — TDM grant blocker,
  root cause (missing FXS UMT mode 1), wire format quirks. **RESOLVED 2026-03-31.**
- [open_source_wrapper_plan.md](open_source_wrapper_plan.md) — original
  feasibility assessment + proposed API. partially outdated — many unknowns now
  resolved, but the call sequence design is still valid.
- [css_crash_analysis.md](css_crash_analysis.md) — crash dump mechanism + type-3
  event panic analysis. led to discovering correct ring buffer message format.

## files — active investigation (BGSC + audio silence, 2026-04-01 to present)

these notes iterate on each other. read them in this order for the full picture:

1. [voice_service_and_app_dsp.md](voice_service_and_app_dsp.md) — establishes
   that app_dsp is the ARM-side BGSC executor, ALL codecs go through AUC
   allocation, there is no bypass. **read first for architectural context.**

2. [bgsc_shared_memory.md](bgsc_shared_memory.md) — shared memory layout, pool
   header, FTAB structure, control word protocol, dispatch loop. contains both
   stable findings and evolving investigation state.

3. [ring_buffer_protocol.md](ring_buffer_protocol.md) — ARM→CSS ring buffer
   message format. type=0 (completion), type=1 (level ready), type=3 (decoder
   frame trigger), type=4 (heartbeat). crash analysis informed the correct format.

4. [css_arm_events.md](css_arm_events.md) — **key discovery:** CSS sends DUA
   async callbacks (cmd=0x7f) on the COMA socket to signal the ARM. the 0xf2
   event drives the module readiness cascade. we need to respond to these.

5. [module_readiness_cascade.md](module_readiness_cascade.md) — the chain from
   ring buffer messages → p_dsp_CallBack → SetModuleReady → RouteCODEC. this is
   what activates the audio routing. depends on findings from css_arm_events.md.

6. [css_level0_dispatch.md](css_level0_dispatch.md) — CSS has its own
   dfl_process_flow identical to the ARM's. for L16, CSS level 0 does ALL audio
   data movement. the ARM is just infrastructure.

7. [init_sequence_comparison.md](init_sequence_comparison.md) — stock vs our init
   side by side. identifies `dfl_module_startup` (element descriptor population)
   as the likely root cause of silence.

8. [codec_table_investigation.md](codec_table_investigation.md) — detailed
   analysis of what `dfl_module_startup` writes. 2137 lines of stock-specific
   data. identifies minimum viable registration hypothesis.

## hardware-validated findings

- DUA response format: cmd=0x81 sync, cmd=0x7f async callbacks
- COMA socket: `recv()` returns EOPNOTSUPP, must use `recvmsg()`
- CSS requires shared memory init before DUA works
- 32 SPVOIPNDA elements confirmed on hardware (matches static analysis)
- TDM assignment UMT blob: 268 bytes, fully decoded
- full DUA audio path works (alloc, UMT, connect, merge, TDM grant)
- BGSC ring buffer: CSS reads our messages (read ptr advances)
- SETCODEC succeeds, CSS produces L16 frames (652 bytes, but silence)
- CSS→ARM cascade events arrive on the DUA COMA socket as cmd=0x7f
- line feed enum: ACTIVE=0, ACTIVE_REV=1, STANDBY=2
