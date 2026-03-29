# open source libcordless replacement — feasibility notes

## goal

bypass the native SIP stack entirely and get PCM audio frames in/out of the
SLIC devices from linux userspace, without needing gs_ata or app_dsp.

## what's available from open source

the linux kernel fork (GPL release) provides:
- full AF_COMA socket implementation (`coma-socket.c`)
- struct cmsg wire format (`include/uapi/linux/coma/coma.h`)
- voice service with /dev/voiceXX devices and RTP session management (`coma-voice.c`)
- TDM service grant/revoke (`coma-tdm.c`)
- TAPI/FXS device interface /dev/fxsXX (hook state, ringing, line feed)
- all cmsg type enums and param structs for voice, datahandler, dvf9a, sharedmem

## what we've reverse engineered from CSS firmware

- **complete DUA command table** (all command IDs 0x01-0x88 mapped!)
  - including the previously-missing connection commands:
    0x04=UnitConnect, 0x05=UnitDisconnect, 0x06=ConnCreate, 0x07=ConnDelete,
    0x08=ConnMerge, 0x09=ConnUnmerge
- DUA message format (16-byte header + params + variable-length data)
- unit type names: UT_SPVOIPNDA, UT_TRACE, UT_FXS
- UID encoding scheme: (type << 8) | instance
- 8 FXS instances (0x0200-0x0207) = 8 phone ports
- 16 SPVOIPNDA instances (0x0000-0x000F) = VoIP DSP channels
- 25 unit element names (codecs, filters, routing blocks)
- pin instance layout (2 pins per unit, elem 35 and 40)
- connection model (create, connect units, merge)
- audio routing table format
- **complete DUA_PARAM code table** (43 params, all with numeric values)
  - key params: UMT_EXEC_GEN (0x10100) for DSP pipeline config,
    USM_DO (0x100ff) for state machine actions, PIN_MUTE/VOLUME for audio control,
    CBK_FUNC (0x1010a) for async event callbacks
- **complete DUAEV event code table** (129 events) including DTMF detection,
  ring detection, hook state, fax/modem detection, etc.
- **complete DUA error code table** (all negative return values mapped)
- DUASM state machine enum (10 operations, values 0-9)

## proposed minimal call sequence

```
== initialization ==
1. socket(PF_COMA, SOCK_SEQPACKET, 0)
2. connect(sock, {AF_COMA, "dua"})
3. send DUA cmd 0x13: ApplInit

== allocate units ==
4. send DUA cmd 0x56: UnitAllocateReq("UT_FXS", port_index)
   → receive FXS UID (e.g., 0x0200 for port 0)
5. send DUA cmd 0x56: UnitAllocateReq("UT_SPVOIPNDA", -1)
   → receive SPVOIPNDA UID (e.g., 0x0000)

== configure DSP pipeline ==
6. send DUA cmd 0x58: UnitSetReq on SPVOIPNDA unit
   - set codec (G.711 = simplest, raw PCM passthrough if possible)
   - set sample rate
   - set frame size
   - register callback (DUA_PARAM_CBK_FUNC)

== create audio path ==
7. send DUA cmd (ConnCreate?) to create a connection
8. send DUA cmd (UnitConnect?) to connect FXS unit to connection
9. send DUA cmd (UnitConnect?) to connect SPVOIPNDA unit to connection

== start voice session (via voice service or /dev/voiceXX) ==
10. open /dev/voiceXX or connect AF_COMA socket to "voice"
11. voice: GET_SESSION → SET_SESSION_FIFOS → START_SESSION

== steady state ==
12. read/write PCM frames from /dev/voiceXX

== teardown ==
13. voice: STOP_SESSION → FREE_SESSION
14. DUA: UnitDisconnect, ConnDelete, UnitFree
```

## remaining unknowns (need further RE or hardware testing)

### critical
- [x] ~~exact command IDs for ConnCreate, UnitConnect, UnitDisconnect, ConnDelete~~ DONE
      ConnCreate=0x06, ConnDelete=0x07, UnitConnect=0x04, UnitDisconnect=0x05,
      ConnMerge=0x08, ConnUnmerge=0x09
- [x] ~~UnitSetReq parameter IDs~~ DONE — full DUA_PARAM table extracted (43 params)
- [ ] what UMT mode values are available for UT_SPVOIPNDA and UT_FXS
      (UMT_EXEC_GEN with mode=N selects a predefined DSP pipeline — need to
      enumerate what mode values exist and what DSP chain each configures)
- [ ] whether the voice service requires DUA to be fully set up first, or
      if we can use TDM grant + voice session directly
- [ ] the exact response/callback message format for parsing DUA replies
- [ ] what p_dua_InitReq (cmd 0x01) does vs p_dua_ApplInit (cmd 0x13)
      and whether both are needed

### important but can discover at runtime
- [ ] available unit elements per type (can use EnumUE cmd 0x7c)
- [ ] available params per unit element (can use EnumUEParam cmd 0x7d)
- [ ] whether G.711 passthrough gives us raw 16-bit PCM or if we need to
      ulaw/alaw decode on the linux side

### nice to have
- [x] ~~DTMF detection callback format~~ — DUAEV_DFC_DTMF_IND (0x1031)
- [x] ~~conference/merge support~~ — ConnMerge cmd 0x08, ConnUnmerge cmd 0x09
- [ ] hook state notification mechanism (probably via /dev/fxsXX TAPI)
- [ ] detailed UMT (unit mode table) format for custom DSP pipeline definition

## alternative approach: TDM-only bypass

if we can figure out the TDM register base addresses from the APP side,
we might be able to memory-map them and read/write samples directly,
skipping CSS entirely for audio data. however:
- we'd still need CSS for SLIC control (ringing, hook detect, line feed)
- the TDM bus must be granted/enabled (maybe possible from linux via COMA TDM service)
- we lose all DSP processing (echo cancel, codecs, AGC, etc.)
- this might conflict with CSS firmware if it's also touching the TDM bus

the DUA/voice path is probably the right approach since it lets CSS handle
the real-time audio processing while we just consume frames from linux.

## implementation approach

a minimal C library that:
1. opens AF_COMA sockets to "dua" and optionally "voice"
2. implements the DUA message packing/unpacking
3. provides a simple API:
   - `dua_init()` — ApplInit
   - `dua_alloc_fxs(port)` → uid
   - `dua_alloc_voip()` → uid
   - `dua_connect(fxs_uid, voip_uid)` → connection
   - `dua_start_audio(connection, codec, rate)` → fd for PCM read/write
   - `dua_stop_audio(connection)`
   - `dua_free(uid)`

this could live in the gstools repo alongside the existing tooling.
