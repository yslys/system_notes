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



## Program Execution
While going through the commit history of eager paging, there is one step that added a syscall to register a process as eager paging candidate, so that user space program can run the process as eager paging.
+ linux-5.13.11/arch/x86/entry/syscalls/syscall_64.tbl:
    + This added a new entry of eagerpaging to the syscall table.
+ linux-5.13.11/fs/exec.c:
    + This file contains the ```execve()``` syscall implementation, which is closely related to program execution.
    + The function ```begin_new_exec()``` is modified. After checking the [patch](https://lkml.org/lkml/2020/5/5/1216), the older version function is ```flush_old_exec()```. Fortunately, this function is mentioned in the book ULK ("The exec Functions" in Chapter 20: Program Execution).
    