# init sequence comparison: stock app_dsp vs our BGSC (2026-04-02)

**context:** side-by-side comparison of stock `p_dsp_app_startup` (FUN_0007e7c4
in app_dsp) vs our `bgsc_init`. the key difference is step 13: stock's
`dfl_module_startup` writes ~2137 lines of data to element descriptors. our
`module_startup` only writes a small subset. see
[codec_table_investigation.md](codec_table_investigation.md) for what's missing.

## stock p_dsp_app_startup (FUN_0007e7c4)

called when the CSS signals that shared memory is ready (dsp_version at
shm+0x10 is non-zero). performs ALL ARM-side DSP framework initialization.

```
step  action                              shared memory effect
────  ──────                              ────────────────────
1     check *(shm+0x10) != 0             reads DSP version
2     *(shm+0x28) = *(shm+0x08)          saves alloc watermark
3     FUN_000844f8(shm + saved_alloc)     stores ring buf hdr addr (global)
4     *(shm+0x08) += 0x10                 advances watermark
5     *(shm+0x2c) = *(shm+0x08)          saves codec state offset
6     store codec state addr              saves to ARM global
7     *(shm+0x08) += 0x79c               advances watermark
8     FUN_00084508(codec, 0x1e7, ...)     init ring buf: [base,end,write,read]
9     BG_struct[0x244] = 0x10             heartbeat interval (ARM-internal)
10    BG_struct[0x248] = 0x10             heartbeat counter (ARM-internal)
11    shm_ptr_struct[1] = 0              ← clears ARM global (NOT shared memory)
12    FOR level = 0..3:                   BG level struct init (ARM-internal):
        .p_n_FLOW = flow_table[level]       flow table from shm+0xb848 area
        .p_n_FLOW_copy = same
        .tick_count = *(flow_table+0x18)
        .field_10 = -1
        .field_14 = 0
13    FUN_00075810: dfl_module_startup    ← WRITES TO SHARED MEMORY! (36KB func)
        calls 21 module startup functions
        populates element descriptors with:
        - codec function table entries (ARM addrs at +0x74-0xe4)
        - module config (group_type, buf_config, arm_module)
        - codec constants (0x7fa1, 5000, etc.)
14    FOR level = 0..3:
        .state = 1 (IDLE)                ARM BG levels now dispatching
15    *(shm+0x14) = DSP_VERSION           version tag
```

## our bgsc_init

```
step  action                              shared memory effect
────  ──────                              ────────────────────
1     check shm_ptr != NULL               basic validation
2     read alloc_wm from shm+0x08
3     read dsp_version from shm+0x10
4     *(shm+0x14) = dsp_version           version tag                    ✓ same
5     *(shm+0x28) = alloc_wm              saves alloc watermark          ✓ same
6     *(shm+0x2c) = alloc_wm + 0x10       codec state offset             ✓ same
7     memset codec state area to 0
8     init ring buffer (vaddr-based)       header + data area             ✓ same
9     *(shm+0x08) = data_end              advances watermark             ✓ same
10    read element table from shm+0xb854   organize into groups           ✓ same
11    module_startup()                      PARTIAL element population     ⚠ PARTIAL
12    send level-ready × 4                 ring buffer messages            ~ different
```

## what's DIFFERENT

### 1. BG level struct initialization (steps 9-12 in stock)
stock initializes 4 BG level structs with flow table pointers from shared
memory. these are ARM-internal (not in shared memory), so they shouldn't
affect the CSS. our BGSC doesn't use BG level structs — we iterate the
element table directly.
**verdict: probably NOT the issue**

### 2. dfl_module_startup (step 13 in stock)
stock's 36KB function calls 21 module startup functions that write extensively
to element descriptors in shared memory. our module_startup writes:
- elem[0]+4 = 0x0e (codec registered)
- common header for elem[2-21] (+0x10 group_type, +0x18 buf_config, +0x20 arm_module)
- function table entries for elem[2] and elem[6]
- frame config for elem[6] and elem[21]

MISSING from our module_startup (present in stock's 2137-line shm diff):
- elem[1]: 3 ARM addresses (+0x14, +0x1c, +0x40)
- elem[3-5]: function tables, config values
- elem[7-11]: various ARM addresses and config
- elem[12]: 5 ARM addresses (+0x28, +0x2c, +0x48, +0x4c, +0x50)
- elem[13-20]: function tables, config values
**verdict: LIKELY the issue — CSS reads these for routing config**

### 3. pool size (shm+0x04)
stock: 0x80000 (512KB). ours: 0x100000 (1MB).
our dua_init_hw sets 0x80000, but the CSS overwrites to 0x100000.
stock's CSS keeps it at 0x80000.
**verdict: unclear, but audio works with stock app_dsp on our DUA init**

### 4. level-ready vs state machine
stock: BG levels go through state machine (2→3→4→5→0x200→1), send
level-ready from state 0x200. we send level-ready immediately at startup.
**verdict: functionally equivalent (CSS receives level-ready either way)**

### 5. elem[0]+0x10: stock=0x02, ours=0x01
might be "number of registered modules" or BG level count. the CSS might
use this to determine how many levels to process.
**verdict: suspicious, needs investigation**

## CSS-side expectations during init

from CSS firmware RE (p_da_InitReq → p_dsp_startup → dfl_startup):

```
1. CSS receives DUA InitReq from ARM
2. p_dsp_startup:
   - p_dsp_LoadRamCode
   - p_dsp_fsi_startup
   - platform_dspapi_init
   - hardware register wait
   - dfl_startup(num_bg_levels):
     - adsp_startup (init ARM→CSS ring buffer reader)
     - init CSS BG level 0 processing struct (flow table from shm)
     - dfl_module_startup (register CSS codec modules: SKO, SSW, etc.)
     - set level 0 state = 1 (IDLE)
     - write version tag to *(shm + 0x10) via shared memory
   - fsi_enable
   - version check
3. p_da_InitReq:
   - sets module bitmask = 0x0016 (modules 1, 2, 4)
   - sets trigger = 0xf0 (for RouteCODEC)
   - continues DUA processing
```

## module readiness cascade (from init to audio routing)

after both CSS and ARM init complete, the module readiness cascade runs:

```
ARM sends level-ready [0, (1<<13)|1, level]
  → CSS p_dsp_CallBack: sets pending flag (bit 8), retry counter = 8

ARM sends heartbeat [0, (4<<13)|0] every 80ms
  → CSS p_dsp_CallBack: decrements retry counter
  → after 8 heartbeats: CheckStartupModeReady → SetModuleReady(2, 0x80)
  → FlxOpen → SetModeInitReq → SetModuleReady(1, 0x80)
  → SetModeReq → SetModuleReady(4, ...)
  → ALL modules ready → RouteCODEC(0x12) → p_dsp_set_codec → AUDIO ROUTING

total time: ~640ms (8 heartbeats × 80ms) per module step
```

## does the cascade actually complete in our system?

**update 2026-04-02 (session 2):** we now know the cascade PARTIALLY works.
CSS sends 0xf2 events via the DUA COMA socket (see [css_arm_events.md](css_arm_events.md)),
confirming that our ring buffer messages DO trigger p_dsp_CallBack. the cascade
progresses through 6 0xf2 events, then 2 0x0e events (elem 35, 40), then stops.

possible failure modes:
1. ~~level-ready doesn't trigger p_dsp_CallBack~~ CONFIRMED WORKING (0xf2 received)
2. ~~heartbeat doesn't advance the cascade~~ CONFIRMED WORKING (countdown fires)
3. our 0xf2 response (level-ready) doesn't match what stock does (I-switch setup)
4. cascade reaches some stage but not RouteCODEC — need CSS trace to verify
5. RouteCODEC fires but the DRT is misconfigured (missing element descriptor data)

## what to investigate next

1. **shm+0x04 pool size**: does changing to 0x80000 affect anything?
2. **elem[0]+0x10**: what does 0x02 vs 0x01 mean? who writes it?
3. **elem[12] full population**: write ALL stock ARM addresses to see
   if any are needed for TDM routing
4. **CSS text trace**: enable trace for the dsp/codec modules during
   init to see if RouteCODEC fires
5. **stock app_dsp trace**: run stock app_dsp with CSS trace enabled
   to capture the exact init message sequence
