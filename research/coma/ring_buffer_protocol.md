# ARM↔CSS ring buffer protocol

## ring buffer location

the ARM→CSS ring buffer is allocated by app_dsp (or comatose_dsp) from the
shared memory pool during init.

layout at the allocation watermark:
```
+0x00: ring buffer header (16 bytes = 4 uint32)
+0x10: ring buffer data area (0x79c bytes = 487 uint32 entries)
```

the header contains USERSPACE VIRTUAL ADDRESSES (the mmap'd pointer):
```c
struct shm_ringbuf_header {
    uint32_t base_vaddr;      // mmap_base + data_offset
    uint32_t end_vaddr;       // base + num_entries * 4
    uint32_t write_vaddr;     // current write position (ARM updates)
    uint32_t read_vaddr;      // current read position (CSS updates)
};
```

**critical:** the header stores userspace virtual addresses, NOT pool-relative
offsets. the CSS does MMU translation from the userspace address to find the
physical memory. writing pool offsets instead causes a CSS panic.

the shared memory pool header stores pointers to this allocation:
- shm+0x28 = saved_alloc_ptr (start of our allocation)
- shm+0x2c = codec_state_ptr (= saved_alloc + 0x10)

## message format

each message: `[instance_addr, (type << 13) | num_params, param0, param1, ...]`

- word 0: instance address (resolved by CSS via p_dspa_find_instance)
- word 1: header — bits 15:13 = message type, bits 7:0 = parameter count
- words 2+: parameters (num_params words)

## known message types

### type=0: initialization/completion events
used for session startup. triggers p_auc_CB_handler state 3→4 transition.

format: `[element_addr + 4, 0x0000, ]` (0 params)

the CSS resolves element_addr+4 via p_dspa_find_instance by checking
(msg_word0 - 4) against ITAB entries. returns instance with ARM flag.

### type=1: level ready
sent during startup after BG level state machine completes.

format: `[0, 0x2001, level_number]` (1 param)

instance = 0 (special, returns 0xBFF). sent for all 4 levels during init.

### type=3: frame production trigger (DECODER ONLY!)
sent periodically (~10ms) for the decoder element to trigger CSS encoder output.

format: `[decoder_element_addr + 4, 0x6001, 0]` (1 param = 0)

**MUST be sent for the DECODER element (elem[21] in group 0), NOT the encoder.**
**MUST have 1 parameter (value 0), NOT 0 parameters.**

sending for wrong elements or with wrong param count causes CSS panic via
SEC interrupt with invalid number (plicu_is_fiq_mode panic).

stock app_dsp sends this continuously every ~10ms during active sessions.

### type=4: heartbeat tick
periodic keepalive from BG level 0.

format: `[0, 0x8000, ]` (0 params)

CSS processes this via p_auc_CB_handler, producing "API callback event=242,
inst=3071" log output. 3071 = 0xBFF (special instance for instance=0).

## CSS processing chain

p_adsp_check_msg_fifo reads messages:
1. reads word 0 (instance_addr) → p_dspa_find_instance → resolved instance
2. reads word 1 (header) → extract type and param count
3. reads params
4. p_dspa_GetModuleID(instance) → module_id
5. event = (type << 13) | (module_id | 0x800)
6. dispatches to i_msg_cb → registered handlers + p_dsp_CallBack

## stock app_dsp ring buffer during active L16 session

observed repeating pattern: `[0xb6d0ab58, 0x00006001, 0x00000000]`
- instance: decoder element addr + 4 (shm+0xb58 = elem[21]+4)
- header: (3 << 13) | 1 = 0x6001 (type=3, 1 param)
- param: 0

this message repeats every ~10ms (every 2nd dispatch cycle at 5ms).

## CSS→ARM communication channels

### COMA socket (confirmed working)

CSS sends DUA async callbacks (cmd=0x7f) on the same AF_COMA "dua" socket that
the ARM opens for DUA init. this carries cascade control events (0xf2) and
element notifications (0x0e). see [css_arm_events.md](css_arm_events.md) for
the full event format and cascade interaction.

our `dua_poll_event()` reads these. they are confirmed to drive the module
readiness cascade — without responding to 0xf2, the cascade stalls.

### CSS→ARM shared memory ring buffer (NOT investigated — known gap)

stock app_dsp's `dfl_calc` reads from a CSS→ARM ring buffer in shared memory
as part of the BG level state machine. this is SEPARATE from the DUA COMA
socket events. it likely carries:
- codec activation commands (I-switch signals)
- state transition triggers
- possibly the 0xf2-equivalent at the shared memory level

our BGSC never reads this buffer. it's unclear whether the COMA socket events
are sufficient, or whether this ring buffer carries additional information
needed for the cascade or audio routing. **this is a priority investigation
item** — if the CSS is writing messages here that we silently drop, it could
explain cascade stalls or missing configuration.
