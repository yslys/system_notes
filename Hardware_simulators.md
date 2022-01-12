## Computer Architecture Performance Evaluation Methods (Chapter 5)

### Tradeoffs in simulation
There are mainly 4 types of simulation:
1. Accuracy: the fidelity of the simulation model with respect to real hardware, i.e., how accurate is the simulation model compared to the real hardware that it models. 
2. Evaluation time: how long it takes to run a simulation. 
3. Development time: how long it takes to develop the simulator. 
4. Coverage: to what fraction of the design space a simulator can explore, e.g., a cache simulator can only be used to evaluate cache performance and not overall processor performance.

Increasing coverage -> more accurate -> more development time, more evaluation time.
Limited coverage -> accuracy is good for the component under study -> limited development time, and short evaluation time.

### 1. Functional simulation
Functional simulation models only the functional characteristics of an ISA — it does not provide any timing estimates.

+ Useful for validating the correctness of a design, rather than evaluating its performance characteristics (**poor accuracy** and **poor coverage**).
+ Very short development time.
+ Short evaluation time - no microarchitecture features need to be modeled.
+ Very long lifetime that can span many development projects. 

Functional simulation can generate instruction and address traces.

What is a trace?
+ A trace is the functionally correct sequence of instructions and/or addresses that a benchmark program produces.

What can we do with the traces?
+ These traces can be used as inputs to other simulation tools — so called (specialized) trace-driven simulators.

Example functional simulators: SimpleScalar’s sim-safe and sim-fast.
  
### 1.1 Alternatives of Functional simulation: Instrumentation (a.k.a Direct execution)
Instrumentation takes a binary and adds code to it - so that when running the instrumented binary on real hardware, the property of interest is collected.

E.g. generate a trace of memory addresses.
+ It suffices to instrument (add code to) each instruction referencing memory in the binary.
+ The code added does the following: compute and print the memory address.

Then, we can run the "instrumented binary" on native hardware -> get a trace of memory addresses.

What is the advantage of instrumentation compared to functional simulation?
+ Instrumentation incurs less overhead. 
  + Instrumentation executes all the instructions natively on real hardware; 
  + In contrast, functional simulation emulates all the instructions -> executes more host instructions per target instruction.

What is the limitation of instrumentation compared to functional simulation?
+ Instrumentation is less portable.
  + Reason: target ISA is typically the same as the host ISA.
  + Solution: using a dynamic binary translator to translate host ISA instructions to target ISA instructions (e.g. Shade).

Two types of instrumentation:
+ Static instrumentation: instruments the binary statically
  + e.g. Atom, EEL (used in the Wisconsin Wind Tunnel II simulator).
+ Dynamic instrumentation: instruments the binary at run time. 
  + e.g. Embra, Shade, Pin


### 1.2 Combination of Functional simulation and Instrumentation
**Instrumentation is fast, while functional simulation is portable.**

Combining those together: functional simulator synthesizer (by Burtscher and Ganusov). See Figure 5.2.
+ Input: 
  + a binary executable.
  + a file containing C definitions of all the supported insturctions.
+ Pass the above two to synthesizer:
  + translates instructions (in the binary) to C statements.
  + user can add simulation code to collect - e.g. a trace of instructions or addresses. 
+ Compiling the synthesized C code:
  + generates the customized functional simulator.

### 2. OS effects
**Functional simulation is often limited to user-level code only,i.e.,application and system library code,
however, it does not simulate what happens upon an operating system call or interrupt.** 

One common approach - ignore interrupts and emulate the effects of system calls. 
+ Manually identifying the input and output register, and memory state, to a system call and invoking the system call natively. 
+ Needs to be done for every system call - tedious and labor-intensive.

Narayanasamy et al. - a technique that automatically captures the side effects of operating system interactions. 
+ What is it?
  + It is an instrumented binary that collects how it changes **register state** and **memory state** for each system call executed (e.g. interrupt, DMA transfer).
  + The memory state change is only recorded if the memory location is later read by a load operation:
    + The instrumented binary maintains a user-level copy of the application’s address space. 
    + A system effect (e.g., system call, interrupt, DMA transfer) will only affect the application’s address space, not the user-level copy. 
    + A write by the application updates both the application’s address space and the user-level copy. 
    + A read by the application verifies whether the data read in the application’s address space matches the data in the user-level copy; 
    + upon a mismatch, the system knows that the application’s state was changed by a system effect, and thus it knows that the load value in the application’s address space needs to be logged. 
    + These state changes are stored in a so called system effect log. 
  + During functional simulation, the system effect log is read when reaching a system call, and the state change which is stored in the log is replayed, i.e., the simulated register and memory state is modified to emulate the effect of the system call. 

+ Because this process does not depend on the semantics of system calls, it is completely automatic, which eases developing and porting user-level simulators. 
+ A side-effect of this technique:
  + it enables deterministic simulation, i.e., the system effects are the same across runs. 

While this facilitates comparing design alternatives, it also comes with its pitfall.

### 3. Full system simulation
User-level simulation is sufficiently accurate for workloads that spend little time executing system-level code. However, for database servers, web servers, email servers, etc., these workloads spend a lot of time executing system-level code -> require full-system simulation. 

Also, the proliferation of multicore hardware -> multi-threaded workload -> performance is affected by OS scheduling decisions -> require full-system simulation.

**Full-system simulation**: 
+ Simulating an entire computer system such that complete software stacks can run on the simulator. 
+ The software stack includes: application software + operating systems, so that the simulation includes I/O and OS activity next
to processor and memory activity. In other words, the user of the full-system simulator is given the illusion to run on real hardware. 
+ E.g. SimOS, Virtutech’s SimICs, AMD’s SimNow, M5, Bochs, QEMU, Embra, and IBM’s Mambo.

Comparison between **full-system simulator** and **user-level functional simulator**:
+ The functionality is the same — both provide a trace of dynamically executed instructions.
+ The only difference: functional simulation simulates user-level code instructions only, whereas fullsystem simulation simulates both user-level and system-level code. 
+ Full-system simulation achieves greater coverage.

### 4. Specialized trace-driven simulation
Specialized trace-driven simulation:
+ Input: instruction traces + address traces (user-level + system level instructions)
+ Output: simulation of specific components (cache or branch predictor) of a target architecture in isolation.

E.g. cache simulation - Dinero IV.

+ Development time and evaluation time are both good.
+ Coverage is limited because only certain components of a processor are modeled. 
+ While the accuracy in terms of miss rate is quite good, overall processor performance accuracy is only roughly correlated with these miss rates because many other factors come into play. 

Nevertheless, specialized trace-driven simulation provides a way to easily evaluate specific aspects of a processor.

### 5. Trace-driven simulation
**Full trace-driven simulation** == **trace-driven simulation**. 

It takes program's instruction traces and address traces, and feeds the full benchmark trace into a detailed microarchitecture timing simulator.

It separates the functional simulation from the timing simulation. This is often useful because the functional simulation needs to be performed only once, while the detailed timing simulation is performed many times when evaluating different microarchitectures. This separation reduces evaluation time somewhat. 

+ Development time: long; Simulation run time: long. 
+ Accuracy: good; Coverage: good.

Disadvantage:
+ 1. Need to store the trace files, which may be huge for contemporary benchmarks and computer programs with very long run times. 
  + Solution: trace compression can be used to address this concern.
+ 2. For modern superscalar processors, they predict branches and execute many instructions speculatively — speculatively executed instructions along mispredicted paths are later nullified. These nullified instructions do not show up in a trace file generated via functional simulation, although they may affect cache and/or predictor contents. Hence, trace-driven simulation will not accurately model the effects along mispredicted paths.


### 6. Execution-driven simulation
In contrast to trace-driven simulation, execution-driven simulation combines functional with timing simulation. 

By doing so, it eliminates the disadvantages of trace-driven simulation: trace files do not
need to be stored, speculatively executed instructions get simulated accurately, and the inter-thread
ordering in multi-threaded workloads is modeled accurately. For these reasons, execution-driven
simulation has become the de-facto simulation approach. 

E.g. SimpleScalar, RSIM, Asim, M5, GEMS, Flexus, and PTLSim.

Although execution-driven simulation achieves higher accuracy than trace-driven simulation, it
comes at the cost of increased development time and evaluation time.
