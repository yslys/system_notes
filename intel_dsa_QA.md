# Intel Data-Streaming Accelerator Q&A with Dave Jiang
This note contains the chat between Dave and me. Many thanks to Dave for his help!

## Memory-mapping to BAR2

> The shared work queue in the accelerator is memory mapped to kernel space, which can be memory mapped by user space as well. After user memory maps to the shared work queue, the user can safely invoke ENQCMD to enqueue the work descriptor to the work queue. 

> May I know if the memory-mapped area is 64 bytes? In other words, the memory mapped area only allows the user to enqueue 1 work descriptor (or 1 batch descriptor) at a time, may I know if I am correct? Thanks in advance.

The memory mapped area is a 4k page. The CPU instruction (ENQCMD) only allows
submission of 1 64B descriptor at a time. So theoretically you can have
64 threads submit a descriptor simultaneously for each cache-line on the
page. The entire 4k page is the same "portal".

### My understanding:
+ Memory-mapped area is a 4 KB page for each portal.
    + Portal: the entry point of each work submission
    + Each work queue has 4 portals, so user can memory-map to 4 pages (total 16 KB) if user wants to submit to 4 different portals of one work queue.
+ Cache line size: 64 Bytes
    + Since each page is 4KB, it contains a total of 64 cache lines.
    + For one particular portal (4KB page) of a work queue, it allows 64 threads to submit descriptors simultaneously for each cache-line on the page.
    + Question: 
        + In [user mode testing code, ```test_batch()``` function](https://github.com/intel/idxd-config/blob/stable/test/dsa_test.c#L33), it calls [```dsa_desc_submit()```](https://github.com/intel/idxd-config/blob/stable/test/dsa_test.c#L151) to submit task descriptor of one batch task node to the portal, which then calls [```dsa_desc_submit()```](https://github.com/intel/idxd-config/blob/stable/test/prep.c#L23) to print out some info about hw, then either calls [```movdir64b()```](https://github.com/intel/idxd-config/blob/stable/test/accfg_test.h#L4) for Dedicated work queue or [```dsa_enqcmd()```](https://github.com/intel/idxd-config/blob/stable/test/dsa.c#L872) for Shared work queue. 
        + [```movdir64b()```](https://github.com/intel/idxd-config/blob/stable/test/accfg_test.h#L4) is implemented as inline assembly, while [```dsa_enqcmd()```](https://github.com/intel/idxd-config/blob/stable/test/dsa.c#L872) allows retrying for at most 4 times if the submission to the portal fails, and each time calls [```enqcmd()```](https://github.com/intel/idxd-config/blob/stable/test/accfg_test.h#L11). The similarity for both is: they all take two same params: ```foo(struct dsa_context *ctx -> void *wq_reg, struct dsa_hw_desc *hw)```. ([```struct dsa_context```](https://github.com/intel/idxd-config/blob/stable/test/dsa.h#L116), and [```void *wq_reg```](https://github.com/intel/idxd-config/blob/stable/test/dsa.h#L126); [```struct dsa_hw_desc```](https://github.com/intel/idxd-config/blob/stable/accfg/idxd.h#L147))
            + As a side note, **it is worth going through [```struct dsa_hw_desc```](https://github.com/intel/idxd-config/blob/stable/accfg/idxd.h#L147) to see how to define the task descriptor with the fields taking up the specified number of bits we want.**
        + Both ```movdir64b()``` and ```enqcmd()``` take two parameters: ```portal``` and ```desc```, meaning the address of the portal to submit to and the address of the task descriptor respectively. Hence, we know that ```portal``` == ```ctx->wq_reg```, ```desc``` == ```hw```.
        + Based on my understanding, if multiple threads trying to submit descriptors to the same portal, current enqcmd does not support distinguishing them by placing them into different cachelines, and how could they be polled on by the hardware? 

## Is it the hardware that handles the enqueue and dequeue to/from the work queue on device?

> ENQCMD requires the hardware to support DMWr (Deferrable Memory Write request) capability. It seems that the accelerator itself has a dispatcher hardware that polls (or wait on an interrupt) on the work submission from IO fabric interface, and handles the newly submitted work descriptor in hardware level, i.e. the hardware decides whether to enqueue the descriptor or not. Hence, it is the hardware itself that does enqueue to the work queues based on the descriptor. May I know if my understanding is correct? Thanks in advance.

So the CPU instruction would write the 64B descriptor to the device portal. 

+ For a shared WQ: 
    + As long as the number of wq enqueued does not exceed the configured wq size, the descriptor is accepted. 
    + If the wq size is exceeded, then the zero flag is set and the descriptor is rejected. 

+ For a dedicated WQ: 
    + Because MOVDIR64B instruction is a posted write, it does not wait for a response from the device. 
    + So if wq size is exceeded, the descriptor is simply dropped without the caller knowing the result. 

The device itself would know a descriptor has been "pushed" to it when the CPU instruction writes the data to the device. The device engine would be continuously pulling the descriptors out of the wq and processing them. 

So theoretically the actual number of descriptors that the device can accept would be **wq->size + number of descriptors the engine can hold and process**.

### My understanding:
+ Shared WQ and Dedicated WQ will have different response when the WQ is full.
    + Shared WQ (ENQCMD): will set the zero flag, and reject the descriptor so that caller will know the result.
    + Dedicated WQ (MOVDIR64B): since MOVDIR64B is a posted write, the descriptor is dropped with no response to the caller.

+ The CPU instructions - ```ENQCMD``` and ```MOVDIR64B``` will write the 64-Byte descriptor to the device's portal. In other words, the two instructions push the descriptor to the portal.
    + According to the description: The ENQCMD instruction allows software to write commands to enqueue registers, which are special device
registers accessed using memory-mapped I/O (MMIO).
    + The portal is the entry point of the task descriptors.

    + The portal itself is a memory-mapped page (both mapped by user and device).
    + So what the instructions do is to write the 64 Byte descriptor to that page.
    + Question:
        + "So the CPU instruction would write the 64B descriptor to the device portal."
            + This is the link to the [```enqcmd()```](https://github.com/intel/idxd-config/blob/stable/test/accfg_test.h#L11)
            + Does it write to the portal in physical memory, or it writes directly to the device?
        + **My understanding: the portal is a physical page, which mapped by both kernel and device register. When user calls ENQCMD to enqueue to one portal, the task descriptor will present both in the page in memory and the device register. But what happens internally?**
        
        + If it writes to the portal only, then how could the device know that there is a new descriptor enqueued? By read the page into a temp buffer and compare with the previous buffer? After the device reads the page, the device should clear the page in memory.
        + If it directly writes to the device, then what happens underneath that allows a write to be directly performed to the device instead of physical memory? Maybe it requires first writing to memory (the page), then transferring to the device. The device itself has a buffer that buffers all the descriptors coming-in, and then enqueue to the WQ one by one?




## How does DSA write completion record to memory?
> My third question regards to how the device writes completions to memory so that user can poll on the result. I happened to find Intel idxd-config repository, which has some use cases of how user can use ENQCMD. Based on my understanding, the completion record (32 bytes) is what the accelerator will write to memory after the computation is finished. May I know if DSA does this by issuing one DMA write at a time for each completion record? If two completion records are contiguous in memory, will DSA do a 64-byte DMA transfer, or still needs to do 32-byte transfer at a time? May I know DSA supports DMA scattered-gather lists operation? Thanks in advance!

So I think you have 2 different questions here. I believe DSA will DMA
32 bytes at a time for completion record. **I don't believe DSA added
logic to optimize larger DMA if CR are contiguous. Probably not worth
the additional hardware logic just for that.**


**DSA supports batch operations. But it does not currently support
scatter-gather for the planned release on Sapphire Rapids platform.**

### My Understanding:
+ Question:
    + How many computations can be performed in parallel in DSA?
    + If 2 or 4, then if more than one computations are finished, then are the writes to memory issued sequentially or in parallel?





## Performance Analysis:
+ Single-request submission:
    + ENQCMD: user calls ```enqcmd()``` each time to the portal, then the device will handle the rest - do computation and after it finishes, write ```Completion Record``` to ```Completion Record Address``` (specified in the task descriptor enqueued) in memory.
        + The hardware manages the work queue.
        + Memory mapped space: ```(#portals/WQ) * (#WQ) * (4KB/WQ)```
        + User knows enqueue result only if: Shared WQ (not Dedicated WQ) + hardware returns.
        + Size to transfer (more official terminology?): ```sizeof(descriptor)```.
        + Result writeback
            + size: ```sizeof(completion record) = 32 bytes```
            + Response time (time for completion): short. 
                + No wait on completion - user can poll on the ```Completion Record Address``` to see the result.
        + Enqueue throughput: 1/time(DMA512BIT)
        + Enqueue latency: time(DMA512BIT) or multiple of it if work queue full.
    + Shared Memory Queue: user directly writes to submission queue, using llring API, i.e. write to the shared queue in memory. 
        + The lling API will manage the queue (software approach).
        + Memory mapped space: ```sizeof(llring + sizeof(descriptor) * nr_desc)```
        + User knows the enqueue result immediately without going through hardware.
        + Size to transfer: the whole memory mapped space - a lot. If DMA transfer granularity is 64 bytes, then the number of DMAs should be ```sizeof(llring + sizeof(descriptor) * nr_desc) / 64```.
        + Result writeback
            + size: ```sizeof(llring + sizeof(descriptor) * nr_desc)```
            + Response time (time for completion): depends.
                + If we define the granularity of computation to be n tasks, then the writeback will be performed after all n tasks completed. Note that n can be one, and we can modify our device to write 1 result back at a time, similar to ENQCMD, but I am not sure if this still satisfies the shared memory queue requirement.
        + Enqueue throughput:
            + Depends on when user rings the doorbell, and the granularity of DMA transfer.
            + For single submission, user enqueues + ring doorbell (write to memory-mapped register): slower than ENQCMD since requires (very likely) more than 1 DMA transfer (since we need to DMA the whole queue). Ringing the doorbell takes up the same time as ENQCMD (directly write the task descriptor to memory-mapped region).
            + Hence, throughput = 1/time(DMA_QUEUE + doorbell) 
        + Enqueue latency: 
            + time(DMA_QUEUE + doorbell)

+ Multi-request submission:
    + ENQCMD: using batch descriptors (granularity = 4)
        + Enqueue throughput: 4/time(latency)
        + Enqueue latency: 
            + Since we are using batch descriptor, internally, it contains VA for 4 task descriptors. Hence, user enqueuing 1 batch descriptor + device reading 4 task descriptors.
            + Latency: 
                + time(DMA512BIT) + time(device reading 4 task descriptors)
                + If granularity of DMA is 64 bytes, then one task descriptor at a time; (optimization: (i) user put task descriptors contiguously to explore spatial locality, or (ii) device does PTW right after it starts DMA the first descriptor, i.e. while DMA the current task descriptor, do PTW to do the access validation of the next task descriptor. Note that DVM already ensures VA==PA, but still requires access validation).
                + If granularity of DMA is 128 bytes, then 2 descriptors at a time. So (i) user put task descriptors contiguously to explore spatial locality would lead to better performance.
                + Will take more time if work queue full. (Once device accepts the batch descriptor and the work queue is full, it guarantees that the 4 task descriptors will be processed - need to be confirmed).
    + Shared Memory Queue: 
        + Enqueue latency:
            + Depends on when user rings the doorbell, and the granularity of DMA transfer.
            + For multiple submissions, suppose user enqueues n tasks.
            + latency = time(DMA_QUEUE + doorbell)
        + Enqueue throughput:
            + n/time(DMA_QUEUE + doorbell)
        + To make shared memory queue catch up with the performance of ENQCMD, needs to make n as large as possible.


> Shared memory queues are the default design for accelerators (and I/O devices), and the benefit of ENQCMD isn’t clear, as it has its own costs (synchronous write or the request, managing individual completions), and a single shared queue limited by the size of the HQ queue vs unlimited queue size for shared memory not shared with other processes.



## My thoughts on design:
Eagerpaging + Identity mapping + eager mmap()


## Questions
separate virtual address spaces cause 
data to be replicated


fill_page_table_manually()

then we set page_increm with the number of new order



Read MEG to see if data is required to read to device or not.