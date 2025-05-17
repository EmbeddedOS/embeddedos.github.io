---
title: "The Linux device driver part I: hardware interaction."
description: >-
  How kernel identifies and interacts with hardware.

author: Cong
date: 2025-02-19 00:01:00 +0800
categories: [kernel, device driver]
tags: [Linux, kernel, device driver model, hardware interaction]
published: false
---

## 1. Objective

- Understand the ways kernel identifies devices: device descriptions and Bus enumerations.
- Understand the ways kernel interacts with hardware: memory map, interrupt, DMA, etc.

## 2. Hardware identifications

To identify devices, kernel must know about the devices that're connected to it. Some devices are detected automatically by their bus drivers, e.g., USB, PCU. Kernel uses bus enumeration to detect and identify them. And for those that are completely silent like UART, GPIO, etc. Kernel uses device descriptions to determine them. Let's explore both ways.

### 2.1. Device descriptions

Every device has its own specification, such as registers, memory addresses, interrupt line. Some devices cannot expose their info themselves to the kernel. So to let kernel notice about those devices, we might have 2 choices. First one is defining the device's information inside drivers, kernel itself. It's easy when develop drivers with hardcoding device information like registers, etc. But this way causes more issues, the kernel become specific, each hardware version, require specific kernel firmware. Every time a device trait changes, the kernel needs to be rebuilt. That brings us to the second choice: Separate the device description outside the kernel. Kernel now becomes more generic, independent with the hardware, and only needs to implement the device description parsers to archive the device information.

Some architectures have their own ways to describe their hardwares. ARM, PowerPC, or embedded devices prefer using DeviceTree, meanwhile, computer systems, like x86, prefer using ACPI table.

#### 2.1.1. DeviceTree

DeviceTree is a data structure and language for describing hardware. By that the kernel doesn't need to hard code details of the machine. The bootloader loads DeviceTree into memory and passes the pointer to the DeviceTree to the Kernel.

##### 2.1.1.1. Specification

The structure is a tree data structure with nodes that represent devices. Each node has name and property/value pairs that describe the characteristics of the device. Take a look to a simple example that nearly complete enough to boot a simple OS, with the platform type, CPU, memory and a single UART:

![DeviceTree Example](assets/img/devicetree_example.png)

Each node has the name in convention: `node-name@unit-address`. The unit-address is specific to the bus type on which node sits and must match  the first address specified in the `reg` property of the node. The root node has no name or unit-address, it is identified by a forward slash (/).

##### 2.1.1.2. Standard properties

There're set of standard properties for device nodes. Here are some notable ones:

- `compatible` -- consists of one or more strings that define the specific model of the device, this list should be used by kernel for device driver selection.
- `status` -- Indicates the operational status of a device. The valid values: `okay`, `disabled`, `reserved`, `fail`, and `fail-sss`.
- `#address-cells` -- used in nodes that has children to indicate number of cells used to encode the address in `reg`.
- `#size-cells` -- used in nodes that has children to indicate number of cells used to encode the size in `reg`.
- `reg` -- The address of the device's resources within the address space defined by its parent bus. Most commonly indicates the offsets and lengths of memory-mapped IO register blocks.

For example:

```text
soc { 
  compatible = "simple-bus";
  #address-cells = <1>;
  #size-cells = <1>;
  ranges = <0x0 0xe0000000 0x00100000>;
  serial@4600 {
    device_type = "serial";
    compatible = "ns16550";
    reg = <0x4600 0x100 0x8600 0x100>;
    clock-frequency = <0>;
    interrupts = <0xA 0x8>;
    interrupt-parent = <&ipic>; 
  }; 
};
```

The `#address-cells = <1>` indicates one address for a cell and `#size-cells = <1>` indicates one length for a cell. And the `reg` contains 2 cells `0x4600 0x100` and `0x8600 0x100`. Another example:


```text
soc { 
  #address-cells = <2>;
  #size-cells = <1>;
  serial@4600 {
    reg = <0x00000000 0x00000001 0x10000>;
  }; 
};
```

The `reg` contains one cell, each cell contains 2 addresses, in this case `0x00000000 0x00000001` and 1 size, in this case `0x10000`.

##### 2.1.1.3. Interrupts

##### 2.1.1.4. Base device node types

All DeviceTree have a root node, and the root includes at least 2 children:

- One `/cpus` node.
- At least one `/memory` node.

Other nodes are optional. Let's see some basic ones:

- `/aliases` -- Define one or more alias properties. For example: `serial0 = "/simple-bus@fe000000/serial@llc500";`.
- `/memory` -- Describes the physical memory layout for the system. If a system has multiple ranges of memory, multiple memory nodes can be created, or the ranges can be specified in the `reg` property of a single memory node.
- `/chosen` -- does not represent a real device in the system but describes parameters chosen or specified by the system firmware at runtime. For example, kernel boot arguments: `bootargs = "root=/dev/nfs rw nfsroot=192.168.1.1 console=ttyS0,115200";`
- `/cpus/cpu*` -- Represents a hardware execution block that is sufficiently independent that it is capable of running OS.

##### 2.1.1.5. Flattened DeviceTree (DTB) format

![DeviceTree DTB Structure](assets/img/devicetree_dtb_structure.png)

The Device Tree header fields:

```c
struct fdt_header { 
  uint32_t magic;
  uint32_t totalsize;
  uint32_t off_dt_struct;
  uint32_t off_dt_strings;
  uint32_t off_mem_rsvmap;
  uint32_t version;
  uint32_t last_comp_version;
  uint32_t boot_cpuid_phys;
  uint32_t size_dt_strings;
  uint32_t size_dt_struct;
};
```

##### 2.1.1.6. Linux and the DeviceTree

#### 2.1.2. ACPI

### 2.2. Bus enumeration

Peripherals are connected to the processor via a Bus. Some buses have supported device enumeration (discovery). The Bus drivers detect new devices automatically.

## 3. Hardware interaction

Assume that the kernel identified its devices, now, how can the kernel communicate with them?

### 3.1. Memory-mapped I/O

### 3.2. Port-mapped I/O

### 3.3. Interrupt

### 3.4. Polling

### 3.5. DMA
