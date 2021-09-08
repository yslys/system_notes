## SoftNIC: A Software NIC to Augment Hardware

### 1.1 Problem:
- NIC h/w evolves slowly, while protocols, apps and system/network architectures evolve fast.
- One solution is through software - implementing NIC features in host network stack, but not good enough.

### 1.2 Paper solution: SoftNIC
- Hybrid software hardware architecture. 
- A programmable platform that allows apps to leverage NIC features implemented in software and hardware.

### 1.3 Paper result
- Single core: multi-10G performance
- Multiple cores: scaling. 

### 2 Introduction
1. Old NICs job: relaying traffic between a server’s CPUs and its network link
2. Current NICs job: hosting advanced NIC features (protocol offloading, packet classification, rate limiting, and virtualization). 
3. Why NICs job change?
    1. New applications with intense requirements - cannot be met by networking stacks - necessitating hardware augmentation; however, hardware evolves slowly.
    2. Virtualization - requires NICs to support switching functionality.
    3. Multi-tenant datacenters & Cloud computing - NICs must provide isolation mechanisms.
4. Problem: smart NICs - complex - support many features - hard to configure, compose or evolve.
5. Since the performance gap between hardware and software network processing getting smaller, the paper argues that a hybrid hardware-software approach can effectively augment or extend NIC features, while providing flexibility and high performance. 
6. SoftNIC:
    1. software-augmented NIC — serves as a fallback or an alternative to the hardware NIC. 
    2. serves as an incubator for next-generation hardware features, as well as a permanent home for features that are too complex for hardware implementation, too specialized to merit scarce NIC resources, or whose performance
needs are easily met with software.
7. Modern NICs offer high throughput (10–40 Gbps); low end-to-end latency (< 10 µs); guarantee performance; isolation among VMs or applications. 
8. SoftNIC must offer similar performance capabilities, but software implementation is hard. Why?
    1. Legacy kernels and hypervisors - slow and inefficient at network processing. 
    2. While specialized packet processing frameworks offer high performance, they lack the rich programmability, or does not guarantee performance.
9. Contributions of this paper:
    1. A new architecture for extending NIC features - developers can use this to develop features in software while incurring minimal performance overhead and leveraging features from the hardware NIC. Application developers can use SoftNIC as a hardware abstraction layer (HAL) to build software that uses NIC
features without worrying about cases where they are unavailable or incomplete.
    2. A scheduling framework - supports a wide range of operator-specified performance isolation policies, and provide fine-grained performance guarantees for applications.
    3. Several use cases of SoftNIC - showing how SoftNIC can improve the flexibility and/or performance of both existing and forward-looking applications.

### 3 SoftNIC design
SoftNIC is a programmable platform that allows applications to leverage software and hardware implementations of NIC features. SoftNIC is implemented as a shim layer between applications and NIC hardware, to achieve the best of both the flexibility of software and the performance of hardware.

### 3.1 Design Goals

