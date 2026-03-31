# claude's notes

reverse engineering notes on the CSS coprocessor and DUA subsystem,
generated during analysis of `_css.elf` (with DWARF symbols) and the
GPL kernel source.

## files

- [css_architecture_overview.md](css_architecture_overview.md) — dual-processor model, COMA message bus, memory layout, ghidra setup
- [dua_protocol.md](dua_protocol.md) — DUA message format, **complete** command table (all IDs mapped), DUASM state machine enum, response format
- [dua_units_and_elements.md](dua_units_and_elements.md) — unit types (UT_SPVOIPNDA, UT_TRACE, UT_FXS), UID encoding, pin instances, DSP elements, connection model, data structure addresses
- [dua_params_and_events.md](dua_params_and_events.md) — **complete** DUA_PARAM code table (43 params), DUAEV event codes (129 events), DUA error codes
- [audio_data_path.md](audio_data_path.md) — full audio path from SLIC through TDM, DSP FIFOs, DUA routing, voice service, to linux
- [key_functions_reference.md](key_functions_reference.md) — address table for ~100 important CSS firmware functions
- [umt_modes.md](umt_modes.md) — UMT bytecode format, all 8 SPVOIPNDA modes and 3 FXS modes decoded, frame size analysis, practical mode selection guide
- [tapi_slic_control.md](tapi_slic_control.md) — TAPI ioctl interface for SLIC hardware control (line feed, hook detect, ringing), kernel module stack (drv_tapi.ko → drv_silabs.ko → Si3217x/Si3226x), minimal init sequence
- [open_source_wrapper_plan.md](open_source_wrapper_plan.md) — feasibility assessment, proposed call sequence, remaining unknowns checklist

- [tdm_grant_investigation.md](tdm_grant_investigation.md) — TDM grant blocker analysis, wire format issue, replacement kernel module, UMT assignment blob structure

## hardware-validated findings (2026-03-29 through 2026-03-30)

- DUA response format confirmed: cmd=0x81 sync (sender_id correlated), cmd=0x7f async callbacks
- DUA is a userspace-registered service (not kernel), registered via AF_COMA connect()
- CSS requires shared memory init (ioctl 0xc0045302 on /dev/sharedmem) before DUA works
- DUA_CMD_INIT_REQ (cmd 0x01) must be sent with shared memory pointer before any operations
- COMA socket recv() returns EOPNOTSUPP — must use recvmsg() instead
- unit allocations work from clean boot after proper init sequence
- 32 SPVOIPNDA elements confirmed on real hardware (matches static analysis exactly)
- DUA variable-length data (e.g. UMT blobs) requires length-prefixed packing
- line feed enum: ACTIVE=0, ACTIVE_REV=1, STANDBY=2 (was wrong in initial code)
- TDM assignment UMT blob fully decoded: 268 bytes, constructible from 5 constants
- full DUA audio path setup works (alloc, UMT, connect, merge) — only TDM grant remains
- wrote replacement TDM kernel module to handle nacks without BUG_ON crash
- TDM grant always rejected with -7 — suspected wire format field ordering issue
