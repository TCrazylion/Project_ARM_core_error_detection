# CPU Error Detection on ARM Platforms

<p align="right">June 10 , 2020</p>

<p align="right">Group Member: Runzhe Tang , Jiayu Zhang</p>

## **1.** **Introduction**

​		All the time, the security and normal operation of computer system depend on software and hardware. However, most current approaches to improving computer systems focus on the software stack. In general, hardware, especially the processor, is treated as a black box, and all results from the processor are unconditionally trusted to be true. But the unconditional trust is not safe. There is still the possibility of hard- ware failure in the life cycle of the existing CPU,such as that the result of the execution of the CPU instruction is not as expected (for example, arithmetic instruction calculation error and status identification bit error).The CPU hardware failure will have a direct and unpredictable impact on the programs running on itself. However, the most widely used ARM cortex-A processor architecture for mobile platforms does not include error detection processing technology. The lack of error detection processing technology makes it impossible for us to effectively detect CPU faults in the mobile platform, which seriously threatens the security and reliability of the platform.

 

## **2.** **Previous Work**

### CPU core error

- #### Definition

  ​		Before investigating how to detect CPU core errors and find effective methods to solve them. We should have a clear idea of the definition of CPU core errors—CPU core can not work properly, it hangs or crashes and can't continue executing instructions. Then I will give several significant cases of CPU failures.

- #### Single Event Effect

  ​		First,high potential and low potential—namely 1 and 0—of the circuit is determined by the accumulated charge on the capacitor. Any form of radiation will harm the unprotected circuit. Then if there are cosmic rays, radiation, sunspot bursts or radioisotope produced during the chip packaging material decay, high-energy particles cause electrons to escape in the form of bombardment or ionization, Single Event Effects（SEE） occurs, causing a CPU soft failure or even a hard failure. Meanwhile unstable voltage and current and circuit noise which may cause clock frequency jitter are also similar hardware failures.

  1.  Soft failure: A binary value "0001 0111" is stored in memory, one bit of it changes because of SEE and becomes "0010 0111"。
  2. Hard failure: Important control bit of CPU core changes coincidently, or soft failures accumulate without correction and eventually the CPU core can't work properly. 
  3. There are millions of transistors in a modern PC,It may only require one transistor to fail before a CPU stops functioning. Very lucky, single-particle incident produces transient current causing equipment damage or single-particle energy level is too high and directly breaks down the circuit. The chance of these two situation happens is too low to take them into account in our daily life. Designers in aerospace industry will usually consider them.

  ​		Then an example will be given to show what will SEE cause and SEE is indeed a problem worthy considering. In 2004, some servers using FPGA chip produces by Xilinx crash randomly. The reason behind it is the radioisotope produced during the chip packaging material decay. The malfunctioning Xilinx FPGA chip uses an inverted package process, and Flip-Chip solder balls are only a few microns away from the transistor's active area on the wafer. A small amount of radioactive isotopes in solder (lead-tin alloy) will undergo alpha decay. This alpha particle will produce a single-particle effect in the circuit, causing soft failures, and eventually causing the server using the chip to crash randomly.

  

- #### Invalid instructions

  ​		Here is a famous bug caused by invalid instruction: Intel f00f Bug. This hexadecimal instruction "f0 0f c7 c8" equals to assembly instruction "lock cmpxchg8b %eax", which can be executed with any level of authority.

1. cmpxchg8b：Compare and swap two 64bit value, and the target should only be values in memory, which means temporary register like eax are not allowed.

2. lock: can only used for "read-modify-write"instruction in memory,so "cmpxchg8b"is not a valid one。

   ​		Ideally, any error above will cause invalid opcode，and then CPU will execute error handling。However, when CPU runs "f0 0f c7 c8" , situation following will happen：

1. Execute" cmpxchg8b %eax">Invalid instruction>invalid opcode>error handling

4. Execute "lock" by mistake

   ​		After executing "lock"，CPU will release "lock" after "write" is executed, however, no executable"write" instruction ，CPU lockup itself.

 

### **Lockstep**

- #### Definition

  ​		Lockstep is a fault-tolerant system using computer redundancy design: multiple CPUs execute the same set of instructions simultaneously, output the results into a comparison logic, and periodically compare whether the results are the same. If they are the same, CPUs continue to run; if they are different, take some measures to diagnose the faulty one and prevent the fault from occurring in time.

- #### Dual-redundant Core Lock-step (DCLS)

  - ##### Operating principle

    ​		A typical Lockstep processor system includes two independent CPUs, usually one is the main CPU, which is responsible for outputting results, and the other is the sub-CPU, which is responsible for detecting the correctness of the output of the main CPU. Meanwhile, it requires fault detection and recovery functions, such as multiple pre-stored instructions that produce known results. When a failure is detected, two CPUs execute pre-stored instruction to determine which CPU fails. If the main CPU fails, it shuts down and the sub-CPU responsible for detection is allowed the output results,namely becomes the main CPU temporarily. Similarly,if the sub-CPU fails, it turns off too. Confirming the emergency response mechanism executed, the faulty CPU then restarts or calls the internal fault repair tool for troubleshooting. If the fault CPU is repaired, it is re-enabled. Otherwise, a fault alarm is prompted and manual troubleshooting or repair is required.

    ​		Some products in the Cortex-A and Cortex-R series have integrated technology similar to lockstep, "Split-lock" ."Split-Lock" can be configured in two modes: 'split mode', two CPUs can execute independently different programs or tasks, so that the processor achieves higher performance; 'lock mode', two CPUs run under lock-step mode, making the processor safer and less prone to error. Meanwhile,if one of the two CPUs fails and can not recover, it can continue to run in degraded mode—only the good CPU will continue to run.

  - ##### Typical considerations

    1. Reset all registers

       ​		A crucial requirement for DCLS is that both cores need to be initialized with the same state. In other words, all registers in the processor should be reset to guarantee the same initial state.
       ​		In many processor designs, the hardware designer may deliberately keep the state of some registers, such as architectural registers, from reset to reduce the power consumption and silicon area. In DCLS, however, non-initialized values from non-reset registers might potentially propagate to outputs, causing false mismatch. To allow DCLS functioning correctly, resetting all registers in the processor by either hardware or software scheme is usually needed.

    2. Avoid common mode failures

       ​		DCLS cannot detect potential failures that can occur at the same point in both cores since the failures do not cause any difference between their outputs. These failures are referred to as common mode failures, which cause false match in the DCLS system.

       ​		Several techniques have been proposed to address common mode failures. One of them is providing temporal diversity to two cores. A common approach to this is delaying the redundant core for few cycles by inserting shift registers into the inputs. With a temporal diversity of even a few cycles, it is less likely that an erroneous trigger occurs at the same point of two cores. 

       ​		Another technique to avoid common mode failures is implementing two cores in different ways. Hardware designers may choose different types of arithmetic logic units (ALU), block implementations or physical designs to implement the redundant core. This keeps the same functionality of two cores but reduce the risk of failures caused by the same erroneous transient pulse from power or signal interface.

  - ##### Specific Module

    ​		Then some modules of lock-step will be introduced to make readers have a clearer view of lock-step. Attention: Circuit designer create different modules for various purposes, modules introduced below are only typical examples of them.

    - Module 1

      ##### <img src="https://i.loli.net/2020/06/10/58P1S6LtuZ3xcOR.png" alt="Fig.1" style="zoom:67%;" />Fig.1

      ##### <img src="https://i.loli.net/2020/06/10/PLy837cDAHjbKSl.png" alt="Fig.2" style="zoom:67%;" />Fig.2

       

      ​		Referring to FIG. 1, in accordance with the teachings of this invention, a receiver function 10 serves to detect and isolate faults in the control outputs of lock-Step processors 1 and 2. A common clock Source (not shown) is coupled to the processors 1 and 2 and to the receiver function 10. Each of the lock-Step processors generates a parity bit for its control output, which parity bit is transmitted to the receiving function along with its control output. Referring now to FIG.2, the receiver function 10 receives each control output and its parity Signal independently. Parity logic 20 detects any parity error in the control output of processor 1 and parity logic 22 detects any parity error in the control output of processor 2. The outputs of parity logic 20 and 22 are coupled as inputs to error detection and isolation logic 24, whose outputs include a control output Select Signal coupled to Selector 26, a processor 1 disable output, a processor 2 disable output, and a lock-Step disable output. In addition to the parity check logic, the receiver function 10 includes bit compare logic 28 which performs a bit-by-bit comparison of control outputs of lock-Step processors 1 and 2 The output of the bit compare logic 28 is coupled as another input to the error detection and isolation logic 24. If the error detection and isolation logic 24 generates errors , it will send a signal to tell the Selector 26 not to output any result from the processor to make sure that the error will not corrupt other parts of the system. Meanwhile the error detection and isolation logic 24 will set error mode to 1, send a signal to disable lock-step and then use a build-in test set to find which processor fails. After getting the results of the test set ,it will decide to disable processor 1 or processor 2, and if the master processor fails, the system will switch master automatically,running in degraded mode at this point, and the logic 24 will set the recovery mode to 0 and tell the Selector 26 to output the result of current master processor.

    - Module 2

      ![image-20200610165902238](https://i.loli.net/2020/06/10/jXeZxtRS8LphwC2.png)**Fig.3**

      ![image-20200610165928041](https://i.loli.net/2020/06/10/ZzKo2cwTEmkhBnr.png)**Fig.4**

      ![image-20200610170124139](https://i.loli.net/2020/06/10/q1L9OsvceWxarB7.png)**Fig.5**

      Structure of SM (Store Memory) :

      - index

        The index is the low 9 bits of the read/write address and serves as the address space of SM

      - ADDR(address)

        ADDR is the 23 higher bits of the address. The address bit and the index bit constitute the address corresponding to the read and write instruction

      - data

        Data,  is the data corresponding to the address

      - valid

        The valid bit, is the valid position 1 of the index in SM for each write operation

      (4-way group association: the SRAM with the same 4-way structure is associated with the same index, and ADDR in the 4-way is compared after the index hit, in order to increase the hit of read operation)

      (In the running state, the processor writes to SM and does not change the memory to ensure that the recovery operation takes up as little system resources as possible)

      At time T0, the two processors runs the instruction at the same time. At the end of T0, it enters the save state of T0 and saves the register state of T0.

      At time T0-T1. Since there is no write operation during this time, the memory of the processor at time T1 is the same as that at time T0, and the state of the register at the end of T1 is saved by the processor.

      At time T1-T2 , there is a read operation, but no write operation, the processor at the end of T2 save state to save the T2 register state.

      At time T2-T3, there is a write operation, followed by a read of the address. In the running state of T3, it will write the address data into SM while keeping the memory unchanged. If a subsequent read operation hits the address of the last write operation, the data for that read operation is extracted from SM and given to the dual processor.

      If the read operation does not hit SM, the data of the read operation is returned directly from memory.
      
      If there are other write operations on SM until SM is full. The processor saves the state directly.
      
      If an error occurs in the T2-T3 segment, the processor's PC jumps back to the saved value at T2, and the registers saved at T2 will be written to the processor's register. At this time, the memory is not saved in the saved state, and the processor's memory state is still the memory in T2 state. The hardware will clear the VALID bit of SM to zero, making the data in SM invalid.
      
      If run to a saved state at the end of T3, the processor will save the registers at T3, and at the same time, it will write the data in SM into memory.
  
- #### Elementary Experiment

  - Hardware
  
  For a chip designer, this is a design-time decision to place two copies of the core logic to establish a Dual-redundant Core Lock-step. However as a software engineer, what we need to do is to find a device that already supports this hardware configuration. That is a member of Hercules series ——TMS570LC43x.
  
  ![image-20200610182406357](https://i.loli.net/2020/06/10/o3sNdQFGS8rZgpf.png)**Fig.6**
  
  - Software
  
    HALCoGen 4.0.7(HAL Code Generator tool)
  
    CCS 8.3.1(Hercules Emulation 6.0.7 and TI Emulators 5.1.641.00)
  
    Hercules SafeTI™ Diagnostic Library 2.20
  
    Other software and material which can be download from www.ti.com and we are not sure whether previous version of these software can work properly or not.
  
  - Content
  
    We write a simple program based on our research of HALCoGen and offical demo to show how lockstep works. The push-button USER SWITCH B will inject a core compare error (CPU mismatch). An on-chip monitor will detect the fault and trigger an error signal causing the ERR LED to light up.
  
    Code below call an official API from Hercules SafeTI™ Diagnostic Library 2.20, which will inject a CPU mismatch error.
  
  ```c
  SL_SelfTest_CCMR5F(CCMR5F_VIMCOMP_ERROR_FORCING_TEST_FAULT_INJECT, TRUE, &failInfoCCMR5F);
  ```
  
  

### **BIST**

- #### Introduction

  ​		Automatic test equipment. A test system that provides a test mode for a test chip and analyzes the chip's response to the test mode to determine whether the chip is good or bad.The test system has one or more test headers containing the circuit to the chip under test. In the test process, the excitation (test vector) is added to measure the chip response output (response), compared with the predicted result in advance, if it is consistent, the chip is good.
  ​		However, the cost of ATE detection was relatively high, so later came BIST, a technology that provides self-test function by embedding relevant functional circuits into the circuit at design time.Compared with ATE detection, the detection speed is fast, the cost is lower than ATE, and the test coverage is large.

  ​		This often requires the integration of additional circuitry and functionality into the design of the circuitry to facilitate self-testing of functionality. This additional capability must be able to generate test patterns and provide a mechanism to determine the output response of the circuit under test (the cut-to-test pattern corresponds to the output response of the fault-free circuit).

- ####  Architecture

  ​		Below is the representative structure of the BIST circuit. The BIST architecture includes two basic functions and two additional functions that are required to perform self-test functions in the system. Two basic features include a test pattern generator (TPG) and an output response analyzer (ORA).When TPG generated a series of patterns for the test CUT, ORA compressed the output response of the CUT into some type of pass/fail indication. Two other features required to use BIST at the system level include a test controller (or BIST controller) and an input isolation circuit. In addition to normal system 1/0 pins, the addition of BIST may require additional IO pins to activate the BIST sequence (BIST startup control signal).Report the results of BIST (pass/fail indication)

##### ![image-20200610212622922](https://i.loli.net/2020/06/10/37eWzOEY6tVw41U.png)Fig.7

- ####  Work Flow

  ​		Activate the BIST sequence by setting BIST Start to logic 1, in which the counters Start counting in sequence. Addressing and reading input test mode from the TPG ROM and expected output response of the corresponding ORA as input test mode from TPG ROM is applied to reduce by input isolation multiplexer, read the output response expected from the ORA comparator will produce logic 1 with any mismatch between expectations and the actual output response, it will be high set input through activities in the output of the pass/fail RS latch logic 1.

   
  
  ![image-20200610212713053](https://i.loli.net/2020/06/10/NBi92Wv6ayPHl3Y.png)**Fig.8**
  
  ​		This is especially true when the set of test vectors and the number of bits and vectors for the expected output response are large. This approach has other drawbacks, too. A minor design change to the CUT may require the regeneration of the test vector and the expected output response. Reprogram the ROM at best and resize the ROM and counter at worst. As a result. Minor design changes to the CUT may require major changes in the BIST implementation. In addition, the comparator is not fully tested in the BIST sequence, requiring additional tests to ensure that the comparator is fail-free .

- ####  Test Pattern Generation

  ​		One approach is to store a good set of test patterns (from the ATPG program) in a ROM on the chip, but this is very expensive in the chip world.

  ​		The LFSR USES the Linear feedback Shift Register (LFSR) to generate pseudo-random tests.This usually requires a million sequences or more tests to achieve high fault coverage, but this approach USES very little hardware and is currently the preferred method of BIST schema generation.

  ​		Pseudo-random test methods are suitable for composite circuits and sequential circuits, but they must have fault simulation, so the test length and fault coverage must be determined.There are usually three different ways to override random and unpredictable failures.A deterministic test generation algorithm is used to generate these random undetectable fault test codes, and these test codes are stored in ROM.The second is to insert testable improved logic for hard-to-detect faults in the circuit.The third is to put weighted random test code into the circuit to make the random difficult fault easy to be measured.

- #### Response Compression

  ​		To determine if the response from the CUT circuit is faulty, we need to compare the response output with the response output of a failure-free circuit. If we compare one bit to another, it will be very inefficient. Therefore, a good method is to compress the response output into a signature, and then compare and judge by the signature.

  ![IMG_256](https://i.loli.net/2020/06/10/LyWjhdJzsZFO6Ib.gif)**Fig.9**

  ​		We need to distinguish between compression and compaction

  ​		Circuit response compression is lossless, because the original output sequence can be completely regenerated from the compressed sequence .

  ​		Compaction, results in information loss, so regenerating the original circuit response in formation is not possible.

  ​		Compression schemes, at present, are impractical for BIST response analysis, because they inadequately reduce the huge volume of data, so we use only compact ion schemes. In mathematical words, compression functions are invertible, but compaction functions are not. Signature analysis is the process of compacting the circuit responses into a very small bit length number, representing a statistical circuit property, for economical on-chip comparison of the behavior of a possibly defective chip with a good machine chip. The signature must preserve as much of the fault information contained in the circuit output response before compaction as possible, and the circuitry used to implement the compacter should be small. All compaction techniques require that the fault-free circuit signature be known.

- #### Advantages & Disadvantages

  ![5](https://i.loli.net/2020/06/10/3fvBUgZCHQiFuKO.gif)**Fig.10**

  ​		The advantages and disadvantages discussed so far are summarized in Fig.5 .Despite the drawbacks, most BIST application case studies show that the benefits of BIST almost always compensate for the costs of BIST (including additional design work, increased chip area, and potential risks to the project).This is especially true when the BIST circuit is designed for off-line system-level testing in the system.

  

### IScanU

- #### Introduction

  ​		We also read a paper on undocumented instruction testing. We find that the instructions in architecture of X86 have no fixed length, while the in the RICS architecture they have fixed length , which results that the X86 instruction detection mechanism can not be applied to RICS. The core idea of this paper is that , the processor behavior at the execution of the instruction word is compared to the behavior specified by ISA, and the operating system signals are used to determine the validity of the instruction word. When an invalid command word is executed, the operating system sends a SIGILL signal to the faulty process.

![image-20200610213313095](https://i.loli.net/2020/06/10/UKXw5Qh7LdAq4yS.png)**Fig.11**

 

- #### An inappropriate case

  ​		There is a very old Intel Pentium CPU Floating-Point Bug. When we perform the following operation,

  RTfloat fValue = 4195835.0f - ( ( 4195835.0f / 3145727.0f ) × 3145727.0f );

  RTdouble dValue = 4195835.0L - ( ( 4195835.0L / 3145727.0L ) × 3145727.0L );

  ​		And we will find that on Intel 486 CPU the result is 0 while on Intel Pentium CPU the result is 256 , and on all the rest Intel CPUs the result is 0.However , the can not be viewed as CPU error.

 

## **3.** **Responsibility**

​		In early work, Runzhe Tang found some significant cases of CPU errors that gave us an preliminary understanding of the nature and causes of CPU errors, as well as their serious consequences. Jiayu Zhang had read some papers on CPU errors detection to get a preliminary understanding of errors detection. We also read the armv8-m manual to get some idea of the cortex-m series RAS. In the middle stage, Runzhe Tang is responsible for the understanding and analysis of Lockstep, Jiayu Zhang is responsible for the BIST part. Finally, we focus on the experiment together.



## **4.** **Future Work**

1. We will work together on the BIST and lockstep part.
2. We will focus on having a deeper understanding of TMS570LC43x using HALCoGen and CCS.



## 5.References

[1]:Chen Hao1(AVIC Xi’an Flight Automatic Control Research Institute,Xi’an,710065,China),"Research on Processor Lockstep Technique",Digital Technology & Application,2012,08

[2]:R. R. Collins, “The intel Pentium f00f bug description and workarounds,” Doctor Dobb’s Journal, 1997.

[3]:M. Lipp, M. Schwarz, D. Gruss, T. Prescher, W. Haas, A. Fogh, J. Horn, S. Mangard, P. Kocher, D. Genkin, Y. Yarom, and M. Hamburg, “Meltdown: Reading kernel memory from user space,” in USENIX Security Symposium 2018, 2018, pp. 973–990.

[4]:Xilinx White Paper 208 (2004)3

[5]:《A Designer’s Guide to Built-In Self-Test》, Charles E. Stroud, University of North Carolina at Charlotte

[6]:《Error detection and fault isolation for lockstep processor systems》,United States Patent, US5915082.