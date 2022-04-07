# Research Overview


## System Overview
```
+-----+       +-----+       +-----+       +-----+
| CPU |       |Accel|       |Accel|  ...  |Accel|
|(MMU)|       |(Mem)|       |(Mem)|       |(Mem)|
+-----+       +-----+       +-----+       +-----+
   |             |             |             |
+-----+       +---------------------------------+
|cache|       |            PCIe Bus             |
+-----+       +---------------------------------+
   |                           |
   |                        +-----+ 
   |                        |IOMMU|
   |                        +-----+
   |                           |
+-----------------------------------------------+
|                   Memory (TB)                 |
+-----------------------------------------------+   
```

## Current Technologies for near-memory accelerators

### 1. DMA from CPU ([2006](https://01.org/blogs/ashokraj/2018/recent-enhancements-intel-virtualization-technology-directed-i/o-intel-vt-d))
+ The work (DMA transfer) is scheduled per chunk by CPU.
+ No overlap of data movement and computation
    + 
    ```
        |data-mov|computation|data-mov|computation|...|
    ```
During the data-movement, Accelerators are idle.

### 2. SVM/SVA/IOMMU (2013)
Shared Virtual Memory (in PCIe); Shared Virtual Addressing; IOMMU. See [Appendix](#appendix) for more details.

+ CPU scheduled work.
+ Data movement and computation can overlap.
    + 
    ```
        |data-mov/computation|data-mov/computation|...|
    ```
+ SVA allows processor and device to use the same virtual addresses avoiding the need for software to translate virtual addresses to physical addresses.
    + User space: initialize a work descriptor, fill in the virtual address of the data to be operated on, then submit to the shared work queue. 
    + Device: will get the work descriptor, with the same virtual address in user space. When the device tries to a read from memory, the device will first do an ATS lookup (check VPN, PFN pairs stored in the device TLB cache), if cache miss, then the VA will be sent to PRI (page request interface) to do a page table walk. If not found, then generates a page fault, which might be batched and issued with a delay.


The overhead of IOMMU is significant, which becomes severe when translation cache miss, page table miss, which causes page fault.

The page fault overhead of IOMMU is worse than that of CPU, since IOMMU tends to batch the page faults until it reaches the watermark, then the page fault will be issued to CPU. CPU then will handle the page fault to bring the corresponding data into memory, build page table, and return the result to IOMMU.




## Our solution
+ Vectorized contiguous memory management
+ Shared Work Queues in memory (shared between multiple users and one device)
    + Our device driver should allocate PASID for each process's task descriptor before submitted to the shared queue.
    + Users submit work descriptor to kernel, which then assigned a PASID and enqueued to the queue.
    + Work descriptor contains the VA of the data to be transferred (read by the device via DMA).
+ DVM to reduce IOMMU translation overhead


## Evaluation and Comparison
In order to communicate with the device, IOMMU is a must. Hence, we cannot let device do computation without IOMMU.

Define a work to be the amount of computation performed by the device at a time. For instance, computing the sum of an array.

+ Current design of interface:
    + What is contained in the work descriptor?
        + length of the array
        + starting DMA address of the array (used by the device; when device tries to a DMA read, it reads from DMA address, which will be passed to IOMMU to do the address translation to obtain the corresponding PA)
        + Question: in the current design of the device, it does not **explicitly** use IOMMU. So what it is that translates the DMA address to the physical address?
        + Question: the above uses DMA address, while SVA uses user VA and translates it into PA using IOMMU. I think our current design is not correct.

+ Baseline:
    + Traditional IOMMU approach with DMA reading from memory to device.

+ Relative:
    + DVM supported IOMMU with 0 translation overhead for IOMMU, with DMA reading from memory to device.
    + This is approximate to the ideal case.


## TODO:
+ Integrate DVM to reduce address translation overhead.
+ Integrate database workload to use our interface.
    + Also need to define "how the workload uses our device" or "how device performs computation".
+ Measure the baseline, compared it to the ideal case where accelerator is fully utilized to see the performance difference.
+ (Maybe) implement ENQCMD command.
+ Think about how to make our design more exciting.

## Previous work - Could be added as reference in our paper
+ [Supporting Address Translation for Accelerator-Centric Architectures](http://web.cs.ucla.edu/~haoyc/pdf/hpca17.pdf)
+ [Virtualizing I/O through the IOMMU](https://pages.cs.wisc.edu/~basu/isca_iommu_tutorial/IOMMU_TUTORIAL_ASPLOS_2016.pdf)


### Useful Links
+ [Mastering the DMA and IOMMU APIs](https://elinux.org/images/3/32/Pinchart--mastering_the_dma_and_iommu_apis.pdf) and [video](https://www.youtube.com/watch?v=n07zPcbdX_w)
+ [Dynamic DMA mapping using the generic device](https://www.kernel.org/doc/Documentation/DMA-API.txt)




## Appendix

### SVA
Below is a brief description of SVA:

#### Background
+ SVA: Shared Virtual Addressing
    + Allow processor and the device to use the same virtual addresses to avoid the need for software to translate VA to PA.
    + The device can directly use the virtual address of user mode applications because IOMMU will handle the address translation (VA -> PA).
    + Not requiring pinning pages for DMA.

+ Three important elements for supporting SVA:
    + PCIe Address Translation Services (ATS)
        + Allows devices to cache translations for virtual addresses.
    + PCIe Page Request Interface (PRI).
        + On an ATS lookup failure, the device uses PRI to request the VA to be paged into CPU page tables.
        + The device must use ATS again to fetch the translation before use.
    + The two elements above allow devices to function much the same way as CPU handling application page faults.
    + IOMMU is required to support the above two features.
        + IOMMU driver uses ```mmu_notifier()``` support to keep the device TLB cache and CPU cache in sync.

#### Shared **Hardware** Workqueues
Some terms:
+ SR-IOV: Single Root I/O Virtualization
+ SIOV: Scalable IOV
+ SWQ: Shared Work Queues
+ PASID: Process Address Space ID, 20-bit number defined by PCI3 SIG (special interest group).
+ RID: PCIe Resource Identifier (Bus/Device/Function)

SIOV permits the use of SWQ by both applications and virtual machines.
    + This allows better hardware utilization vs. hard partitioning resources that can result in under utilization.
    + SIOV uses PASID to let hardware distinguish different work being executed in hardware by SWQ interface.

PASID
    + Allows IOMMU to track I/O on a per-PASID granularity.
    + IOMMU also uses the PCIe Resource Identifier to track I/O.
    + Kernel must allocate a PASID on behalf of each process which will use ENQCMD, and program it into the new MSR (Model-Specific Register, control register) to communicate the process identity to platform hardware.

#### ENQCMD
Atomically submits a work descriptor to a device. Work descriptor includes:
+ operation to be performed
+ virtual addresses of all parameters
+ virtual address of a completion record
+ PASID of the current process.

Non-posted semantics:
+ Requires that a bus transaction responds with a completion response to indicate success or failure of the transaction, and is naturally much slower than posted since it requires a round trip delay similar to read bus transactions.
+ ENQCMD works with non-posted semantics.
+ ENQCMD carries a status back if the command was accepted by hardware. So that submitter can know whether retry is needed.

ENQCMD is the glue that ensures applications submit commands to hardware.
+ Kernel allocates a PASID on behalf of each process that will use ENQCMD, and program the PASID into the new MSR (Model-Specific Register) to communicate the process identity to platform hardware.
+ ENQCMD uses the PASID stored in this MSR to tag requests from this process. 

When a user submits a work descriptor to a device using the ENQCMD: 
+ The PASID field in the descriptor is auto-filled with the value from MSR_IA32_PASID.
+ Requests for DMA from the device are also tagged with the same PASID. 
+ The platform IOMMU uses the PASID in the transaction to perform address translation. 
+ The IOMMU APIs setup the corresponding PASID entry in IOMMU with the process address used by the CPU (e.g. %cr3 register in x86).

The MSR must be configured on each logical CPU before any application thread can interact with a device. Threads that belong to the same process share the same page tables, thus the same MSR value.

#### How are shared work queues different?
Traditionally, in order for userspace applications to interact with hardware, there is a separate hardware instance required per process. 

For example, consider doorbells as a mechanism of informing hardware about work to process. Each doorbell is required to be spaced 4k (or page-size) apart for process isolation. This requires hardware to provision that space and reserve it in MMIO. This doesn't scale as the number of threads becomes quite large. 

The hardware also manages the queue depth for Shared Work Queues (SWQ), and consumers don't need to track queue depth. If there is no space to accept a command, the device will return an error indicating retry.

A user should check Deferrable Memory Write (DMWr) capability on the device and only submits ENQCMD when the device supports it. In the new DMWr PCIe terminology, devices need to support DMWr completer capability. In addition, it requires all switch ports to support DMWr routing and must be enabled by the PCIe subsystem, much like how PCIe atomic operations are managed for instance.

SWQ allows hardware to provision just a single address in the device. When used with ENQCMD to submit work, the device can distinguish the process submitting the work since it will include the PASID assigned to that process. This helps the device scale to a large number of processes.


### dma_alloc_coherent
```dma_alloc_coherent()``` calls ```dma_alloc_attrs()```, which may call different functions