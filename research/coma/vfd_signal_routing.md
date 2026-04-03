# VFD signal routing and the missing audio link (2026-04-02)

## what VFD is

VFD appears to stand for "voice/feed/data" or similar. it's a per-FXS-port
subsystem that manages signal routing between TDM hardware and the DSP pipeline.
it has both a CSS-side component (switchVfdState) and an ARM-side component
(VFD_IN_APP FIFO initialization in libcordless.so).

## CSS-side: switchVfdState

`switchVfdState` at 0x0202c754 in _css.elf. called from `p_Do_FXS` when
`UnitSetReq(fxs_uid, elem=0x3b, USM_DO, 1)` is sent.

```c
switchVfdState(instance, on=1):
  p_da_SwitchInstance(u16_Instance, 0)     // deactivate
  p_da_SwitchInstance(u16_Instance, 0x10)  // PENDING (DFL_CW_POST_CALC_2)
  p_da_SetDSPValue(u16_Instance, ...)      // configure DSP routing
  p_da_SwitchInstance(u16_Instance, 1)     // ACTIVE
```

each call resolves to `dfl_set_I_switch(instance, value)` which writes directly
to the element's control word via the CSS ITAB.

### VFD instance handles (per FXS port)

VFD struct base: `DAT_0202c7cc` → 0x024358d0 (CSS DDR). stride = 0x638 per port.

per-port struct contains 3 CSS-internal DFL instance handles:
```
+0x00: SSW instance (signal switch write)
+0x04: SSR instance (signal switch read)
+0x08: main instance
```

for FXS port 0:
- SSW: 0x080E → ITAB[0x0E] → **0x02109bd0 (CSS DTCM, not shared memory!)**
- SSR: 0x080C → ITAB[0x0C] → **0x02109b68 (CSS DTCM, not shared memory!)**
- main: 0x080F → ITAB[0x0F] → 0xb6ef36f0 (shared memory)

**critical finding:** the SSW and SSR signal routing elements live in CSS
DTCM, not shared memory. they are invisible to the ARM. the CSS level 0
dispatch accesses them directly. `switchVfdState` activates them correctly
but we cannot observe or verify this from the ARM side.

### ITAB resolution

```
dfl_get_itab_entry(instance):
  level = (instance >> 12) & 7
  index = instance & 0x3ff
  return level_table[level][index]
```

level 0 ITAB base: 0x02102968 (at `DAT_0201e604` → 0x02102ea8 → [0] = 0x02102968)

## ARM-side: VFD_IN_APP (from boot log)

libcordless.so contains VFD_IN_APP initialization code that runs during stock
boot. from `research/logs/boot_full.txt`:

```
##### VFD_IN_APP  - vfd_init called 
##### VFD_IN_APP Shared mem Init succeesful 
##### VFD_IN_APP - value_array[0] b6cdf6f0, value_array[1] 0, fifo_css_shmem_ptr 0xb6cdf6f0 
##### VFD_IN_APP - After dspfifo_init
##### VFD_IN_APP - fifo_css_shmem_ptr 0xb6cdf6f0, vfdfifo_shmem_ptr 0x1815e0
switchVfdState: value of vfdMemSize is 492
```

this runs once per FXS port (8 times). it:
1. initializes shared memory ("Shared mem Init succeesful")
2. calls `dspfifo_init` with shared memory pointers
3. stores `fifo_css_shmem_ptr` (a shared memory virtual address)
4. stores `vfdfifo_shmem_ptr` (a CSS-side shared memory offset like 0x1815e0)
5. calls `switchVfdState` which prints "value of vfdMemSize is 492"

the `vfdfifo_shmem_ptr` values (0x1815e0, 0x182768, 0x184310, ...) are CSS-side
offsets with stride ~0x11B0 (4528 bytes) per port. these might be the VFD FIFO
data structures in shared memory.

**this ARM-side VFD FIFO initialization is something `app_dsp` does that we
DON'T do.** it may be the missing link that configures the shared memory FIFOs
connecting CSS signal routing (SSW/SSR) to the voice encoder.

## what we know works

1. RouteCODEC fires (DRT register 0 = 0x01) — audio routing enabled ✓
2. VFD DUA call succeeds (elem=59 result=11 per FXS port) ✓
3. SETCODEC activation works (encoder, decoder, buffer go ACTIVE) ✓
4. CSS AUC encoder runs, produces L16 frames ✓
5. dual-buffer offset applied correctly ✓
6. TDM hardware configured identically to stock ✓

## what we know doesn't work

7. audio frames contain silence (zeros)
8. CSS-level elements (e[2]-e[9]) remain INACTIVE (cw=0) in shared memory
   (but the REAL signal routing elements SSW/SSR are in CSS DTCM, invisible)

## what we don't know

1. **what the ARM-side VFD FIFO init does** — libcordless.so's vfd_init creates
   FIFOs in shared memory. are these the data path between CSS signal routing
   and the voice encoder? does the CSS read these FIFO pointers during audio
   routing?

2. **whether stock app_dsp also does VFD FIFO init** — the boot log shows
   VFD_IN_APP from libcordless.so (loaded by gs_ata). does app_dsp do the
   same? or is it only gs_ata/libcordless that does this?

3. **whether dfl_module_startup's element descriptor writes affect CSS-DTCM
   elements** — our module_startup writes to shared memory elements. but the
   SSW/SSR live in CSS DTCM. does the CSS's own dfl_module_startup (called
   during dfl_startup) populate the DTCM elements? if so, they might already
   be properly configured.

4. **whether there are additional DUA commands** stock sends during session
   start that we're missing. gs_ata might do per-call setup beyond what
   lib_dua_init does.

## next steps

1. **RE the VFD FIFO init in libcordless.so** — find the functions that print
   "VFD_IN_APP" and trace what they write to shared memory. search for
   `vfd_init`, `dspfifo_init`, `switchVfdState` strings.

2. **check if stock app_dsp does VFD FIFO init** — search app_dsp binary for
   VFD-related strings or functions.

3. **compare shared memory at 0x1815e0+ region** between stock and our dumps
   to see what VFD FIFO structures look like.

4. **trace CSS voice_start_rtp more carefully** — understand exactly what path
   audio takes from TDM through CSS DTCM signal routing to the voice CFIFO.
   the SSW/SSR elements in DTCM might read config from shared memory.
