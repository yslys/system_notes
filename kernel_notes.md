# Kernel Notes

## Terminologies:
+ BIOS: Basic Input/Output System
+ GDT: Global Descriptor Table
+ LDT: Local Descriptor Table


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
    + What does BIOS contain?
        + 

## mm
