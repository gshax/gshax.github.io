# TDM grant investigation (2026-03-31, latest update)

## status: root cause identified — missing DSP pipeline setup

**update 2026-03-31:** we traced through the CSS firmware dispatch chain and
compared the stock `lib_dua_init()` sequence against libcomatose. the bytecodes
are identical (confirmed by raw memory comparison), but we were missing a critical
prerequisite: **UMT static mode 1 must be executed on each FXS unit BEFORE
sending the TDM assignment**, because mode 1 creates the DSP FIFOs that the TDM
assignment maps timeslots to. without mode 1, the FIFOs don't exist and the TDM
assignment writes to an empty routing table.

### previous blockers (may be resolved by the fix)

1. **UMT bytecode via DUA does not execute** — the TDM assignment blob sent via
   `DUA_PARAM_UMT_IMMEDIATE` is acknowledged by the CSS but never runs. manual
   assignment via the CSS debug shell (`tdm assign` commands) DOES work.
   **hypothesis: the FIFOs referenced in the blob don't exist yet, so the
   assignment succeeds structurally but has no routing effect.**

2. **`tdm_enable()` panics the CSS** — after manual assignment + successful grant
   via CSS shell, `tdm_enable()` crashes. the handler_private_data struct is mostly
   zeros (only +0x28 channel count is set by commitTDMAssignment). the base address,
   IRQ, and other fields at their correct offsets are needed for tdm_enable to work.
   **hypothesis: commitTDMAssignment double-buffer swap may work correctly once
   FIFOs exist — the base addr/IRQ/clock fields might come from the FIFO's parent
   TDM instance data, which mode 1 initializes.**

## the root cause (confirmed via CSS debug shell)

the CSS's `tdm_grant()` reads from `tdm_private_data[id]` which points to an
all-zero buffer in DDR (`0x022db770`). the actual TDM init data lives in DTCM
(`0x02103b88`) where `tdm_init()` wrote the base address, IRQ, and clock config.
the pointer table was never updated to point at the init data.

### CSS memory layout (confirmed by memory dump)

```
tdm_private_data @ 0x022dd078 (pointer table, 3 entries):
  [0] = 0x022db770  → points to tdm_buffer[0] (all zeros in DDR)
  [1] = 0x022dbfc8  → points to tdm_buffer[1]
  [2] = 0x022dc820  → points to tdm_buffer[2]

tdm_init target @ 0x02103b88 (actual init data in DTCM, stride 0x1c):
  TDM0 @ 0x02103b88:
    +0x00: 0x0202d9f5 (ISR function pointer)
    +0x04: 0x05f00000 (TDM0 hardware base address)
    +0x08: 0x00000020 (IRQ 32)
    +0x18: 0x0000000b (clock config)
```

the pointer table points to the wrong place. `tdm_grant` reads from the pointer
table, finds zeros at +0x14 (granted), +0x1c (channels), etc., and rejects.

### TDM hardware state (confirmed via CSS shell `tdm regs 0`)

```
TDM_ENABLE  (0x00) = 0x00000000  (disabled)
TDM_CFG_1   (0x04) = 0x00001807  (slots=8 in bits[4:0]=7, configured by kernel)
TDM_CFG_2   (0x08) = 0x00000302
TDM_FSYNC   (0x10) = 0x00000000
TDM_FSYNC_D (0x14) = 0x0000000F
TDM_FIFO_WM (0x18) = 0x00000E0E
TDM_INT_EN  (0x20) = 0x0000000F
```

the kernel's gs-tdm driver (built-in) configured the TDM hardware at boot.
slots=8 is correct. but the CSS's software state doesn't reflect this.

## what we've confirmed through brute-force testing

### COMA message field ordering

the kernel source has `struct tdm_msg_grant {type, id, cookie, channels, sample_size, rate}`
but the CSS expects the fields as `{type, id, rate, channels, sample_size, cookie}`.

**evidence:**
- with original struct order: always -7 (channel mismatch, because rate=0 at cookie position)
- with swapped struct (rate before cookie): error changed from -7 to -6 (rate=0 → now rate check fires)
- putting rate value in id position: error changed to -1 (id >= 3)
- **conclusion: the struct layout in the GPL source DOES NOT match the compiled binary**

however, this is a SECONDARY issue. even with correct field ordering, `tdm_grant`
checks channels first, and `channels=0` in the instance data means nothing passes.

### CSS shell `tdm grant` command

the shell's `tdm grant` prints "TDM 0 granted" but `tdm stats` shows `granted: no`.
the shell function calls `shell_tdm_grant → tdm_grant` which silently fails (returns
non-zero) but the shell ignores the return value.

### the "error setting the rate" issue

`shell_tdm_set_rate` and `shell_tdm_get_rate` also fail because they read from
the same `tdm_private_data` pointer table which points to all-zero buffers.

## what the stock firmware does differently

from boot log analysis and RE:

1. **t=1.77s**: kernel gs-tdm configures TDM hardware (registers)
2. **t=15.96s**: CSS firmware loads, `css_init_board()` calls `tdm_init()`
3. **t=17-32s**: `app_dsp` does full DUA init via `lib_dua_init()`:
   - `duasync_init()` → opens COMA "dua" socket
   - allocates all 8 FXS units
   - per-FXS: UnitSetReq with various params (elem=-2, elem=0x13, elem=0x3b)
   - `UnitConnectReq(uid, -3)` for each FXS
   - allocates all 16 SPVOIPNDA units
   - connects SPVOIPNDA units to FXS connections
4. **t=32.46s**: `dua_set_fxs_tdm()` sends TDM assignment UMT blob then writes
   to `/proc/gs/css_own_tdm0` → `dvf_tdm_grant()` → `coma_tdm_grant()` → CSS ACKs

the key difference: on stock, the DUA setup does UnitSetReq with elem=-2 (SETTOG),
elem=0x13, elem=0x3b BETWEEN allocations, and connects each FXS with connId=-3.
we skip all of these intermediate steps. **one of these UnitSetReq calls may be
what initializes the TDM instance data pointer linkage.**

## what we're missing (UPDATED 2026-03-31)

our OLD init sequence (broken):
1. sharedmem init ✓
2. DUA InitReq + ApplInit ✓
3. allocate all 8 FXS ✓
4. connect(-3) per FXS ✓
5. TDM assignment UMT ← accepted but FIFOs don't exist!
6. TDM grant ✗

stock init sequence (lib_dua_init, fully decoded from ghidra):
```
per-FXS unit (×8):
  1. UnitAllocateReq(type=2, spec=i)
  2. UnitSetReq(uid, elem=-2, 0x10100, mode=1, 0)  ← UMT_EXEC_GEN mode 1!!
  3. UnitSetReq(uid, elem=0x13, 0x100FF, dtmf_data, 12)  ← USM_DO: DTMF config
  4. UnitSetReq(uid, elem=0x3b, 0x100FF, 1, 0)  ← USM_DO: VFD state
  5. UnitConnectReq(uid, -3)
  6. UnitSetReq(uid, elem=-1, 0x1010A, callback_ptr, 0)  ← CBK_FUNC

per-VOIP unit (×16):
  1. UnitAllocateReq(type=0, spec=i)
  2. UnitConnectReq(voip_uid, fxs_conn_id)

then:
  dua_set_fxs_tdm() → UMT_IMMEDIATE with TDM assignment + /proc grant
```

**the critical missing step is #2: UMT_EXEC_GEN with mode=1.** this runs the
FXS DSP pipeline setup bytecode at 0x0203581F in _css.elf, which uses UMT
opcodes 26/10/9 to create per-port DSP elements including the FIFOs that TDM
assignment maps timeslots to.

NEW init sequence (fixed in libcomatose):
1. sharedmem init
2. DUA InitReq + ApplInit
3. per-FXS: allocate → **UMT mode 1** → connect(-3)
4. allocate 16 VOIP units
5. TDM assignment UMT_IMMEDIATE
6. TDM grant

## RESOLVED (2026-03-31): audio working!

all three blockers fixed. bidirectional audio confirmed on hardware after full
reboot. the fix required:
1. FXS UMT mode 1 per unit (creates DSP FIFOs)
2. proper async callback waiting (sync response ≠ completion)
3. init callback drain after ApplInit
4. correct VOIP-to-FXS connection topology (2 VOIP per FXS conn)
5. conn_merge for intercom routing

remaining issue: TDM grant only works after full device reboot, not after
css_reset alone. the stock kernel's BUG_ON in the TDM NACK handler kills the
COMA kthread, and this doesn't recover from just reloading CSS firmware.
fix: use comatose_tdm.ko which handles NACKs properly, or find a way to
restart the kernel's COMA kthread.

## next steps

1. **fix css_reset workflow** — either deploy comatose_tdm.ko or find the
   kthread restart mechanism so we don't need full reboots
2. **investigate audio quality** — check for dropouts, echo, level issues
3. **hook detection** — wire up SLIC hook state to DUA event callbacks
4. **asterisk channel driver** — the end goal

## tools available

- `css_shell` — CSS debug console (works after DUA init)
- `tdm_diag` — minimal DUA setup with clean shutdown
- `comatose_tdm.ko` — replacement TDM kernel module with proper nack handling
- `dua_intercom` — full intercom setup (blocks, hangs CSS on exit)

## CSS debug shell useful commands

- `tdm stats N` — TDM instance state (granted, rate, channels, routing)
- `tdm regs N` — hardware register dump
- `tdm info` — number of TDM instances
- `tdm grant N rate ch bps` — grant (hex values, silently fails)
- `tdm read N offset` / `tdm write N offset value` — hardware register access
- `tdm clock N on/off` — enable/disable TDM clock
- `tdm assign ...` — TDM routing commands
- `memory dump addr size` — read CSS memory (hex)
- `memory set addr size byte` — write byte to CSS memory
- `version` — firmware version

## subagent audit findings (2026-03-30)

a fresh-context audit of all notes produced these corrections and insights:

### corrected false assumptions

1. **"UnitConnectReq(uid, -3) might link FXS to TDM"** — INCORRECT. connId=-3
   creates a DUA audio routing connection via `p_duax_CreateConn(2)`, not a TDM
   link. the TDM linkage is done by the TDM assignment blob's commit function.

2. **"pData=NULL for operation 1 in per-FXS init"** — INCORRECT. the decompiler
   showed NULL but the assembly at 0xe698 shows `mov r3,#0x1`. pData=1, meaning
   this is `DUA_PARAM_PIN_MUTE` with value=1 (mute enabled).

3. **"the UMT blob is working (step 5 ✓)"** — INCORRECT as stated. DUA returns OK
   meaning the message was received, NOT that the bytecode executed. confirmed by
   CSS shell: `tdm stats 0` shows `channels configured: 0` after our UMT blob,
   but `channels configured: 8` after manual `tdm assign` commands.

4. **"tdm_private_data pointers are never updated"** — PARTIALLY INCORRECT.
   `commitTDMAssignment` DOES update the pointer table by swapping to a
   double-buffer. the issue is that the commit never runs because the UMT
   bytecode doesn't execute.

### per-FXS operations (FULLY DECODED 2026-03-31)

the 6 operations stock lib_dua_init does per FXS unit, with resolved param codes
from ghidra analysis of libcordless.so at 0x0001e538:

1. `UnitSetReq(uid, elem=-2, 0x10100=UMT_EXEC_GEN, mode=1, dataSz=0)`
   → **executes FXS UMT static mode 1** (DSP pipeline setup)
   → mode 1 bytecode at 0x0203581F in _css.elf (genModes[1] for FXS type)
   → uses UMT opcodes 26 (8×, elems 0-7), 10, 9 to create DSP elements
   → **creates the FIFOs that TDM assignment references** (0x080d, 0x0810, etc.)

2. `UnitSetReq(uid, elem=0x13, 0x100FF=USM_DO, dtmf_config, 12)`
   → dispatches USM handler for DTMF config (elem 0x13)
   → data: {1, 0xFFFF, 2, 0x1E, 0x1E} packed as 12 bytes

3. `UnitSetReq(uid, elem=0x3b, 0x100FF=USM_DO, 1, dataSz=0)`
   → dispatches USM handler for VFD state (elem 0x3b)

4. `UnitConnectReq(uid, connId=-3)`
   → creates DUA audio routing connection (NOT a TDM link)

5. `UnitSetReq(uid, elem=-1, 0x1010A=CBK_FUNC, 0x0001E350, dataSz=0)`
   → sets application callback function pointer on the FXS unit type

**step 1 is the critical missing piece for TDM.** our earlier research incorrectly
identified this as "DUA_PARAM_PIN_MUTE" — it is actually UMT_EXEC_GEN (0x10100)
which dispatches through p_ugt1_GenSetFunc to execute the FXS genModes[1] bytecode.
the DAT value 0x0001e8d4 was 0x00010100, not 0x10006 (PIN_MUTE).

### key experimental results

**manual TDM assign via CSS shell WORKS:**
```
tdm assign start
tdm assign 0 fifo 0 tx 810
tdm assign 0 fifo 0 rx 80d
... (all 8 channels)
tdm assign commit
→ tdm stats 0 shows "channels configured: 8" with correct FIFO mappings
```

**CSS shell grant after manual assign:**
- channels=8 at +0x28 is accepted (no more -7 error!)
- but `tdm_enable()` panics the CSS
- the handler_private_data struct at 0x022db770 is still mostly zeros
- only the routing info and channel count at +0x28 are populated
- `tdm_enable` needs base addr (+0x04), rate (+0x18), etc. which are all 0

**the two remaining blockers are:**
1. why doesn't the UMT blob execute via DUA? the CSS shell's `tdm assign` uses
   the same underlying functions. possibly `DUA_PARAM_UMT_IMMEDIATE` stores
   but doesn't execute, or the FXS unit's SetFunc ignores this param.
2. the handler_private_data needs its full initialization (base addr, IRQ, clock,
   rate) before `tdm_enable` can work. this data exists at DTCM 0x02103b88
   (written by `tdm_init`) but the pointer table points to empty DDR buffers.
   `commitTDMAssignment` swaps to a double-buffer, but the double-buffer is
   initialized by copying from the current (empty) handler_private_data.
