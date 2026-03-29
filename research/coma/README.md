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
