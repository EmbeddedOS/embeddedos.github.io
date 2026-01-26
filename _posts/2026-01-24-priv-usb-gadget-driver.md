---
title: "USB gadget drivers"
description: >-
  Master USB gadget to make your system become whatever USB device you want.

author: Cong
date: 2025-11-15 00:01:00 +0800
categories: [kernel, usb-gadget]
tags: [linux, kernel, usb, gadget, drivers]
image:
  path: assets/img/invisible_process.png
  alt: USB gadget driver
published: false
---

The USB controller found in most of our PCs can only act as hosts on a USB bus. But on embedded devices, the USB controller can act as a host, as a device, or as both.

Acting like a device is called "USB gadget" by Linux. Several USB gadget drivers that are alreadly available:

- Ethernet.
- Serial.
- Mass storage.
- MIDI.
- Printer.

## Basic communication flow

1. Plug & Detect: When USB device is plugged, the USB Host Controller (UHC) sends an interrupt to kernel - USB core, alerting it of new device connection.

2. Enumration: Who are you? Linux USB core sends **Standard Control Requests** to the device to gather identity data:
  - Vendor ID.
  - Product ID.
  - Device Class/Subclass.
  - Configurations & Endpoints.

3. Device matching: Based on those info, Linux match VID, PID and CLASS to the driver. Driver `probe()` that device.

4. Load the driver, and init user interfaces: `/dev/sdb`, `/dev/ttyACMx`, etc.

5. User space start talking.

## USB info

### classes

USB class and subclass: [USB-IF](https://www.usb.org/defined-class-codes)

Basic class:

- `0x01`: Audio.
- ``

### USB structure

USB devices are structured hierarchically, comprising configurations, interfaces, and endpoints to manage data transfer.

Hierarchy: Device -> Configuration(s) -> Interface(s) -> Endpoint(s).

- A USB Configuration (Configuration Descriptor): Define power consumption (mA) and groups interfaces. One device can have multiple configurations but only one active at a time.
- USB interface: Defines a specific function of the device.


NOTE: The host always init the communication, for example: in case of networking, host polls for an endpoint to check any data IN, if any, it requests to get the data.

```mermaid
sequenceDiagram
    autonumber

    participant HostApp as Host App<br/>(ping/ssh/curl)
    participant HostNet as Host Network Stack
    participant HostDrv as USB Net Driver<br/>(cdc_ncm / rndis_host)
    participant HCD as USB Host Controller<br/>(xhci-hcd)
    participant Cable as USB Cable
    participant UDC as UDC HW<br/>(3550000.usb)
    participant UDCdrv as UDC Driver<br/>(tegra-xudc)
    participant Gadget as Gadget Core<br/>(usb_composite)
    participant Func as Ethernet Function<br/>(f_ncm / f_rndis)
    participant DevNet as Device Network Stack
    participant DevApp as Device App<br/>(sshd/httpd)

    %% Enumeration
    HostDrv->>HCD: Detect device attach
    HCD->>UDC: USB reset
    HostDrv->>UDC: GET_DESCRIPTOR
    UDC->>HostDrv: Device + Config descriptors
    HostDrv->>HCD: SET_CONFIGURATION
    HCD->>UDC: Activate interfaces/endpoints
    UDC->>Func: set_alt(), enable endpoints

    %% RX path (Host -> Device)
    HostApp->>HostNet: Send packet (ping)
    HostNet->>HostDrv: skb -> USB frame
    HostDrv->>HCD: Submit OUT URB
    HCD->>Cable: USB OUT transaction
    Cable->>UDC: USB packet
    UDC->>UDCdrv: Interrupt / DMA complete
    UDCdrv->>Func: usb_request->complete()
    Func->>DevNet: skb -> netif_rx()
    DevNet->>DevApp: Packet delivered

    %% TX path (Device -> Host)
    DevApp->>DevNet: Send reply packet
    DevNet->>Func: ndo_start_xmit(skb)
    Func->>UDCdrv: Queue IN usb_request
    UDCdrv->>UDC: Prepare IN data
    HCD->>UDC: IN token (poll)
    UDC->>Cable: Send data
    Cable->>HCD: USB IN data
    HCD->>HostDrv: URB complete
    HostDrv->>HostNet: skb received
    HostNet->>HostApp: Reply delivered
```
