# DUA (digital unit allocation) protocol

## overview

DUA is the CSS service responsible for audio routing and DSP pipeline configuration.
it manages "units" (audio endpoints like FXS ports and VoIP channels), "connections"
between them, and the DSP processing elements in the signal path.

libcordless.so on the APP side is the primary userspace client of this service.
it communicates via AF_COMA sockets connected to the "dua" service.

## DUA message format

messages sent to the DUA service have this layout:

```c
struct dua_msg {
    uint32_t sender_id;     // [0] caller/session id
    uint32_t unknown;       // [1] flags or version
    uint32_t cmd;           // [2] command type (see table below)
    uint32_t num_params;    // [3] parameter count
    uint32_t params[];      // [4+] command-specific integer params
};
// optionally followed by variable-length string/data references
```

total minimum size: 16 bytes (verified: `dua_process()` checks `len < 0x10`)

### variable-length data packing

some commands (0x56-0x59, 0x4a, 0x4b, etc.) pack string and data references
after the fixed header using `dua_unpack_string_ref()` and `dua_unpack_data_ref()`.
the `num_params` field ([3]) indicates how many fixed params are present, and the
remaining data follows as length-prefixed strings/blobs.

## command table

### numeric UID commands (low-level)

recovered by tracing the switch8 jump table in `dua_process()` at 0x022700a0
and the branch cascade that follows it.

| cmd | CSS handler | params (from msg[4+]) | notes |
|-----|-------------|----------------------|-------|
| 0x01 | `p_dua_InitReq` | params[4..7] | line/channel init (distinct from ApplInit) |
| 0x02 | `p_da_GoIdleReq` | mode=params[4] | put DSP/audio to idle |
| 0x04 | `p_dua_UnitConnectReq` | uid=(s16)params[4], connId=params[5] | connect unit to connection |
| 0x05 | `p_dua_UnitDisconnectReq` | uid=(s16)params[4], connId=params[5] | disconnect unit |
| 0x06 | `p_dua_ConnCreateReq` | (none) | create a new connection, returns connId |
| 0x07 | `p_dua_ConnDeleteReq` | connId=params[4] | delete connection |
| 0x08 | `p_dua_ConnMergeReq` | connId1=params[4], connId2=params[5] | merge (conference) |
| 0x09 | `p_dua_ConnUnmergeReq` | connId=params[4] | unmerge connection |
| 0x0a | `p_dua_UnitAllocateReq` | unitType=(s16)params[4], unitSpec=params[5] | allocate unit by numeric type (0-2) |
| 0x0c | `p_dua_UnitFreeReq` | uid=(s16)params[4] | free a previously allocated unit |
| 0x0d | `p_dua_UnitSetReq` | uid=(s16)params[4], elem=params[5], param=params[6], data... | set unit parameter |
| 0x0e | `p_dua_UnitGetReq` | uid=(s16)params[4], elem=params[5], param=params[6], data... | get unit parameter |
| 0x0f | `p_dua_DialDTMFReq` | uid=params[4], elem=params[5], flags=params[6], data... | send DTMF tones |
| 0x10 | `p_dua_StopDTMF` | uid=params[4], elem=params[5] | stop DTMF generation |
| 0x11 | `p_dua_PlayMelodyReq` | uid=params[4], elem=params[5], nMelos=params[6], ... | play melody/tone pattern |
| 0x12 | `p_dua_StopMelody` | uid=params[4], elem=params[5] | stop melody playback |
| 0x13 | `p_dua_ApplInit` | (none) | application initialization — call first! |

note: cmd 0x03 is a noop (jumps directly to response/exit). cmds 0x00 and 0x0b appear unused.

### string-based commands (high-level, probably what libcordless uses)

| cmd | CSS handler | prototype | notes |
|-----|-------------|-----------|-------|
| 0x4a | `p_duasc_Number` | (name_str) | lookup numeric ID by name |
| 0x4b | `p_duasc_Name` | (uid, ...) | lookup name by numeric ID |
| 0x55 | `p_duasc_ConnUnmergeReq` | (connId) | unmerge a connection |
| 0x56 | `p_duasc_UnitAllocateReq` | (type_str, flags) | allocate unit by type name string |
| 0x57 | `p_duasc_UnitFreeReq` | (type_str, flags) | free unit by type name string |
| 0x58 | `p_duasc_UnitSetReq` | (type_str, id, elem_str, param_str, data, dataSz) | set param by name |
| 0x59 | `p_duasc_UnitGetReq` | (type_str, id, elem_str, param_str, maxSz) | get param by name |
| 0x7b | `p_duasc_EnumUT` | (**pszName, *pNum, numUT) | enumerate unit types |
| 0x7c | `p_duasc_EnumUE` | (**pszName, *pNum, uid, elemIdx) | enumerate unit elements |
| 0x7d | `p_duasc_EnumUEParam` | (**pszName, *pNum, uid, elem, paramIdx) | enumerate params |
| 0x88 | `text_trace_configure` | (...) | debug tracing configuration |

note: the string-based commands for ConnCreate, ConnDelete, ConnMerge, UnitConnect,
UnitDisconnect exist as `p_duasc_*` wrappers but their command IDs in the 0x4c-0x54
range couldn't be individually resolved (second unrecovered switch table). they
resolve the string unit type name via `p_duasc_UID()` and then call the corresponding
`p_dua_*` function. for practical purposes, use the numeric commands (0x04-0x09)
directly since we know the UID encoding scheme.

### DUASM state machine enum

the internal state machine IDs used by `p_dua1_Trigger()`, recovered from
disassembly of each `p_dua_*Req` function:

| DUASM | value | handler function |
|-------|-------|-----------------|
| DUASM_CONNCREATE | 0 | p_dua1sm_ConnCreate |
| DUASM_CONNDELETE | 1 | p_dua1sm_ConnDelete |
| DUASM_UNITCONNECT | 2 | p_dua1sm_UnitConnect |
| DUASM_UNITDISCONNECT | 3 | p_dua1sm_UnitDisconnect |
| DUASM_CONNMERGE | 4 | p_dua1sm_ConnMerge |
| DUASM_CONNUNMERGE | 5 | p_dua1sm_ConnUnmerge |
| DUASM_UNITALLOCATE | 6 | p_dua1sm_UnitAllocate |
| DUASM_UNITFREE | 7 | p_dua1sm_UnitFree |
| DUASM_UNITSET | 8 | p_dua1sm_UnitSet |
| DUASM_UNITGET | 9 | p_dua1sm_UnitGet |

## response format

responses are sent back via `dua_send_message()`, which calls
`coma_create_message(2, 2, NULL, 0, buf, len)` — service_id=2, msg_type=2.

the response is prepared by `dua_prep_sendmsg()` which builds a response buffer
with the original sender_id, a result code, and optional string data.

async callbacks go through `dua_send_cbk()` / `dua_send_unit_cbk()` for events
like connection state changes, DTMF detection, etc.

## special parameter: DUA_PARAM_CBK_FUNC

UnitSetReq with param name "DUA_PARAM_CBK_FUNC" registers or deregisters a
callback for async events on a unit. setting data to NULL deregisters.
this is how the APP side receives notifications (hook state, DTMF, etc.).
