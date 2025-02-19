---
title: "AArch64 Exception models"
description: >-
  Understand exception and privilege model in AArch64.

author: Cong
date: 2025-02-19 00:01:00 +0800
categories: [ARM, exception]
tags: [AArch64, exception models]
image:
  path: assets/img/service_call_routing.png
  alt: aarch64 exception levels.
published: false
---

Targets:

- Understand exception and privilege model in AArch64.
- Be able to develop low-level code, such as boot code or kernels, particularly, to manage exceptions.

Requirements:

- Understand Assembly syntax or willing to learn 😛.

## 1. Privilege and Exception levels

In modern system, software is split into different modules, each with a different level of access to resources. For example, splitting between the OS kernel and user programs. For stability and security, the OS need to perform actions that we don't want user programs do it directly. So the kernel needs a high level of access to resources, whereas the apps need limited ability to configure the system. *Privilege dictates which resources a software entity can access*.

The AArch64 enable this *spit* by implementing different levels of privilege.

The current privilege can only change when processor takes or returns from an exception. Therefore, these privilege levels are referred to as *Exception Level*.

### 1.1. Exception levels

The name for privilege in Aarch64 is Exception level, shortly, EL. The ELs are numbered, `EL<x>` with `x` between 0 and 3. The higher level of privilege the higher number.

The architecture does not specify what software uses which EL, and not all levels are required. EL0 and EL1 are mandatory, whereas EL2 and EL3 are optional. This makes the architecture become more popular with the big range of applications. Here is a common implementation:

```text
ELO |       Application       |
EL1 |         Rich OS         |
EL2 |        Hypervisor       |
EL3 | Firmware/Secure Monitor |
```

The EL can only change when:

- Taking an exception.
- Returning from an exception.
- Processor reset.
- During Debug state.
- Exiting from Debug state.

When an exception is taken, EL can increase or stay the same. Conversely, when returning an exception, EL can only decrease or stay the same.

### 1.2. Types of privilege

#### 1.2.1. Memory

ARM Cortex-A families implements a virtual memory system, in which a Memory Management Unit (aka. MMU) allows software to assign attributes (read/write permissions, etc.) to regions of memory. With these attributes, the memory accesses can be separated into privileged and unprivileged accesses.

The MMU configuration is stored in System registers, current EL also decide you have the  access those registers or not.

#### 1.2.2. Registers

In AArch64, there are 2 categories of register:

- General registers that are used instruction processing.
- System registers that are used to configure the processor and control systems like MMU and exception handling.

The General registers are able to access from all ELs, meanwhile, the system registers require privilege to access.

The system registers names end with `_ELx`. The `_ELx` suffix specifies the minimum privilege necessary to access the register. For example, `VBAR_EL1`, the vector base address register, needs at least EL1 privilege to access the register.

There are many system registers with similar functions that have names that differ only by their Exception level suffix. They are independent registers. For example, there is a system control register `SCTLR` for each EL, each register control the arch features for that EL: `SCTLR_EL1`, `SCTLR_EL2`, and `SCTLR_EL3`.

Higher Exception levels have the privilege to access registers that control low levels. We normally don't do that, however, sometimes we do access that for special features, for examples, virtualization, context switching, etc.

There are some special system registers when handling exceptions:

- `ELR_ELx`: Exception Link Register, holds the address of the instruction which caused the exception.
- `ESR_ELx`: Exception Syndrome Register, the reasons for the exception.
- `FAR_ELx`: Fault Address Register, holds the virtual faulting address.
- `HCR_ELx`: Hypervisor Configuration Register, controls virtualization settings and trapping of exceptions to EL2.
- `SCR_ELx`: Secure Configuration Register, controls secure state and trapping of exceptions to EL3.
- `SCTLR_ELx`: System Control Register, controls standard memory, system facilities, report functions's status.
- `SPSR_ELx`: Saved Program Status Register, holds the saved processor state when an exception is taken to this ELx.
- `VBAR_ELx`: Vector Base Address Register, holds the exception base address for any exception that is taken to ELx.

## 2. Exception types

The exception model in AArch64 is a complicated topic, we only discuss basic things in here, first thing first, the concept. *Exceptions are conditions or system events* that usually require remedial action or an update of system status by privileged software to ensure smooth functioning of the system. Therefore, they can cause the currently executing program to be suspended.

> The exception concept in other architectures might be different, for example, x86, exceptions are defined as a type of interrupt that are generated by the CPU when an `error` occurs.
{: .prompt-info }

Exceptions are used for many different reasons, including the following:

- Emulating virtual devices.
- Virtual memory management.
- Handling software errors.
- Handling hardware errors.
- Handling interrupts.
- Debugging.
- *Performing calls to different privilege or security states*.
- *Handling across different Execution states*.

One of these use cases is for implementing the system call, where we can request from different privilege.

When an exception is taken, instead of moving to the next instruction, the processor stops current execution and branches to a piece of code to deal with the request. This code is know as an *exception handler*.

### 2.2. Exception types

There are two types of exceptions:

- Synchronous exceptions are exceptions that can be caused by the instruction that is currently executed. Different causes of synchronous exceptions:
  1. Invalid instruction and trap exceptions.
  2. Memory accesses.
  3. Exception-generating instructions.
  4. Debug exceptions.
- Asynchronous exceptions are exceptions that are generated externally, like: Interrupt, SError, IRQ and FIQ, Virtual interrupts, etc.

### 2.2. Exception generating instructions

There are instructions that intentionally cause an synchronous exception to be generated and taken. These instructions are used to implement *system call interfaces* to allow less privileged software to request services from more privileged software.

![Service call routing](assets/img/service_call_routing.png)

- The Supervisor Call (`svc`) instruction enables a user program at EL0 to request an OS service at EL1.
- The Hypervisor Call (`hvc`) instruction, enables the OS to request hypervisor services at EL2 (available of the Virtualization Extensions are implemented).
- The Secure Monitor Call (`smc`) instruction, enables the Normal world to request Secure word services from firmware at EL3 (available of the Security Extensions are implemented).

### 2.3. Exception handler

![Exception handler](assets/img/exception_handler.png)
