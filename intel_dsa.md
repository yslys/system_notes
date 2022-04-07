# Intel Data Streaming Accelerator Architecture Specification

## Table of contents
+ [Keywords](#keywords)
+ [Overview](#2-overview)
    + [Goal](#goal)
    + [Infrastructure Features](#infrastructure-features)
    + [Data and Control Operations](#data-operations-and-control-operations)
+ [Intel Data Streaming Accelerator Architecture](#3-intel-data-streaming-accelerator-architecture)
    + [Intro](#intro)
    + [Register and Software Programming Interface](#register-and-software-programming-interface)
    + [Descriptors (work/batch descriptors)](#descriptors)
    + [Work Queues](#work-queues)
    + [Shared Work Queue](#shared-work-queue-swq)

## Keywords
+ SoC: System on a Chip
+ Root Complex: In a PCI Express (PCIe) system, a root complex device connects the CPU and memory subsystem to the PCI Express switch fabric composed of one or more PCIe or PCI devices.
+ DMA Remapping hardware unit: IOMMU
+ ATC: Address Translation Cache
+ QoS: Quality of Service
+ IOV: IO Virtualization
+ TLP: Transport Layer Packets for PCIe


## 2 Overview
### Goal
The goal of DSA is to provide **higher performance for data mover and transformation operations, while freeing up CPU cycles for higher level functions**. 

Intel DSA hardware supports highperformance data mover capability to/from volatile memory, persistent memory, memory-mapped I/O, and through a Non-Transparent Bridge (NTB) in the SoC to/from remote volatile and persistent memory on another node in a cluster. 

It provides a **PCI Express** compatible programming interface to the Operating System and can be controlled through a device driver.

### Infrastructure Features
+ Shared Virtual Memory (SVM): 
    + allows user level applications to submit commands to the device directly, with **virtual addresses** in the descriptors. It supports translating virtual addresses to physical addresses using IOMMU including handling page faults. The virtual address ranges referenced by a descriptor may span multiple pages. **Intel DSA also supports the use of physical addresses, as long as each data buffer specified in the descriptor is contiguous in physical memory**.

+ Partial descriptor completion: 
    + With SVM, an operation may encounter a page fault during address translation. Software can control whether the device is to continue processing after waiting for resolution of a page fault or terminate processing of a descriptor that encounters a page fault and proceed to the next descriptor. If processing of a descriptor is terminated, the completion record indicates to software the amount of work completed and information about the page fault so that software can resolve the fault and restart the operation from the point where it stopped.

+ Block on fault: 
    + As an alternative to Partial descriptor completion, when the device encounters a page fault it can coordinate with system software to resolve the fault and continue the operation transparently to the software that submitted the descriptor.

+ Batch processing: 
    + A Batch descriptor points to an array of work descriptors (i.e., descriptors with actual data operations). When processing a batch descriptor, the device fetches the work descriptors from the specified virtual memory address and processes them.

+ Stateless device: 
    + Descriptors are designed so that all information required for processing the descriptor comes in the descriptor itself. This allows the device to store little client specific state which improves its scalability. The only exception is the completion interrupt message, when used, because it must be configured by trusted software.

+ Cache allocation control: (?)
    + This allows applications to specify whether output data is allocated in the cache or is sent to memory without cache allocation. Completion records are always allocated in the cache.

+ Shared Work Queue (SWQ) support: (?) 
    + Shared Work Queues (SWQ) enable scalable work submission using Deferrable Memory Write transactions, which indicate whether the work was accepted into the WQ.
    + Work Queue as a SWQ can be shared by multiple software components.

+ Dedicated Work Queue (DWQ) support: 
    + Dedicated Work Queues (DWQ) enable high-throughput work submission using **64-byte** Memory Write transactions.
    + Work Queue as a DWQ is assigned to a single a single software component at a time.

+ QoS support:
    + Intel DSA supports several features that allow the kernel driver to separately control access to device resources by different guests and applications.
    + In the field of computer networking and other packet-switched telecommunication networks, quality of service refers to traffic prioritization and resource reservation control mechanisms rather than the achieved service quality. Quality of service is the ability to provide different priorities to different applications, users, or data flows, or to guarantee a certain level of performance to a data flow.

+ Intel® Scalable IOV support: 
    + Intel Scalable IO Virtualization improves scalability of device assignment, allowing a VMM to share the device across many more VMs than would be possible using SR-IOV.
    + DSA architecture is designed to support Intel Scalable I/O Virtualization. The device can be shared directly with multiple VMs in a secure and isolated manner to achieve high throughput.

+ Persistent Memory features: 
    + Configuration registers and descriptor flags allow software to indicate writes to durable memory (persistent memory) and specify the durability and ordering semantics to the SoC.

### Data Operations and Control Operations
Section 2.2.2 & 2.2.3

## 3 Intel Data Streaming Accelerator Architecture

### Intro
From large to small:
+ Multi-socket server platform can support multiple SoCs.
+ Per SoC, it supports 1 or more DSA device instances.
+ Each instance 
    + is exposed as a single [Root Complex Integrated Endpoint](https://en.wikipedia.org/wiki/Root_complex)
    + is under the scope of a DMA Remapping hardware unit (IOMMU).
+ 1 IOMMU for 1 or more device instance(s)

Address Translation and IOMMU:
+ DSA has an Address Translation Cache(ATC).
    + ATC interacts with DMA Remapping hardware using 
        + PCI-SIG-defined Address Translation Services (ATS), 
        + Process Address Space ID (PASID), and 
        + Page Request Services (PRS) capabilities.
+ PASID TLP prefix
    + added to upstream requests to support both Shared Virtual Memory (SVM) and Intel Scalable I/O Virtualization. The device utilizes the DMA Remapping hardware (IOMMU) to **translate DMA addresses to host physical addresses**. Depending on the usage, a DMA address can be:
        + a Host Virtual Address (HVA), 
        + Guest Virtual Address (GVA), 
        + Guest Physical Address (GPA), or 
        + I/O Virtual Address (IOVA). 

+ Intel DSA supports additional PCI Express capabilities, including Advanced Error Reporting (AER) and MSI-X.

+ DSA device at a conceptual level: Figure 3-1.
    + Downstream work requests from clients are received on the I/O fabric interface.
    + Upstream read, write, and address translation operations are sent on that interface.
    + The device includes: 
        + configuration registers 
        + Work Queues (WQ) to hold descriptors submitted by software
            + Two types of descriptors: batch descriptor (bd), work descriptor(wd)
                + If it is batch descriptor, it will be sent to the batch processing unit.
                + The batch processing unit processes Batch descriptors by reading the array of descriptors from memory.
                + But in the end, the batch descriptor will still result in several work descriptors and sent to the arbiter.
            + WQ configuration allows software to configure each WQ as either a SWQ or DWQ.
            + WQ configuration allows software to control which WQs feed into which engines and the relative priorities of the WQs feeding each engine.
        + arbiters used to implement QoS and fairness policies
            + arbiter will send the selected **work descriptor** (not batch descriptor) to the work descriptor processing unit.
        + processing engines
        + an address translation and caching interface
            + will be used by both batch processing unit and work descriptor processing unit.
        + a memory read/write interface
            + will be used by both batch processing unit and work descriptor processing unit.
        + Work descriptor processing unit has the following stages: 
            + read memory to get the data
            + perform the requested operation on the data
            + generate output data
            + write output data, completion records, and interrupt messages

### Register and Software Programming Interface
This section discusses how software interacts with DSA - the hardware-exposed registers.

DSA is software compatible with PCIe configuration. User can use memory-mapped I/O registers to check status and control device operation. 
    + capability register 
    + configuration register
    + work submission registers (portals)
are accessible through the MMIO regions defined by the BAR0 and BAR2 registers (section 9.1.1). Each portal is on a separate 4K page so that they may be independently mapped into different address spaces (clients) using CPU page tables.

### Descriptors
Software specifies work for the device using descriptors. Descriptors specify: 
    + the type of operation for the device to perform
    + addresses of data and status buffers
    + immediate operands
    + completion attributes
        + specify the address to write the completion record, and optionally, the information needed to generate a completion interrupt.
    + etc.

See chapter 8 for descriptor formats and details.

DSA does not maintain client specific state on the device, i.e. the descriptor contains all information to process a descriptor. This improves shareability of the device among user-mode applications, as well as among different virtual machines or machine containers in a virtualized system.

+ Work descriptor:
    + contains an operation and associated parameters.
+ Batch descriptor:
    + contains the address of an array of work descriptors.

**Software prepares the descriptor in memory and submits the descriptor to a Work Queue (WQ) of the device.** The device dispatches descriptors from the work queues to the engines for processing. When an engine completes a descriptor or encounters certain faults or errors that result in an abort, it notifies the host software by either writing to a completion record in host memory, issuing an interrupt, or both.

### Work Queues
Work queues (WQs) are **on-device storage** to contain descriptors that have been submitted to the device. 

+ WQ Capability register:
    + stores the number of work queues and the amount of work queue storage available on the device.
    + indicates support for Dedicated and Shared modes.
    + Software configures how many work queues are enabled and divides the available WQ space among the active WQs.

+ Configure the WQs:
    + Use WQ Configuration Table.
    + Can configure the mode of each WQ while the WQ is Disabled.
    + Software configures the size of each WQ before enabling the device. Unused WQs have a size of 0. 
    + Section 9.2.19: configuring WQs, the command register, and enabling Work Queues.

+ Submitting descriptors to WQs:
    + Descriptors are submitted to work queues via **special registers called portals**.
    + **Portal**: the entrance, or gate to enter the WQ. 
    + Each **portal** is in a separate 4 KB page in device MMIO space. 
    + **4 portals** per WQ:
        + Unlimited MSI-X Portal 
        + Limited MSI-X Portal
        + Unlimited IMS Portal
        + Limited IMS Portal
    + We need to know the address of the portal so that we know where to submit a descriptor. It allows the device to determine which WQ to place the descriptor in, whether the portal is limited or unlimited, and which interrupt table to use for the completion interrupt.
    + Section 3.3.1 (SWQ): the usage of limited and unlimited portals.
    + Section 3.7 (Interrupts): the usage of MSI-X and IMS portals
    + No difference between the limited and unlimited portals for Dedicated WQs.
    + No difference between MSI-X or IMS portal for descriptors that do not request interrupt.

IMS portals do not exist if IMS is not supported, so a descriptor written to an address that would normally correspond to an IMS portal is discarded without reporting an error. If the descriptor was submitted with a non-posted write (CPU needs to wait for a write completion response, and in this case, IMS is not supported, so the response will be failure, i.e. Retry), a Retry response is returned.

+ Posted Write:
    + A posted write is a computer bus write transaction that does not wait for a write completion response to indicate success or failure of the write transaction. For a posted write, the CPU assumes that the write cycle will complete with zero wait states, and so doesn't wait for the done. This speeds up writes considerably. For starters, it doesn't have to wait for the done response, but it also allows for better pipelining of the datapath without much performance penalty.
+ Non-posted Write:
    + A non-posted write requires that a bus transaction responds with a write completion response to indicate success or failure of the transaction, and is naturally much slower than a posted write since it requires a round trip delay similar to read bus transactions.


### Shared Work Queue (SWQ)
+ Shared Work Queue 
    + accepts work submission using the PCIe-defined Deferrable Memory Write Request (DMWr). 
    + DMWr is a **64-byte non-posted write** that waits for a response from the device before completing.
    + Device returns Success if the descriptor is accepted into the work queue, or Retry if the descriptor is not accepted due to WQ capacity or QoS. 
        + This allows multiple clients to directly and simultaneously submit descriptors to the same work queue. 
    + On Intel CPUs, DMWr is generated using the ENQCMD or ENQCMDS instructions. 
        + ENQCMD and ENQCMDS instructions return the status of the command submission in EFLAGS.ZF flag; 0 indicates Success, and 1 indicates Retry.

+ Configure a SWQ to reserve some of the WQ capacity
    + by setting the WQ Threshold field in the WQCFG register. 

+ Work submission via limited/unlimited portal:
    + For limited portal:
        + Work submission via limited portal is accepted until the number of descriptors in the SWQ reaches the configured threshold. 
    + For unlimited portal:
        + Work submission via an unlimited portal is accepted unless the SWQ is completely full. 
        + The unlimited portals are intended to be used only by privileged software when a work submission to the corresponding limited portal returns Retry. 
        + User-mode and guest software typically only have access to limited portals.


If DMWr returns Success, the descriptor has been accepted by the device and queued for processing. If DMWr returns Retry, software can try re-submitting the descriptor to the SWQ, or if it was a user-mode client using a limited portal, it can request that the kernel-mode driver submit the descriptor on its behalf using an unlimited portal. This helps avoid denial of service and provide forward progress guarantees. See chapter 7 for more information on software use of the limited and unlimited portals.

+ Identification of Clients:
    + Clients are identified by the device using a 20-bit ID called process address space ID (PASID). 
        + The PASID capability must be enabled to use SWQs. 
        + The PASID is used by the device to look up addresses in the Address Translation Cache and to send address translation or page requests to the IOMMU. 
        + In Shared mode, the PASID for each descriptor is contained in the PASID field of every descriptor. 
            + ENQCMD instruction: copies the PASID of the current thread from the IA32_PASID MSR into the descriptor 
            + ENQCMDS instruction: allows supervisor mode software to copy the PASID into the descriptor. 
    + Refer to section 1.2 in Intel Architecture Instruction Set Extensions Programming Reference for the use of PASID and the ENQCMD and ENQCMDS instructions.


### Dedicated Work Queue (DWQ)
+ Submitting work to a Dedicated Work Queue:
    + Software uses a **64-byte memory write** transaction with **write atomicity**. 
        + This transaction may complete faster than DMWr due to the **posted** nature of the write operation.

+ Software's job:
    + The device depends on software to provide flow control based on the number of slots in the work queue.
    + Software is responsible for 
        + tracking the number of descriptors submitted and completed, 
        + detecting a work queue full condition. 
    + If software erroneously submits a descriptor to a dedicated WQ when there is no space in the work queue, the descriptor is dropped.
        + The error is reported in the Software Error Register.
        + Software Error Register is still a register, but stores values about software errors.

+ Instruction to submit work to DWQ: MOVDIR64B
    + It generates a non-torn 64-byte write.

+ Optional PASID:
    + With dedicated WQs, the use of PASID is optional. 
        + If PCI Express PASID capability is not enabled, PASID is not used. 
        + If the PASID capability is enabled, the WQ PASID Enable field of the WQ Configuration register controls whether PASID is used for each DWQ. 
        Since the MOVDIR64B instruction does not fill in the PASID
as the ENQCMD or ENQCMDS instructions do, the PASID field in the descriptor is ignored. When PASID is
enabled for a DWQ, the device uses the WQ PASID field of the WQ Configuration register to do address translation. The WQ PASID field must be set by the driver before enabling a work queue in dedicated mode.
Although dedicated mode doesn’t support the sharing of a single DWQ by multiple clients, Intel DSA can
be configured to have multiple DWQs and each of the DWQs can be independently assigned to clients.
DWQs can be configured to have the same or different QoS levels.

