# TAPI / SLIC hardware control (APP-side)

## overview

SLIC control is completely separate from the COMA/DUA/CSS audio stack.
it goes through TAPI (Telephony API) ioctls on `/dev/fxsXX` devices,
handled by kernel modules that talk directly to Silicon Labs SLICs over SPI.

the TAPI interface is derived from Infineon/Lantiq's VINETIC SDK (TAPI 2.x),
heavily mutated through grandstream's internal fork. ioctl numbers differ from
any publicly available VINETIC SDK version.

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

## key ioctls for /dev/fxsXX

(from research/ioctl.md — ioctl numbers are grandstream's mutated VINETIC values)

### essential for basic operation

| ioctl | name | direction | purpose |
|-------|------|-----------|---------|
| 0x710f | IFX_TAPI_CH_INIT | write | initialize channel — call first |
| 0x7101 | IFX_TAPI_LINE_FEED_SET | write | set line feed state (power the line) |
| 0x7184 | IFX_TAPI_LINE_HOOK_STATUS_GET | read | get hook state (on/off hook) |
| 0x7187 | IFX_TAPI_RING_START | write | start ringing |
| 0x7188 | IFX_TAPI_RING_STOP | write | stop ringing |

### configuration

| ioctl | name | size | purpose |
|-------|------|------|---------|
| 0x7102 | IFX_TAPI_RING_CFG_SET | varies | configure ring waveform parameters |
| 0x7103 | IFX_TAPI_RING_CADENCE_HR_SET | varies | set ring cadence pattern (high-res) |
| 0x7185 | IFX_TAPI_RING_MAX_SET | 4001 B | set max ring REN |
| 0x7104 | IFX_TAPI_PCM_CFG_SET | varies | configure PCM interface |
| 0x7106 | IFX_TAPI_PCM_ACTIVATION_SET | varies | enable/disable PCM |
| 0x7145 | IFX_TAPI_PCM_VOLUME_SET | 4004 B | set audio volume |
| 0x7109 | IFX_TAPI_LINE_IMPEDANCE_SET | 4004 B | set line impedance |
| 0x7149 | IFX_TAPI_LINE_DC_FEED_SET | 4004 B | set DC feed params |
| 0x712e | IFX_TAPI_LINE_HOOK_VT_SET | 4004 B | set hook thresholds |

### caller ID / tones

| ioctl | name | size | purpose |
|-------|------|------|---------|
| 0x71b0 | IFX_TAPI_CID_CFG_SET | 4004 B | configure caller ID |
| 0x71b2 | IFX_TAPI_CID_TX_SEQ_START | 4004 B | start CID transmission |
| 0x71b5 | IFX_TAPI_CID_TX_INFO_STOP | 4004 B | stop CID transmission |
| 0x719b | IFX_TAPI_TONE_LOCAL_PLAY | 4001 B | play local tone |
| 0x71c5 | IFX_TAPI_TONE_NET_PLAY | 4001 B | play network-side tone |
| 0x71a4 | IFX_TAPI_TONE_STOP | varies | stop tone |

### events

| ioctl | name | purpose |
|-------|------|---------|
| 0x71c0 | IFX_TAPI_EVENT_GET | poll for pending events (hook, DTMF, etc.) |
| 0x71c1 | IFX_TAPI_EVENT_ENABLE | enable specific event type |
| 0x71c2 | IFX_TAPI_EVENT_DISABLE | disable specific event type |

### diagnostics

| ioctl | name | purpose |
|-------|------|---------|
| 0x4e0f | VINETIC_GR909_START | start GR.909 line test |
| 0x4e11 | VINETIC_GR909_RESULT | get GR.909 test results |

## /dev/slic_bsp ioctls

| ioctl | name | purpose |
|-------|------|---------|
| 0x0 | HT_BSP_VERSION | get BSP version string |
| 0x1 | HT_BSP_INIT | initialize BSP, returns model/SLIC info |
| 0x2 | HT_BSP_NOOP | no-op |
| 0x3 | HT_BSP_RESET_ASSERT | assert SLIC reset |
| 0x4 | HT_BSP_RESET_CLEAR | clear SLIC reset |

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

## SiLabs ProSLIC API (from drv_silabs.ko symbols)

the kernel module wraps Silicon Labs' ProSLIC API:

### line control
- `ProSLIC_SetLinefeed(state)` — set line feed (power state)
- `ProSLIC_ReadHook()` — read hook switch state
- `ProSLIC_SetMuteStatus(mute)` — mute/unmute
- `ProSLIC_SetPower(mode)` — power mode

### ringing
- `ProSLIC_RingStart()` — start ring
- `ProSLIC_RingStop()` — stop ring
- ring waveform tables: `sineRingFreqTable`, `trapRingFreqTable`
- ring trip tables: `sineRingtripTable`, `trapRingtripTable`

### audio
- `ProSLIC_TXAudioGainSetup()` — transmit gain
- `ProSLIC_RXAudioGainSetup()` — receive gain
- `ProSLIC_DCFeedSetup()` — DC feed configuration
- `ProSLIC_GetDCFeePreset()` — query DC feed settings

### chip-specific (Si3217x)
- `Si3217x_SetLinefeed()` (220 B)
- `Si3217x_ReadHook()` (60 B)
- `Si3217x_RingStart()` (100 B)
- `Si3217x_DCFeedSetup()` (84 B)
- `Si3217x_dbgSetGain()` (844 B) — debug gain config

### chip-specific (Si3226x)
- `Si3226x_DCFeedSetup()` (1020 B)
- `Si3226x_RingStart()` (64 B)
- `Si3226x_dbgSetGain()` (1100 B)
- `Si3226x_dbgSetRinging()` (692 B)
- `Si3226x_LineMonitor()` (704 B)

### SLIC registers accessed directly
- `LINEFEED` (0x1E) — line feed state
- `RINGCON` (0x3C) — ring control
- `RINGTALO`/`RINGTAHI` — ring trip thresholds
- `RINGTILO`/`RINGTIHI` — ring trip levels

## minimal SLIC init sequence (for open source wrapper)

```
1. open /dev/slic_bsp
2. ioctl(bsp_fd, HT_BSP_INIT, &result)  — init BSP, get SLIC count
3. for each port:
   a. open /dev/fxsN
   b. ioctl(fxs_fd, IFX_TAPI_CH_INIT, ...)  — init channel
   c. ioctl(fxs_fd, IFX_TAPI_LINE_FEED_SET, STANDBY)  — power standby
4. to go off-hook ready:
   ioctl(fxs_fd, IFX_TAPI_LINE_FEED_SET, ACTIVE)  — power active
5. to detect hook state:
   ioctl(fxs_fd, IFX_TAPI_LINE_HOOK_STATUS_GET, &status)
6. to ring:
   ioctl(fxs_fd, IFX_TAPI_RING_START, ...)
   ioctl(fxs_fd, IFX_TAPI_RING_STOP, ...)

note: the TAPI_EVENT_GET ioctl or poll() on the fd can be used for
async event notification (hook changes, DTMF, ring trip, etc.)
```

## important caveat about ioctl numbers

the ioctl numbers documented here are from grandstream's mutated VINETIC fork.
they do NOT match any publicly available version of the Infineon/Lantiq TAPI SDK.
the numbers were likely renumbered during the port from VINETIC to the DVF99
platform. the research in ioctl.md may also be incomplete — there could be
additional undocumented ioctls. the kernel modules have symbols that could help
identify any missing ones.
