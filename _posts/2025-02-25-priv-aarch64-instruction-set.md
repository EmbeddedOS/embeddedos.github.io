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

### Operator

### Branch

- `bl` branch to other instruction, the `LR` will load the address of next instruction after `bl`, using this indicate you will return in the future.
- `b` branch without return, you just want to jump to other instruction.
