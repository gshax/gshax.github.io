# CSS crash dump analysis

## crash dump mechanism

when the CSS panics, `platform_panic_swi` (at 0x022a138c) saves:
1. 12 CP15 coprocessor registers (Control, TTB, Domain, DFSR, IFSR, FAR, etc.)
2. ITCM copy (35 words from 0x08a00000)
3. BMP register copy (14 words from 0x08300000)
4. 4-word gap
5. more register data
6. saved ARM registers (r0-r12, sp, lr) and panic parameters

the dump is written to CSS virtual address 0x023639c8 = CSS DDR offset 0x1639c8.

## physical memory layout

from device tree and dmesg:
- CSS DRAM: physical 0x4f400000, size 8MB (reserved-memory node "css-dram")
- shared memory: physical 0x4ff00000, size 1MB (reserved-memory node "shared-dram")
- CSS DDR virtual 0x02200000 maps to physical 0x4f400000
- crash dump physical: 0x4f400000 + 0x1639c8 = **0x4f5639c8**

## using the css_crash tool

```
/tmp/css_crash    # reads from /dev/mem, decodes registers and finds LR
```

## crash analysis: type-3 event panic (2026-04-01)

**symptom:** CSS panic when sending type=3 (0x6000) ring buffer events for
encoder elements during active voice session.

**crash dump:**
- fault address: 0xfffffffb (near-NULL, -5)
- data fault status: 0x01 (translation fault)
- LR (crash location): 0x0202ac45 = `plicu_is_fiq_mode` + 0x15

**call chain:** `css_idle` → `do_swi` → `plicu_is_fiq_mode(nr >= 64)` → panic

**root cause:** our type-3 event message was processed by `p_adsp_check_msg_fifo`
→ `p_dspa_find_instance` → `p_dspa_GetModuleID` → event constructed. then
`i_msg_cb` dispatched to `p_auc_CB_handler` AND `p_dsp_CallBack`. in
`p_dsp_CallBack`, the 0x6000 branch called `(*puVar4)(stored_instance, 0xf2)`
which triggers a SEC (semaphore exchange center) interrupt. our instance
(resolved from p_dspa_find_instance with ARM flag 0x800 set) was too large
for the SEC interrupt table (max 64).

**fix:** stopped sending type-3 events for encoder elements. discovered that
stock app_dsp sends type-3 events for the DECODER element only, with format
[decoder_addr+4, 0x6001, 0] (1 parameter, not 0).

## p_dspa_find_instance protocol

the ring buffer message's first word (instance address) is resolved by:
```c
// scans all 4 BG levels' ITAB tables
// checks: (msg_word0 - 4) == *itab_entry
// returns: element_index | 0x800 | (level << 12)
```

so the message must contain `element_control_word_address + 4`.

## instance number encoding

```
bits [14:12] = BG level (0-3)
bit  [11]    = ARM flag (0x800) — required for ARM-side instances
bits [9:0]   = element index within level
```

## p_dspa_GetModuleID

reads a per-level module table to get a module_id byte for each element:
- level table at CSS 0x02031c38: [offset, count] per level
- module byte array at CSS 0x02031c48
- module_id = table[level_offset + element_index] | 0x800
- repeating pattern of 22 bytes per SPVOIPNDA group

## SEC/interrupt mapping

the SEC trigger `(*puVar4)(instance, event_code)` expects instance < 64.
ARM-flagged instances (0x800+) are too large → panic.
this means type-3 events should NOT be sent for arbitrary elements via
the ring buffer. the stock type-3 events work because the decoder element
resolves to a valid SEC channel.
