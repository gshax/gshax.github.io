# CSS level 0 dispatch — the audio data mover (2026-04-02)

**context:** the DSP framework has 4 BG (background) levels. level 0 runs on
the CSS processor; levels 1-3 run on the ARM (app_dsp/our BGSC). both sides
use the same `dfl_process_flow` dispatch code and share element descriptors in
shared memory. see [bgsc_shared_memory.md](bgsc_shared_memory.md) for the ARM
side and [voice_service_and_app_dsp.md](voice_service_and_app_dsp.md) for the
overall architecture.

## key discovery

the CSS has its OWN `dfl_process_flow` at 0x020126e4 — **identical code** to
the ARM's dispatch at 0x000885b0. same FTAB structure, same control word
protocol, same calc function dispatch.

```
CSS level 0: dfl_process_flow @ 0x020126e4 (CSS firmware)
ARM levels 1-3: dfl_process_flow_asm @ 0x000885b0 (app_dsp)
```

## architecture

```
CSS level 0 dispatch:
  - runs on CSS processor
  - processes CSS-side FTAB entries with CSS-internal calc_func pointers
  - handles: TDM data reception, signal routing, buffer management
  - reads element descriptor CONFIG from shared memory

ARM levels 1-3 dispatch:
  - runs on ARM processor (our BGSC)
  - processes ARM-side FTAB entries with ARM process-local calc_func pointers
  - handles: codec processing (encoding/decoding)
  - for L16: all noops

both dispatches access the SAME element descriptors in shared memory (same
cw_ptr values). the element descriptors are the shared interface between
the CSS and ARM dispatchers.
```

## for L16 (passthrough): CSS does all the real work

since the ARM codec functions are noops for L16, ALL audio data movement
happens on CSS level 0. the ARM just:
- acknowledges control word changes (clear pending bits)
- sends ring buffer messages (type=0 completion, type=3 decoder ready)
- does NOT move any audio data

the CSS level 0 dispatch:
- reads TDM audio from hardware
- routes it through the DSP pipeline (signal blocks: SSW, SSR, SU2)
- writes encoded frames to the voice CFIFO

**confirmed 2026-04-02:** TDM hardware registers and `optimized_tdm_handler`
ISR counters are IDENTICAL between stock app_dsp (working audio) and our
comatose_dsp (silence). the TDM ISR is NOT involved in the voice audio path
at all — counters are zero even with stock producing real audio. the CSS level 0
dispatch handles audio data movement directly, reading configuration from
element descriptors in shared memory.

**implication:** the silence is caused by CSS level 0 dispatch not having the
correct configuration in element descriptors, and/or our ARM dispatch
interfering with CSS level 0 dispatch on shared control words.

## why silence: CSS level 0 reads config from descriptors

the CSS level 0 calc functions read element descriptor fields to determine:
- where to read audio data (input buffer pointer)
- where to write audio data (output buffer pointer)
- which codec/signal processing to apply
- routing connections between elements

these fields are populated by `dfl_module_startup` on the ARM side (stock
app_dsp: FUN_00075810, 36KB, 21 callees). without them, the CSS level 0
creates voice sessions and produces frames, but the frames contain silence
because the routing pipeline has no configured endpoints.

see [codec_table_investigation.md](codec_table_investigation.md) for detailed
analysis of what `dfl_module_startup` writes and what we're missing.

additionally, audio routing requires RouteCODEC to fire (sets DRT global flags),
which requires the module readiness cascade to complete — see
[module_readiness_cascade.md](module_readiness_cascade.md).

## FIFO I/O path (from CSS firmware RE)

```
p_dsp_fifo_write(instance, data, samples)
  → p_dspa_GetParameterAddress(instance, 4, &fifo_ptr)
    → dfl_get_itab_entry(instance) + 4  // descriptor+4 = buf_ptr_a
  → p_dspa_FIF_fw(samples, fifo_ptr, data)  // write to *buf_ptr_a

p_dsp_fifo_read(instance, data, samples)
  → similar, probably uses offset 8 (buf_ptr_b)
```

the FIFO write goes to `descriptor+4` (buf_ptr_a). for the buffer element
(elem[22]), this is the pointer we differentiate by +0x140.

## next: trace CSS level 0 FTAB

need to find:
1. the CSS level 0 FTAB (flow table) — CSS-internal, but entries reference
   shared memory element descriptors
2. which CSS calc functions handle audio routing
3. what descriptor fields they read (which offsets matter)

the CSS FTAB pointer might be at the shared memory flow table header
(shm+0xb848+4 = 0x0046a724, a CSS address).
