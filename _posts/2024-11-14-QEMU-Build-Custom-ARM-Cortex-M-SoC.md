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

### 1.1. Overview about the SoC

The target emulated machine will be STM32F407 Discovery 1 board that based on STM32F407 SoC and Cortex-M4 core. Let's take a look to some basic concepts:

#### 1.1.1. Bus protocol and bus interface

In cortex Mx professor the bus interfaces are based on the Advanced Micro-controller Bus Architecture (`AMBA`) specification. The specification is designed by ARM to specify standards for on-chip communication inside the SoC. Several bus protocols are supported:

- `AHB`: AMBA High performance Bus that is mainly used for **high speed communication** with peripherals and memory.
- `APB`: AMBA Peripheral Bus that is used for **lower speed communication**, peripherals that are not required high operations are connected to this bus.

The Cortex-M4 core communicates with external world through various interfaces: `I-Bus`, `D-Bus`, `S-Bus`. By using different interfaces, the processor can access to different memory region.

- `I-Bus`: Instruction fetch.
- `D-Bus`: Data access.
- `S-Bus`: System interface to access peripheral memory, vendor specific regions.

```text
 __________________       ________
|Cortex-M4    I-bus|<--->| AHB Bus|<-----------> Flash.
| 168-MHz     D-bus|<--->| Matrix |<-----------> SRAM.
|             S-bus|<--->|________|<-----------> DMA1, DMA2.
|__________________|             \<-----AHB3---> External memory (SRAM, NOR Flash, NAND Flash).
                                  \<----AHB2---> Camera FIFO, USB OTG FS, RNG.
                                   \<---AHB1---> GPIOx.
                                         \<----> AHB/APB1 <---> DAC, USART, UART, WDG, CAN, SPI, I2C, etc.
                                          \<---> AHB/APB2 <---> TIM, USART, ADC, SPI, etc.
```

#### 1.1.2. Memory map

The bus interfaces provides 32-bit address and 32-bit data channels, so the processor can access up to 2^32 addressable memory. Memory map of the Cortex Mx processor `0x00000000` -> `0xFFFFFFFF`:

```text
|----------------------------|0xFFFFFFFF
|Vendor-specific memory 511MB|0xE0100000
|----------------------------|0xE00FFFFF
|Private peripheral bus   1MB|0xE0000000
|----------------------------|0xDFFFFFFF
|External device          1GB|0xA0000000
|----------------------------|0x9FFFFFFF
|External RAM             1GB|0x60000000
|----------------------------|0x5FFFFFFF
|Peripherals            0.5GB|0x40000000
|----------------------------|0x3FFFFFFF
|SRAM                   0.5GB|0x20000000
|----------------------------|0x1FFFFFFF
|CODE                   0.5GB|0x00000000
```

512MB of the `CODE` region contains different type of code memories: Flash, ROM, etc. 1MB of Flash memory reside in the region that start from the address `0x08000000`. Processor be default fetches the vector table information from this region right after reset.

The `SRAM` (Static-RAM) region is the next 512MB that contains 112KB + 16KB SRAM started from `0x20000000` and then reserved region. This region can also contain executable code.

Core Coupled Memory (CCM) is a special SRAM block available in the STM32 F2, F3, and F4 families. CCM can be faster than the standard SRAM, but it is usually smaller. To use, this region need to be define in linker script and configure by user firmware.

The Peripherals region is used for almost on chip peripherals. For example:

- 9 GPIOx peripherals: `0x40020000` -> `0x400223FF`
- 14 Timer peripherals:
  - TIM9, TIM10, TIM11: `0x40014000` -> `0x40014BFF`
- 2 DMAx peripheral: `0x4002 6000` -> `0x4002 67FF`

#### 1.1.3. Clock sources

The STM32F407 has 2 main clock sources:

- HSI (High Speed Internal): internal RC Oscillator that is used by default. The HSI clock signal is generated from an internal 8MHz RC Oscillator.
- HSE (High speed External): The clock source take from external resonator.

```text
 _____                  System Clock Mux
|_HSI_|--------------------->|\                    /------------------------> Ethernet clock.
           \                 | \                  /
 _____      \                |  \     ________   /
|_HSE_|------\-------------->|   \-->|_SYSCLK_|-----AHB Prescaler-----------> Peripheral clocks.
            \ \              |   /                                 \ \-APB1-> Peripheral clocks.
             \ \      __     |  /                                   \--APB2-> Peripheral clocks.
              \ \----|  \    | /
               \     |PLL|-->|/
                \----|__/ \
                           \------------------------------------------------> 48MHz clock.
                            \-----------------------------------------------> I2S clock.
```

By default the SoC use HSI as a source for SYSCLK. Before use any peripherals, its clock should be enable.

Some other clock sources like LSI, LSE (Low Speed) are used for RTC, IWDG.

### 1.2. Target emulator

#### 1.2.1. Memory

- The SoC support 128KB SRAM start from `0x20000000`.
- The 1MB Flash start from `0x08000000`.
- The 64KB of CCM (core coupled memory) data RAM start from `0x10000000`.

#### 1.2.2. Clock

By default after resetting, the HSI is selected as a source clock with value 16MHz.

#### 1.2.3. Peripherals

The STM32F407 SoC have various peripherals, some of them are not supported (or developed) by QEMU yet. Let's start with two basic peripherals: USART and GPIOx. The SoC has 4 USARTs/2 UARTs along with 9 GPIOx.

## 2. Understand the loading flow

1. Launch the qemu and specify the target machine, for example in our case `stm32f407g_disc`:

    ```bash
    qemu-system-arm -machine stm32f407g_disc -kernel boot.elf
    ```

2. QEMU start running:
   - Loading all modules in the QoM tree.
   - Running the `type_init()` callbacks.
   - Initialize all types by calling `class_init` callbacks, that also included our machine type `stm32f407g_disc_class_init()` and soc type `stm32f407_soc_class_init()`.
   - Look up to our machine type `stm32f407g_disc` in the tree and create an object for the type, so the constructor for our machine type will be called `stm32f407g_disc_init()`.
   - In the our machine constructor, we start initializing the machine.
   - Create a source clock with frequency.
   - Create an `stm32f407_soc` device object. The corresponding `stm32f407_soc` will be called. In the constructor we do:
     - Add and initialize object's runtime properties: CPU, Peripherals (uart, spi, timer, etc.), Interrupt, system clock, reference clock.
   - Connect our device with the clock.
   - Plug our device to the default sysbus and realize the device. In this time the realize callback will be called.
     - Initialize memories: flash, ram, sram.
     - Realize object properties: CPU, peripherals.
     - Mapping peripherals to memory.
     - Connect peripheral interrupt line.
   - Load the kernel.
