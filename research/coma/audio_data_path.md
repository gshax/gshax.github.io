# audio data path through the CSS

## high-level flow

```
analog phone ←→ SLIC chip
                   ↕
              TDM bus (hardware)
                   ↕ tdm_read_sample() / tdm_write_sample()
              optimized_tdm_handler()
                   ↕ p_da_DSPFifoWrite() / p_da_DSPFifoRead()
              DSP FIFOs
                   ↕
              DUA audio routing (ART - audio routing table)
                   ↕ signal blocks, codec pipeline
              VoIP DSP pipeline (SPVOIPNDA unit)
                   ↕
              COMA voice service
                   ↕ encoder/decoder CFIFOs
              /dev/voiceXX on Linux
                   ↕
              RTP stack (gs_ata)
```

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

### optimized_tdm_handler(p)

the main audio pump loop, called periodically (likely from ISR context):

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

these FIFOs connect the TDM handler to the DSP processing pipeline.

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
- `voice_process_message()` — handles incoming voice cmsg from linux
- `voice_get_session()` / `voice_free_session()` — session lifecycle
- `voice_set_session_*()` — FIFO and RTP configuration
- `voice_session_process()` — per-session frame processing
- `voice_send_dtmf()` / `voice_send_evt()` — in-band events
- `create_voice_message()` — builds response messages

the voice service manages encoder/decoder CFIFOs that carry compressed audio
frames between the CSS DSP and the linux kernel's /dev/voiceXX devices.
