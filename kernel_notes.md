# Kernel Notes

+ ULK: Understanding the Linux Kernel

## Terminologies:
+ BIOS: Basic Input/Output System
+ MBR: Master Boot Record
    + Typically the first sector of the disk, or the partition of the disk.
+ IDT: Interrupt Descriptor Table
+ idtr register: IDT (Interrupt Descriptor Table) Register
+ PIC: Programmable Interrupt Controllers
+ cr0: control register ([src](https://wiki.osdev.org/CPU_Registers_x86))
+ PE bit: Protected mode Enabled bit
+ PG bit: Paging bit
+ bss: block starting symbol - the portion of an object file, executable, or assembly language code that contains statically allocated variables that are declared but have not been assigned a value yet.
+ LDT: Local Descriptor Table
+ GDT: Global Descriptor Table
+ Segment Descriptor: an 8-byte entry in LDT or GDT that describes the segment characteristics.
    + Segment descriptor fields: p. 38
    + Code Segment Descriptor
    + Data Segment Descriptor
    + Task State Segment Descriptor (TSSD)
    + Local Descriptor Table Descriptor (LDTD)
+ gdtr register: GDT (Global Descriptor Table) Register
    + stores the address and size of the GDT in memory.
+ ldtr register: LDT (Local Descriptor Table) Register
    + stores the address and size of the currently used LDT.
+ Logical Address: \[segment (16bit) | offset (32bit)\]
    + Logical Addr ---SegmentationUnit---> Linear Addr (virt addr) ---PagingUnit---> Physical Addr.
+ cs, ss, ds, es, fs, gs registers (ULK p. 37)
    + code segment, stack segment, data segment
    + The rest three are general purpose registers.
+ CPL: Current Privilege Level
    + 2-bit field in cs register (code segment register) that specifies CPL of the CPU. 0 - kernel mode, highest privilege level; 3: user mode, lowest privilege level.
+ PAE: Physical Address Extension (huge pages, 2MB per page)
+ ELF: Executable and Linking Format
    + The standard Linux executable format
    + An executable format is described by an object of type ```linux_binfmt```, such type provides 3 methods:
        + load_binary: sets up a new execution environment for current process by reading the info stored in an executable file.
        + load_shlib
        + core_dump


## Bootstrap
Resource: *Appendix A, Understanding the Linux Kernel*.

1. BIOS
2. Boot loader
3. setup()
4. startup_32()
5. start_kernel()


#### BIOS (Basic Input/Output System)
+ RESET pin in CPU is raised.
+ After RESET, some registers (cs - code segment, eip - instruction pointer ([link](https://stackoverflow.com/questions/17777146/what-is-the-purpose-of-cs-and-ip-registers-in-intel-8086-assembly))) are set.
+ Code at address 0xfffffff0 is executed. 0xfffffff0 is mapped to ROM (Read-Only Memory, persistent memory chip).
    + What stores in ROM?
        + BIOS - the set of programs that includes several interrupt-driven procedures in the booting phase to handle the hardware devices.
    + BIOS procedures must be executed in real mode (not protected mode).
    + What is real mode address?
        + \[segment | offset\]: PA = seg\*16+offset
+ 4 operations in BIOS
    + (1) POST (Power-On Self-Test): execute tests on H/W to establish which devices are present, whether they are working properly.
    + (2) Initialize H/W: guarantee all H/W operate w/o conflicts on IRQ lines and I/O ports.
    + (3) Search for an OS to boot: may access the first sector (boot sector) of hard disk.
    + (4) After hard disk is found, copy the boot sector into RAM, starting from PA 0x00007c00, then jump into that address to execute the code (for boot).

#### Boot Loader
+ Definition: the program invoked by BIOS to load the image of OS into RAM.
+ Booting from hard disk (LILO - Linux Loader):
    + MBR is the first sector of the partition of the hard disk, which includes the partition table, and a small program that loads the first sector of the partition containing the OS to be started.
    + (1) In MBR (or boot sector), has a small bootloader, which is loaded into RAM at address 0x00007c00 by BIOS.
    + (2) The small boot loader moves itself to address 0x00096a00 in RAM, sets up the real mode stack (from 0x00098000 to 0x000969ff), loads the second part of LILO into RAM at address 0x00096c00 and jumps into it.
    + (3) The second part of LILO copies the kernel into RAM.
        + (3.1) Load first part of kernel:
            + Invokes a BIOS procedure to load an initial portion of the kernel image from disk: first 512 bytes of kernel image are put at address 0x00090000 in RAM, the code of setup() function is put at 0x00090200 in RAM.
        + (3.2) Load the rest of kernel:
            + Invokes a BIOS procedure to load the rest of kernel image into RAM at either address: (i) low address 0x00010000 (small kernel image compiled with make zImage), or (ii) high address 0x00100000 (big kernel image compiled with make bzImage).
    + (4) Jumps to setup() code.

#### setup() Function
+ Where is setup() function located?
    + The code of setup() is assembly, which has been placed by the linker at offset 0x200 of kernel image, i.e. 0x00090200 in RAM.
+ What does setup() do?
    + Initialize H/W devices, set up the environment for the execution of the kernel.
+ BIOS has initialized H/W already, why setup() does the same thing?
    + Linux reinitializes the devices in its own manner to enhance portability and robustness.
+ Steps of setup(): 
    + Only some important steps are listed, for more info, refer to the book p. 839
    + Invokes a BIOS routine that builds a table in RAM describing the layout of physical memory.
    + Reinitializes the disk controller and determines the hard disk parameters.
    + Checks for an IBM Micro Channel bus (MCA).
    + If the kernel image was loaded low in RAM (at physical address 0x00010000), the function moves it to physical address 0x00001000. If the kernel image was loaded high in RAM, the function does not move it (remains 0x00100000). This step is necessary because to be able to store the kernel image on a floppy disk and to reduce the booting time, the kernel image stored on disk is compressed, and the decompression routine needs some free space to use as a temporary buffer following the kernel image in RAM.
    + Sets up a provisional (temporary) Interrupt Descriptor Table (IDT) and a provisional Global Descriptor Table (GDT).
    + Resets the floating-point unit (FPU), if any.
    + Reprograms the Programmable Interrupt Controllers (PIC) to mask all interrupts, except IRQ2 which is the cascading interrupt between the two PICs.
    + Switches the CPU from Real Mode to Protected Mode by setting the PE bit in the cr0 status register. The PG bit in the cr0 register is cleared, so paging is still disabled.
    + Jumps to the startup_32() assembly language function.

#### startup_32() Function
Recall that after setup(), depending on whether the kernel image is loaded high (big image) or low, the kernel image is at address 0x00100000 (for high), or 0x00001000 (for low) in RAM.

There are two startup_32() functions:
+ arch/i386/boot/compressed/head.S
+ arch/i386/kernel/head.S - the decompressed kernel image

The first startup_32():
+ (1) Initializes the segmentation registers and a provisional stack.
+ (2) Clears all bits in the eflags register.
+ (3) Fills the area of uninitialized data of the kernel identified by the _edata and _end symbols with zeros ("Physical Memory Layout" in Chapter 2).
+ (4) Invokes the decompress_kernel() function to decompress the kernel image. If the kernel image was loaded low, the decompressed kernel is placed at physical address 0x00100000. Otherwise, if the kernel image was loaded high, the decompressed kernel is placed in a temporary buffer located after the compressed image. The decompressed image is then moved into its final position, which starts at physical address 0x00100000.
+ (5) Jumps to physical address 0x00100000 (for both loaded low and high).

The second startup_32():
+ Sets up the execution environment for the first Linux process (process 0).
+ (1) Initializes the segmentation registers with their final values.
+ (2) Fills the bss segment of the kernel with zeros ("Program Segments and Process Memory Regions" in Chapter 20).
+ (3) Initializes the provisional kernel Page Tables contained in swapper_pg_dir and pg0 to identically map the linear addresses to the same physical addresses ("Kernel Page Tables" in Chapter 2).
+ (4) Stores the address of the Page Global Directory in the cr3 register, and enables paging by setting the PG bit in the cr0 register.
+ (5) Sets up the Kernel Mode stack for process 0 ("Kernel Threads" in Chapter 3).
+ (6) Once again, the function clears all bits in the eflags register.
+ (7) Invokes setup_idt() to fill the IDT with null interrupt handlers ("Preliminary Initialization of the IDT" in Chapter 4).
+ (8) Puts the system parameters obtained from the BIOS and the parameters passed to the operating system into the first page frame ("Physical Memory Layout" in Chapter 2).
+ (9) Identifies the model of the processor.
+ (10) Loads the gdtr and idtr registers with the addresses of the GDT and IDT tables.
+ (11) Jumps to the start_kernel( ) function.


#### start_kernel() Function
The start_kernel( ) function completes the initialization of the Linux kernel. Nearly every kernel component is initialized by this function.

+ The scheduler is initialized by invoking the sched_init() function (Chapter 7).
+ The memory zones are initialized by invoking the build_all_zonelists() function ("Memory Zones" in Chapter 8).
+ The Buddy system allocators are initialized by invoking the page_alloc_init() and mem_init() functions ("The Buddy System Algorithm" in
Chapter 8).
+ The final initialization of the IDT is performed by invoking trap_init() ("Exception Handling" in Chapter 4) and init_IRQ() ("IRQ data structures" in Chapter 4).
+ The TASKLET_SOFTIRQ and HI_SOFTIRQ are initialized by invoking the softirq_init() function ("Softirqs" in Chapter 4).
+ The system date and time are initialized by the time_init() function ("The Linux Timekeeping Architecture" in Chapter 6).
+ The slab allocator is initialized by the kmem_cache_init() function ("General and Specific Caches" in Chapter 8).
+ The speed of the CPU clock is determined by invoking the calibrate_delay() function ("Delay Functions" in Chapter 6).
+ The kernel thread for process 1 is created by invoking the kernel_thread() function. In turn, this kernel thread creates the other kernel threads and executes the /sbin/init program ("Kernel Threads" in Chapter 3).


## What is done in H/W and what is done in Linux
The consistency between the cache levels is implemented at the hardware level. Linux ignores these hardware details and assumes there is a single cache.

When the cr3 control register of a CPU is modified, the hardware automatically invalidates all entries of the local TLB, because a new set of page tables is in use and the TLBs are pointing to old data.


## mm

### Linux Memory Models:
In Linux, each page frame is represented by ```struct page```. Given an virtual address, CPU needs to ask TLB if it contains the PTE of the mapping: ```(VPN, PFN)```. So at the end, we know what PFN is. However, although there is a one-to-one mapping between PFN and ```struct page```, PFN 0 does not necessarily map to page 0. There is another level of mapping that stores the mapping: ```(PFN, struct page)```.

Linux provides us some macros to retrieve either PFN or page: ```page_to_pfn``` and ```pfn_to_page```. So now the question is - how are they mapped?

Answer: **It depends on the memory model we use**.

[Source](http://www.wowotech.net/memory_management/memory_model.html)

#### UMA and NUMA:
+ Similarities:
    + Both are shared memory models.
    + Both targeting multi-processors.
+ What is UMA?
    + UMA (Uniform Memory Access).
    + Only one memory controller.
    + Multiple processors share one RAM. Time to access memory for each processor is the same.

+ What is NUMA?
    + NUMA (Non-Uniform Memory Access).
    + Multiple memory controllers.
    + Time to access memory depends on how far from the memory to the processor is, i.e. local memory access is faster than remote memory access.
    + Typically we use the notion ```node``` to denote one processor and one local memory pair. Hence, remote memory must reside in another node.

#### FLat Memory Model
From any processor's point of view, when it tries to access physical memory, the physical address space is contiguous without holes.

In this case, kernel maintains a page array (```struct page *```) called ```mem_map```, with length being the number of pages in memory. Each ```struct page``` maps to a page frame (PFN), with a constant offset (offset could be 0). There is only one node, i.e. only one ```struct pglist_data```. No need a page table.

To convert PFN to ```struct page``` is simple in this case.

#### Discontiguous Memory Model (Multi-nodes + Flat/node)
From any processor's point of view, when it tries to access physical memory, the physical memory contains some holes, and thus discontiguous.

Discontiguous memory model is designed for NUMA architectures, and it is an extension for FLat memory model. It has multiple nodes, i.e. multiple ```struct pglist_data```. Each ```struct pglist_data``` points to an array of contiguous ```struct page```s. The mapping between each page structure (```struct page```) and the physical page frame (PFN) is the same as FLat memory model, i.e. constant offset. We can use the macro ```NODE_DATA()``` to get the corresponding pointer to a ```struct pglist_data```:
```
#include <linux/mmzone.h>
// @nid: node id.
static inline struct pglist_data *NODE_DATA(int nid) {
    return &contig_page_data;
}
```
Inside the ```struct pglist_data *``` we obtained, it has a field: ```struct page *node_mem_map```, which is the starting address of the array of contiguous pages within this node, similar to ```mem_map``` in Flat memory model.

To convert PFN to ```struct page```, we first need to obtain the node id (based on PFN), then pass the node id to ```NODE_DATA()``` to get ```struct pglist_data *```, then get the starting address of an array of contiguous ```struct page```s by accessing ```struct page *node_mem_map```. Finally, we could obtain the pointer to the corresponding ```struct page``` by using offset.

Conclusion: Discontiguous memory model - several (number of nodes) arrays of contiguous pages, each array of pages is not contiguous.

#### Sparse Memory Model (for memory hot plug/remove)
From Flat memory model to Discontiguous memory model, the number of chunks of contiguous pages changes from 1 (flat) to the number of nodes (discontiguous), i.e. from 1 ```mem_map``` array to multiple ```struct page *node_mem_map``` arrays. It seems to work well (and indeed works well for systems without hot plugging). But Discontiguous memory model needs to change so that it supports memory hot plugging.

+ What is hot plugging?
    + It is the addition of a component to a "running computer system" without significant interruption to the operation of the system.
    + Hot plugging a device does not require a restart of the system.
    + This is especially useful for systems that must always stay running, e.g. a server.
+ Hot-pluggable devices:
    + HDD: hard disk drives
    + SSD: solid-state drives
    + USB flash drive: (Universal Serial Bus) flash drive
    + Mice, keyboards
+ Memory hot(un)plug ([src](https://www.kernel.org/doc/html/latest/admin-guide/mm/memory-hotplug.html))
    + allows for increasing and decreasing the size of physical memory available to a machine at runtime. In the simplest case, it consists of physically plugging or unplugging a DIMM at runtime, coordinated with the operating system.

For a single node, say it has memory of size 32GB, and is currently running to provide some support for our personal purposes. What if we want to increase/decrease the size of memory while not impacting the currently running applications? This will make the previously contiguous ```node_mem_map``` within a single node not contiguous anymore, since for that single node, the memory might be hot-plugged or hot-unplugged while that node is running.

In other words, say there are two memory device connecting to a single node. If using discontiguous memory model, Linux will regard those two memory devices as a single one, i.e. only 1 ```struct page *node_mem_map``` (array of logically contiguous but physically separated into two pieces ```struct page```s). If the memory is hot-(un)pluggable, then if we remove one piece of memory, it is hard for Linux to figure out which part of ```node_mem_map``` should be cleared.

Here comes Sparse memory model, which further divides the memory space of one node into **sections**, so that each section is hot-(un)pluggable; meanwhile, the array of ```struct page```s are contiguous within each section.

The data structure for one section is ```struct mem_section```, not ```struct pglist_data``` (for node) anymore. I tried to find any relation between ```pglist_data``` and ```mem_section```, but did not find it; hence, I assume there is no relation between ```pglist_data``` and ```mem_section```.

To convert PFN to ```struct page```: [source code](https://elixir.bootlin.com/linux/v5.17.1/source/include/asm-generic/memory_model.h#L39)
```
#define __pfn_to_page(pfn)                          \
({	unsigned long __pfn = (pfn);                    \
    struct mem_section *__sec = __pfn_to_section(__pfn);	\
    __section_mem_map_addr(__sec) + __pfn;          \
})
```
Before we start discussing the details, we need to know how PFN is structured:
```
Recall that Virtual address consists of:
VA = |VPN|offset_within_page|
VPN --> TLB / PageTable Walk --> PFN

How does kernel use PFN?
PFN consists of two parts: section index and page index.
PFN = |section_index|page_index|
section index: which section this PFN belongs to
page index: the index into the array of struct pages corresponding to the page 
            we want

page_index: PFN_SECTION_SHIFT bits (maximum of 2^PFN_SECTION_SHIFT pages)
```
+ There is a static array of ```mem_section *``` defined in [```mm/sparse.c```](https://elixir.bootlin.com/linux/v5.17.1/source/mm/sparse.c#L27), i.e. ```struct mem_section **mem_section;```. 
+ Each ```struct mem_section``` has a field ```unsigned long section_mem_map```, which is a pointer to an array of ```struct page```s.
+ Right shift PFN ```PFN_SECTION_SHIFT``` bits to obtain section number. (Note: ```#define PAGES_PER_SECTION       (1UL << PFN_SECTION_SHIFT)```)
+ Use section number to index the static array (```mem_section **```), to find the corresponding ```struct mem_section *```.
+ Inside ```struct mem_section *```, retrieve ```section_mem_map```.
+ Use page index to index the ```section_mem_map``` array to get the page pointer we want.

## Process ```struct task_struct```
An **Execution Context**, which can be independently scheduled, has its own process descriptor - ```struct task_struct```, i.e. process descriptor pointers (in the case of address). This structure is defined in ```include/linux/sched.h```.

#### Memory Descriptor
All information related to the process address space is included in an object called the *memory descriptor*, which is of type ```struct mm_struct```, i.e. ```struct mm_struct *mm```. This structure is defined in ```/include/linux/mm_types.h```. (Note, in ULK, it states it is defined in ```include/linux/sched.h```, but only in v2.6 kernel.)

Another thing worth mentioning is - ```struct page```, ```struct vm_area_struct``` are also defined in ```/include/linux/mm_types.h```.

Given a process, the pointer to the memory descriptor can be retrieved by ```current->mm```.

+ Inside ```struct mm_struct```, we added a boolean variable ```bool eagerpaging``` to indicate whether current process's address space is using eagerpaging or not.
+ Where we added such boolean variable?
    + there is a struct inside ```struct mm_struct```. In other words, it looks like the following:
    +
    ```
        struct mm_struct {
            struct {
                // something here
                bool eagerpaging; // newly added
            } __randomize_layout;
            unsigned long cpu_bitmap[];
        };
    ```
    + What is ```__randomize_layout```?
        + [Article from LWN](https://lwn.net/Articles/722293/)
        + [patch](https://lore.kernel.org/lkml/1495829844-69341-7-git-send-email-keescook@chromium.org/T/#maef5a5badd660e2064cdd48fc3e95c361c0aa4ee). (Search for mm_struct.)
        + This marks most of the layout of ```mm_struct``` as randomizable, but leaves cpu bitmap untouched at the end.
    + What is designated layout?    
        + [patch](https://lore.kernel.org/lkml/1495829844-69341-4-git-send-email-keescook@chromium.org/)
        + This allows structure annotations for requiring designated initialization in GCC 5.1.0 and later: [link to GCC doc on Designated Initializers](https://gcc.gnu.org/onlinedocs/gcc/Designated-Inits.html)
        + The structure randomization layout **plugin** will be using this to help identify structures that need this form of initialization.

#### Program Execution
While going through the commit history of eager paging, there is one step that added a syscall to register a process as eager paging candidate, so that user space program can run the process as eager paging.
+ linux-5.13.11/arch/x86/entry/syscalls/syscall_64.tbl:
    + This added a new entry of eagerpaging to the syscall table.
    + But on RISC-V, it is slightly different
    + TODO:
+ linux-5.13.11/fs/exec.c:
    + This file contains the ```execve()``` syscall implementation, which is the only syscall for program execution.
    + The function ```begin_new_exec()``` is modified. After checking the [patch](https://lkml.org/lkml/2020/5/5/1216), the older version function is ```flush_old_exec()```. Fortunately, this function is mentioned in the book ULK ("The exec Functions" in Chapter 20: Program Execution).
        + ```sys_execve()``` service routine takes in 3 parameters in user space: address of executable pathname, address of NULL-terminated array (each string is a cmdline argument), address of NULL-terminated array (each string is an environment variable in the ```NAME=value``` format).
        + ```sys_execve()``` copies the executable pathname to a newly allocated page frame, then invokes ```do_execve()``` function, which then calls ```load_binary``` method.
        + The ```load_binary``` method corresponding to an executable file format invokes ```flush_old_exec()``` to release almost all resources used by the previous computation.

#### Process termination
+ In Linux 2.6, there are two syscalls that terminate a user mode application:
    + ```exit_group()```: terminates a full thread group, i.e. a whole multithreaded application
    + ```_exit()```: terminates a single process, regardless of any other process in the thread group of the victim. The main function inside is ```do_exit()```.
    + In ```do_exit()```, we added an if-statement to check if the current process is eagerpaging process, if yes, then calls ```clear_eagerpaging_process()``` function on ```current->comm```.






## Eager paging porting:
Eager paging is a bit different from demand paging.
### Core idea of demand paging:
User space code has a set of virtual addresses, and when user accesses those set of virtual addresses, if the pages corresponding to those virtual addresses are not in memory, it will cause page faults. Hence, in order to solve that, need to find some free pages, add page table entries to the page table, then the user mode code can continue.

### Core idea of eager paging:
Instead of triggering page faults, when you allocate virtual addresses (using malloc or mmap()), you also allocate physical pages corresponding to the virtual addresses. Since the target is to minimize the address translation overhead, eager paging allocates the physical pages contiguously.

The buddy allocator is used by Linux to perform memory allocations. The buddy allocator maintains a list of free physical pages. Such info could be retrieved by looking at ```/proc/buddyinfo```:
```
$ cat /proc/buddyinfo

Node 0, zone      DMA      1      0      0      1      2      1      1      0      0      1      3
Node 0, zone    DMA32      0      1      3      2      5      5      5      4      3      1    963
Node 0, zone   Normal    297    227    156     86     55     20      9      9      6      2   1318
```
Starting from the fourth column (the numbers), it shows the number of pages available for various sizes (4KB, 8KB, 16KB, 32KB, ...). Hence, for zone Normal, in this case, it contains 297 4KB pages (since page size is 4KB), 227 8KB blocks, 156 16KB blocks, etc.. There are a total of **11** columns of numbers (```MAX_ORDER``` in ```include/linux/mmzone.h```), so the last column represents the number of 4MB blocks, which is what is by default in the Linux kernel. We enlarge it to be 20, so that the largest size of the blocks buddy allocator can keep track of is 2GB.

```populate```: means allocate physical memory and create the page tables.


### Enable greater buddy allocator MAX_ORDER --> required for eager paging's contiguous block allocations
+ linux-5.16.15/arch/riscv/include/asm/sparsemem.h
    + ```SECTION_SIZE_BITS```: the number of bits of the size of each section of memory (Sparse memory model).
    + [resource](https://lore.kernel.org/lkml/1465821119-3384-1-git-send-email-jszhang@marvell.com/)
    + [resource](https://docs.kernel.org/vm/memory-model.html)
    https://www.tsz.wiki/linux/memory/common/modle/modle.html#_0-%E6%A6%82%E8%BF%B0
    https://lwn.net/Articles/789304/
+ linux-5.16.15/include/linux/mmzone.h
    + ```#define MAX_ORDER 20```: previously 11. It defines the maximum order of contiguous physical pages that could be requested by kernel buddy memory allocator. 11 means 4MB blocks, and 20 means 2GB blocks.


### Adding a syscall to register a process as eagerpaging process.
We need to add a syscall so that we could register a specific process to use eager paging instead of normal paging (demand paging)


[resource](https://www.kernel.org/doc/html/v4.10/process/adding-syscalls.html#:~:text=several%20other%20architectures%20share%20a%20generic%20syscall%20table.%20Add%20your%20new%20system%20call%20to%20the%20generic%20list%20by%20adding%20an%20entry%20to%20the%20list%20in%20include/uapi/asm%2Dgeneric/unistd.h%3A)

Eager paging routines in place only for ANONUMOUS mappings. mm_eager(……) --> allocates large contiguous buddy blocks and pre-populates mappings (similar to mm_populate but with contiguity in place)