---
title: "Build and emulate your custom ARM cortex-M SoC with QEMU"
description: >-
  Let's build your own QEMU to emulate ARM cortex-M SoC based.

author: Cong
date: 2024-11-22 00:00:00 +0800
categories: [QEMU, SOC]
tags: [QEMU, SOC, ARM, CORTEX-M]
image:
  path: assets/img/qemu_logo.jpg
  alt: QEMU.
---

## 1. Target machine

The target machine to be added:

- STM32F407 Discover 1 board.
- STM32F407 SoC based Cortex-M4 core.
- RAM:
- Peripherals:
- Flash:
- Interrupt controller:

## 2. Understand the loading flow

1. Launch the qemu and specify the target machine, for example in our case `stm32f407g_disc`:

    ```bash
    qemu-system-arm -machine stm32f407g_disc -kernel boot.elf
    ```

2. QEMU start running:
   - Loading all modules in the QoM tree.
   - Running the `type_init()` callbacks.
   - Initialize all types by calling `class_init` callbacks, that also included our machine type `stm32f407g_disc_class_init()` and soc type `stm32f407_soc_class_init()`:
   - Look up to our machine type `stm32f407g_disc` in the tree and create an object for the type, so the constructor for our machine type will be called `stm32f407g_disc_init()`.
   - In the our machine constructor, we start initializing the machine.
     - Create an `stm32f407_soc` object.
