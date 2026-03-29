# UMT (unit mode table) analysis

## what UMTs are

UMTs are bytecode programs that configure the DSP processing pipeline for a unit.
when you call `UnitSetReq` with `DUA_PARAM_UMT_EXEC_GEN` and a mode number,
the CSS executes the corresponding UMT bytecode to set up the signal processing chain.

the UMT bytecode format is:
```
u16 opword: bits[4:0] = opcode (0-27), bits[15:5] = element index (11 bits)
followed by: variable-length params (u8, u16, or u32 depending on opcode+param position)
terminated by: 0x0000 (END) or 0xFFFF (STOP)
```

after the bytecode terminator, additional configuration data follows that the
bytecode operations reference. this data varies per mode and contains things
like frame sizes and DSP routing parameters.

## opcode groups

the dispatch handler `p_ugt1_DoUmop` at 0x020290bd groups opcodes:

| opcodes | handler | likely purpose |
|---------|---------|---------------|
| 0-8 | 0x02029173 | DSP element install/configure operations |
| 9-18 | 0x02029193 | DSP element parameter set operations |
| 19 | 0x0202923d | special operation |
| 20-22 | 0x02029255 | another group |
| 23 | 0x020292c5 | individual |
| 24 | 0x020292ff | individual |
| 25 | 0x0202931d | individual |
| 26 | 0x02029343 | individual |
| 27 | 0x0202933f | individual |

## SPVOIPNDA generic modes (8 modes)

mode table pointer: `G_ugt_UnitType[0]` field [20] → 0x02035938

### mode 0 — "full pipeline init"

bytecode (19 bytes, 11 operations):
```
op1(elem=ALL)           — likely "reset all elements"
op31(elem=1799, 0x00ff) — configure element 1799
op27(elem=0, 0x1500)    — some global setting
op0(elem=2032)          — install/enable element
op31(elem=183, 0xfe00)  — configure
op31(elem=255, 0xe100)  — configure
op31(elem=55, 0xffe2)   — configure
op16(elem=2, 0)         — set param
op0(elem=856)           — install/enable element
op4(elem=32)            — install element
op16(elem=1776, 255)    — set param
STOP
```

this is the longest and most complex mode — likely a full cold-start initialization
of the VoIP DSP pipeline including codec, echo canceller, and signal processing.

### modes 1, 2, 3 — "narrowband codec variants"

identical bytecode (6 bytes):
```
op1(elem=ALL)           — reset all
op6(elem=1808, 0x50ff)  — install codec element 1808 with param 0x50ff
END
```

these three modes have the same bytecode but different post-bytecode config data.
the key difference is a frame size value:
- mode 1: 0x00a0 = 160 samples (20ms at 8kHz)
- mode 2: 0x0140 = 320 samples (40ms at 8kHz, or 20ms at 16kHz)
- mode 3: 0x01e0 = 480 samples (60ms at 8kHz, or 30ms at 16kHz)

**these are likely the standard narrowband VoIP modes with different ptime values.**

### modes 4, 5, 6 — "wideband codec variants"

identical bytecode (15 bytes):
```
op1(elem=ALL)           — reset all
op6(elem=1808, 0x00ff)  — install codec (different param from modes 1-3!)
op3(elem=0, 0x6b00)     — routing/connection setup
op4(elem=32)            — install element
op16(elem=2032, 255)    — set param
STOP
```

different post-bytecode data, key differences:
- mode 4: 0x0140 = 320 samples
- mode 5: 0x0280 = 640 samples
- mode 6: 0x0280 = 640 samples (same as 5)

the `op6` param is 0x00ff vs 0x50ff for narrowband — the 0x50 likely sets a
narrowband flag. **these are likely wideband (16kHz) modes:**
- mode 4: 20ms wideband
- mode 5: 40ms wideband
- mode 6: 40ms wideband (possibly different codec)

### mode 7 — "minimal/passthrough"

bytecode (12 bytes):
```
op1(elem=ALL)           — reset all
op1(elem=1808)          — reset codec element specifically
op31(elem=15, 0x0000)   — configure element 15
op0(elem=864)           — install element 864
op4(elem=32)            — install element
op16(elem=8, 0)         — set param to 0
END
```

this mode is notably different from all others — it doesn't use `op6` to install
a codec. it resets the codec element, installs element 864 (different from normal),
and sets a param to 0. **this might be a raw/passthrough mode or a special-purpose
configuration like a tone-only mode.**

## FXS generic modes (3 real modes + 2 garbage)

mode table pointer: `G_ugt_UnitType[2]` field [20] → 0x02035998

### mode 0 — "FXS init"

```
op1(elem=ALL)               — reset all
op31(elem=855, 0x0604)      — configure FXS-specific element
op16(elem=120, 0)           — set param
END
```

basic FXS port initialization.

### mode 1 — "FXS config A"

```
op26(elem=0, 0x0044)       — specialized FXS operation
END
```

minimal reconfiguration.

### mode 2 — "FXS config B"

```
op26(elem=0, 0x0044)       — same as mode 1
END
```

identical to mode 1 (might differ in pre-bytecode config data).

### modes 3-4 — INVALID

the pointers at 0x022a1624 and 0x022a1634 point into DDR_RAM3 string data,
not valid UMT bytecode. these are likely beyond the actual mode count.
the real FXS nGenModes is probably 3, not 5.

## key element indices observed

| elem | appears in | likely identity |
|------|-----------|----------------|
| ALL (0xFFFFF...) | all modes | wildcard / "all elements" |
| 1808 | SPVOIP modes 1-7 | main codec element (install point) |
| 1799 | SPVOIP mode 0 | signal processing config |
| 2032 | SPVOIP modes 0, 4-6 | DSP output routing |
| 856 | SPVOIP mode 0 | signal processing block |
| 864 | SPVOIP mode 7 | alternative processing block |
| 855 | FXS mode 0 | FXS-specific signal element |
| 120 | FXS mode 0 | FXS config element |
| 32 | SPVOIP modes 0, 4-7 | common element (maybe probe/monitor) |
| 15 | SPVOIP mode 7 | config element |

## practical implications

for a minimal audio setup, the call sequence would be:

1. allocate UT_SPVOIPNDA → get uid (e.g. 0x0000)
2. allocate UT_FXS → get uid (e.g. 0x0200)
3. **UnitSetReq(spvoip_uid, -1, DUA_PARAM_UMT_EXEC_GEN, &mode, 4)**
   - mode=1 for 8kHz 20ms narrowband (most basic, G.711-compatible)
   - mode=4 for 16kHz 20ms wideband
   - mode=7 for possible raw/passthrough
4. **UnitSetReq(fxs_uid, -1, DUA_PARAM_UMT_EXEC_GEN, &mode, 4)**
   - mode=0 for FXS init
5. ConnCreate → get connId
6. UnitConnect(fxs_uid, connId)
7. UnitConnect(spvoip_uid, connId)
8. set up voice session via /dev/voiceXX or voice COMA service

**mode 1 is probably the safest starting point** — it's a standard narrowband
20ms frame config that should work with G.711 μ-law/a-law.

**mode 7 is the most interesting for raw audio** — if it's truly a passthrough
mode, it might give uncompressed PCM directly.
