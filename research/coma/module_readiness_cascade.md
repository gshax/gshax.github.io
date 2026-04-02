# module readiness cascade — the audio routing trigger (2026-04-02)

**context:** this describes the internal CSS chain that activates audio routing.
it is driven by ARM→CSS ring buffer messages (level-ready, heartbeat) documented
in [ring_buffer_protocol.md](ring_buffer_protocol.md), and sends CSS→ARM events
via the DUA COMA socket as documented in [css_arm_events.md](css_arm_events.md).
the cascade only works if element descriptors are properly populated by the ARM's
`dfl_module_startup` — see [codec_table_investigation.md](codec_table_investigation.md).

## complete chain from ring buffer messages to audio routing

### 1. ring buffer message dispatch (CSS: p_adsp_check_msg_fifo)

```
ARM sends: [instance_addr, (type << 13) | num_params, params...]

CSS reads message from ring buffer:
  → p_dspa_find_instance(instance_addr)
    compares (instance_addr - 4) against ITAB entries
    returns: index | 0x800 | (level << 12)  — the instance HANDLE
  → p_dspa_GetModuleID(instance_handle)
    returns: module_id | 0x800
  → event = (type << 13) | module_id
  → i_msg_cb dispatches to registered callback based on module_id
```

### 2. p_dsp_CallBack processes events

the DSP module's callback handles our type=0 and type=3 events:

**type=0 (completion event):**
```
set flag bit 8 ("pending init")
set retry counter = 8
store instance
```

**type=3 (decoder ready), when bit 8 set:**
```
decrement retry counter
when counter reaches 0:
  clear bit 8
  if (startup_mode_bit set):
    → p_da02_CheckStartupModeReady()
  else:
    → send event 0xf2 to callback
```

### 3. module readiness cascade

p_da_InitReq(mode=5) sets up:
- modules bitmask = 0x0016 (modules 1, 2, 4 must report ready)
- trigger value = 0xf0 (for RouteCODEC)

cascade progression:
```
CheckStartupModeReady → SetModuleReady(2, 0x80) → clears bit 2

type=3 processing → bit 30 check → FlxOpen → SetModeInitReq
  → SetModuleReady(1, 0x80) → clears bit 1, sets bit 31

type=3 processing → bit 31 check → SetModeReq → ...
  → SetModuleReady(4, ...) → clears bit 4

bitmask = 0 → ALL READY
```

### 4. RouteCODEC — the audio routing activation

when all modules report ready (bitmask == 0) AND trigger == 0xf0:

```
p_da_SetCODEC(0)           // configure codec descriptors
p_da_RouteCODEC(0x12)      // set PCM-to-DRT routing (0x12 = two channels)
p_da_SetCODEC(3)           // activate codec

RouteCODEC(0x12):
  stores 0x12 at config+0x13
  calls SetCODEC with stored mode

SetCODEC:
  extracts two codec indices from 0x12 (nibbles: 2, 1)
  looks up codec descriptors from table
  calls p_dsp_set_codec(desc_a, desc_b)
  
p_dsp_set_codec:
  stores codec descriptors
  sets DRT global flags (bits 0-3)
  → enables audio data path between TDM and encoder
```

### 5. CSS level 0 dispatch now routes audio

with DRT global flags set, the CSS level 0 dispatch's signal routing
modules (SSW, SSR, etc.) know to:
- read TDM audio from hardware
- route through DSP pipeline
- write to encoder output buffer
- encoder output → voice CFIFO → /dev/voiceN

## key insights

1. p_dspa_find_instance subtracts 4 from our instance address before ITAB
   lookup. our messages send `cw + 1` (= base + 4), CSS computes base.
   THIS SHOULD MATCH the ITAB entries.

2. the entire cascade is driven by our type=0 and type=3 messages. each
   type=3 event advances the state machine one step. 8 type=3 events
   are needed after each type=0 to trigger the next module-ready step.

3. the cascade takes ~80ms minimum (8 × 10ms type=3 events) per module.
   with 3 modules, the full cascade takes ~240ms.

4. if ANY step fails (bad instance lookup, wrong event routing, missing
   flag), the cascade stops and audio routing never activates.

## verification approach

if RouteCODEC fires, specific shared memory changes should be visible:
- DRT global flags become non-zero
- codec descriptors stored in CSS config struct
- possibly: specific element descriptor fields change

if the cascade is stuck, we need to determine WHICH step fails.
options:
- enable CSS text trace for the dsp module
- check shared memory for cascade side-effects
- add diagnostic ring buffer messages and check CSS responses
