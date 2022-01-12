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

+ Useful for validating the correctness of a design, rather than evaluating its performance characteristics (poor accuracy and poor coverage).
+ Very short development time.
+ Short evaluation time - no microarchitecture features need to be modeled.
+ Very long lifetime that can span many development projects. 

Functional simulation can generate instruction and address traces.

What is a trace?
+ A trace is the functionally correct sequence of instructions and/or addresses that a benchmark program produces.

What can we do with the traces?
These traces can be used as inputs to other simulation tools — so called (specialized) trace-driven simulators.

Example functional simulators: SimpleScalar’s sim-safe and sim-fast.
  
