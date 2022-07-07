## Motivation
While doing research, I have to simulate a PCIe device, and add it to the riscv board. However, mainline gem5 only has one PCIe device, named CopyEngine, and there is no existing configuration (the riscv board configuration) that has such PCIe device connected. Therefore, I need to figure it out myself. This file has two purposes:
1. Introduction and explanation of PCI-e devices.
2. How to add a PCIe device to gem5.

