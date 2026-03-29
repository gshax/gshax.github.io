# CSS coprocessor architecture overview

## processor model

the ht818 has a dual-processor architecture:
- **APP** (application processor) — arm core running linux, hosts gs_ata/app_dsp/etc
- **CSS** (communication subsystem) — arm core running proprietary RTOS firmware (`_css.elf`)

the two processors communicate via **COMA (cordless manager)**, a message-passing framework
with a kernel driver on the linux side and a matching implementation in the CSS firmware.

## CSS firmware overview

- binary: `materials/css/_css.elf` (3MB, ARM executable)
- **has DWARF debug symbols** — full function names, types, source file paths, variable names
- ~3300 functions
- source tree was `src/d53/drv/` (visible in debug info FILE entries)
- includes a debug shell with interactive commands (fragile, many commands crash)

## memory layout

| region | address | size | purpose |
|--------|---------|------|---------|
| ITCM | 0x08000000 | 272 KB | instruction tightly coupled memory (boot code) |
| DTCM | 0x08100000 | 128 KB | data tightly coupled memory |
| AHBRAM | 0x08200000 | 128 KB | auxiliary shared ram (dvf99 only) |
| BMP | 0x08300000 | 288 B | bitmap module MMIO registers (NOBITS in ELF) |
| DRT | 0x09000000 | 128 B | digital radio transmit MMIO registers (NOBITS in ELF) |
| VIRT_ITCM | 0x020008f0 | ~214 KB | virtual mapping of instruction memory |
| VIRT_DTCM | 0x02100000 | ~76 KB | virtual mapping of data memory |
| DDR_RAM | 0x02200000 | ~2.8 MB | shared DDR (code + data, dynamically allocated by kernel) |

the CSS firmware runs primarily from virtual addresses in the 0x02xxxxxx range.
the kernel maps physical TCM regions via ioremap and allocates DDR via dma_alloc_coherent.

### ghidra setup notes

the ELF program headers are broken (segment 1 claims VMA 0x40000000, which is bogus).
load using **section headers** instead. manually create volatile memory blocks for:
- BMP at 0x08300000 (0x120 bytes)
- DRT at 0x09000000 (0x80 bytes)

these are needed for register name constants to resolve properly in the decompiler.

## COMA message bus

### transport

- bidirectional **CFIFOs** (circular FIFOs) in DMA-coherent shared memory
- one l2c (linux-to-css) and one c2l (css-to-linux) FIFO per service
- lock-free single-reader/single-writer ring buffers
- **SEC** (semaphore exchange center) hardware interrupts for signaling

### message format

```c
struct cmsg {
    int type;                // message type identifier
    unsigned int params_size; // size of parameter block
    unsigned int payload_size; // size of payload data
    char body[0];            // params followed by payload
};
```

### services registered on CSS

| service | purpose |
|---------|---------|
| coma (id 0) | core service registration/deregistration |
| dua | digital unit allocation — audio routing and DSP configuration |
| voice | RTP session management, codec FIFO setup |
| tdm | TDM bus grant/revoke |
| dvf9a | hardware interrupt routing |
| crypto | crypto operations |
| datahandler | DECT data calls |
| sharedmem | shared memory initialization |

### kernel-side interfaces

- **AF_COMA sockets**: `socket(PF_COMA, SOCK_SEQPACKET, 0)` + `connect()` with service name
  - requires CAP_SYS_ADMIN or CAP_NET_ADMIN
  - `struct sockaddr_coma { sa_family_t family; char service[16]; };`
- **/dev/voiceXX**: character devices for voice RTP frame I/O
- **/dev/fxsXX**: TAPI interface for SLIC control (ring, hook, line feed)

## CSS services in detail

### voice service
defined in kernel: `coma-voice.c`, header: `cmsg-voice.h`

message types (enum):
- CMSG_VOICE_REQUEST_GET_SESSION (0)
- CMSG_VOICE_REQUEST_SET_SESSION_FIFOS (2)
- CMSG_VOICE_REQUEST_START_SESSION (4)
- CMSG_VOICE_REQUEST_STOP_SESSION (6)
- CMSG_VOICE_REQUEST_FREE_SESSION (8)
- CMSG_VOICE_REQUEST_SEND_DTMF (10)
- CMSG_VOICE_RECEIVE_DTMF (12)
- CMSG_VOICE_DATA
- (odd numbers are REPLY variants)

params struct: `cmsg_voice_params` with session_id, line_id, result, and a union of config types.

CSS-side handler: `voice_process_message()` at 0x0229ffbc (676 bytes)

### TDM service
CSS-side handler: `tdm_service_process_message()` at 0x0202d8b8

message types: grant (0) and revoke (1)
- grant params: id, rate, channels, sample_size
- responses: ack or nack

the kernel sends TDM_GRANT/TDM_REVOKE via COMA; CSS calls `tdm_grant()`/`tdm_revoke()`.
