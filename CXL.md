## Systems Architecture
- CPUs have multiple memory channels that allow for parallel access to RAM modules. 
- NUMA: In multi-socket systems, each CPU may have its own local memory bank, and accessing remote memory is slower.

## Memory
- Taking memory offline:
  - Taking memory offline for specific workloads is a strategy to increase predictability, reduce contention, and tailor memory access patterns, which can ultimately lead to better system performance, efficiency, and resource utilization.
  - in servers/HPC systems, we offline memory to reduce contention for memory bandwidth.
  - Resource Allocation for High Priority Tasks: Some workloads, like real-time analytics or machine learning model inference, are sensitive to memory latency and require predictable memory access times. By taking certain memory regions offline and dedicating them to these tasks, you ensure that these applications have exclusive access to the memory, reducing latency and improving response times.
  - Reducing Cache and Memory Contention: In shared memory systems, multiple applications and processes may contend for memory resources, leading to cache thrashing and inefficient memory utilization. Isolating memory for specific tasks mitigates this contention, enabling more consistent memory access patterns and reducing cache misses, which is especially beneficial in multi-core systems.
  - Improved Memory Bandwidth and Throughput: High-performance workloads, like large data analytics, require significant memory bandwidth. By taking certain memory modules offline for general usage, these applications can utilize the full bandwidth without interference, leading to higher throughput and lower memory access latency.
  - Optimizing for Non-Uniform Memory Access (NUMA) Architectures: In NUMA systems, taking memory offline from one node and dedicating it to local workloads on that node can reduce cross-node memory access latency. This NUMA-locality optimization is critical for workloads that benefit from low-latency access to memory close to the processor, as it minimizes the delays associated with accessing remote memory.
  - Tailoring Memory Characteristics for Workload Needs: Different workloads have distinct memory needs, like low-latency access for databases or large, high-bandwidth access for high-performance computing tasks. Taking memory offline for these workloads allows system administrators to optimize memory parameters—such as page size or memory access priority—according to specific requirements, making the memory subsystem more efficient for particular applications.
  - Reducing Power Consumption for Specialized Applications: Certain applications, like embedded systems or edge devices, benefit from lower power consumption by isolating and tuning specific memory banks. By taking other memory offline, these systems can limit power use to only the necessary resources, which is beneficial for energy-constrained environments.

