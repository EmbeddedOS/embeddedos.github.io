---
title: "The eBPF Virtual Machine"
description: >-
  Let's discover the eBPF VM layer, how eBPF programs are built and loaded into kernel.

author: Cong
date: 2025-11-08 00:01:00 +0800
categories: [eBPF, virtual-machine]
tags: [eBPF, network, security, tracing, virtual-machine]
image:
  path: assets/img/ebpf-vm.png
  alt: eBPF virtual machine.
published: true
---

An eBPF program is a set of eBPF bytecode instructions, (eBPF code can be written directly in this bytecode, but higher-level programming languages are easier to deal with). We could say, eBPF is written in C and then compiled to eBPF byte code.

This bytecode runs in an eBPF virtual machine within the kernel.

## The eBPF Virtual Machine

The eBPF VM, is a software sandbox inside the kernel that provides a secure and isolated environment to run the verified bytecode directly. Like any VMs that translate the program machine code into native machine code, the eBPF VM takes in a program in the form of eBPF bytecode instructions, and these have to be converted to native machine instructions that run on the CPU. And don't be confused with the KVM (Kernel-based virtual machine), that turns kernel act like a hypervisor.

Historically the eBPF VM was only available from within the kernel and was used to filter network packets only (Berkley Packet Filter). The user space interference was only exposed via the `bpf()` system call and `uapi/linux/bpf.h` since kernel v3.18.

### eBPF registers

eBPF bytecode consists of a set of instrctions, and those instructions act on eBPF registers (software virtual registers)

### `bpf()` system call

### The JIT compiler

### The verifier

### Attach to an event

## A demonstration

Benefits of this approach:

- Define its own eBPF instruction set -> not depends on the archicture -> CO_RE.


that able to access a subset of kernel functions and memory.