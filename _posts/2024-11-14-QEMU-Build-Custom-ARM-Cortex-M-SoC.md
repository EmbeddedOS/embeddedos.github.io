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

## 2. Loading flow

1. You launch the qemu and specify the target machine, for example `stm32f407g_disc`:

    ```bash
    qemu-system-arm -machine stm32f407g_disc -kernel boot.elf
    ```

2. QEMU load the machine module and its dependencies that are configured before:

    ```text
    config STM32F407G_DISC
        bool
        default y
        depends on TCG && ARM
        select STM32F407_SOC
    ```
