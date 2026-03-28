# BLISS HACK — Final Combined Xbox One Fat Boot ROM Payload

## Overview

**Double Voltage Glitch (Bliss)** → eFuses extraction + Hypervisor patch + Full firmware dump
Includes automated UART raw binary output and USB trigger readiness.

* Based on: *Markus Gaasedelen RE//verse 2026*
* Rebuilt by: **Taylor Christian Newsome**
* Date: **2026-03-28**

---

## ⚠️ Warning

All memory addresses in this payload are **placeholders**.

* No public zero-day addresses are available
* You **must replace** them with real reverse-engineered values
* Failure to do so will result in non-functional execution

---

## Assembly Payload

```asm
.syntax unified
.arch armv7-a
.thumb

.globl _start
_start:
    ; Disable interrupts and caches
    cpsid   if
    mrc     p15, 0, r0, c1, c0, 0
    bic     r0, r0, #0x1000
    bic     r0, r0, #0x0004
    mcr     p15, 0, r0, c1, c0, 0
    dsb
    isb

    ; Set up stack (SRAM)
    ldr     sp, =0x30000000

    ; 1. Dump eFuses
    ldr     r0, =0xF800E000
    bl      dump_efuses

    ; 2. Decrypt bootchain
    bl      decrypt_bootchain

    ; 3. Patch hypervisor
    ldr     r0, =0x40000000
    bl      patch_hypervisor

    ; 4. Dump firmware via UART
    bl      dump_firmware_via_uart

    ; 5. Completion message
    adr     r0, done_msg
    bl      uart_puts

    b       .

done_msg:
    .ascii  "BLISS HACK COMPLETE - Full firmware dumped!\r\n"
    .byte   0
```

---

## eFuse Dump Routine

```asm
dump_efuses:
    push    {r4-r7, lr}
    ldr     r4, =0x30001000
    mov     r5, #0
    mov     r6, #128
1:
    ldr     r0, =0xF800E000
    add     r0, r0, r5, lsl #2
    ldr     r7, [r0]
    str     r7, [r4, r5, lsl #2]
    add     r5, #1
    cmp     r5, r6
    bne     1b
    pop     {r4-r7, pc}
```

---

## Bootchain Decryption Stub

```asm
decrypt_bootchain:
    ; TODO:
    ; Use eFuse-derived keys to decrypt SP1, SP2, 2BL
    bx      lr
```

---

## Hypervisor Patch

```asm
patch_hypervisor:
    ldr     r0, =0x40001000
    mov     r1, #0xE320F000
    str     r1, [r0]
    bx      lr
```

---

## Firmware Dump via UART

```asm
dump_firmware_via_uart:
    push    {r4-r8, lr}
    ldr     r4, =0x08000000
    ldr     r5, =0x04000000
    mov     r7, #0

dump_loop:
    ldr     r8, [r4, r7]
    mov     r0, r8
    bl      uart_putc
    mov     r0, r8, lsr #8
    bl      uart_putc
    mov     r0, r8, lsr #16
    bl      uart_putc
    mov     r0, r8, lsr #24
    bl      uart_putc

    add     r7, #4
    cmp     r7, r5
    blo     dump_loop

    pop     {r4-r8, pc}
```

---

## UART Helpers

### Print String

```asm
uart_puts:
    push    {r4, lr}
    mov     r4, r0
1:
    ldrb    r0, [r4], #1
    cmp     r0, #0
    beq     2f
    bl      uart_putc
    b       1b
2:
    pop     {r4, pc}
```

### Send Character

```asm
uart_putc:
    ldr     r12, =0xF8001000

1:
    ldr     r1, [r12, #0x18]
    tst     r1, #0x20
    bne     1b

    strb    r0, [r12]
    bx      lr
```

---

## End of Payload

---

## Repository

GitHub:
https://github.com/SleepTheGod/Xbox-One-Bliss-Hack/blob/main/bliss.asm

---

## Notes for Researchers

* Requires precise timing for voltage glitching
* UART base + FIFO flags may differ depending on SoC revision
* Hypervisor patch likely requires multiple instruction overwrites
* Bootchain decryption is the **critical missing component**

---

## Suggested Next Steps

* Reverse BootROM memory map
* Identify real eFuse controller offsets
* Implement AES + SHA routines for bootchain
* Add USB trigger synchronization for glitch timing
* Expand UART to DMA for faster dumping

---
