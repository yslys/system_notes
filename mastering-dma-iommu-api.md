# Mastering DMA and IOMMU APIs
This file serves as the notes for the talk at ELC (Embedded Linux Conference) 2014 by Laurent Pinchart.

+ [Recording](https://www.youtube.com/watch?v=n07zPcbdX_w)
+ [PPT](https://elinux.org/images/3/32/Pinchart--mastering_the_dma_and_iommu_apis.pdf)

The notes will be based on the PPT, and will be indexed using the page number of the pdf.

The talk will be focusing on DMA mapping. DMA engine is described in documentation ([link](https://www.kernel.org/doc/html/latest/driver-api/dmaengine/index.html)).

## Table of contents
+ [6 Memory Access](#6-memory-access)
+ [Communication between CPU and devices](#communication-between-cpu-and-devices)
    + [From the CPU side:](#from-the-cpu-side)
        + [8 Simple case (20th century with 8-bit microcontrollers)](#8-simple-case-20th-century-with-8-bit-microcontrollers)
        + [10 CPU += Write buffer](#10-cpu--write-buffer)
        + [12 CPU += L1 Cache](#12-cpu--l1-cache)
        + [14 CPU += L2 Cache + Multiple(CPU core + L1 Cache)](#14-cpu--l2-cache--multiplecpu-core--l1-cache)
        + [8, 10, 12, 14](#8-10-12-14)
        + [16 Architecture += Cache coherent interconnect (CCI)](#16-architecture--cache-coherent-interconnect-cci)
        + [Conclusion - from CPU side](#conclusion---from-cpu-side)
    + [From the Device side:](#from-the-device-side)
        + [18 Device += MMU(what we called IOMMU) + cache](#18-device--mmuwhat-we-called-iommu--cache)
        + [19 Real systems today](#19-real-systems-today)
+ [Memory Mappings](#memory-mappings)
    + [Types of memory mappings](#types-of-memory-mappings)
        + [Fully Coherent memory](#22-fully-coherent)
        + [Write Combining](#23-write-combining)
        + [Weakly Ordered](#24-weakly-ordered)
        + [Non-Coherent](#25-non-coherent)
+ [Cache Management](#cache-management)
+ [DMA Mapping API](#dma-mapping-api)
    + [34 linux/dma-mapping.h](#34-linuxdma-mappingh)
    + [37 DMA coherent mapping](#37-dma-coherent-mapping)
        + [dma_alloc_coherent()](#dmaalloccoherent)
        + [dma_alloc_attrs()](#dmaallocattrs)
    + [48 DMA mask](#48-dma-mask)
    + [51 User space mapping]





## 6 Memory Access
DMA is all about memory access when you have several agents in the system that can accept the same piece of memory (physical memory).

+ If you get a piece of memory that is only accessed by the CPU, you do not need the DMA mappings.
+ If you have a piece of memory that is only accessed by the device (private to the device), then you do not need to care about DMA mappings either.
+ But when you have memory that is accessed by multiple agents (devices), then you need to care about DMA mappings.

## Communication between CPU and devices
The CPU writes to memory, and the data written will be read by the device. 

The memory controller is connected to the CPU, memory, and the device, which serves as the interconnect of the three components.

### From the CPU side:

#### 8 Simple case (20th century with 8-bit microcontrollers)
See the figure on page 8. The CPU has to first write to memory through a memory controller, then the device can read from that memory directly. In this case, we do not have cache or write buffers, etc.

#### 10 CPU += Write buffer
See the figure on page 10. The hardware is getting more complicated and **software** has to deal with that complexity. In this case, the first piece of added complexity is a write buffer, which is used by the CPU to buffer writes so that **CPU can continue executing instructions while the writes are slowly flushed to memory**.

Before the device can read from memory, CPU has to make sure that the write buffer is flushed to memory - the data is actually written, so that the device can see the same data. Otherwise the device will not be able to see that.

#### 12 CPU += L1 Cache
CPU needs to flush the data in cache to memory before device can access it.

#### 14 CPU += L2 Cache + Multiple(CPU core + L1 Cache)
With multiple CPU cores, each CPU core has its own private L1 cache. There is also a shared L2 cache. If CPU core0 wants to write to memory, the data might be in two places: core0 L1 cache, and the L2 cache. 

Hence, core0 has to flush data in core0 L1 cache, and L2 cache before device can access it.

#### 8, 10, 12, 14
The software has to make sure the data is in memory before it can tell the device to read from memory.

What we will see next is that the hardware takes over the responsibility so that it simplifies the software complexity.

With Cache coherent interconnect, it becomes the interconnect that bridges CPU, device and memory controller together. The memory controller will connect the cache coherent interconnect and memory together. (See p. 15 for more details of the architecture).

#### 16 Architecture += Cache coherent interconnect (CCI)
In order to deal with cache coherence problems, on multi-core systems, it adds one level between the CPU and the memory controller - the cache coherent interconnect.

The idea of CCI is that the devices' view of the memory will always be coherent with the CPUs' view of the memory. Recall that when CPU writes to memory, the data might be in the cache. 

With CCI, when the device tries to read from memory, it will first go through CCI and CCI will look inside cache to see if there is no piece of data in cache that is more recent than what is available in memory.

Hence, CCI takes over the responsibility so that it simplifies the software complexity.

#### Conclusion - from CPU side
With MMUs, CPU can have a virtual view of the memory, i.e. the addresses are virtual addresses, which will be translated into physical address before accessing memory.

### From the Device side:
What we have discussed above is all from the CPU side (add more cores, more levels of caches). You can also have caches, MMUs on the device side so that **the device can have a virtual view of the memory** as well.

#### 18 Device += MMU(what we called IOMMU) + cache
We can add some complexities to the device as well: the device can also have cache and MMUs (IOMMU). Between the device and Cache coherent interconnect, we add an IOMMU. The CCI is not a must, so it is optional.

We need to program the IOMMU in order for the device to be able to access the memory.

Hence, the CPU can write data to memory, and then the device can read the data from memory directly (if there is CCI).

#### 19 Real systems today
It is more complex in real systems today:
+ Multiple CPU cores that can be grouped in clusters. 
    + CPU contains different levels of caches, with some local caches to the CPU core, and some shared between CPU cores.
+ Multi-layer reordering bus matrix:
    + connects all the pieces together
    + reorder writes and reads to memory - a complex problem.

With a complex system, there is no concept of "before and after", i.e. no sequential view of the system anymore. What we have is the concept of **who can see what data in memory and how to synchronize between them**. So there is lots of complexity in the hardware that we need to deal with.

## Memory mappings
We have physical memory in our system, and that memory is not owned by anyone. We cannot access the physical memory before we give the rights to either the CPU or to the devices to access the memory.

**Giving a view of the memory and giving the rights to access the memory to an agent in the system (CPU, devices, etc.)** is **memory mapping** for that agent.

### Types of memory mappings
There are several types of memory mappings:
+ [Fully Coherent memory](#22-fully-coherent)
+ [Write Combining](#23-write-combining)
+ [Weakly Ordered](#24-weakly-ordered)
+ [Non-Coherent](#25-non-coherent)

#### 22 Fully coherent 
For any agent in the system, when it writes to memory, the write will immediately be visible to all the other agents in the system. The view is coherent, so no need to worry about the case where the data to be written is in cache.

Several ways to achieve fully coherent memory mapping:
+ Use a cache coherent interconnect in hardware.
+ Or we can just disable the cache.

We cannot think about memory being cached or not - caching is not a property of the physical memory. **Caching is a property of the memory mapping - it is a property of how an agent (CPU or devices) sees the memory and accesses the memory.** So when we map the memory to the CPU, a physical page can be accessed using a corresponding virtual address. Later when CPU issues a write to memory, whether the write is cached or not depends on the type of our mapping. In the case of fully coherent memory mapping, we do not need to worry about or handle whether the write is cached or not, since fully coherent memory mapping ensures that the write is immediately visible to all the other agents. Even if the data is cached, when another agent tries to read from memory, CCI will be able to handle it and make sure the data the other agent gets is the latest copy.

Here comes the definition of fully coherent from the PPT (p. 22):
> Coherent (or consistent) memory is memory for which a write by either the device or the processor can immediately be read by the processor or device without having to worry about caching effects.

> Consistent memory can be expensive on some platforms, and the minimum allocation length may be as big as a page.

#### 23 Write Combining
Write combining memory mapping is mostly like fully coherent except that **writes can be buffered**. When an agent (CPU or device) writes data to memory, the data can be buffered, and then later the data will be written to memory in burst or fast access.

This memory type is typically used for (but not restricted to) graphics memory, where we want to draw a frame in a piece of memory, then render that with hardware.

We set a bit in the mapping to indicate that we can combine the writes in a buffer.

Here comes the definition of write combining from the PPT (p. 23):
> Writes to the mapping may be buffered to improve performance. You need to make sure to flush the processor's write buffers before telling devices to read that memory. 

#### 24 Weakly Ordered
When an agent writes to/reads from memory, those set of instructions might be reordered. So if the agent wants to keep those in-order, needs to add sync points or barriers to make sure that the data goes to memory instead of staying in cache.

Most systems support cached weakly ordered memory mappings.

#### 25 Non-Coherent
Non-coherent memory mapping is the mapping that can be cached where there is no guarantee on the order of operations. So the software has to handle the synchronization of every component in the system.


Here comes the definition of non-coherent mapping from the PPT (p. 25)
> This memory mapping type permits speculative reads, merging of accesses and (if interrupted by an exception) repeating of writes without side effects. Accesses to non-coherent memory can always be buffered, and in most situations they are also cached (but they can be configured to be uncached). There is no implicit ordering of non-coherent memory accesses. When not explicitly restricted, the only limit to how out-of-order non-dependent accesses can be is the processor's ability to hold multiple live transactions.

> When using non-coherent memory mappings you are guaranteeing to the platform that you have all the correct and necessary sync points for this memory in the driver.

## Cache management
According to the previous section (memory mappings), we know that we have to handle the cache coherence problem for some memory mappings.

In Linux kernel:
+ ```include/asm-generic/cacheflush.h```: 
    + includes some APIs for cache management, especially for flushing the cache and making sure that data ends up in memory (from CPU's point of view, i.e. user space mappings and kernel space mappings).
    + does not include the cache management for devices. 
    + There is no explicit support for different levels of caches.
+ ```arch/arm/include/asm/outercache.h```:
    + This header takes device into account (i.e. from the devices' point of view).
    + This is architecture specific.
    + 14:21 of the video, did not hear it clearly: what level of the cache (?)

**Important Rule:** Never use the above headers in drivers.
+ ```outercache.h``` is completely architecture specific (only for ARM). So if you use that header, then your driver will never be portable across different architectures.
+ ```cacheflush.h``` is more generic, but it is targeted for specific purposes: syncing the memory when you will map and unmap the memory for CPU, and modifying the configuration of the MMU.

So those APIs are not suitable for DMA. What we need to use is DMA Mapping API.

## DMA Mapping API
Several purposes for those APIs:
+ Allocate memory that is suitable for DMA operations.
    + Not all memory in the system is suitable to be accessed by both CPU and devices.
+ Map the allocated memory (DMA memory) to different agents of the system
    + The agents include CPU (user space processes, kernel space processes), and the devices.
    + You may need a buffer to be accessible from user space and also map it to devices.
    + So it is really mapping and unmapping that piece of memory to/from different agents in the system, and giving/removing access to that memory for different agents in the system.
+ Synchronize memory between CPU and device domains.
    + Synchronize the access to the memory.
    + CPU writes to memory and that API is used to let CPU give ownership of the memory to the device since the device needs to access that memory. 

DMA mapping APIs are stored in the header file ```include/linux/dma-mapping.h```. That header is the file we can include in our device drivers.

### 34 linux/dma-mapping.h
```
linux/dma-mapping.h
│
├─ linux/dma-attrs.h             // the attributes that come with DMA mappings
├─ linux/dma-direction.h         // the directions of DMA (memory <-> device)
├─ linux/scatterlist.h
│
#ifdef CONFIG_HAS_DMA
└─ asm/dma-mapping.h
#else
└─ asm-generic/dma-mapping-broken.h // if platform does not support DMA, that header
                                    // will make sure the driver still compiles
                                    // but it will not link, so it will not crash
                                    // everything at compilation time.
#endif
```
If we need DMA in our driver, we need to make sure that our driver depends on ```CONFIG_HAS_DMA```, otherwise it will not work.


Below is the same file for ARM (architecture specific part of the implementation) on p. 35:
```
linux/dma-mapping.h
│
├─ linux/dma-attrs.h
├─ linux/dma-direction.h
├─ linux/scatterlist.h
│
└─ arch/arm/include/asm/dma-mapping.h
 │
 ├─ asm-generic/dma-mapping-common.h
 └─ asm-generic/dma-coherent.h
```
It includes a couple of headers that have helper functions for generic parts.

### 37 DMA coherent mapping

#### dma_alloc_coherent()
In ```include/linux/dma-mapping.h```:
+ ```dma_alloc_coherent()```: create a DMA coherent mapping.
+ ```dma_free_coherent()```: create a DMA coherent mapping.

In detail:
```
void *
dma_alloc_coherent(struct device *dev, size_t size,
                   dma_addr_t *dma_handle, gfp_t flag);
```
This routine allocates a region of ```size``` bytes of coherent memory. It also returns a ```dma_handle``` which may be cast to an unsigned integer the same width as the bus and used as the device address base of the region.

**dma_handle**: this is not physical address. When using DMA mapping APIs, we do not touch physical memory. In many cases that ```dma_handle``` corresponds to physical memory, but when there is an IOMMU that remaps addresses between the device and physical memory, ```dma_handle``` and physical address will not be identical.

Hence, from the CPU's perspective, the return value is the virtual address for that mapping; from the device's perspective, the ```dma_handle``` is the device's view of the memory (the virtual address of the mapping from device's view). 

Returns: a pointer to the allocated region (in the processor's virtual address space) so that the kernel driver can accept that memory, or NULL if the allocation failed.

Note: coherent (or consistent) memory (```dma_alloc_coherent()```) can be expensive on some platforms, and the minimum allocation length may be as big as a page, so you should consolidate your requests for consistent memory as much as possible. 

If you need lots of small buffers for DMA, the simplest way to do that is to use the ```dma_pool``` calls, which will allocate pages and then split them into small pieces.

```
void
dma_free_coherent(struct device *dev, size_t size,
                  void *cpu_addr, dma_addr_t dma_handle);
```
Free memory previously allocated by dma_free_coherent(). Unlike with CPU memory allocators, calling this function with a NULL cpu_addr is not safe.

#### dma_alloc_attrs()
```
void * dma_alloc_attrs(struct device *dev, size_t size,
                dma_addr_t *dma_handle, gfp_t flag, struct dma_attrs *attrs);
void dma_free_attrs(struct device *dev, size_t size,
             void *cpu_addr, dma_addr_t dma_handle, struct dma_attrs *attrs);
```
These two functions extend the coherent memory allocation API (dma_alloc_coherent()) by allowing the caller to specify attributes for the allocated memory. When @attrs is NULL, the behaviour is identical to the dma_*_coherent() functions.

Not all attributes are implemented on every architecture, so it can be considered a bug if you specify the attribute while the architecture does not support it. Below is the list of attributes:
+ Allocation Attributes 
    + DMA_ATTR_WRITE_COMBINE
        + Tell the allocator to allocate memory that has a write combining mapping for the CPU.
    + DMA_ATTR_WEAK_ORDERING
        + Tell the allocator to allocate weakly ordered mapping. (for cell platform only)
    + DMA_ATTR_NON_CONSISTENT
        + Tell the allocator to allocate non-consistent mapping. (e.g. this attribute is not implemented on ARM architecture)
    + DMA_ATTR_WRITE_BARRIER
    + DMA_ATTR_FORCE_CONTIGUOUS
        + When we allocate memory and want it to be accessible by a device that has an IOMMU, then there is no need to allocate physically contiguous memory. But by specifying this attribute, we are forcing that the memory is allocated contiguously, which will reduce the overhead of using IOMMU for non-contiguous memory.
+ Allocation and mmap Attributes
    + DMA_ATTR_NO_KERNEL_MAPPING
        + This is used when we know in advance that we do not need to touch the memory inside the kernel, i.e. we do not need a memory mapping between kernel space and physical memory, then we can use this attribute.
        + By setting this attribute, there is no need to create the mapping between kernel and memory. Creating that mapping can be expensive.
        + So by using this attribute, we know that the buffer in memory is only shared by devices (e.g. one device writes to that buffer and another device reads from that buffer).
+ Map Attributes
    + DMA_ATTR_SKIP_CPU_SYNC
        + This is used so that we can skip cache operations.
        + For instance, if the memory is fully coherent, then we do not need to handle the cache in that case, and we can use this attribute.
        + For instance, if the buffer is allocated with non-consistent memory mapping, then we can use this attribute. 
        + Need to be careful because when we need to handle the cache, we should not set this attribute.

All attributes are optional. An architecture that doesn't implement an attribute ignores it and exhibit default behaviour. (See Documentation/DMA-attributes.txt)

### 48 DMA mask
Now that we have allocated memory, we should be aware that not all of the system memory can be used for DMA. For instance, we have a 64-bit system, or even a 32-bit system with the large physical address extension, then the physical address can be more than 32 bits. But there can be some devices that can only do DMA in the low 32 bits of the address space.

In order to solve that, we need to tell the DMA mapping subsystem that the device has the restrictions and we need to check that the device's capability for DMA actually matches what the system can provide.

#### dma_set_mask() & dma_set_coherent_mask()
```
/* asm/dma-mapping.h */
int dma_set_mask(struct device *dev, u64 mask),
/* linux/dma-mapping.h */
int dma_set_coherent_mask(struct device *dev, u64 mask);
int dma_set_mask_and_coherent(struct device *dev, u64 mask);
```

+ ```dma_set_mask()```:
    + Set the device DMA mask for non-coherent mappings.
    + If the device can do DMA for the first 4GB of the address space, then the DMA mask (which is a bitmask) will be a 32-bit bitmask (all 1's).

+ ```dma_set_coherent_mask()```:
    + Set the device DMA mask for coherent mappings.

+ ```dma_set_mask_and_coherent()```:
    + Set the same device DMA mask for both coherent and non-coherent mappings.
    + It is not typical to have two different masks.

The above functions set the dma mask inside ```struct device```:
```
/* linux/device.h */
struct device {
    ...
    u64 *dma_mask;
    u64 coherent_dma_mask;
    ...
};

/* linux/dma-mapping.h */
int dma_coerce_mask_and_coherent(struct device *dev, u64 mask);
```
```dma_coerce_mask_and_coherent``` first initialize the pointer ```dma_mask``` so that it points to 64-bit value ```coherent_dma_mask```, then it calls the above functions to set the corresponding dma mask value.


### 51 User space mapping
We have seen how to allocate memory maps, and how to restrict the address space, we can then talk about how to map the allocated memory to user space.

```
/* asm-generic/dma-mapping.h */
/* Implemented on arm, arm64 and powerpc */
int dma_mmap_attrs(struct device *dev, struct vm_area_struct *vma,
                    void *cpu_addr, dma_addr_t dma_addr, size_t size,
                    struct dma_attrs *attrs);
/* Wrappers */
int dma_mmap_coherent(...);
int dma_mmap_writecombine(...);
```
Two wrapper functions are all based on ```dma_mmap_attrs()```. The wrapper functions map coherent DMA memory OR write-combine DMA memory previously allocated by ```dma_alloc_attrs()``` into user space. The DMA memory must not be freed by the driver until the user space mapping has been released.

Creating multiple mappings with different types (coherent, write-combined, weakly ordered or non-coherent) produces undefined results on some architectures. Care must be taken to specify the same type attributes for all calls to the dma_alloc_attrs() and dma_mmap_attrs() functions for the same memory.


### 55 DMA Streaming mapping
Instead of allocating coherent buffers for DMA using ```dma_alloc_coherent()```, we can allocate DMA memory using ```__get_free_pages()``` or ```kmalloc()``` that will guarantee to return physically contiguous memory. Then we can map the memory to the device by ourselves.

```
/* linux/dma-direction.h */
enum dma_data_direction {
    DMA_BIDIRECTIONAL = 0,
    DMA_TO_DEVICE = 1,
    DMA_FROM_DEVICE = 2,
    DMA_NONE = 3,
};

/* asm-generic/dma-mapping.h */
dma_addr_t
dma_map_single_attrs(struct device *dev, void *ptr, size_t size,
                     enum dma_data_direction dir, struct dma_attrs *attrs);
void
dma_unmap_single_attrs(struct device *dev, dma_addr_t addr, size_t size,
                       enum dma_data_direction dir, struct dma_attrs *attrs);
dma_addr_t dma_map_single(...);
void dma_unmap_single(...);
```

We can also map a page by using:
```
dma_addr_t
dma_map_page(struct device *dev, struct page *page,
                size_t offset, size_t size,
                enum dma_data_direction dir);
void
dma_unmap_page(struct device *dev, dma_addr_t addr,
                size_t size, enum dma_data_direction dir);
```


## IOMMU Integration
The dma mapping functions handle it for us when we map a buffer to a device, it will program the IOMMU of the device, so that the device can access the buffer correctly.

But if we want to deal with IOMMU ourselves:
```
/* linux/iommu.h */
struct iommu_domain * iommu_domain_alloc(struct bus_type *bus);
void iommu_domain_free(struct iommu_domain *domain);
int iommu_attach_device(struct iommu_domain *domain,  struct device *dev);
void iommu_detach_device(struct iommu_domain *domain, struct device *dev);
int iommu_map(struct iommu_domain *domain, unsigned long iova, 
                phys_addr_t paddr, size_t size, int prot);
size_t iommu_unmap(struct iommu_domain *domain,
                    unsigned long iova, size_t size);
```