# Arm Neoverse V1 Core

The micro-architecture details of the Arm Neoverse core, which is a super pipelined super-scalar processor which has an in-order frontend and out-of-order backend.

![Screenshot 2025-04-28 144542](https://github.com/user-attachments/assets/9a57d3e3-46dc-4a50-9f09-2fbb57c1a6c1)

![Screenshot 2025-04-28 164557](https://github.com/user-attachments/assets/a9c0b7da-591e-4038-9938-6fbcee4ff24b)

To Tune an application code : 
 Detects the code bottleneck which is where most of the cycles are spent.
 Measure micro-architectural metrics that help deep dive into the bottlenecking CPU component for further analysis  

Stage 1 : Top Down Analysis : Hotspot Detection 
  Helps to characterize the distribution of cycles spent by the processor.  
  Locate Bottleneck.
  Percentage of the total execution bandwidth of the processor : 
  - Operations retiring    :  This indicates the cycles that were used and efficient.
  - Branch Misspeculation  :  Executed operations that didn’t retire due to a pipeline flush. The inefficient cycles spent for recovering from the mis-speculation, refilling the pipeline from the correct location.
  - Frontend Stalls        :  stalled due to resource constraints in the frontend unit.
  - Backend Stalls         :  stalled due to resource constraints in the backend unit.

The total execution bandwidth of the processor can be measured in execution slots for operations.  
Micro-architectural parameter  : The number of slots supported by the core determines the execution bandwidth of the processor for top-down accounting. 

Stage 2 : Micro-architecture Exploration
  Analysis of the bottlenecking CPU resource.  
  A set of CPU resource effectiveness metrics in metric groups per resource.  
  Misses Per Kilo Instructions (MPKI) .
  Miss Ratios


Neoverse V1 PMU Events Selection for Workload Characterization

![image](https://github.com/user-attachments/assets/447ed595-18de-45c3-8ce9-ed2eff4b59af)

![image](https://github.com/user-attachments/assets/8b862c03-9f8c-436d-8738-bd67680d3030)

Arm recommends collecting all the metrics that are in Stage 1 and Stage 2 for workload characterization.   

![Screenshot 2025-04-28 150524](https://github.com/user-attachments/assets/a795bc1a-6ca0-42be-897a-c066c5c00aaa)

## Stage1 : Top Down Analysis

![Screenshot 2025-04-28 151243](https://github.com/user-attachments/assets/679b85d1-b792-485e-af72-2885bc171ae0)

![Screenshot 2025-04-28 151318](https://github.com/user-attachments/assets/cc8eb741-92bf-439f-be6b-be00c0c0ee12)

**UStress:** Micro-architecture Metrics Validation Workload Suite

Top-down level 1 metrics provide clear indication of the bottlenecking part of the processor pipeline. :  Locating the bottleneck or hotspot of the program
  
To stress some of the major CPU resources  : 
- **Branch:** branch_direct_workload, branch_indirect_workload, call_return_workload.
- **Data Cache:** l1d_cache_workload, l2d_cache_workload.
- **Instruction Cache:** l1i_cache_workload.
- **Data TLB:** l1d_tlb_workload.
- **Arithmetic Execution Units:**    div32_workload,  fpdiv_workload,  mul64_workload
- **Memory Subsystem:** memcpy_workload, store_buffer_full_workload, load_after_store_workload  (RAW Hazard)

![Screenshot 2025-04-28 151805](https://github.com/user-attachments/assets/338e455d-f397-41df-a0ed-994cd819c4b0)

From above following Observations can be made : 
- **Branch Tests                    :**  Different branch types result in pipeline flushes ----->  bad speculation-related stalls :  high percentage on bad_speculation metric (34% ~ 63%)
- **Data Cache Tests                :**  Stall the backend of the processor. Show a high percentage of backend_ bound metrics (>90%).
- **Instruction Cache Tests         :**  Stalls the frontend of the processor.  frontend_bound metric (58%).
- **Data TLB tests                  :**   Heavy data TLB miss that would cause stalls in the processor’s backend caused by delays in the memory address translation stage.  backend_bound metric (61%).
- **Arithmetic execution unit tests :**  Process various arithmetic operations of different latency requirements.  Backend_bound metric (55% ~ 89%).
- **Memory subsystem Test           :** Stress on the Load Store Units. High percentage of backend_bound metric (63% for memcpy_workload, 52% for store_buffer_full_workload). load_after_store_workload(RAW) results in high bad_speculation (36%) because the code triggers many speculative loads due to abandoned due to mispredicted data addresses. The test memcpy_workload copies memory block smaller than L1D Cache efficiently in batch, which results in high retiring. 


## Stage 2 :  Micro-architecture Exploration

- **High frontend_bound metric :** The pipeline stalls in the in-order frontend division of the processor. The branch prediction unit, fetch latency due to instruction cache misses and translation delays caused by Instruction TLB walks. (ITLB metrics)
- **High backend_bound metric  :** The pipeline stalls in the processor’s backend. Execution units, data cache misses and translation delays caused by data TLB walks.
- **Bad Speculation metric     :**The pipeline stalls caused by flushes or machine clears that break the pipeline needing a control flow change.  Branch mis-predictions.
- **High retiring metric       :**  Inefficiency in terms of underutilization of the micro-architectural capabilities.


### MPKI – Misses Per Kilo Instructions 
Misses Per Kilo Instructions is a set of metrics that can be derived to normalize the misses in CPU components, mainly branches, caches and TLBs, against the total instructions executed.   
This helps with comparison across different implementations  of CPU Architecture. 

![Screenshot 2025-04-28 153805](https://github.com/user-attachments/assets/57d6d720-254f-4db7-8f53-4d80275444d8)


The takeaway points from the above results : 
- The branch tests show relatively high branch MPKI values.
- The data cache tests relatively high L1D MPKI and L2 MPKI for tests l1d_cache_workload and l2d_cache_workload respectively, along with some pressure in the frontend.
- The instruction cache test shows relatively high L1I MPKI and branch MPKI, matching the expected behavior for a frontend_bound workload.
- Explore the branch and L1I cache effectiveness metrics further to determine the root cause.
- The pressure in one CPU resource can cause pressure in other components. Example : The L1I_MPKI > 1000 --------> L1I_CACHE_REFILL  is greater than INST_RETIRED. In this case check the INST_SPEC against the INST_RETIRED.
- Data TLB test shows relatively high L1D TLB MPKI.
- Arithmetic execution unit tests show very low MPKI. This doesn't matter much as this core is bound in the backend.
- Memory subsystem unit tests show low MPKI as the test data always hits the L1D cache.

### Miss Ratio : 
To calculate the ratio of the misses in the CPU components, mainly branches, caches and TLBs, against the total accesses in those components.  
The efficiency of each CPU component in the pipeline and help to root cause issues.  

![Screenshot 2025-04-28 154904](https://github.com/user-attachments/assets/6b36ddaa-c19f-4bea-82ec-41950ea202aa)

The Takeaways point from this : 
- Branch tests show relatively high branch mis-prediction ratio (30% ~ 50%) values.
- Data Cache tests show high L1D cache miss ratio (>95%) for l1d_cache_workload and high L2 miss rate for l2d_cache_workload.
- Instruction Cache test l1i_cache_workload shows relatively high L1I cache miss ratio (>80%) and high branch misprediction rate (74.82%) as expected.
- Data TLB test l1d_tlb_workload shows relatively high L1D TLB miss ratio (>95%), as expected.
- Arithmetic execution unit tests mostly show high L2 cache miss ratio (~18%), but the corresponding MPKI is very low. The high miss ratio is due to very few L2 accesses, but not a true bottleneck.
- L2 miss ratio of the memory subsystem tests are due to few L2 accesses, not high misses. 



The Neoverse V1 Micro-architecture has a variety of execution units that can process five types of operations:  
- Branch.
- Single-cycle Integers.
- Multicycle Integers.
- Load/Store Unit with AGU.
- Advanced Floating-point/SIMD operation.

![Screenshot 2025-04-28 155801](https://github.com/user-attachments/assets/01a673ef-1b0f-4a54-809e-49991d3ebcc2)


From above Figure following conclusion can be made : 
- **Branch operations               :** into immediate, indirect, and return branches, counted by events BR_IMMED_SPEC, BR_INDIRECT_SPEC, and BR_RETURN_SPEC respectively.
- **Branch Instruction              :** Mis-prediction Test :  (2.41% ~ 11.67%). Inspecting assembly code shows there should at least be 10% ~ 12.5% (1/10 ~ 1/8) branch instructions. Neoverse V1 supports both INST_RETIRED and BR_RETIRED events to compare the ratio of retired branch instructions against total retired instructions.
- **Data Cache tests                :** This shows a large proportion of load operations (32.41%) as well as integer operations (34.77%) as expected.
- **Instr Cache tests               :**  The high scalar integer operations (>80%). The test continuously incrementing and dereferencing a function pointer, which are integer operations. The speculated integer operations are much higher than the retired ones.
- **Data TLB test                   :** shows relatively high scalar integer operations and a mix of loads and branch operations (integer_dp=66.62%, load=16.61%, branch=16.73%).
- **Arithmetic execution unit tests :** (mul and div tests) show high scalar operation percentage (83.3%) for scalar integer tests and high floating point operations (66.5%) for the FP tests (fpmul and fpdiv). For tests that are double to integer conversion, we see a mix of scalar fp (40.0%) as well as scalar integer (40.0%) as expected.
- **Memory subsystem test           :** memcpy_workload has a greater proportion of load and store operations, as expected. For the store_buffer_full_workload test, high store (32.5%) is observed while the load_after_store_workload test shows a high proportion of load operations (42.2%). 




