# TAPI / SLIC hardware control (APP-side)

## overview

SLIC control is completely separate from the COMA/DUA/CSS audio stack.
it goes through TAPI (Telephony API) ioctls on `/dev/fxsXX` devices,
handled by kernel modules that talk directly to Silicon Labs SLICs over SPI.

the TAPI interface is derived from Infineon/Lantiq's VINETIC SDK (TAPI 2.x),
heavily mutated through grandstream's internal fork.

## kernel module stack

```
drv_tapi.ko (115 KB)    — TAPI layer, ioctl dispatch, ring state machine
    │
drv_silabs.ko (373 KB)  — SiLabs ProSLIC API, hardware register access
    │
bsp_ht.ko (13 KB)       — board support, GPIO, IRQ, SPI access
    │
Si3217x / Si3226x       — physical SLIC chips (SPI bus)
```

all modules live at: `materials/mtdblock/ht818base/lib/modules/slic/`
they have decent symbol coverage (exported kernel symbols).

## device files

- `/dev/slic_bsp` — BSP control (init, reset, version)
- `/dev/fxs0` through `/dev/fxs7` — per-port FXS control (8 ports on HT818)
- note: TAPI driver drops battery when all fds to a port close

## ioctl encoding (IMPORTANT)

GS uses TWO ioctl encoding styles, validated via `Get_IOCTL_Str()` in
drv_tapi.ko at 0x00014d20:

- **simple**: `(type << 8) | nr` where type = 0x71 ('q'). used for ioctls
  with plain int args (line feed, hook status, ring start/stop, events).
- **_IOW encoded**: full Linux `_IOW(type, nr, size)` with direction and
  size bits. used for ioctls that take struct pointers.
  - `0x4001xxxx` = _IOW with 1-byte size field
  - `0x4002xxxx` = _IOW with 2-byte size field
  - `0x4004xxxx` = _IOW with 4-byte (pointer) size field

previously we assumed all ioctls used simple encoding. this was WRONG
for several ioctls (LINE_TYPE_SET was 0x7107 but is actually 0x40047147;
0x7107 is PCM_ACTIVATION_GET).

## complete ioctl table (validated against Get_IOCTL_Str)

### /dev/fxsXX — simple encoding (plain int arg)

| ioctl    | name                        | arg          | notes |
|----------|-----------------------------|--------------|-------|
| `0x7100` | IFX_TAPI_VERSION_GET        | char*        | |
| `0x7101` | IFX_TAPI_LINE_FEED_SET      | int (enum)   | see line feed section |
| `0x7102` | IFX_TAPI_RING_CFG_SET       | struct*      | ring waveform params |
| `0x7103` | IFX_TAPI_RING_CADENCE_HR_SET| struct*      | cadence bitmap, MUST call before RING_START |
| `0x7104` | IFX_TAPI_PCM_CFG_SET        | struct*      | per-channel PCM config |
| `0x7105` | IFX_TAPI_PCM_CFG_GET        | struct*      | read PCM config |
| `0x7106` | IFX_TAPI_PCM_ACTIVATION_SET | int          | enable/disable PCM |
| `0x7107` | IFX_TAPI_PCM_ACTIVATION_GET | int*         | **NOT LINE_TYPE_SET!** |
| `0x7108` | IFX_TAPI_TONE_LEVEL_SET     | int          | |
| `0x710c` | IFX_TAPI_METER_CFG_SET      | struct*      | |
| `0x710d` | IFX_TAPI_METER_START        | —            | |
| `0x710e` | IFX_TAPI_METER_STOP         | —            | |
| `0x710f` | IFX_TAPI_CH_INIT            | 0            | channel init, call first |
| `0x7110` | IFX_TAPI_EXCEPTION_MASK     | int          | |
| `0x7111` | IFX_TAPI_PCM_IF_CFG_SET     | struct*      | PCM interface config |
| `0x7112` | IFX_TAPI_DEBUG_REPORT_SET   | int          | |
| `0x711f` | IFX_TAPI_RING_CFG_GET       | struct*      | read ring config |
| `0x7183` | IFX_TAPI_RING               | —            | blocking ring |
| `0x7184` | IFX_TAPI_LINE_HOOK_STATUS_GET| int*        | 0=on-hook, 1=off-hook |
| `0x7187` | IFX_TAPI_RING_START         | 0            | non-blocking ring start |
| `0x7188` | IFX_TAPI_RING_STOP          | 0            | |
| `0x7189` | IFX_TAPI_LINE_LOGICAL_POLARITY_STATE_GET | int* | polarity state |
| `0x71a1` | IFX_TAPI_TONE_BUSY_PLAY     | —            | |
| `0x71a2` | IFX_TAPI_TONE_RINGBACK_PLAY | —            | |
| `0x71a3` | IFX_TAPI_TONE_DIALTONE_PLAY | —            | |
| `0x71a4` | IFX_TAPI_TONE_STOP          | int          | |
| `0x71c0` | IFX_TAPI_EVENT_GET          | tapi_event_t*| 16-byte event struct |
| `0x71c1` | IFX_TAPI_EVENT_ENABLE       | tapi_event_t*| |
| `0x71c2` | IFX_TAPI_EVENT_DISABLE      | tapi_event_t*| |

### /dev/fxsXX — _IOW encoding (struct pointer arg)

| ioctl          | name                      | notes |
|----------------|---------------------------|-------|
| `0x40017185`   | IFX_TAPI_RING_MAX_SET     | max ring count |
| `0x40027186`   | IFX_TAPI_RING_CADENCE_SET | simple cadence |
| `0x4001719b`   | IFX_TAPI_TONE_LOCAL_PLAY  | tone table index; **25 = dialtone** |
| `0x400171ab`   | IFX_TAPI_TONE_LOCAL_STOP  | |
| `0x400171ac`   | IFX_TAPI_TONE_NET_STOP    | |
| `0x400171c5`   | IFX_TAPI_TONE_NET_PLAY    | |
| `0x40047109`   | IFX_TAPI_LINE_IMPEDANCE_SET | |
| `0x40047124`   | IFX_TAPI_MAP_DATA_ADD     | |
| `0x40047125`   | IFX_TAPI_MAP_DATA_REMOVE  | |
| `0x4004712a`   | IFX_TAPI_MAP_PHONE_ADD    | |
| `0x4004712b`   | IFX_TAPI_MAP_PHONE_REMOVE | |
| `0x4004712e`   | IFX_TAPI_LINE_HOOK_VT_SET | hook thresholds |
| `0x4004713e`   | IFX_TAPI_TEST_HOOKGEN     | |
| `0x4004713f`   | IFX_TAPI_TEST_LOOP        | |
| `0x40047142`   | IFX_TAPI_PHONE_VOLUME_SET | |
| `0x40047143`   | IFX_TAPI_MAP_PCM_ADD      | |
| `0x40047144`   | IFX_TAPI_MAP_PCM_REMOVE   | |
| `0x40047145`   | IFX_TAPI_PCM_VOLUME_SET   | |
| `0x40047146`   | IFX_TAPI_LINE_LEVEL_SET   | |
| `0x40047147`   | IFX_TAPI_LINE_TYPE_SET    | FXS/FXO config struct |
| `0x40047148`   | IFX_TAPI_LASTERR          | |
| `0x40047136`   | IFX_TAPI_TONE_TABLE_CFG_SET | |
| `0x400471b0`   | IFX_TAPI_CID_CFG_SET      | caller ID config |
| `0x400471b1`   | IFX_TAPI_CID_TX_INFO_START| |
| `0x400471b2`   | IFX_TAPI_CID_TX_SEQ_START | CID+ring sequence |
| `0x400471b5`   | IFX_TAPI_CID_TX_INFO_STOP | |
| `0x400471d6-e1`| IFX_TAPI_FXO_*            | FXO ioctls (HT818 has no FXO) |

### /dev/slic_bsp

| ioctl | name               | purpose |
|-------|--------------------|---------|
| `0x0` | HT_BSP_VERSION     | get BSP version string |
| `0x1` | HT_BSP_INIT        | init BSP, returns model/SLIC info |
| `0x2` | HT_BSP_NOOP        | no-op |
| `0x3` | HT_BSP_RESET_ASSERT| assert SLIC reset |
| `0x4` | HT_BSP_RESET_CLEAR | clear SLIC reset |

HT_BSP_INIT returns:
```c
struct ht_bsp_init_result {
    char* model;          // kernel pointer to model name string
    int slic_count;       // number of SLIC chips
    int slic_channels;    // channels per SLIC
    int daa_count;        // DAA (FXO) chips
    int daa_channels;     // channels per DAA
    int ren;              // max ringer equivalence number
    int unk0, unk1;
};
```

## line feed states (verified from drv_silabs.ko RE)

gs_ata has its OWN internal enum, remapped in Nuvoton::setLineState
(FUN_00020eec in gs_ata) before calling ioctl 0x7101:

```
gs_ata enum → ioctl value → ProSLIC LINEFEED register (0x1E)
    0       →     4        →  0 (OPEN)           — disabled, line off
    1       →     2        →  1 (FWD_ACTIVE)     — "standby" (same as active!)
    2       →     0        →  1 (FWD_ACTIVE)     — active, battery on
    3       →     1        →  5 (REV_ACTIVE)     — reversed polarity
    4       →    23        →  5 (REV_ACTIVE)     — GS custom reversed
```

full ioctl→ProSLIC mapping from IFX_TAPI_LL_ALM_CPE_Line_Mode_Set
(0x000401d4 in drv_silabs.ko):

| ioctl val | ProSLIC state | meaning |
|-----------|---------------|---------|
| 0 (ACTIVE) | 1 FWD_ACTIVE | battery on, phone works |
| 1 (ACTIVE_REV) | 5 REV_ACTIVE | reversed polarity |
| 2 (STANDBY) | 1 FWD_ACTIVE | **same as ACTIVE electrically!** |
| 3 (HIGH_IMPEDANCE) | — | rejected, returns -1 |
| 4 (DISABLED) | 0 OPEN | line off |
| 6 (NORMAL_AUTO) | 1 | same as ACTIVE |
| 7 (REVERSED_AUTO) | 5 | same as ACTIVE_REV |
| 8, 9 | — | rejected (drv_silabs default case) |
| 10 (RING_BURST) | 4 RINGING | |
| 11 (RING_PAUSE) | 1 FWD_ACTIVE | |
| 21 | 2 FWD_OHT | on-hook transmission |
| 22 | 6 REV_OHT | reversed on-hook transmission |
| 23 (GS custom) | 5 REV_ACTIVE | |
| 24 (GS custom) | 0 OPEN | + disables hook interrupt |

**STANDBY does NOT implement low-power standby.** grandstream maps it
to the same ProSLIC state as ACTIVE. drv_tapi.ko tracks it logically
for polarity management only.

## TAPI events (verified on hardware)

event struct = 16 bytes (4 x uint32), from IFX_TAPI_EventFifoGet():
```c
struct {
    uint32_t id;       // event type | subtype
    uint16_t ch;       // channel
    uint16_t more;     // more events in FIFO?
    uint32_t data[2];  // event-specific data
};
```

event IDs match upstream Infineon SDK (verified via Get_IFX_TAPI_EVENT_ID_Str):

| event ID     | name | notes |
|--------------|------|-------|
| `0x20000002` | FXS_RINGBURST_END | |
| `0x20000003` | FXS_RINGING_END | |
| `0x20000004` | FXS_ONHOOK | |
| `0x20000005` | FXS_OFFHOOK | |
| `0x20000006` | FXS_FLASH | |
| `0x30000001` | PULSE_DIGIT | data byte1 = pulse count |
| `0x31000001` | DTMF_DIGIT | data byte2=ASCII, byte1=index, byte0=action |
| `0x32000001` | CID_TX_SEQ_START | |
| `0x32000002` | CID_TX_SEQ_END | |
| `0xF2000005` | FAULT_LINE_OVERTEMP | |
| `0xF2000006` | FAULT_LINE_OVERCURRENT | |
| `0xF2000007` | FAULT_LINE_OVERVOLTAGE | |

### DTMF digit data format (byte layout of data[0]):
```
bits 23-16: ASCII character ('0'-'9', '*', '#')
bits 15-8:  digit index (1-9, 0xa='0', 0xb='*', 0xc='#')
bits 7-0:   action (1 = key down)
```

### pulse digit data format:
```
bits 15-8: pulse count (1-9 = digits 1-9, 11 = digit '0')
bits 7-0:  always 0x00
```
note: digit '0' reports as count=11, not 10. GS quirk.

## ringing state machine (from gs_ata RE)

gs_ata's ring sequence (Nuvoton::startRing at FUN_00020d38):
```
1. ioctl(fd, 0x7103, cadence_struct)    // RING_CADENCE_HR_SET — always first!
2. ioctl(fd, 0x7187, 0)                 // RING_START
```

gs_ata's ring stop (Nuvoton::stopRing at FUN_00021fe0):
```
1. ioctl(fd, 0x7188, 0)                 // RING_STOP
2. if on-hook AND call type 9:
   setLineState(STANDBY)                // conditional restore
```

**critical: do NOT set line feed state around ringing.** the driver
manages line feed transitions internally. setting ACTIVE before
RING_START confuses drv_tapi.ko's internal state machine and causes
the octuple_ring_sema to get stuck.

ring cadence struct (each bit = 50ms, 1=ring 0=silent):
```c
struct {
    uint8_t data[40];     // periodic cadence bitmap
    int32_t nr;           // valid bits in data[]
    uint8_t initial[40];  // initial cadence (played once)
    int32_t initialNr;    // valid bits in initial[]
};
```
standard NA: 2s on / 4s off = data[0..4]=0xff, nr=120.

## gs_ata Nuvoton class

the SLIC abstraction in gs_ata is a C++ class called `Nuvoton` (RTTI: `7Nuvoton`).
detects both Si32260 (SiLabs) and N682386 (Nuvoton) chips.

key methods (from string xrefs):
- `Nuvoton::setLineState` (FUN_00020eec) — enum remap + LINE_FEED_SET
- `Nuvoton::startRing` (FUN_00020d38) — simple ring
- `Nuvoton::startRingWithCID` (FUN_00021404) — CID+ring via CID_TX_SEQ_START
- `Nuvoton::stopRing` (FUN_00021fe0) — ring stop + conditional line restore
- `Nuvoton::loadRing` (FUN_00023e38) — parse cadence string → bitmap
- `Nuvoton::setRingCadence` (FUN_00020c94) — ioctl 0x7103
- `Nuvoton::run` (FUN_000245c0) — main event loop (select + EVENT_GET)
- `Nuvoton::getHookStatus` (FUN_00021dac) — ioctl 0x7184
- `Nuvoton::getLogicPolarityState` (FUN_000225b0) — ioctl 0x7189

## SiLabs ProSLIC API (from drv_silabs.ko symbols)

the kernel module wraps Silicon Labs' ProSLIC API:

### line control
- `ProSLIC_SetLinefeedStatus(state)` — set LINEFEED register (0x1E)
- `ProSLIC_GetLinefeedStatus(&state)` — read current state
- `ProSLIC_ReadHook()` — read hook switch state
- `ProSLIC_EnableOneInterrupt(9)` / `ProSLIC_DisableOneInterrupt(9)` — hook IRQ

### ringing
- `ProSLIC_RingSetup(preset)` — configure ring waveform
- `ProSLIC_GetRingPreset()` — get ring preset config
- ring waveform tables: `sineRingFreqTable`, `trapRingFreqTable`

### audio
- `ProSLIC_TXAudioGainSetup()` — transmit gain
- `ProSLIC_RXAudioGainSetup()` — receive gain
- `ProSLIC_DCFeedSetup()` — DC feed configuration

### ProSLIC LINEFEED register values (Si32260)
```
0 = OPEN (line off)
1 = FORWARD_ACTIVE (battery on, normal polarity)
2 = FORWARD_OHT (on-hook transmission)
3 = TIP_OPEN
4 = RINGING
5 = REVERSE_ACTIVE (reversed polarity)
6 = REVERSE_OHT
```

## operational notes

- TAPI driver drops line battery when all fds to a port close. asterisk
  driver will need to keep fds open for managed ports.
- tapi_port_open() must NOT send CH_INIT/LINE_TYPE_SET when running
  alongside app_dsp — reinitializing clobbers driver state.
- tone table indices are populated by gs_ata at boot. index 25 = dialtone.
  indices are sparse (index 1 fails).
- the upstream drv_tapi-3.13.0 source in materials/ is the Infineon SDK,
  NOT what's on the device. the device runs GS's mutated fork (binary only).
  ioctl numbers, enum values, and behavior may differ.
