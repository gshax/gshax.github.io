# key CSS firmware function reference

quick reference for important functions in _css.elf, organized by subsystem.

## COMA (message bus)

| function | address | purpose |
|----------|---------|---------|
| coma_request_register | 0x0201de29 | register a new service on CSS |
| coma_request_deregister | 0x0201ddad | deregister a service |
| coma_cmsg_alloc | 0x0201db89 | allocate space for outgoing message |
| coma_cmsg_commit | 0x0201dbdd | commit message to CFIFO |
| coma_cmsg_send | 0x0201dc09 | alloc + commit combined |
| coma_process_message | 0x0201dd65 | dispatch incoming COMA core messages |
| coma_receive | 0x022776f9 | receive message from CFIFO |
| coma_create_message | 0x02277611 | create and send a message |
| coma_signal | 0x0227772d | trigger SEC interrupt to wake other processor |

## DUA (audio routing)

### message handling
| function | address | purpose |
|----------|---------|---------|
| dua_process_message | 0x02270739 | entry point — unwraps cmsg, calls dua_process |
| dua_process | 0x0227004f | main command dispatcher (408 lines, big switch) |
| dua_send_message | 0x02281911 | send response back via COMA |
| dua_send_cbk | 0x02281805 | send async callback to APP |
| dua_send_unit_cbk | 0x02281941 | send unit-specific callback |

### state machine
| function | address | size | purpose |
|----------|---------|------|---------|
| p_dua1_Trigger | 0x02025069 | — | queue a state machine request |
| p_dua1_ReqStart | 0x02025045 | — | start processing queued request |
| p_dua1sm_UnitAllocate | 0x02025605 | 478 B | allocate a unit instance |
| p_dua1sm_UnitFree | 0x02025d81 | 264 B | free a unit instance |
| p_dua1sm_UnitConnect | 0x02025809 | 678 B | connect unit to connection |
| p_dua1sm_UnitDisconnect | 0x02025ac1 | 688 B | disconnect unit |
| p_dua1sm_UnitGet | 0x02025ea1 | 164 B | get unit parameter |
| p_dua1sm_UnitSet | 0x02025f55 | 298 B | set unit parameter |
| p_dua1sm_ConnCreate | 0x0202512d | 32 B | create connection |
| p_dua1sm_ConnDelete | 0x02025151 | 38 B | delete connection |
| p_dua1sm_ConnMerge | 0x0202517d | 566 B | merge connections |
| p_dua1sm_ConnUnmerge | 0x020253c5 | 556 B | unmerge connections |

### string-based API (duasc)
| function | address | purpose |
|----------|---------|---------|
| p_duasc_UnitAllocateReq | 0x02272001 | alloc unit by type name string |
| p_duasc_UnitFreeReq | 0x0227201d | free unit by type name string |
| p_duasc_UnitSetReq | 0x02272039 | set param by name |
| p_duasc_UnitGetReq | 0x02272093 | get param by name |
| p_duasc_UnitConnectReq | 0x02271fb1 | connect by name |
| p_duasc_UnitDisconnectReq | 0x02271fd1 | disconnect by name |
| p_duasc_ConnCreateReq | 0x02271fa1 | create connection |
| p_duasc_ConnDeleteReq | 0x02271fa9 | delete connection |
| p_duasc_ConnMergeReq | 0x02271ff1 | merge |
| p_duasc_ConnUnmergeReq | 0x02271ff9 | unmerge |
| p_duasc_EnumUT | 0x02271f81 | enumerate unit types |
| p_duasc_EnumUE | 0x02271f89 | enumerate unit elements |
| p_duasc_EnumUEParam | 0x02271f91 | enumerate element parameters |
| p_duasc_Number | 0x02271f71 | name → numeric ID lookup |
| p_duasc_Name | 0x02271f79 | numeric ID → name lookup |
| p_duasc_DialDTMFReq | 0x022720d9 | dial DTMF by name |
| p_duasc_StopDTMF | 0x02272113 | stop DTMF |
| p_duasc_PlayMelodyReq | 0x02272141 | play melody |
| p_duasc_StopMelody | 0x0227217b | stop melody |
| p_duasc_Mute | 0x022721a9 | mute |

### internal helpers
| function | address | purpose |
|----------|---------|---------|
| p_duax_UnitCheck | 0x020277d9 | validate unit type + find free instance |
| p_duax_UnitType | 0x02027fdd | UID → unit type struct |
| p_duax_GetUnitHdl | 0x02026541 | UID → unit handle index |
| p_duax_GetUnitConnHdl | 0x02026491 | UID + connId → unit connection handle |
| p_duax_CreateConn | 0x02026385 | allocate connection slot |
| p_duax_DeleteConn | 0x020263f5 | free connection slot |
| p_duax_ConnPinSet | 0x02026329 | set pin connection state |
| p_duax_GetPinSB | 0x02026449 | get signal block for a pin |
| p_duax_UnitCloneable | 0x02027899 | check if unit can be cloned |
| p_duax_TmpArtPathApp | 0x02026195 | build temp audio routing path |
| p_duax_CopyConnAttr | 0x02026361 | copy connection attributes |
| p_GetNumber | 0x02282f01 | name → number lookup via bsearch |

## TDM (time-division multiplexing)

| function | address | purpose |
|----------|---------|---------|
| tdm_grant | 0x02270aa7 | grant/activate a TDM bus |
| tdm_revoke | 0x02270b6f | revoke TDM bus access |
| tdm_enable | 0x02271051 | low-level TDM enable |
| tdm_enable_clock | 0x02270e79 | enable TDM clock |
| tdm_read_sample | 0x0202d805 | read 16-bit sample from TDM register |
| tdm_write_sample | 0x0202daad | write 16-bit sample to TDM register |
| tdm_rx_available | 0x0202d821 | check rx frames available |
| tdm_tx_free | 0x0202da89 | check tx slots free |
| tdm_get_bank_size | 0x0202d799 | get TDM bank size |
| tdm_read_reg | 0x02270f33 | read TDM hardware register |
| tdm_write_reg | 0x02270f63 | write TDM hardware register |
| tdm_set_on_off | 0x02270f8f | enable/disable TDM |
| tdm_get_assignment | 0x022709eb | get TDM slot assignment info |
| tdm_get_rate | 0x02270ee5 | get TDM sample rate |
| optimized_tdm_handler | 0x02020505 | main audio pump (TDM ↔ DSP FIFO) |
| tdm_service_process_message | 0x0202d8b8 | COMA message handler |
| _tdm_cmd | 0x022759ed | shell command interface (3786 bytes!) |
| shell_tdm_grant | 0x02270aa7 | shell wrapper for grant |

## audio / DSP

| function | address | purpose |
|----------|---------|---------|
| p_da_DSPFifoRead | 0x02022c45 | read from DSP FIFO |
| p_da_DSPFifoWrite | 0x02022c5d | write to DSP FIFO |
| p_da_SetAudioRouter | 0x02022f3d | program audio routing table |
| p_da_SetCODEC | 0x02022ffd | select codec hardware |
| p_da_RouteCODEC | 0x02022f25 | route PCM to DRT |
| p_da_SetSBCollect | 0x020230a9 | set signal block collect mode |
| p_da_SetSBExecute | 0x020230dd | execute signal block |
| p_da_SetSBPrepare | 0x020230f1 | prepare signal block |
| p_da_SetTOG | 0x02023109 | set tone generator |
| p_da_SetV0dB | 0x0202312d | set 0dB reference level |
| p_da_SetupDialling | 0x02023161 | configure dial tone generation |

## FXS / SLIC

| function | address | purpose |
|----------|---------|---------|
| p_Do_FXS | 0x02020605 | FXS state machine event handler |
| p_Trans0_FXS | 0x02020de1 | FXS state transition |
| p_MuteAutoOffFxs | 0x02020b95 | auto mute off |
| p_MuteOffFxs | 0x02020c39 | manual mute off |
| p_suppressFxsTone | 0x02028941 | suppress FXS tone |
| p_updateMuterFxsState | 0x02029ce5 | update muter state |
| p_dat1_InitLine | 0x020233d9 | initialize phone line |

## voice / RTP (CSS side)

| function | address | purpose |
|----------|---------|---------|
| voice_process_message | 0x0229ffbc | COMA voice message handler |
| voice_get_session | 0x0229fe71 | allocate voice session |
| voice_free_session | 0x0229fe0d | free voice session |
| voice_session_process | 0x022a0644 | per-session processing |
| voice_request_start | 0x022a0338 | start voice stream |
| voice_send_dtmf | 0x022a0531 | send DTMF via RTP |
| voice_send_evt | 0x022a05d5 | send generic RTP event |
| voice_set_session_fifos | 0x022a0709 | configure session FIFOs |
| voice_rtp_query | 0x022a0445 | query RTP state |
| create_voice_message | 0x022777a5 | build voice response message |
| rtp_control_poll | 0x0229dcd1 | poll RTP control channel |
| p_rtp_pktrecv | 0x02294751 | RTP packet receive |

## codec initialization

| function | address | purpose |
|----------|---------|---------|
| Init_G_729A_Decoder | 0x02002d04 | G.729A decoder init |
| Init_G_729A_Encoder | 0x02002ea0 | G.729A encoder init |
| vadcng_Esl_Vad_Init | 0x02003114 | VAD/CNG init |
| adpcm_init | 0x022719d9 | ADPCM init |
| adpcm_on | 0x0201befd | enable ADPCM |
| adpcm_off | 0x0201bee5 | disable ADPCM |

## system / utility

| function | address | purpose |
|----------|---------|---------|
| p_dua_ApplInit | 0x02283815 | DUA application init |
| DUAPPL_INIT | 0x02017645 | application-level init hook |
| p_sys5_DisableArmInts | — | disable interrupts (critical section) |
| p_sys5_RestoreArmInts | — | restore interrupts |
| p_aec_recv_egress | 0x02282f14 | AEC egress path |
| p_aec_recv_ingress | 0x02282fa4 | AEC ingress path |
