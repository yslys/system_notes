## Useful links:
+ [Lecture of Virtual Machines by UCSD](http://www.cs.cmu.edu/~dga/15-440/F10/lectures/vm-ucsd.pdf)
+ [CSE 291 Virtualization @UCSD](https://cseweb.ucsd.edu/~yiying/cse291j-winter20/reading/)
+ [Lecture of Virtualizing Memory](https://cseweb.ucsd.edu/~yiying/cse291j-winter20/reading/Virtualize-Memory.pdf)
+ [Nested page tables](https://www.anandtech.com/show/2480/10)
+ [Virtualization and Memory Management on YouTube](https://www.youtube.com/watch?v=e6Zc4hcJjAY)
+ [I/O virtualization 1](https://www.youtube.com/watch?v=2yhcITe_xa0)
+ [I/O virtualization 2](https://www.youtube.com/watch?v=gwMrdCONERo)
+ [Intel® Virtualization Technology for Directed I/O (VT-d): Enhancing Intel platforms for efficient virtualization of I/O devices](https://software.intel.com/content/www/us/en/develop/articles/intel-virtualization-technology-for-directed-io-vt-d-enhancing-intel-platforms-for-efficient-virtualization-of-io-devices.html)
## Memory virtualization
Several terms:

+ Guest OS: on a machine, we could install multiple OSs, which are called guest OSs.
+ Guest virtual addr: the virtual addr used by the guest OS
+ Guest physical addr: the physical addr used by the guest OS (not the actual physical address on Host)
+ Host OS: the OS running on the hardware
+ Host physical addr: the actual physical addr of the memory

#### What on earth is Memory Virtualization?

It is mapping the guest virtual addr to the guest physical addr, and mapping the guest physical addr to host physical addr

#### Two ways of memory virtualization

Two approaches - software approach; hardware approach.

+ Software approach - **hypervisor**
	+ Shadow Paging (two tables)
		+ details: use a shadow page table. First table: (GVA -> GPA), and the shadow of the previous page table - (GPA -> HPA)
		+ Use the table (GVA -> GPA) to prevent guest OS directly access the hardware PT.
+ Hardware approach
	+ Nested Page Table (AMD processors)
	+ Extended Page Table (Intel processors)
	+ details: 
		+ For the guest OS, there's one page table (GVA -> GPA), then that page table is linked to another page table (GPA -> HPA), maintained by the virtual machine.
		+ One more level of virtualization (one more virtual machine on the virtual machine, which could be viewed as the virtual host machine), 

+ What's the difference?
	+ software approach - the page tables are maintained by the software - hypervisor
	+ hardware approach - the page tables are maintained by the virtual machine itself.

## Memory Reclaimation
Several terms:

+ Overcommitment
	+ Example: 32 GB of physical memory on host, 3 VMs with each having 16 GB (48 in total)
+ Deduplication
	+ solution to overcommitment
	+ optimization technique
	+ detail: share the pages that have the same content in different VMs
+ Ballooning
	+ Use case: shift some of the pages acquired by VM1 to VM2
		+ the pages shifted are NOT important to VM1
		+ managed by hypervisor
	+ Problem: how could hypervisor know which pages are NOT important to VM1?
	+ Solution: Ballooning
	+ Details:
		+ Use a balloon module that attached to each VM (one balloon per VM)
		+ For any VM, if more pages are assigned to it, then its balloon will inflate (充气，膨胀)
		+ ...............less pages...........................................deflate
	+ Does the hypervisor decide which page is NOT important?
		+ No. It is the VM (Guest OS) itself to decide which pages are NOT important - it will deallocate such pages, wuch deallocated pages will be shifted by the hypervisor to another VM.

## I/O Virtualization

#### How it works
1. After installation of the Guest OS, it will probe for all the available I/O devices.
2. The probing will **trap the hypervisor**, then hypervisor will report back all the available I/O devices (also in disk spaces).
3. The Guest OS will load device drivers and try to use such available I/O devices.

+ Before the problem, we need to know something more:
	+ In the computer, the hard disk is divided into different partitions, and there is a partition for I/O devices. 

+ Problem:
	+ After hypervisor reports all available disk partition of I/O devices, the Guest OS thinks it owns the entire partition. But in reality, there might be more VMs sharing such partition.

+ Solution:
	+ The hypervisor creates one **file** on the disk partition for each VM, and makes such VM believe that the entire partition is owned by itself (the VM).
	+ It is the job of the hypervisor to translate the commands from the Guest OS s.t. they are suitable for running on the actual disk partition.

#### DMA
It is like a co-processor that performs the input/output on behalf of a processor. So the processor won't need to intervene to perform I/O for the computer.

Whenever a processor wants to do some I/O, it passes the physical address to DMA, and then DMA goes to the physical address space to fetch the data and sends it back to some input or output device.

+ Problem: 
	+ Multiple VMs, but only one DMA. DMA deals with the **actual** physical address, but each VM has its own **virtual** physical addresses.
	+ We need a mapping between actual PA of DMA and virtual PA of VM.
+ Solution:
	+ IOMMU

#### IOMMU
I/O MMU uses page tables to map memory addresses that a device wants to use with physical addresses that DMA uses. [Picture](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit)

+ Device Isolation: The hypervisor sets the page table in VM, in such way, the device performing DMA does not interfere with the memory that does not belong to that VM. 

+ Device Pass-through: IOMMU allows physical device to be directly assigned to a VM.

+ IOMMU ensures the interrupt generated by device reach the corresponding VM with the corresponding interrupt number.
	+ Example: a printer generates an interrupt, while there are more than one VMs running on the host OS. IOMMU will ensure that such interrupt directly notify the corresponding VM with the correct interrupt nr.

#### Device Domains
Use one specific virtual machine to handle I/O - such VM is the device domain of I/O device.

All Guest OSs can direct their I/O calls (by sending requests) to that VM (to the Device Domain). How? 
+ The hypervisor issues a command that will tell the Device Domain what the guest OS wants
+ and then the Device Domain will do the thing for the Guest OS.

Now, let's see an example: SR-IOV

#### Direct I/O: Single Root I/O Virtualization (SR-IOV in network devices)
Virtualize a single device to trick every VM into believing that it has exclusive access to its own device.

Instead of creating multiple copies of the hardware of the host OS, it creates multiple virtual copies of I/O devices. It could then allocate such copies of I/O devices to different VMs, and those VMs believe that they have exclusive access to its own device.

![](https://github.com/yslys/OS_IMS/blob/main/SR-IOV.png)

+ Problem: 
	+ when guest device driver provides DMA buffers to VF, it can only provide guest physical address of the buffer
	+ NIC cannot access the DMA buffer memory using guest physical address alone
+ Solution:
	+ IOMMU - Guest PA -> Host PA; without needing to go to the host OS stack.

Now, we could start looking at the paper.


## Outline

+ Direct I/O
	+ **Advantage**: the best performance **I/O virtualization** method
	+ **Problem**: requires hypervisor to statically pin the entire [guest memory](https://www.vmware.com/content/dam/digitalmarketing/vmware/en/pdf/techpaper/perf-vsphere-memory_management.pdf) (refers to the memory that is visible to the guest OS)
	+ **Effect**: hinders efficiency of memory management
	+ **Solution**: virtual IOMMU (vIOMMU)

+ vIOMMU
	+ **Advantage**: emulation of its DMA remapping capability carries sufficient info about guest DMA buffers, allowing the hypervisor to do fine-grained pinning of guest memory
	+ **Problem**: emulation cost too high
	+ **Effect**: cannot reliably eliminate static pinning in direct I/O
	+ **Solution**: coIOMMU - (vIOMMU + cooperative DMA buffer tracking mechanism)

+ coIOMMU
	+ a new vIOMMU architecture for efficient memory management with a cooperative DMA buffer tracking mechanism. 
	+ **cooperative DMA buffer tracking**:
		+ a dedicated interface for hypervisor and guest to **exchange DMA buffer info** over a **shared DMA tracking table (DTT)**.
			+ orthogonal to the costly DMA remapping interface
		+ smart pinning
		+ lazy pinning
		+ (to minimize the impact on the performance of direct I/O)
	+ **Result**: dramatically improves the efficiency of memory management in wide direct I/O usages with negligible cost. Moreover, the desired semantics of DMA remapping can be sustained when
cooperative tracking is enabled alongside.


## Introduction
#### Direct I/O 
+ the best performance [**I/O virtualization**](https://gcallah.github.io/OperatingSystems/VirtualIO.html) method
+ cornerstone capability in data centers and clouds
+ guest could directly interact with I/O devices without the intervention from software intermediary.
+ **Problem**: requires hypervisor to statically pin the entire guest memory
+ **Effect**: hinders efficiency of memory management
+ **Solution**: virtual IOMMU

#### [IOMMU](https://terenceli.github.io/%E6%8A%80%E6%9C%AF/2019/08/04/iommu-introduction)
+ prevents DMA attacks in Direct I/O by providing the **DMA remapping**.
	+ DMA-remapping translates the address of the incoming DMA request to the correct physical memory address and perform checks for permissions to access that physical address, based on the information provided by the system software.
+ Each assigned device is associated with an IOMMU page table (IOPT), configured by the hypervisor in a way that **only the memory of the guest that owns the device is mapped**.
+ IOMMU walks the IOPTs to validate and translate DMA requests
	+ achieving inter-guest protection among directly assigned devices.
+ **Limitation:** Most devices do not tolerate DMA faults - guest buffers must be pinned in host memory and mapped in IOPT before they are accessed by DMAs. 

#### Problem with IOMMU
But, the hypervisor does not know which pages are mapped by the guest when it is eliminated from the direct I/O path. Hence, it has to pin the entire guest memory upfront, i.e. a static pinning, which is **coarse-grained pinning**.
- **Problem:** This heavily hinders the efficiency of memory management and worsens memory utilization.
- **Reason:** Because pinned pages cannot be reclaimed for other purposes.
- **Solution:** vIOMMU

#### Virtual IOMMU (slow, but secure)
+ providing vIOMMU to the guest 
	+ allows fine-grained pinning of guest memory for efficient memory management
	+ primary purpose: help the guest protect itself against buggy drivers

+ Remember IOMMU provides **DMA remapping** to prevent DMA attacks?
	+ The hypervisor emulates the DMA remapping interface by:
		+ i) walking the virtual IO page table => identify the affected buffers
		+ ii) pinning and unpinning the buffers in the host memory
		+ iii) mapping and unmapping them in the physical IOMMU to enforce protection as desired by the guest.
	+ Naturally, the emulation leads to a fine-grained pinning scheme if the guest always uses the virtual IOMMU to remap its DMA buffers.

#### Problem with vIOMMU
It is not a reliable solution for fine-grained pinning, only used in limited circumstances.
+ Virtual DMA remapping are disabled by most guests in public cloud.
+ **Reason:** Because significant emulation cost may be incurred due to **frequent mapping operations in the guest**. Such cost could be **alleviated** through side-core emulation, or para-virtualized extension. (Well, they still have some new problems, see below):
	+ Side-core emulation (security issues)
		+ requires an additional CPU core to perform the emulation
		+ can only achieve optimal performance with deferred IOTLB invalidation => compromised security.
	+ Para-virtualized extension (performance issues)
		+ reduces the virtualization overhead with optimized interfaces
		+ but still involves large number of VM-exits at the time of guest DMA mappings/unmappings => limiting the performance.

Therefore, vIOMMU is only used when intra-guest protection is valued over the overhead of DMA remapping. (slow, but secure)

#### What the authors propose
Mixing the requirements of protection (security) and pinning (performance) through the same costly DMA remapping interface, is needlessly constraining. 
- Protection is a guest requirement.
- Pinning is for host memory management. 
The two do not always match, thus favoring one may easily break the other. Instead, they aim to provide a reliable solution for fine-grained pinning by decoupling it from protection.

#### Here comes coIOMMU
+ A new vIOMMU architecture
+ Helps hypervisor achieve efficient memory management in direct I/O.
+ A mechanism for cooperative DMA buffer tracking, orthogonal to the costly DMA remapping interface
+ Allows the hypervisor and guest to communicate over a DMA tracking table (DTT) located in a shared memory region.
	+ Guest: records the mapping status of its DMA buffers in the DTT.
	+ Hypervisor: walks the DTT to identify the corresponding pinning requirement.
+ Minimizes the number of notifications from the guest, with two optimizations:
	+ smart pinning, which heuristically pins frequently used pages and timely shares its pinning status with the guest, to enable precise notification in guest-mapping operations.
	+ lazy unpinning, which asynchronously unpins guest pages to eliminate notifications in guest-unmapping operations.

In conclusion: the new mechanism does not affect the desired semantics of DMA remapping. It can be enabled with or without DMA remapping, as a reliable and standard **interface** to achieve fine-grained pinning in direct I/O.
