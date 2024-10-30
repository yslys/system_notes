## Systems Architecture
- CPUs have multiple memory channels that allow for parallel access to RAM modules. 
- NUMA: In multi-socket systems, each CPU may have its own local memory bank, and accessing remote memory is slower.

## Memory
- Taking memory offline: reserve memory region for specific workloads.
  - Aim: reduce contention, increase memory bandwidth
  - How: reserve regions for a specific application, so that no other applications access that memory region to introduce contention.
  - Q&A: [link](https://chatgpt.com/share/6722aa46-8e58-8005-b89d-685a9ad178cc)
