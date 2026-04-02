# CSS→ARM event communication (2026-04-02)

## discovery: CSS sends events via the DUA COMA socket

the CSS communicates DSP framework events to the ARM via DUA async callbacks
(cmd=0x7f) on the same AF_COMA "dua" socket that app_dsp (or our code) opens
for DUA init. these use the same wire format as DUA operation callbacks but
carry cascade control events (0xf2) and element notifications (0x0e).

we were NOT reading them — adding a socket poll (via `dua_poll_event()` in
libcomatose) revealed them immediately.

**note:** this is distinct from the ARM→CSS ring buffer in shared memory
(documented in [ring_buffer_protocol.md](ring_buffer_protocol.md)). the ring
buffer carries messages FROM the ARM TO the CSS (level-ready, heartbeat,
decoder acks). the COMA socket carries events FROM the CSS TO the ARM. there
may also be a CSS→ARM ring buffer in shared memory that stock app_dsp reads
in its `dfl_calc` state machine — this is a known gap in our understanding
(see [ring_buffer_protocol.md](ring_buffer_protocol.md) "CSS→ARM ring buffer"
section).

## observed events after BGSC startup

```
event 1: DUA async callback
  marker:   0xdeadbeef
  uid:      0xfffffeff (broadcast/special)
  instance: 0x00000bff (default DSP module instance)
  result:   0x0000f203 → event 0xf2, sub=3
  
  THIS IS THE CASCADE EVENT! the CSS sends 0xf2 after our level-ready
  + 8 heartbeats trigger the countdown in p_dsp_CallBack.

event 2: DUA async callback
  uid:      0x00000000
  elem:     0x00000023 (= 35, element index)
  result:   0x00000e02 → event 0x0e, sub=2

event 3: DUA async callback
  uid:      0x00000000
  elem:     0x00000028 (= 40, element index)
  result:   0x00000e02 → event 0x0e, sub=2
```

## the cascade stall explained

the module readiness cascade in p_dsp_CallBack:

1. our level-ready [0, (1<<13)|1, level] → matches p_dsp_CallBack type=0
   (via default instance 0x0BFF, event = 0x2BFF, base = -0x2BFF)
2. our heartbeat [0, (4<<13)|0] every 80ms → matches type=3
   (event = 0x8BFF)
3. after 8 heartbeats: retry counter → 0
4. bit 1 of state byte NOT set (bVar1 = 0x40)
5. → callback(instance, 0xf2) instead of CheckStartupModeReady
6. callback dispatches via 0x02100048 → sends DUA async to ARM
7. ARM receives 0xf2 on COMA socket ← CONFIRMED!
8. ARM is supposed to respond → CSS sets bit 1 → cascade continues

step 8 is what we're missing. stock app_dsp responds. we don't.

## event format in DUA async callback

```
word 0-1: zeros (DUA header padding)
word 2:   0x0000007f = cmd 0x7f (async callback)
word 3:   num_params (typically 4)
word 4:   0xdeadbeef = async marker
word 5:   UID (0xfffffeff for broadcast, 0 for unit-specific)
word 6:   element/instance index
word 7:   result code (event << 8 | sub_type, or packed differently)
```

## what stock ARM does with 0xf2

stock app_dsp's FUN_0007ee20 (the "second state machine") handles 0xf2.
it triggers the BG level state machine: 2→3→4(I-switch)→5→0x200→1.
the I-switch setup sets PREPARE_RUN bits on control words. the CSS reads
these to confirm ARM initialization. then level-ready is sent at state 0x200.

our attempts to replicate:
- level-ready only → 0xf2 loops (CSS doesn't accept without I-switch)
- I-switch + level-ready → 0xf2 loops (our dispatch clears PREPARE_RUN
  before CSS sees it, because our dispatch runs continuously unlike stock
  which pauses during I-switch)
- limited responses (max 5) → cascade progresses! after 6 0xf2 events,
  CSS gives up and sends 0x0e events for elem 35/40 instead
- I-switch with dispatch pause → too disruptive, breaks voice_open

the correct fix is likely: implement proper BG level state machine with
dispatch pause during I-switch, matching stock's state 2→3→4→5→0x200→1.

## observed cascade pattern

```
startup:
  level-ready x4 → heartbeat every 80ms
  
~640ms later (8 heartbeats):
  CSS sends 0xf2 (#1) → we respond level-ready
  CSS sends 0xf2 (#2) → we respond level-ready
  ... (repeats up to our limit of 5)
  CSS sends 0xf2 (#6) → we DON'T respond
  
CSS gives up on 0xf2, moves to next stage:
  CSS sends 0x0e for elem 35 (type=2, UnitSet indication)
  CSS sends 0x0e for elem 40 (type=2, UnitSet indication)
  
(no more events after this)
```

the 0x0e events are DUA UnitSet indications for specific elements (elems 35
and 40 = pin elements, see [dua_units_and_elements.md](dua_units_and_elements.md)).
they might be the CSS confirming element configuration, or they might need
responses to complete the cascade toward RouteCODEC.

**relationship to the module readiness cascade:** the 0xf2 events ARE the
cascade mechanism described in [module_readiness_cascade.md](module_readiness_cascade.md).
the cascade was designed to work via ARM→CSS ring buffer messages + CSS→ARM
callbacks. our ring buffer messages (level-ready, heartbeat) trigger the CSS
countdown. when the countdown finishes, CSS sends 0xf2 back via the COMA
socket. the ARM is expected to respond (stock's FUN_0007ee20 triggers the BG
level state machine), and only after several rounds does the cascade progress
to RouteCODEC which activates audio routing.

## key addresses

- CSS state struct: 0x02103174 (CSS BSS, byte 1 = cascade control)
- CSS callback table: 0x02100048 (CSS BSS, dispatches events)
- default DSP instance: 0x0BFF (used for level-ready/heartbeat/events)
- cascade event base: 0x2BFF (= DAT_020248a4 negated)
- p_dspa_find_instance default: DAT_02024e40 = 0x0BFF
