# DUA unit types, instances, and elements

## unit type table (G_ugt_UnitType at 0x021002c4)

there are exactly 3 unit types. the string names are stored in `asc_ugt_UT` at 0x022c03ec.

| index | name | instances | UID range | purpose |
|-------|------|-----------|-----------|---------|
| 0 | UT_SPVOIPNDA | 16 | 0x0000 - 0x000F | VoIP speech processing / DSP pipeline |
| 1 | UT_TRACE | ~3 | 0x0100 - 0x0102 | debug/tracing |
| 2 | UT_FXS | 8 | 0x0200 - 0x0207 | FXS port (SLIC interface) |

## UID encoding

```
UID = (unitType << 8) | instanceIndex

examples:
  0x0000 = SPVOIPNDA instance 0
  0x0005 = SPVOIPNDA instance 5
  0x0200 = FXS instance 0 (port 1)
  0x0207 = FXS instance 7 (port 8)
```

decoding: `unitType = uid >> 8`, `instanceIndex = uid & 0xFF`
validated in `p_duax_UnitType()`: unitType must be < 3.

the 8 FXS instances directly correspond to the HT818's 8 phone ports.
the 16 SPVOIPNDA instances are the VoIP DSP channels (supports up to 16 simultaneous calls,
though the HT818 has 8 ports — extra instances likely for conference/transfer scenarios).

## pin instances (G_ugt_PinInst at 0x0203034c)

each unit instance has two pins (connection endpoints), with element IDs 35 and 40.

### SPVOIPNDA pins (uid 0x0000-0x000F)

| uid | elem | pin type (v3) | DSP instance (v4) |
|-----|------|---------------|-------------------|
| 0x0000 | 40 | 0x100 | 6 |
| 0x0000 | 35 | 0x102 | 4 |
| 0x0001 | 40 | 0x100 | 10 |
| 0x0001 | 35 | 0x102 | 8 |
| ... | ... | ... | increments by 4 per instance |
| 0x000F | 40 | 0x100 | 66 |
| 0x000F | 35 | 0x102 | 64 |

pattern: DSP instance for elem 40 = 6 + (uid * 4), elem 35 = 4 + (uid * 4)

### FXS pins (uid 0x0200-0x0207)

| uid | elem | pin type (v3) | DSP instance (v4) |
|-----|------|---------------|-------------------|
| 0x0200 | 40 | 0x101 | 70 |
| 0x0200 | 35 | 0x103 | 68 |
| 0x0201 | 40 | 0x101 | 76 |
| 0x0201 | 35 | 0x103 | 74 |
| ... | ... | ... | increments by 6 per instance |
| 0x0207 | 40 | 0x101 | 112 |
| 0x0207 | 35 | 0x103 | 110 |

pattern: DSP instance for elem 40 = 70 + (fxs_idx * 6), elem 35 = 68 + (fxs_idx * 6)

### pin type interpretation

- 0x100 / 0x102: SPVOIPNDA pin types (likely ingress / egress)
- 0x101 / 0x103: FXS pin types (likely ingress / egress)
- elem 40 and elem 35 are the two directions of audio flow per unit

## unit elements (UEs) — DSP processing blocks

from `asc_ugt_UE` at 0x022c0324 (200 bytes, 25 entries of 8 bytes each):

### codec elements
| name | likely codec |
|------|-------------|
| UE_G1E | G.711 (a-law / μ-law) |
| UE_G2E | G.726 |
| UE_G3E | G.723.1 |
| UE_G6E | G.726 variant |
| UE_G9E | G.729A |
| UE_AMRWD | AMR wideband |
| UE_ILBCE | iLBC |
| UE_OPUSE | Opus |

### signal processing elements
| name | likely purpose |
|------|---------------|
| UE_DLY | delay / buffer |
| UE_HPF_B | high-pass filter (DC removal) |
| UE_LVLRX | RX level control / AGC |
| UE_SLE | SLIC echo (line echo canceller?) |
| UE_SKOSFVOIP_10 | VoIP speech kernel, 10ms frames |
| UE_SKOSFVOIP_30 | VoIP speech kernel, 30ms frames |
| UE_SKP_B | speech kernel processing |

### routing / switching elements
| name | likely purpose |
|------|---------------|
| UE_SSR | signal switch read |
| UE_SSW | signal switch write |
| UE_SSW_TDM | signal switch write — TDM side |
| UE_SU2 | subscriber unit (connects to TDM?) |

### other elements
| name | likely purpose |
|------|---------------|
| UE_CIT | caller ID / tone transmission |
| UE_OPI | output processing interface |
| UE_PRB | probe / monitoring point |
| UE_PRB_C | probe (capture variant?) |
| UE_TPE | tone / pattern engine (ring tones, busy tones, etc.) |

## connection model

connections link units together. from the code:

- max 16 connections (loop limit 0x10 in `p_duax_CreateConn`)
- each connection is 12 bytes (0x0c)
- connection `kind`: 1 = normal, 2 = dynamic/internal
- connections track source/dest unit handles

### connection workflow (from state machine functions)

1. `ConnCreate` — allocate a connection slot
2. `UnitConnect` — attach a unit (by UID) to a connection
   - looks up unit handle via `p_duax_GetUnitHdl(uid)`
   - if unit already connected, may clone it (`p_duax_UnitCloneable` / clone logic)
   - creates the connection if needed (`p_duax_CreateConn(2)`)
3. `ConnMerge` — merge two connections (for conferencing)
4. `UnitDisconnect` — detach a unit
5. `ConnDelete` — free the connection slot

### audio routing tables (ART)

the DUA maintains audio routing tables that map signal block (SB) pairs:
- `p_da_SetAudioRouter(pst_ART, numEntries)` — writes sorted routing entries
- each entry has: u8_CallNr, u8_DestSB, u8_SrcSB
- entries are sorted by a composite key for binary search at runtime
- `p_duax_TmpArtPathApp` builds temporary ART paths during connection setup

## related global data objects

| symbol | address | size | purpose |
|--------|---------|------|---------|
| G_dua1_UnitInst | 0x022e1628 | 1120 B | unit instance array (35 × 32 bytes) |
| G_dua1_UnitConn | 0x022e1c88 | 192 B | connection array (16 × 12 bytes) |
| G_dua1_PinInst | 0x022e1170 | 1152 B | pin instance array |
| G_dua1_PinConn | 0x022e1a88 | 512 B | pin connection state |
| G_dua1_UCH | 0x022e1d48 | 176 B | unit connection handles |
| G_dua1_SM | 0x022d5630 | 40 B | state machine state |
| G_dua1_UnitHdl | 0x022e15f0 | 54 B | unit handle lookup |
| G_dua1_UnitIdx | 0x022d5608 | 6 B | unit type index offsets |
| G_ugt_UnitType | 0x021002c4 | 96 B | unit type definitions (3 × 32 bytes) |
| G_ugt_PinInst | 0x0203034c | 576 B | pin instance definitions |
| G_ugt_PinType | 0x0203031c | 48 B | pin type definitions |
| G_ugt_DspElem | 0x02030640 | 5624 B | DSP element definitions |
| dua_adpcm_usage | 0x022d5718 | 6 B | ADPCM resource tracking |
| dua_apu_usage | 0x022d571e | 6 B | APU resource tracking |
| dua_drt_usage | 0x022d5724 | 6 B | DRT resource tracking |
