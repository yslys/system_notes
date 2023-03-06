## Motivation
While doing research, I have to simulate a PCIe device, and add it to the riscv board. However, mainline gem5 only has one PCIe device, named CopyEngine, and there is no existing configuration (the riscv board configuration) that has such PCIe device connected. Therefore, I need to figure it out myself. This file has two purposes:
1. Introduction and explanation of PCI-e devices.
2. How to add a PCIe device to gem5.

## Introduction and Explanation of PCIe devices
Paper: [Simulating PCI-Express Interconnect for Future
System Exploration](https://par.nsf.gov/servlets/purl/10078086)

+ What is PCIe?
    + PCI-e interconnect: a interconnection technology **within a single computer node** that connects off-chip devices (NICs, accelerators like GPUs, storage devices like SSDs) to the processor chip. 

+ PCIe - the bottleneck of I/O performance
    + PCI-e bandwidth and latency are often the bottleneck in the processor, memory and device interactions and impacts the overall performance of the connected devices.
    + The I/O performance - important in the performance and power efficiency of such modern systems

+ Types of I/O devices
    + NIC, GPU, SSD, NVM, other accelerators, etc.

+ PCI and PCI-e?
    + PCI bus: is shared by several devices
    + PCI-e: 
        + provides a virtual point-to-point connection between a device and a processor,
        + enabling the **processor to simultaneously communicate with multiple devices**.
        + Higher bandwidth between processor and device.

Now, let's take a deep look into the difference between PCI bus and PCIe.

### PCI bus
PCI is a parallel bus (Bus 0, Bus 1, Bus 2, ...) interconnect where multiple I/O devices (endpoints) share a single bus, clocked at 33 or 66 MHz. Different PCI buses are connected together via PCI Bridges to form a hierarchy. Each device on a PCI Bus is capable of acting as a bus master, and thus DMA transactions are possible without the need for a dedicated DMA controller.
+ Each PCI Bus can support up to 32 devices. 
+ Each device is identified by a bus, device and function number. 
+ A PCI device maps its configuration registers into a configuration space, that is an address space exposing the PCI specific registers of the device to enumeration software and the corresponding driver.
+ Enumeration software is a part of BIOS and operating system kernel that attempts to detect the devices installed on each PCI Bus (slot), by reading their **Vendor ID** and **Device ID** configuration registers. The enumeration software performs a depth-first search to discover devices on all the PCI buses in the system. This allows a device driver to identify its corresponding device and communicate with it through a memory mapped I/O.

A PCI bus does not support split transactions.



### PCI address space
https://www.youtube.com/watch?v=ihgMcP2353I

1. PCI configuration space
2. PCI memory-mapped space
3. PCI I/O-mapped space


https://www.youtube.com/playlist?list=PLBTQvUDSl81dTG_5Uk2mycxZihfeAYTRm






+ What was gem5 and what is gem5 after this paper?
    + Previously: all devices are connected through a xBar to the processor chip.
    + Now: we have a detailed off-chip interconnection model, which models the interactions between processor and off-chip devices.


