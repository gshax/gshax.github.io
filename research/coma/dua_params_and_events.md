# DUA parameter codes and event IDs

extracted from the master name-number lookup table at 0x022ba59c (2993 entries total,
used by `p_GetNumber()` via bsearch). these are the symbolic names that the string-based
DUA API (cmd 0x58 UnitSetReq, cmd 0x59 UnitGetReq) uses to identify parameters.

## DUA_PARAM codes (for UnitSet/UnitGet)

### special / low-level params

| name | value | notes |
|------|-------|-------|
| DUA_PARAM_MIN_CODE | -5 | minimum valid param code |
| DUA_PARAM_SIMBMP | -4 | simulate bitmap |
| DUA_PARAM_SIMDSP | -3 | simulate DSP |
| DUA_PARAM_SETTOG | -2 | set tone generator (freq, duration, volume) |

### address-offset params (direct DSP register access)

| name | value | notes |
|------|-------|-------|
| DUA_PARAM_ADDR_OFFS_First | 0x0000 | first DSP address offset |
| DUA_PARAM_ADDR_OFFS_Iswi | 0x2000 | input switch address offset |
| DUA_PARAM_ADDR_OFFS_Dptr | 0x2001 | data pointer address offset |
| DUA_PARAM_ADDR_OFFS_Last | 0xFFFF | last DSP address offset |

any param code in range 0x0000-0xFFFF is treated as a direct DSP value
set/get via `p_da_SetDSPValue()`. this is how individual DSP registers
within a unit element's processing block are configured.

### unit-level params (0x10000+)

| name | value (hex) | purpose |
|------|------------|---------|
| DUA_PARAM_UNIT | 0x10000 | base for unit-level params |
| DUA_PARAM_PIN_CAPS | 0x10001 | get/set pin capabilities |
| DUA_PARAM_PIN_SET_WEIGHTS | 0x10002 | set pin routing weights |
| DUA_PARAM_PIN_RES_WEIGHTS | 0x10003 | reset pin weights |
| DUA_PARAM_PIN_SET_CONNATTR | 0x10004 | set connection attributes |
| DUA_PARAM_PIN_RES_CONNATTR | 0x10005 | reset connection attributes |
| **DUA_PARAM_PIN_MUTE** | **0x10006** | **mute a pin** |
| **DUA_PARAM_PIN_SOFTMUTE** | **0x10007** | **soft mute (gradual)** |
| **DUA_PARAM_PIN_VOLUME** | **0x10008** | **set pin volume level** |
| DUA_PARAM_SFSWITCH | 0x1007f | subflow switch |
| DUA_PARAM_ISWITCH | 0x10080 | input switch |
| DUA_PARAM_DSP_ARRAY_VALUE | 0x10081 | set DSP array element |
| DUA_PARAM_DSP_STRUCT_VALUE | 0x10082 | set DSP struct field |
| **DUA_PARAM_USM_DO** | **0x100ff** | **invoke unit state machine action** |
| **DUA_PARAM_UMT_EXEC_GEN** | **0x10100** | **execute generic UMT (unit mode table)** |
| DUA_PARAM_UMT_EXEC_STAT | 0x10101 | execute static UMT |
| DUA_PARAM_UMT_EXEC_DYN | 0x10102 | execute dynamic UMT |
| DUA_PARAM_UMT_IMMEDIATE | 0x10103 | immediate UMT execution |
| DUA_PARAM_UMT_LOAD_DYN | 0x10104 | load dynamic UMT data |
| DUA_PARAM_GET_FREE_UNITS | 0x10105 | query free unit count |
| DUA_PARAM_UNIT_GET_NUM | 0x10106 | get unit numeric ID |
| DUA_PARAM_GET_CONN_OF_UID | 0x10107 | get connection ID for a UID |
| DUA_PARAM_GET_UID_OF_CONN | 0x10108 | get UID for a connection ID |
| DUA_PARAM_DA_PARAMS | 0x10109 | DA (digital audio) parameters |
| **DUA_PARAM_CBK_FUNC** | **0x1010a** | **register/deregister async callback** |
| DUA_PARAM_CBK_PARAM | 0x1010b | callback parameter |
| DUA_PARAM_SETFUNC | 0x1010c | custom set function |
| DUA_PARAM_GETFUNC | 0x1010d | custom get function |
| DUA_PARAM_MEM8 | 0x10110 | read/write 8-bit memory |
| DUA_PARAM_MEM16 | 0x10111 | read/write 16-bit memory |
| DUA_PARAM_MEM32 | 0x10112 | read/write 32-bit memory |
| DUA_PARAM_PRIVATE | 0x18000 | start of private/custom params |
| DUA_PARAM_LAST_PREDEF | 0x1ffff | end of predefined param range |

### key params for audio setup (bold above)

for getting audio flowing, the most important params are likely:
- **DUA_PARAM_UMT_EXEC_GEN** (0x10100) — this is how you configure the DSP pipeline.
  UMTs (unit mode tables) define the complete DSP processing chain for a unit.
  the `mode` value selects which predefined pipeline to use.
- **DUA_PARAM_USM_DO** (0x100ff) — triggers unit state machine actions (like start/stop)
- **DUA_PARAM_PIN_MUTE/VOLUME** — basic audio control
- **DUA_PARAM_CBK_FUNC** (0x1010a) — register for async events

## DUAEV event codes (for callbacks/responses)

### core events (0-14)

| name | value | notes |
|------|-------|-------|
| DUAEV_OK | 0 | success |
| DUAEV_READY | 1 | system ready |
| DUAEV_INIT_IND | 2 | init complete |
| DUAEV_CONNCREATE_IND | 3 | connection created |
| DUAEV_CONNDELETE_IND | 4 | connection deleted |
| DUAEV_UNITCONNECT_IND | 5 | unit connected |
| DUAEV_UNITDISCONNECT_IND | 6 | unit disconnected |
| DUAEV_CONNMERGE_IND | 7 | connections merged |
| DUAEV_CONNUNMERGE_IND | 8 | connections unmerged |
| DUAEV_UNITALLOCATE_IND | 9 | unit allocated |
| DUAEV_UNITFREE_IND | 10 | unit freed |
| DUAEV_SET_IND | 11 | set complete |
| DUAEV_GET_IND | 12 | get complete |
| DUAEV_PIN_MUTED | 13 | pin muted |
| DUAEV_PIN_UNMUTED | 14 | pin unmuted |

### USM (unit state machine) events (0x1000-0x1fff)

| name | value | notes |
|------|-------|-------|
| DUAEV_CID_START | 0x1000 | caller ID start |
| DUAEV_CID_IND | 0x1001 | caller ID data received |
| DUAEV_CID_PROGRESS | 0x1002 | caller ID in progress |
| DUAEV_CID_TIMEOUT | 0x1003 | caller ID timeout |
| DUAEV_CIT_START | 0x1010 | caller ID transmit start |
| DUAEV_CIT_IND | 0x1011 | CIT indication |
| DUAEV_CIT_ERROR | 0x1012 | CIT error |
| DUAEV_CIT_TIMER | 0x1013 | CIT timer event |
| DUAEV_CPD_START | 0x1020 | call progress detection start |
| DUAEV_CPD_CONT_IND | 0x1021 | CPD continuous tone detected |
| DUAEV_CPD_BUSY_IND | 0x1022 | CPD busy tone detected |
| DUAEV_CPD_TIMEOUT | 0x1023 | CPD timeout |
| DUAEV_CPD_TIMER | 0x1024 | CPD timer event |
| DUAEV_DFC_START | 0x1030 | DTMF/frequency collection start |
| DUAEV_DFC_DTMF_IND | 0x1031 | **DTMF digit detected** |
| DUAEV_DFC_DTMF_TIMEOUT | 0x1032 | DTMF detection timeout |
| DUAEV_DFC_CAS_IND | 0x1033 | CAS (channel associated signaling) |
| DUAEV_DFC_CAS_TIMEOUT | 0x1034 | CAS timeout |
| DUAEV_GTD_START | 0x1040 | general tone detection start |
| DUAEV_GTD_FAX_IND | 0x1041 | **fax tone detected** |
| DUAEV_GTD_FAX_TIMEOUT | 0x1042 | fax detection timeout |
| DUAEV_GTD_MODEM_IND | 0x1043 | **modem tone detected** |
| DUAEV_GTD_MODEM_TIMEOUT | 0x1044 | modem detection timeout |
| DUAEV_RCD_START | 0x1050 | ring cadence detection start |
| DUAEV_RCD_DTMF_IND | 0x1051 | ring cadence DTMF |
| DUAEV_RCD_DTMF_TIMEOUT | 0x1052 | ring cadence timeout |
| DUAEV_RGD_START | 0x1060 | ring detection start |
| DUAEV_RGD_RING_START | 0x1061 | **ring started** |
| DUAEV_RGD_RING_END | 0x1062 | **ring ended** |
| DUAEV_RGD_RING_RPAS | 0x1063 | ring RPAS signal |
| DUAEV_RGD_LINE_REVERSAL | 0x1064 | **line polarity reversal** |
| DUAEV_SLC_START | 0x1070 | SLIC control start |
| DUAEV_SLC_IND | 0x1071 | SLIC indication |
| DUAEV_SLC_TIMEOUT | 0x1072 | SLIC timeout |
| DUAEV_SLC_TIMER | 0x1073 | SLIC timer |
| DUAEV_USM_LAST | 0x1fff | end of USM events |

### param events (0x2000-0x2fff)

| name | value | notes |
|------|-------|-------|
| DUAEV_PARAM_BASE | 0x2000 | param event base |
| DUAEV_PARAM_VOIP_RECV_PACKET | 0x2001 | VoIP packet received |
| DUAEV_PARAM_VOIP_RTCP_REPORT | 0x2002 | RTCP report |
| DUAEV_PARAM_VOIP_DTMF | 0x2003 | VoIP in-band DTMF |
| DUAEV_MEM_DUMP | 0x2004 | memory dump event |
| DUAEV_PARAM_LAST | 0x2fff | end of param events |

### warning events (0x3000-0x3fff)

| name | value | notes |
|------|-------|-------|
| DUAEV_WARNING_BASE | 0x3000 | |
| DUAEV_CW_OVERFLOW_WARN | 0x3001 | connection weight overflow |
| DUAEV_UNIT_NOT_CONNECTED | 0x3002 | unit not connected warning |
| DUAEV_SET_PIN_CAPS_SKIPPED | 0x3003 | pin caps set skipped |
| DUAEV_SOFTMUTE_TIMEOUT | 0x3004 | soft mute timeout |
| DUAEV_MAX_ARCID_OFL | 0x3005 | ARC ID overflow |
| DUAEV_MAX_ART_or_PINART_OFL | 0x3006 | ART overflow |
| DUAEV_UMOP_WAITING | 0x3007 | UMT operation waiting |

### trace events (0x4000+)

| name | value | notes |
|------|-------|-------|
| DUAEV_TRACE_DA | 0x4000 | DA subsystem trace |
| DUAEV_TRACE_DSP | 0x4001 | DSP trace |

## DUA error codes (negative return values)

| name | value | notes |
|------|-------|-------|
| DUA_ERR_NONE | -3 | sentinel / no error state |
| DUA_ERR_UNDEF | -4 | undefined error |
| DUA_INIT_FAIL | -5 | initialization failed |
| DUA_NO_FREE_CONNECTION | -6 | all 16 connection slots full |
| DUA_UNIT_CONN | -7 | unit already connected |
| DUA_UNIT_FREE | -8 | unit already free |
| DUA_INVALID_UNIT | -9 | bad UID |
| DUA_CONNECT_ERROR | -10 | connection failed |
| DUA_DISCONN_ERROR | -11 | disconnection failed |
| DUA_RANGE_ERROR | -12 | value out of range |
| DUA_UNITS_OVERFLOW | -13 | max units exceeded |
| DUA_NO_FREE_UNIT | -15 | no free instances of requested type |
| DUA_PARAMVALUE_RANGE | -16 | param value out of range |
| DUA_PARAMTYPE_WRONG | -17 | wrong param type |
| DUA_MEMORY_LOW | -18 | memory allocation failed |
| DUA_BAD_DYN_MODE_DEF | -19 | bad dynamic mode definition |
| DUA_INVALID_ELEM | -20 | invalid element ID |
| DUA_BUFFERSIZE_ERROR | -21 | buffer too small |
| DUA_INVALID_UNITTYPE | -22 | unit type not 0-2 |
| DUA_INVALID_PIN | -23 | invalid pin |
| DUA_INVALID_DSPINST | -24 | invalid DSP instance |
| DUA_NO_DSPINST_FOR_UNITELEM | -25 | no DSP instance for element |
| DUA_INVALID_GEN_MODE | -26 | invalid generic mode |
| DUA_GETFUNC_REQUIRED | -27 | custom get function required |
| DUA_SETFUNC_REQUIRED | -28 | custom set function required |
| DUA_REQ_PENDING | -29 | request already pending |
| DUA_NOT_CAPABLE | -30 | unit not capable |
| DUA_INVALID_CONNECTION | -31 | bad connection ID |
| DUA_FATAL | -32 | fatal error |
| DUA_UNIT_NOT_CLONEABLE | -35 | unit can't be cloned |
| DUA_MISSING_PARAM | -37 | required param missing |
| DUA_INVALID_PARAMCODE | -36 | unknown param code |
