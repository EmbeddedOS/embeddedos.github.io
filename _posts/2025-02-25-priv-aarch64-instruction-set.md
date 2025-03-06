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

### Load/Store

- `ldr w0, =0xfff` -- Loads 0xfff into w0.
- `ldr x8, [x10]` -- Loads the value from the address in x10 into x8.
- `ldr x2,=place` -- Loads the address of place into x2.
- `ldrb	w0, [x0]` -- Loads first byte from memory stored in x0 into w0.
- `ldr x0, [Xn/sp, #imm]` -- `ldr x0, [sp, #16]` -- Load the value from the address sp + 16 into x0, `ldr x0, [x3, #16]` -- Load the value from the address of x3 + 16 into x0.

### Stack operation

- Create a stack frame when entering a block code: `sub sp, sp, #16` -- Create a frame with 16 bytes. (enough for 2 64=bit registers).
- Store to stack frame: `str x0, [sp, 8]` -- Store x0 to second 64 bit register in stack frame.
- Load from stack frame: `ldr x0, [sp, 8]` -- Load second 64 bit register into x0.
- Delete a stack frame when exiting a block code: `add sp, sp, #16` -- Remove a frame with 16 bytes.

### Operator

- `sub x0, x0, #20` -- sub x0 with 20 and store to x0, the value should be in range 0-4095. If the value is too big, using below commands instead.
- `sub x1, x1, x2` -- sub x1 to x2 and store to x1.
- `subs  x1, x1, x2` -- sub x1 to x2 and store to x1. the SUBS instruction updates the N, Z, C and V flags according to the result.

### Branch

- `bl` branch to other instruction, the `LR` will load the address of next instruction after `bl`, using this indicate you will return in the future.
- `b` branch without return, you just want to jump to other instruction.
- `bne new_instruction` -- branch if not equal. If zero flag is clear -> jump to new_instruction. If zero flag is set -> execute next instruction.

## Condition expression

- `cmp x0, #5` -- Compare value x0 with 5, if equal, set the zero flag, otherwise, clear it.
