---
title: "AArch64 instruction set"
description: >-
  Aarch64 instruction set

author: Cong
date: 2025-02-19 00:01:00 +0800
categories: [Aarch64, Assembly]
tags: [Aarch64, Assembly, Instruction Set]
published: false
---

## AArch64 Instruction set

- [x0] -- Refer to the value is stored in the memory that is hold in x0. Use this like get *ptr in C.

### Load/Store

- `ldr w0, =0xfff` -- Loads 0xfff into w0.
- `ldr x8, [x10]` -- Loads the value from the address in x10 into x8.
- `ldr x2,=place` -- Loads the address of place into x2.
- `ldrb w0, [x0]` -- Loads first byte from memory stored in x0 into w0.
- `ldr x0, [Xn/sp, #imm]` -- `ldr x0, [sp, #16]` -- Load the value from the address sp + 16 into x0, `ldr x0, [x3, #16]` -- Load the value from the address of x3 + 16 into x0.
- `ldp w3, w7, [x0]` - Loads the value is stored in memory hold by x0, into w3, and loads [x0 + 4] into w7.
- `str x0, [sp, #-8]!` - Add sp with -8 and push x0 into.
- `stp d0, d1, [x4]` - Store d0 to [x4] and d1 to [x4 + 8].
- `stp x19, x20, [x8], #16` - Equal to Store `stp x19, x20, [x8]` and then Add `add x8, x8, #16`.
- `stp x19, x20, [x8, #16]!` - Equal to Add  `add x8, x8, #16` and then Store `stp x19, x20, [x8]`.
- `stp x0, x1, [sp, #-16]!` - Sub stack to make a frame of 16 bytes and then Push x0, and x1 onto stack.
- `ldp x0, x1, [sp], #16` - pop x0, x1 from the stack and then add sp to remove the frame.

### Stack operation

- Create a stack frame when entering a block code: `sub sp, sp, #16` -- Create a frame with 16 bytes. (enough for 2 64=bit registers).
- Store to stack frame: `str x0, [sp, 8]` -- Store x0 to second 64 bit register in stack frame.
- Load from stack frame: `ldr x0, [sp, 8]` -- Load second 64 bit register into x0.
- Delete a stack frame when exiting a block code: `add sp, sp, #16` -- Remove a frame with 16 bytes.
- `stp x0, x1, [sp, #-16]!` - Sub stack to make a frame of 16 bytes and then Push x0, and x1 onto stack.
- `ldp x0, x1, [sp], #16` -pop x0, x1 from the stack and then add sp to remove the frame.

### Operator

- `sub x0, x0, #20` -- sub x0 with 20 and store to x0, the value should be in range 0-4095. If the value is too big, using below commands instead.
- `sub x1, x1, x2` -- sub x1 to x2 and store to x1.
- `subs x1, x1, x2` -- sub x1 to x2 and store to x1. the SUBS instruction updates the N, Z, C and V flags according to the result.

### Moving

- `ldr x0, =label`
- `adr x0, label` - Form PC-relative address, `label` relative address (runtime address) to x0.

### Branch

- `bl` branch to other instruction, the `LR` will load the address of next instruction after `bl`, using this indicate you will return in the future.
- `b` branch without return, you just want to jump to other instruction.
- `bne new_instruction` -- branch if not equal. If zero flag is clear -> jump to new_instruction. If zero flag is set -> execute next instruction.

### Condition expression

- `cmp x0, #5` -- Compare value x0 with 5, if equal, set the zero flag, otherwise, clear it.

## AArch64 registers

### Special registers

- Zero register: `xzr`, `wzr`.
- Program counter: `pc`.
- Stack pointer: `SP_EL0`, `SP_EL1`, `SP_EL2`, `SP_EL3` -- Holds the stack pointer associated with ELx. at a time, `sp` register will refer to one of them.
- Program status: `SPSR_EL1`, `SPSR_EL2`, `SPSR_EL3`.
- Exception Link register: `ELR_EL1`, `ELR_EL2`, `ELR_EL3` -- Hold the address to return after handling exception.
- Exception Syndrome register: `ESR_EL1`, `ESR_EL2`, `ESR_EL3` -- Exception syndrome register.
- Fault address register: `FAR_ELx` -- Hold the address that relates to triggering exception, for example MMU faults.

### general registers

- `x0`..`x30` -- general purpose registers.
- `x30` -- map to Link register -- hold the function return address.

## AArch64 Calling conventions

- `x31` (SP) -- Stack pointer or a zero register, depending on context.
- `x30` (LR) -- Procedure link register, used to return from subroutines.
- `x29` (FP) -- Frame pointer.
- `x19` to x28 -- Callee-saved.
- `x18` (PR) -- Platform register. Used for some operating-system-specific special purpose, or an additional caller-saved register.
- `x16` (IP0) and x17 (IP1) -- Intra-Procedure-call scratch registers.
- `x9` to `x15` -- Local variables, caller saved.
- `x8` (XR) -- Indirect return value address.
- `x0` to `x7` -- Argument values passed to and results returned from a subroutine.
