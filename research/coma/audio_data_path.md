# audio data path through the CSS

## high-level flow

```
analog phone ←→ SLIC chip                         ← TAPI ioctls (tapi_slic_control.md)
                   ↕
              TDM bus (hardware)                   ← tdm_grant activates (tdm_grant_investigation.md)
                   ↕
              CSS level 0 dispatch                 ← dfl_process_flow @ 0x020126e4 (css_level0_dispatch.md)
              (signal routing via element          ← element descriptors configured by BGSC module_startup
               descriptors in shared memory)
                   ↕
              DUA audio routing (ART)              ← RouteCODEC activates this (module_readiness_cascade.md)
                   ↕ signal blocks, codec pipeline
              VoIP DSP pipeline (SPVOIPNDA unit)
                   ↕ (for L16: CSS does all data movement, ARM codec funcs are noops)
              COMA voice service                   ← voice_service_and_app_dsp.md
                   ↕ encoder/decoder CFIFOs (20KB each, DMA-coherent)
              /dev/voiceXX on Linux                ← kernel coma-voice.c chardev
                   ↕
              RTP stack (gs_ata / our code)
```

**IMPORTANT correction (2026-04-02):** the `optimized_tdm_handler` ISR and its
DSP FIFO write/read path documented below are NOT the mechanism used for the
voice service audio path. TDM ISR sample counters are zero even with stock
app_dsp producing real audio through /dev/voiceN. the TDM hardware is also
identically configured between stock and our code (same registers, same values).

the audio data movement between TDM hardware and the encoder input buffer
happens entirely through the **CSS level 0 dispatch** — the same dfl_process_flow
mechanism that runs on the ARM for levels 1-3. the CSS level 0 FTAB entries
contain calc functions for signal routing (SSW, SSR, SU2) that read from TDM
hardware and write to DSP buffers. these calc functions read configuration from
**element descriptors in shared memory** — the same descriptors that the ARM's
`dfl_module_startup` populates.

**critical dependencies for audio to flow:**
1. FXS UMT mode 1 must create DSP FIFOs (resolved 2026-03-31)
2. module readiness cascade must complete and trigger RouteCODEC (resolved 2026-04-02)
3. RouteCODEC sets DRT global flags at 0x09000000 = 0x01 (resolved 2026-04-02)
4. BGSC must populate element descriptors so CSS level 0 dispatch knows how to
   route audio — **THIS IS THE REMAINING BLOCKER**
5. ARM dispatch must NOT interfere with CSS level 0 dispatch on shared elements
6. voice session must be started (SETCODEC ioctl on /dev/voiceN)

see [module_readiness_cascade.md](module_readiness_cascade.md) for the cascade
chain and [codec_table_investigation.md](codec_table_investigation.md) for the
element descriptor registration requirements.

**the FXS-to-FXS intercom (dua_intercom) is a separate audio path** that works
through DUA audio routing tables (ART) alone, bypassing the voice service and
TDM ISR entirely. intercom audio is routed at the hardware/DRT level between
FXS ports without ARM involvement.

## TDM layer (bottom)

### hardware interface

- up to 3 TDM bus instances (id 0-2)
- ISR handlers: `tdm0_isr`, `tdm1_isr`
- instance data: `tdm_instance_data` struct, size 0x1c per instance

### tdm_read_sample(instance, *sample)

reads one 16-bit PCM sample from TDM hardware:
```c
*sample = (int16_t)(*(reg_base + 0x3c) >> shift);
// reg_base = *(instance * 0x1c + instance_data_base + 4)
// shift = *(instance * 0x1c + instance_data_base + 0x14) & 0xff
```

### tdm_write_sample(instance, sample)

writes one 16-bit PCM sample to TDM hardware:
```c
*(reg_base + 0x38) = (int)sample << shift;
```

### tdm_grant(id, rate, channels, sample_size)

activates a TDM bus:
- validates: id < 3, channels <= 32, sample_size < 17, rate != 0
- checks not already granted
- registers threshold notifier
- enters critical section
- validates channel count matches hardware config
- stores rate, channels, sample_size in instance data
- calls `tdm_enable(id, channels, sample_size)`
- NOTE: does NOT call `tdm_enable_clock` — the CMU clock is managed separately

### optimized_tdm_handler(p) — NOT used for voice audio path!

**IMPORTANT (verified 2026-04-02):** this ISR's sample counters are zero even
with stock app_dsp producing real audio. it is NOT the mechanism that moves
audio between TDM hardware and the voice encoder. audio data movement for the
voice service path happens through the CSS level 0 dispatch (dfl_process_flow)
which uses signal routing calc functions (SSW, SSR, SU2) reading configuration
from element descriptors in shared memory.

the ISR is documented here for reference as it EXISTS in the firmware, but its
role in the overall audio path is unclear — it may be used for a different audio
mode or may be a fallback path that the DVF101 platform doesn't use.

**RX path** (SLIC → DSP):
```
for each available rx frame:
    for each channel:
        tdm_read_sample(id, &sample)
        if fifo exists:
            p_da_DSPFifoWrite(fifo, &sample, 1)
        if channels != bank_size:
            tdm_inc_rx_bank(id)
```

**TX path** (DSP → SLIC):
```
for each free tx slot:
    for each channel:
        if tx_handler exists:
            p_da_DSPFifoRead(fifo, &sample, 1)
        else:
            use last_sample (comfort noise / silence)
        tdm_write_sample(id, sample)
```

each channel has separate rx/tx handler structs with:
- `fifo` — DSP FIFO instance number
- `fn` — handler function pointer (optional)
- `last_sample` — last written sample (for tx fill)

## DSP FIFO layer

thin wrappers around `p_dsp_fifo_read()` / `p_dsp_fifo_write()`:
- `p_da_DSPFifoRead(instance, data, samples)`
- `p_da_DSPFifoWrite(instance, data, samples)`

these FIFOs are referenced by TDM assignment (FIFO IDs like 0x080D) and by the
CSS level 0 dispatch. the CSS level 0 calc functions access them via the
`dfl_get_itab_entry` → `dfl_get_param_addr` parameter system, reading buffer
pointers from element descriptors populated by `dfl_module_startup`.

## codec / audio routing layer

### p_da_SetCODEC(codec_mask)

selects codec hardware based on a bitmask. routes through `p_dsp_set_codec()`.
the codec selection is tied to DRT (digital radio transmit) configuration.

### p_da_SetAudioRouter(pst_ART, numEntries)

programs the audio routing table — a sorted array of `{callNr, destSB, srcSB}` entries.
entries are insertion-sorted by a composite 32-bit key for efficient binary search.
this is what actually wires signal blocks together at runtime.

## FXS (SLIC) control

### p_Do_FXS(pSM, usmState, unit, elem, ind, ...)

FXS unit state machine handler. processes events for UIDs 0x0200-0x0207:
- DTMF mute control (elem 0x13)
- CIT (caller ID transmission) start (elem 0x12)
- VFD state switching (elem 0x3b)
- various indication types trigger state transitions

### related functions
- `p_Trans0_FXS` — FXS state transition handler
- `p_MuteAutoOffFxs` / `p_MuteOffFxs` — mute control
- `p_suppressFxsTone` — tone suppression
- `p_updateMuterFxsState` — muter state update
- `p_dat1_InitLine(lineId)` — initialize a phone line's DTMF/dial state

## voice service (COMA, top of CSS stack)

CSS-side functions:
- `voice_process_message()` @ 0x0229ffbc — dispatches incoming voice cmsg
- `voice_start_session()` @ 0x022a0cc4 — routes to rtp/t38/aec handlers
- `voice_start_rtp()` @ 0x022a0a58 — fills RTP_DATA, calls p_rtpapp_Start
- `voice_get_session()` / `voice_free_session()` — session lifecycle
- `voice_set_session_fifos()` — maps CFIFO physical addresses via MMU
- `voice_send_dtmf()` / `voice_send_evt()` — in-band events
- `create_voice_message()` — builds response messages

the voice service manages encoder/decoder CFIFOs that carry codec-encoded audio
frames between the CSS DSP and the linux kernel's /dev/voiceXX devices.

**critical**: the START_SESSION reply is NOT sent synchronously. the full
chain is documented in [voice_service_and_app_dsp.md](voice_service_and_app_dsp.md):
p_rtpapp_Start → p_rtpapp_unit_mode_set (async DUA) → callback →
p_rtpapp_start_enc_dec → p_rtpapp_ENCDECstart (AUC alloc) → BGSC start →
p_rtpapp_SetSesionStatus (sends reply). this requires ARM-side BGSC threads
to be running. without BGSC, this chain stalls at AUCChannelPairStart.

ALL codecs including L16 go through AUC channel allocation. there is NO
passthrough/bypass path. the CSS is compiled with -DDSPonARMonly=1, meaning
all codec encode/decode runs on the ARM in app_dsp's BGSC threads. however,
for L16 specifically, the ARM codec functions are **pure no-ops** — all actual
audio data movement happens on the CSS in level 0 dispatch. the ARM just needs
to maintain BGSC infrastructure (ring buffer messages, control word protocol,
element descriptor registration). see [bgsc_shared_memory.md](bgsc_shared_memory.md).
