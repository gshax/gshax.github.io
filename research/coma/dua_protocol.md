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

## response format (confirmed on hardware 2026-03-29)

responses use the same `struct dua_msg` layout:

```c
struct dua_response {
    uint32_t sender_id;   // echoed from request (used for correlation!)
    uint32_t flags;       // always 0
    uint32_t cmd;         // 0x81 = sync response, 0x7f = async callback
    uint32_t num_params;  // 1 = error, 2+ = success
    uint32_t params[];
    // followed by null-terminated string (name for enum, empty otherwise)
};
```

### sync responses (cmd=0x81)
- `sender_id` matches the request — used to correlate responses to requests
- `num_params=1, params[0] < 0` means DUA error (see `enum dua_error`)
- `num_params >= 1, params[0] >= 0` means success
- for EnumUT/EnumUE: params contain index + id, followed by name string

### async callbacks (cmd=0x7f)
- `sender_id=0`
- `params[0]` = callback reference (e.g. 0xDEADBEEF from InitReq)
- these arrive unsolicited and must be drained to get to sync responses
- a burst of async callbacks follows InitReq (one per FXS/VOIP port)

## initialization sequence (confirmed on hardware)

**CRITICAL: the following steps must be performed in order or the CSS will panic.**

### phase 1: DUA transport setup
1. open `/dev/sharedmem` and call `ioctl(fd, 0xc0045302, params)`
   - this sends `CMSG_SHAREDMEM_INIT` to CSS, which sets up its MMU mapping
   - returns physical address and size of shared memory region
2. `mmap()` the shared memory, zero it, set header:
   - `*(shm+4) = size`
   - `*(shm+8) = 0x30` (offset)
3. connect AF_COMA socket to "dua" (this registers the DUA service on the CSS)
4. send `DUA_CMD_INIT_REQ` (cmd 0x01) with 5 params:
   - params = {5, 0xffffffff, 0xffffffff, 0xdeadbeef, shm_ptr}
   - the CSS uses the shared memory pointer for DUA data structures
5. send `DUA_CMD_APPL_INIT` (cmd 0x13)
6. DUA is now ready for unit operations

### phase 2: unit setup (stock lib_dua_init at 0x0001e538 in libcordless.so)

per FXS unit (×8, in order):
1. `UnitAllocateReq(type=2, spec=i)`
2. `UnitSetReq(uid, elem=-2, 0x10100=UMT_EXEC_GEN, mode=1, 0)`
   **← CRITICAL: creates DSP pipeline + FIFOs needed for TDM**
   mode 1 bytecode at 0x0203581F (_css.elf genModes[1] for FXS type)
3. `UnitSetReq(uid, elem=0x13, 0x100FF=USM_DO, dtmf_config, 12)`
4. `UnitSetReq(uid, elem=0x3b, 0x100FF=USM_DO, 1, 0)`
5. `UnitConnectReq(uid, -3)`
6. `UnitSetReq(uid, elem=-1, 0x1010A=CBK_FUNC, callback_ptr, 0)`

per VOIP unit (×16):
1. `UnitAllocateReq(type=0, spec=i)`
2. `UnitConnectReq(voip_uid, fxs_conn_id)`

### phase 3: TDM setup (stock dua_set_fxs_tdm at 0x0001ef9c)
1. `UnitSetReq(fxs_uid, elem=-2, 0x10103=UMT_IMMEDIATE, tdm_blob, 268)`
2. write "1" to `/proc/gs/css_own_tdm0` → triggers COMA TDM grant

skipping steps 1-4 of phase 1 causes CSS panic because the DUA handler dereferences
uninitialized shared memory pointers. skipping phase 2 step 2 causes TDM assignment
to write to nonexistent FIFOs.

discovered by tracing `app_dsp` main() → `FUN_00018a18` (sharedmem init) →
`FUN_000155d0` (duasync_init) → `FUN_00015744` (DUA InitReq with 0xdeadbeef).
phase 2/3 decoded 2026-03-31 from ghidra analysis of libcordless.so + _css.elf.

## COMA socket quirk

the kernel's COMA socket returns `EOPNOTSUPP` (errno 95) on plain `recv()`
but works correctly with `recvmsg()`. this appears to be a bug in the socket
layer where the `MSG_OOB` flag check in `coma_sock_recvmsg` triggers incorrectly
for `recv()` but not `recvmsg()`. always use `recvmsg()`.

## special parameter: DUA_PARAM_CBK_FUNC

UnitSetReq with param name "DUA_PARAM_CBK_FUNC" registers or deregisters a
callback for async events on a unit. setting data to NULL deregisters.
this is how the APP side receives notifications (hook state, DTMF, etc.).
