# HSA Queuing Model
https://www.youtube.com/watch?v=mL5Z5ziKULE&t=52s

HSA: Heterogeneous System Architecture

HSA Queuing is the mechanism with which one component in the system (e.g. CPU) offloads work to another component in the system (e.g. GPU, accelerators, etc.).

## Classic work offload
User, OS, GPU:

+ User calls ```mmap()``` or ```malloc()``` for memory
    + User first requests a buffer which will be storing the data that will be consumed by GPU. 
    + The buffer allocation is handled by OS, which allocates the buffer and return to user.
+ User enqueues to the queue
    + OS may invoke the driver for GPU to schedule the task, and enqueue the task to GPU
+ GPU starts computing
+ GPU finishes computing and interrupts OS
+ OS notifies the user so that user knows the task has finished
+ User reads the result from memory. 


## HSA Queuing

### Requirements
HSA Queuing requires 4 mechanisms to enable lower overhead job dispatch:
+ Shared Virtual Memory
+ System coherency (cache coherency)
+ Signaling (user level)
    + The ability of one agent in the system to signal another agent without any invocation of a higher privilege level piece of software.
+ User mode queuing
    + The ability of user mode application to dispatch work to another agent without going through an OS or driver. 

#### Shared Virtual Memory
CPU and GPU have their own virtual address space. Hence, on this type of system, you can have different virtual addresses on GPU and CPU pointing to the same physical address in memory. You can also have same virtual addresses pointing to different physical addresses. Sometimes there is a special region of physical memory that is just for GPU.

The disadvantage of that is: suppose user has requested a chunk of memory, with virtual address ```u_va```, and the corresponding physical address is ```u_pa```. GPU needs to read that buffer to obtain the data to operate. However, GPU has not yet created a (VA, PA) mapping in memory. Hence, user needs to call ```cudaMalloc``` to create the mapping between GPU and memory. Hence, in total, there are two buffers allocated in memory, one for CPU, the other for GPU.

User needs to write to the buffer for CPU, and then tell GPU to copy the data 

Since GPU does not share the VA or PA with CPU, in order to create a mapping between GPU's VA and memory PA, user needs 

 Since GPU needs to read that buffer to get the data to compute, GPU might have another set of VA, PA pair - ```d_va```, ```d_pa```

Shared Virtual Memory provides a consistent view of virtual memory across the CPU and GPU, i.e. for the same virtual address on both CPU and GPU, they all point to the same physical address. This precludes having private physical memory for GPU (or any HSA agent)

+ Pros:
    + No mapping tricks, no copying back-and-forth between different PA addresses.
    + Allows to send pointers (not data) back and forth between HSA agents.