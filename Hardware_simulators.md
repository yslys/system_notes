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
