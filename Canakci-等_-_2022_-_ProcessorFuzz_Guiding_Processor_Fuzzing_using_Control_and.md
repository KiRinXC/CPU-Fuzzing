# ProcessorFuzz: Guiding Processor Fuzzing using Control and Status Registers

Sadullah Canakci

scanakci@bu.edu

Boston University

Boston, USA

Chathura Rajapaksha

chath@bu.edu

Boston University

Boston, USA

Anoop Mysore Nataraja

mysanoop@uw.edu

University of Washington

Seattle, USA

Leila Delshadtehrani

delshad@bu.edu

Boston University

Boston, USA

Michael Taylor

Prof.taylor@gmail.com

University of Washington

Seattle, USA

Manuel Egele

megele@bu.edu

Boston University

Boston, USA

Ajay Joshi

joshi@bu.edu

Boston University

Boston, USA

## Abstract

As the complexity of modern processors has increased over the years, developing effective verification strategies to identify bugs prior to manufacturing has become critical. Undiscovered micro-architectural bugs in processors can manifest as severe security vulnerabilities in the form of side channels, functional bugs, etc. Inspired by software fuzzing, a technique commonly used for software testing, multiple recent works use hardware fuzzing for the verification of Register-Transfer Level (RTL) designs. However, these works suffer from several limitations such as lack of support for widely-used Hardware Description Languages (HDLs) and misleading coverage-signals that misidentify "interesting" inputs.

Towards overcoming these shortcomings, we present Proces-sorFuzz, a processor fuzzer that guides the fuzzer with a novel CSR-transition coverage metric. ProcessorFuzz monitors the transitions in Control and Status Registers (CSRs) as CSRs are in charge of controlling and holding the state of the processor. Therefore, transitions in CSRs indicate a new processor state, and guiding the fuzzer based on this feedback enables ProcessorFuzz to explore new processor states. ProcessorFuzz is agnostic to the HDL and does not require any instrumentation in the processor design. Thus, it supports a wide range of RTL designs written in different hardware languages.

We evaluated ProcessorFuzz with three real-world open-source processors - Rocket, BOOM, and BlackParrot. ProcessorFuzz triggered a set of ground-truth bugs ${1.23} \times$ faster (on average) than DI-FUZZRTL. Moreover, our experiments exposed 8 new bugs across the three RISC-V cores and one new bug in a reference model. All nine bugs were confirmed by the developers of the corresponding projects.

## KEYWORDS

processor, greybox fuzzing, verification, coverage

## 1 INTRODUCTION

As the complexity of processor designs has continuously grown over the years, verification has become one of the most challenging tasks in processor manufacturing. The state-space of a complex processor is extremely large, while the processor vendors have limited time and resources for verification. An exhaustive verification (i.e., testing each and every scenario) is an unrealistic goal to achieve, and therefore, a high-quality verification methodology is essential to discover bugs before fabrication. A timely, pre-silicon bug discovery can circumvent potentially millions-of-dollars of losses [33]. Otherwise, undiscovered bugs can manifest as severe security vulnerabilities in both proprietary and open-source processors such as transient execution vulnerabilities (e.g., Spectre [40], Foreshadow [74]), x86's guest privilege escalation bug [77], Intel's TSX bug [36] that breaks KASLR, Intel's machine check vulnerability [34] that enables denial-of-service attacks, and Pentium's FOOF [16] and FDIV bugs [19].

Broadly, the verification techniques can be divided into two categories - static and dynamic. Static verification techniques [7, 12, 55] aim to prove that the implementation is accurate with respect to a specification. Due to the well-known state explosion problem [18] of these techniques, dynamic verification techniques $\lbrack 5,{20},{23},{56}$ , 70, 75] are commonly used as part of the processor verification process. Dynamic verification involves simulating a Design Under Test (DUT) with a test input and analyzing the behavior of the DUT during or after simulation to identify bugs. Recent works $\left\lbrack  {{32},{42},{72}}\right\rbrack$ demonstrate that Coverage-based Greybox Fuzzing (CGF), a widely-used software testing technique, can be adapted as a dynamic verification technique to identify bugs in a processor design if certain differences between hardware and software are addressed.

Prior works on processor fuzzing mainly focus on addressing two major challenges. First, code coverage metrics used for fuzzing software programs (basic block, branch coverage, etc.) are not well-suited for fuzzing hardware [32, 70]. Second, a bug in a processor design does not result in an observable anomaly (i.e., crash) during testing as opposed to many software programs which can indicate the presence of bugs by throwing memory violation errors or raising exceptions.

To address the first challenge, researchers have introduced a variety of coverage metrics such as multiplexer toggle coverage, register coverage, etc $\left\lbrack  {{32},{42},{44},{54}}\right\rbrack$ that are tailored for hardware. In the context of a processor, the processor is effectively a complex Finite State Machine (FSM) that consists of a large number of states. Exploring different states in 'processor FSM' is the key to identifying bugs in the processor. Therefore, hardware-specific coverage metrics mainly aim to guide the fuzzer towards different uncovered 'processor FSM' states. These metrics take the hardware intrinsic (e.g., wire connections) into account rather than merely the code structure of the hardware. For instance, DIFUZZRTL [32], a state-of-the-art processor fuzzer, introduces register coverage metric where the goal is to monitor value changes in registers that control multiplexer selection signals. The intuition is that a particular value in these registers represents a unique state in the 'processor FSM' and guiding the fuzzer based on this feedback explores additional FSM states.

DIFUZZRTL's register coverage metric improves on prior works $\left\lbrack  {1,{42},{53}}\right\rbrack$ in terms of scalability, efficiency, and precision. However, we make a key observation that the register coverage can be a highly misleading metric for a processor fuzzer. Specifically, we find that DIFUZZRTL monitors many datapath registers which have minimal control over the current FSM state of the processor. The coverage increase resulting from the datapath registers does not provide meaningful information related to the current FSM state of the processor. This results in a scenario where inputs that affect datapath register coverage are incorrectly being classified as 'interesting' inputs, which in turn leads to wasted fuzzing time.

To address the second challenge, existing processor fuzzers $\left\lbrack  {{27},{32},{38},{43}}\right\rbrack$ adapt differential testing from the software domain to the hardware domain. Differential testing in software compares outputs of multiple programs that have the same functional behavior and checks for inconsistencies. In the hardware domain, the results of an Register Transfer Level (RTL) simulator are compared with those of an Instruction Set Architecture (ISA) simulator. An RTL simulator is used to simulate the execution of an instruction stream on the detailed microarchitecture implementation of the processor. The ISA simulator is used to simulate the functional behavior of the processor design and used as a reference model. A difference in the execution output of RTL simulation and ISA simulation indicates a potential bug in the processor.

In this work, we present ProcessorFuzz, a processor fuzzer that implements two novel features. First, ProcessorFuzz uses a new coverage metric called CSR-transition coverage to effectively guide processor fuzzing towards exploring unique processor states. Specifically, it monitors transitions in Control and Status Registers (CSRs) that form the core of the architecture specifications. Our intuition is that certain CSRs dictated by ISA readily expose the current 'processor FSM' state (e.g., current privilege mode, the event that caused floating mode exception), and thus the transitions in these CSRs signify a new 'processor FSM' state.

ProcessorFuzz's second feature is that it uses ISA simulation to rapidly determine if a test input is interesting. Prior works rely on RTL simulation for the same goal, which is time-consuming. In fact, this problem gets compounded if the coverage guidance is misleading and results in the execution of repetitive test inputs. ISA simulation is significantly faster than RTL simulation ${}^{1}$ . Hence, Pro-cessorFuzz can efficiently eliminate repetitive test inputs and focus on as many qualitatively distinct test input patterns as possible to expose bugs faster. Another benefit of this design feature is that ProcessorFuzz is agnostic to the hardware description language (HDL) used for designing the processor. Unlike prior works [32, 42], ProcessorFuzz does not require any HDL-specific hardware instrumentation because it identifies interesting inputs using ISA simulation. Hence, processors expressed in different HDLs (VHDL, System Verilog, etc.) can easily utilize ProcessorFuzz as a verification tool without having to worry about integration issues.

We evaluate ProcessorFuzz using a variety of widely-used open-source RISC-V based processors Rocket Core [3], BOOM [10], and BlackParrot [61]. Here Rocket Core [3] and BOOM [10] have been designed using Chisel HDL, while BlackParrot [61] has been designed using SystemVerilog. In addition, these processors vary in microarchitectural implementations such as their pipeline depths, execution type (i.e., in-order and out-of-order execution), etc. We compare the bug-finding effectiveness of ProcessorFuzz against the state-of-the-art register coverage guided DIFUZZRTL. On average, for the bugs found by DIFUZZRTL, ProcessorFuzz triggers bugs ${1.23} \times$ faster than DIFUZZRTL. In addition, ProcessorFuzz revealed 8 new bugs in widely-used open-source processors and one new bug in a reference model.

In summary, we make the following contributions:

- We propose ProcessorFuzz, a new processor fuzzing mechanism. ProcessorFuzz uses a novel CSR-transition coverage (CTC) metric, to effectively guide processor fuzzing towards interesting processor states.

- We propose to use the ISA simulator as part of a coverage feedback mechanism to rapidly identify interesting test inputs, thereby accelerating the bug-finding process.

- We demonstrate the practicality of ProcessorFuzz using 3 different open-sourced RISC-V processors and present eight new bugs identified in those three different processor designs and one new bug in a reference model.

- In the spirit of open science and to facilitate reproducibility of our experiments, we will make our source code of ProcessorFuzz publicly available.

## 2 BACKGROUND AND MOTIVATION

In this section, we first briefly explain coverage-based greybox fuzzing (CGF) for software. Next, we provide a brief background of how CGF is adapted as a hardware fuzzing method (specifically for processor fuzzing).

### 2.1 Coverage-based Greybox Fuzzing

Fuzzing has gained broad adoption in the software community due to its effectiveness in bug discovery, scalability, and practicality $\left\lbrack  {{25},{26},{50}}\right\rbrack$ . Fuzzing is the process of repeatedly running a Program Under Test (PUT) with a large number of random inputs to discover bugs in software. One of the widely-used fuzzing variants is CGF which utilizes the coverage feedback collected from the PUT at runtime. In each run of the PUT, CGF records coverage (e.g., basic block coverage, edge coverage, etc.) to determine if the input is 'interesting', i.e., whether it leads to increased coverage. If so, CGF applies a set of mutations to the 'interesting' input to generate new inputs which are then fed to the PUT in the next fuzzing rounds. Here, the intuition is that generating new inputs from coverage increasing ones would cover even more unexplored code. CGF instruments the code of the program (either statically or dynamically) with the necessary book-keeping logic to record coverage during the program execution.

---

${}^{1}$ As a reference point, ISA simulation is ${79} \times$ faster than RTL simulation for an open-source RISC-V based processor (i.e., BOOM [10]).

---

### 2.2 Adapting CGF for Processor Fuzzing

Recent works $\left\lbrack  {{32},{42},{72}}\right\rbrack$ show that CGF can be adapted as a dynamic verification method for hardware including processors. In this section, we briefly explain two important aspects when adapting CGF to processor fuzzing.

## Hardware Execution.

In the case of CGF for software, the fuzzing target is a software program that can be directly executed on a host machine with a test input after compilation. However, hardware (e.g., a processor) is not directly executable on the host machine. A hardware design is implemented with an RTL abstraction and must be simulated with an RTL simulator to evaluate a test input. RTL describes the hardware design in terms of data transfer between registers and the logical operations between the registers. The RTL design is usually expressed with an HDL (e.g., Verilog, VHDL). The RTL simulator can provide cycle-accurate information regarding the real-time behavior of the RTL design.

Bug Detection. Most software fuzzers focus on bugs that manifest as memory safety violations such as segmentation faults. These types of bugs are relatively easy to detect because they cause an observable anomaly (i.e., crash) in program behavior. However, fuzzing to find semantic bugs (e.g., logic errors) is harder than discovering memory violations because defining semantic violations is a highly domain-specific task. For these types of bugs, researchers proposed differential testing $\left\lbrack  {6,{49},{51},{64}}\right\rbrack$ . Differential testing compares the output of multiple programs that have the same functionality and checks for inconsistent behaviors. This approach is used by processor fuzzers $\left\lbrack  {{27},{32},{43}}\right\rbrack$ as well. In particular, the processor fuzzer provides the same input to both the RTL simulator and the reference model. Here, the reference model is an ISA simulator that mimics the behavior of all the ISA-level operations. The ISA simulator is a software model of the hardware and does not require any low-level microarchitectural details (e.g., the pipeline depth, buffer size). For a given program, it computes the values of architectural registers and memory state after the execution of each instruction. In contrast, RTL simulator is cycle-accurate and realizes the effect of executed instructions in the microarchitectural level such as the available packets in a buffer or branch prediction result of a conditional branch. The hardware fuzzer extracts an execution trace log from both the ISA simulator and the RTL simulator for the same input and cross-checks the traces. Here, the execution trace contains the final memory states and architectural registers. Any mismatch in the traces is considered a potential bug in the processor and is marked for further investigation by the verification engineer.

### 2.3 DIFUZZRTL's Register Coverage

DIFUZZRTL [32], the current state-of-the-art hardware fuzzer, uses this CGF approach to identify bugs in processors. DIFUZZRTL proposes a feedback strategy that aims to capture FSM state transitions during RTL simulation. The strategy follows a two-stage approach as depicted in Figure 1. In stage ①, it performs static analysis to identify a small set of registers in each RTL module and instruments the RTL with necessary hardware logic to record register coverage at simulation time. At a high level, DIFUZZRTL monitors a register if its value is directly or indirectly used to control a multiplexer selection signal. DIFUZZRTL creates a circuit graph of the RTL design where nodes and edges of this graph represent circuit elements (e.g., multiplexers, wires, ports, registers) and connections, respectively. Then, it recursively performs a backward data-flow analysis for each multiplexer's selection signal and identifies any register in the traversed path. In stage ②, DIFUZZRTL monitors value changes in the identified registers during the RTL simulation. For each clock cycle, DIFUZZRTL hashes all the values in the identified registers into a coverage map to represent the current FSM state. If a new hash value is observed, DIFUZZRTL increases register coverage to signify that the current test is interesting for further mutations.

DIFUZZRTL’s register coverage improves prior work $\left\lbrack  {{42},{44}}\right\rbrack$ in terms of scalability, efficiency, and precision. However, using register coverage metric for hardware fuzzing can be highly misleading. At a high level, we observe that a subset of registers leads to misleading coverage increase, and therefore, misguides the hardware fuzzer. We provide more details using an example (illustrated in Figure 1) from the open-source RISC-V-based Rocket Core [3].

In particular, in the multiplication unit of Rocket Core, there is a 130-bit remainder register in the MulDiv module that indirectly controls 98 mux selection signals. Therefore, DIFUZZRTL identifies this register to monitor during fuzzing. ${}^{2}$ The change in the value of remainder results in an increase in coverage. In Figure 2, we demonstrate the coverage increase resulted from the remainder register during a 24-hour fuzzing session. First, in Figure 2a, we depict the coverage progress of different modules in the Rocket core. Clearly, the MulDiv module (multiplication unit of Rocket core) dominates the module-wise register coverage. ${62}\%$ of overall register coverage results from the MulDiv module at the end of 24- hours. Figure 2b further shows the contribution of the remainder register to the coverage increase in the MulDiv module. Compared to all other registers in the MulDiv module, remainder register is clearly major factor that causes increase in register coverage. Indeed, our further analysis showed that the multiplication of two different numbers (see code snippet in List 1) increases the register coverage (i.e., explores a new state) even after $2\mathrm{M}$ iterations.

---

void main (   ) \{

	unsigned int num1, num2, res ;

	for ( int i = 0; i < 2000000; i++ ) \{

		num1 = i ; num2 = i + 1 ;

		res = num1 * num2;

	\}

---

Listing 1: Code snippet for testing multiplication unit.

---

${}^{2}$ DIFUZZRTL applies some optimizations to reduce search space. As one of their optimizations, it is able to track only a subset of bits of a register and therefore; ultimately tracks 98 -bit in remainder register.

---

![0196f2be-ea78-7594-b960-28a9b8cd3eda_3_147_234_1505_450_0.jpg](images/0196f2be-ea78-7594-b960-28a9b8cd3eda_3_147_234_1505_450_0.jpg)

Figure 1: Overview of DIFUZZRTL's coverage feedback strategy. DIFUZZRTL detect multiplexer select signals in the RTL design and trace them back to find the registers that affect these select signals (①). These registers are then hashed together to represent the FSM state of each module(2). DIFUZZRTL detects data-path registers such as remainder from Rocket core MulDiv module which increase the state space of the module significantly.

![0196f2be-ea78-7594-b960-28a9b8cd3eda_3_155_873_708_311_0.jpg](images/0196f2be-ea78-7594-b960-28a9b8cd3eda_3_155_873_708_311_0.jpg)

Figure 2: DIFUZZRTL's register coverage breakup for Rocket core over time.

Broadly, as pointed out with the above example, DIFUZZRTL monitors and uses coverage information from registers even if they are mostly involved in datapath-related operations and have minimal control over the current FSM state of the hardware. Unfortunately, data-path registers (e.g., remainder) increase search space significantly, yet the coverage increase resulting from data-path registers indeed does not provide meaningful information to the fuzzer related to the current hardware state. Therefore, it is not interesting to keep an input for further mutations if it increases coverage based on data-path registers. In our work, we present a new coverage metric that aims to tackle this problem.

## 3 PROCESSORFUZZ

In this section, we present the design of ProcessorFuzz, a fuzzing mechanism tailored for processors. We first provide a high-level overview of the different stages of ProcessorFuzz. Then, we outline our reasoning for using ISA simulation instead of RTL simulation to evaluate coverage. Finally, we explain the details of our CSR transition coverage metric and how ProcessorFuzz uses this metric to guide the fuzzing procedure.

### 3.1 Design Overview

We illustrate the design overview of ProcessorFuzz in Figure 3. In stage (1), ProcessorFuzz is provided with an empty seed corpus. It populates the seed corpus by generating a set of random test inputs in the form of assembly programs that conforms to the target ISA. Next, ProcessorFuzz chooses a test input from the seed corpus in stage (2) and subsequently applies a set of mutations (such as removing instructions, appending instructions, or replacing instructions) on the chosen input in stage (3). For these three stages, ProcessorFuzz uses the same methods applied by a prior work [32]. In stage (4), ProcessorFuzz runs an ISA simulator with one of the mutated inputs and generates an extended ISA trace log. A typical trace log generated by the ISA simulator contains (for each executed instruction) a program counter, the disassembled instruction, current privilege mode, and a write-back value as detailed in Section 2. The extended ISA trace log additionally includes the value of CSRs for each executed instruction. The Transition Unit (TU) receives the ISA trace log in stage (5). The TU extracts the transitions that occur in the CSRs. Each observed transition is cross-checked against the Transition Map (TM). The TM is initially empty and populated with unique CSR transitions during the fuzzing session. If the observed transition is not present in the TM, it is classified as a unique transition and added to the TM. In case the current test input triggers at least one new transition, the input is deemed interesting and added to the seed corpus for further mutations. If, however, there are no new transitions triggered, the input is discarded. In stage (6), ProcessorFuzz runs the RTL simulation of the target processor with the mutated input only if the input is determined as interesting. The RTL simulation also generates an extended RTL trace log similar to the extended ISA trace log. The extended RTL trace log contains the same information as the extended trace log. The ISA trace log and the RTL trace log are compared in stage (7). Any mismatch between the logs signifies a potential bug that needs to be confirmed by a verification engineer usually by manual inspection.

![0196f2be-ea78-7594-b960-28a9b8cd3eda_4_145_229_1503_499_0.jpg](images/0196f2be-ea78-7594-b960-28a9b8cd3eda_4_145_229_1503_499_0.jpg)

Figure 3: ProcessorFuzz Design: ProcessorFuzz runs the ISA simulator with an input generated by the mutation engine and outputs an extended ISA trace log that contains CSR values. The transition unit extracts CSR transitions, determines if a transition is new by checking the transition map, and stores new ones in the transition map. ProcessorFuzz runs the RTL simulation only with interesting inputs and creates an RTL trace log to be compared with the ISA log for bug detection.

<table><tr><td>#</td><td>PC</td><td>Instruction</td><td>[Privileged</td><td>Unprivileged ]</td></tr><tr><td>1</td><td>0x045c</td><td>sret</td><td>[8000000a00006000,00,0f, b100,</td><td>0,001</td></tr><tr><td>2</td><td>0x283c</td><td>sraiw s5, s0, 6</td><td>[8000000a00006020,00,0f, b100,</td><td>0.001</td></tr><tr><td>3</td><td>0x2840</td><td>fdiv.s fs11, ft0, fa7</td><td>[8000000a00006020,00,0f, b100,</td><td>0.001</td></tr><tr><td>4</td><td>0x2844</td><td>fence iorw, iorw</td><td>[8000000a00006020,00,0f, b100,</td><td>0,03]</td></tr><tr><td>5</td><td>0x2848</td><td>fsqrt.s ft0, ft5</td><td>[8000000a00006020,00,0f, b100,</td><td>0.031</td></tr><tr><td>6</td><td>0x284c</td><td>fcvt.wu.s s6, fs5</td><td>[8000000a00006020,00,0f, b100,</td><td>0.031</td></tr><tr><td>7</td><td>0x2850</td><td>addi a0, a0, 1344</td><td>[8000000a00006020,00,0f, b100,</td><td>0.13]</td></tr><tr><td>8</td><td>0x2854</td><td>fsgnj.s ft4, ft3, fa3</td><td>[8000000a00006020,00,0f, b100, ]</td><td>0,13]</td></tr></table>

Figure 4: Extended trace log generated by the ISA simulator. The values (in hexadecimal) of a subset of CSRs in Table 5 in Appendix are included within the square brackets in the given order; mstatus, mcause, scause, medeleg, frm, and fflags. Transitions are color coded; red and blue for mstatus and fflags CSR transitions, respectively.

### 3.2 Feedback from the ISA Simulation

One design feature of ProcessorFuzz is that it relies on the ISA simulation to determine if a test input is interesting as opposed to prior works that rely on the RTL simulation. Specifically, ProcessorFuzz runs the ISA simulator with each input obtained from the mutation engine and collects necessary feedback (i.e., CSR transitions which we detail in the following subsections) from the simulator. Proces-sorFuzz later processes the collected feedback to determine if the input should be ignored or used by the RTL simulator.

We use the ISA simulator to capture the CSR transitions for two main reasons. First, ISA simulators are generally much faster in executing a given program in comparison to executing that program on a processor using the RTL simulation. For instance, we observed that the RISC-V Spike ISA simulator is, on average ${79} \times$ faster than the RTL simulation of the RISC-V BOOM processor. This speedup provides a considerable advantage as ProcessorFuzz can then quickly identify if a test input is interesting without performing the slow RTL simulation. Eliminating inputs with similar characteristics help ProcessorFuzz to achieve faster bug discovery times as shown in Section 4. Indeed, ProcessorFuzz discovered all the bugs found by the existing processor fuzzer (i.e., DIFUZZRTL).

Second is the reduced effort needed to instrument the simulator. A simulator needs to be instrumented to generate an extended trace log with the selected CSRs. An ISA simulator can be easily instrumented by extending the already available trace logic with the selected CSRs. The same instrumented ISA simulator can be used to fuzz any processor design as long as it has been designed for the same ISA target. In contrast, instrumenting RTL designs for tracking the coverage metrics requires extensive effort. Moreover, instrumentation in one HDL does not readily translate to other HDLs. Additionally, as shown in Section 4, ProcessorFuzz incurs limited instrumentation overhead during fuzzing (only $< 1\%$ in ISA simulator) as opposed to prior works [73] that instrument processor RTL and result in higher runtime overheads (e.g., 71% by TheHuzz [73] and %97 by RFUZZ [42]).

### 3.3 CSR-transition Coverage

3.3.1 Description of the Metric. As described in Section 2.3, DIFUZ-ZRTL's register coverage technique monitors many datapath registers (e.g., remainder register) to determine the current FSM state, which leads to large state space. Hence, guiding the fuzzing procedure with DIFUZZRTL's register coverage metric can be highly misleading when fuzzing processors. To test the processor with as many qualitatively distinct input patterns as possible, we propose a novel CSR transition-based coverage metric.

CSRs are system registers in an ISA specification. These registers are used to control (e.g., delegated exceptions) or hold information (e.g., state of the floating-point unit) about the current architectural state of the processor. Our intuition for using CSRs is as follows. A processor is a complex FSM where CSRs have direct control over the current processor state. Architectural state of the processor (held in the register file and status registers) represents the state of a program running in the processor. A value change in a CSR often signifies an architectural state change such as a value change in a CSR that stores exception code or privilege level. Therefore, ProcessorFuzz aims to realize the current state of the processor by monitoring transitions in CSRs to guide the fuzzer towards interesting processor states.

CSRs are part of both an ISA simulator and the RTL design of a processor. Hence, CSR transitions can be extracted either from the ISA simulation or the RTL simulation. As detailed before, Proces-sorFuzz uses the ISA simulator to capture CSR transitions. Specifically, to extract a CSR transition, ProcessorFuzz monitors the CSR values resulting from the execution of the previous and current instructions and checks if they differ. If so, ProcessorFuzz uses the transition to determine if the input is interesting as detailed in the following subsections. Here, we provide a concrete example to illustrate how ProcessorFuzz identifies a CSR transition in the ISA trace log. Consider the extended ISA trace log shown in Figure 4. The CSR value changes after execution of the sret instruction shown in Line 1 , which can be seen by comparing the entries in Line 1 and Line 2 of the 'Privileged' column. Specifically, we observe a CSR-transition in mstatus CSR from ${0x8000000a00006000}$ to ${0x8000000a00006020}$ as highlighted in red in Figure 4. Overall, from Figure 4, we represent the CSR transition caused by sret instruction as $\left( {{S}_{0},{S}_{1}}\right)$ $= \left( {{8000000}\mathrm{a}{00006000000}\mathrm{{fb}}{100000},{8000000}\mathrm{a}{00006020000}\mathrm{{fb}}{100000}}\right)$ , where ${S}_{0}$ and ${S}_{1}$ are defined as the concatenated CSR values before and after the transition, respectively.

3.3.2 Why Transitions Instead of Values? DIFUZZRTL determines the current processor state based on the register coverage as detailed in Section 2.3. For each newly covered FSM state, DIFUZZRTL's register coverage only stores the current state of the processor and does not consider the previous state. Unfortunately, this design choice can lead to important test inputs being discarded by the fuzzer and the fuzzer can potentially miss out on the discovery of a bug. We illustrate this in detail below. Figure 5 represents a subset of the abstract states associated with a real-world bug (Bug 2 in Table 2) that we identified in an open-source RISC-V processor.

In the figure, the processor starts out in the N0 state. The bug triggers in the N2 state only if the previous state is N1. It does not trigger when the previous state is N0 . During a coverage-guided fuzzing session, if both N1 (through P0 transition) and N2 (through P2 transition) are covered individually, there will not be a coverage increase for the denoted P1 state transition. And so, the unique P1 transition is not particularly driven towards. Thus, the fuzzing session fails to trigger the bug. Contrarily, by monitoring transitions, we can detect $\mathrm{P}1$ as a new transition even though $\mathrm{N}1$ and $\mathrm{N}2$ states are already covered. Overall, we monitor new transitions in CSRs rather than just identifying unique CSR values to improve the sensitivity of the feedback metric. Indeed, our rationale is similar to widely-used software fuzzers [26, 47]'s rationale that monitors edges in a program instead of basic blocks. We provide the details on how ProcessorFuzz extracts CSR transitions in the next subsection.

![0196f2be-ea78-7594-b960-28a9b8cd3eda_5_320_1681_379_311_0.jpg](images/0196f2be-ea78-7594-b960-28a9b8cd3eda_5_320_1681_379_311_0.jpg)

Figure 5: Abstract state diagram for triggering Bug 2 listed in Table 2

3.3.3 CSR Selection Criteria. An ISA specification usually specifies a large number of CSRs ${}^{3}$ . Monitoring all available CSRs for transitions can mislead the fuzzer (as we show in Section 4) because not all CSRs provide distinctive information regarding the current processor state. As an example, consider instret CSR that holds the total number of retired instructions. Considering this CSR results in a scenario where each committed instruction by the processor results in a CSR transition. Effectively, ProcessorFuzz would identify any test input as interesting since the instret CSR causes a transition after each committed instruction. However, a test would rarely result in a bug because of a change in committed instruction count. To aid ProcessorFuzz in determining qualitatively different inputs, we introduce the following two criteria when selecting the CSRs that ProcessorFuzz monitors transitions. First, we select CSRs that contain status information about the processor (criteria C1). These CSRs are important because they directly reveal the current status of the processor. As an example, we select a CSR that stores the cause for an exception taken by the processor (e.g., mstatus). If a test case results in an exception, ProcessorFuzz analyzes the cause and differentiates it from another test case that has a different exception reason (e.g., misaligned load/store attempt or access faults due to unauthorized privilege mode). Second, we select any CSR that is used to set a certain configuration in the processor (criteria C2). Here, we aim to realize if the processor behaves as expected under different configurations. For instance, the value of medeleg can be changed to determine which traps can be delegated to lower privilege levels (e.g., the load access fault handled in supervisor mode instead of machine mode). This way, ProcessorFuzz aims to realize if processor designs can perform correctly under different configurations (e.g., different exception delegations) for a particular processor status (e.g., an exception). In Table 5 in Appendix, we list all the CSRs in the RISC-V ISA that we used for identifying transitions in the current implementation of ProcessorFuzz based on the aforementioned two criteria $\mathrm{C}1$ and $\mathrm{C}2$ . We also provide all the CSRs that we excluded (e.g., instret) along with details why they are not considered as part of ProcessorFuzz's current design (Table 6 in Appendix).

Apart from these two criteria, CSR selection can be further limited depending on the features supported by the target processor or the desired scope of verification. For example, if the target processor does not support interrupts within the testing framework, any CSRs related to the configuration or status of interrupts can be excluded. Similarly, if we only want to verify the functionality of the floating-point unit in the processor, only floating-point CSRs can be monitored to identify transitions. We quantitatively demonstrate this capability of ProcessorFuzz in Section 4.2.

---

${}^{3}$ As a reference point, RISC-V ISA defines up to 4096 CSRs

---

![0196f2be-ea78-7594-b960-28a9b8cd3eda_6_221_232_1357_159_0.jpg](images/0196f2be-ea78-7594-b960-28a9b8cd3eda_6_221_232_1357_159_0.jpg)

Figure 6: Workflow of the ProcessorFuzz transition unit.

### 3.4 Transition Unit

As shown in Figure 3, the TU takes an extended ISA trace log as input and communicates with the TM to output whether the trace log contains any new transitions. We describe the complete work-flow of the TU in Figure 6. As a first step, the TU extracts all CSR transitions in the trace log based on the description in Section 3.3. Then, ProcessorFuzz applies a filter to remove unnecessary transitions. Next, the TU groups the transitions to reduce the state space. We describe how the TU filters and groups the transitions in the rest of this subsection.

Filtering Transitions. We note that the number of possible CSR transitions can be large depending on the cumulative width of the selected CSRs. However, not all CSR transitions represent interesting architectural state changes that are relevant for testing processors. For instance, a test program running on the target processor can write to a CSR that contains processor status, e.g. mstatus CSR in RISC-V ISA. This could get identified as a new CSR transition. If the write operation is legal, the processor continues the execution of the program and eventually overwrites the CSR with the updated status. Overall, the type of transitions that occur from writes to status CSRs do not affect the architectural state of the processor. Thus, ProcessorFuzz filters out transitions that occur from explicit writes to status CSRs.

Grouping Transitions. ProcessorFuzz provides the flexibility to customize the CSR-transition coverage metric to be suitable for verifying different Architectural Units (AUs) individually. Specifically, ProcessorFuzz allows a designer to group CSR transitions of AUs, thereby considering them as independent events. Grouping transitions improves the exploration of CSR transitions within each group. As a result, the fuzzer is able to generate tests targeted towards individual AUs and verify them thoroughly. This is a useful feature for a verification engineer as AUs in a processor can be individually verified as an initial step of verification. For example, privileged and unprivileged architectures in a RISC-V processor can be verified individually by grouping transitions as shown in Figure 4. Identifying and fixing the bugs in each AU before fuzzing the processor as a whole can reduce the overall verification effort. Transition Map ProcessorFuzz maintains a transition map to store CSR-transitions. Each transition is stored in the map as a tuple: $\left( {{I}_{m},{S}_{0},{S}_{1}}\right)$ where ${I}_{m}$ is the mnemonic of the instruction whose execution resulted in the CSR transition. ${S}_{0}$ and ${S}_{1}$ are CSR values before and after the transition as defined in subsection 3.3. Revisiting the same example given in subsection 3.3, privileged CSR-transition caused by sret instruction can be represented as (sret, 8000000a00006000000fb1000000, 8000000a00006020000fb1000000). Similarly, ProcessorFuzz converts the unprivileged CSR-transition in lines 3 and 4 in Figure 4 to (fdiv.s, 0000, 0003).

We include instruction mnemonic in the aforementioned tuple because the same transition can be triggered by different instructions. For example, both floating-point division and floating-point square-root instructions can trigger the same transition in flags CSR in RISC-V ISA due to invalid operations. Nevertheless, only the invalid operation of floating-point division instruction might contain a bug. Therefore, we tag each transition with the mnemonic of the instruction that triggered it to uniquely identify transitions triggered by different instructions. Only the mnemonic of the instructions is included to ignore repetitive transitions that get triggered by different operands of the same instruction.

Once tuples are created, the map is queried to check whether the detected transition is new or a duplicate. Tuples that are identified to contain new transitions are added to the map while marking the current test input as interesting. The transition map is empty at the beginning of a fuzzing session and maintained throughout the session.

### 3.5 RTL Simulation and Trace Comparison

If the TU determines that the current input results in a unique CSR transition, ProcessorFuzz launches the RTL simulation and generates the extended RTL trace log. ProcessorFuzz then compares the extended RTL trace log with the extended ISA trace log. Any difference between these logs signifies a potential bug in the processor design and needs to be investigated further by a verification engineer. In case the input does not result in a unique transition, ProcessorFuzz discards the input and proceeds to the next fuzzing iteration.

## 4 EVALUATION

In this section, we evaluate the effectiveness of ProcessorFuzz using real-world processor designs. First, we provide the details of our evaluation setup. Then, we assess the bug-finding capability of ProcessorFuzz using ground-truth bugs and compare Processor-Fuzz's performance against DIFUZZRTL. Specifically, we analyze if ProcessorFuzz can expose the same set of bugs reported by DI-FUZZRTL in a more efficient way. Finally, we describe the list of new real-world bugs that ProcessorFuzz identified along with the severity of the bugs.

### 4.1 Evaluation Setup

4.1.1 Implementation Details. ProcessorFuzz has two main implementation steps; generation of an extended trace log using the ISA simulator and building the TU (see Figure 3). For the former, we extended SPIKE [68] open-source ISA simulator to store the values of monitored CSRs (see Table 5 in Appendix) for each executed instruction during the ISA simulation. The instrumentation overhead of SPIKE is ${0.4}\%$ in terms of lines of $\mathrm{C} +  +$ code, while the runtime overhead is 0.15%. The TU is implemented as a Python library. For the RTL simulation of all processors designs, we used Verilator [67], an open-source RTL simulator. We used the same mutation engine (see Figure 3) as provided by DIFUZZRTL's open-source repository. Using the same engine is important since our goal is to compare two coverage feedback mechanisms (i.e., register coverage and CSR-transition coverage) rather than input generation mechanisms. We separated transitions belonging to frm and fflags to separate floating-point operations from the rest of the CSRs.

4.1.2 Processor Designs. In our evaluation, we use three real-world open-source processors designed using the open-standard RISC-V ISA.

RISC-V Rocket Core. Rocket core is an open-source, general-purpose, in-order, RISC-V processor core that can be generated using the Rocket Chip SoC Generator framework [3]. Rocket core is designed in Chisel HDL [4], and is shown to integrate well with custom hardware accelerators. Rocket core has been taped out multiple times [3] and is capable of booting Linux. Essentially, it is well-tested. We used Spike [68] as a reference model to verify the correctness during fuzzing. The commit version of the Rocket core that we used is 148d5d2.

RISC-V BOOM Core. BOOM [10] core can also be generated from the same Rocket Chip SoC Generator framework [3] and is also designed in Chisel HDL. BOOM is an out-of-order, superscalar RISC-V processor core and capable of booting Linux. BOOM has also been taped out [11]. We used Spike ISA simulator to verify the correctness during fuzzing. The commit version of the BOOM core that we used is 148d5d2.

RISC-V BlackParrot Core. BlackParrot [61] is an open-source 64-bit RISC-V core, designed in the industry-standard SystemVer-ilog HDL. BlackParrot is an ideal candidate for hosting accelerator fabrics and for hardware research owing to its tiny, modular, and friendly design approach. BlackParrot is silicon-validated and is in active development. We used Dromajo [71] as a reference model to expose the bugs in BlackParrot for the bc3b48b commit version.

4.1.3 Settings. We compared ProcessorFuzz with two different settings of DIFUZZRTL. The first setting is no-cov-difuzzrtl where DIFUZZRTL fuzzing framework is used without any coverage guidance (i.e., as a blackbox fuzzer). For all the cores that we evaluated, we successfully used this setting as a comparison point. The second setting is reg-cov-difuzzrt1 where DIFUZZRTL fuzzing framework relies on register coverage as a guidance mechanism. While this setting is applicable to Rocket and BOOM Cores, it is not the case for BlackParrot Core. This is because DIFUZZRTL's register coverage passes do not support SystemVerilog. They are tailored for FIRRTL [35], an intermediate representation (IR) used by Chisel HDL, which is used to design Rocket and BOOM cores. We tried to convert SystemVerilog to FIRRTL using an open-source tool (i.e., Yosys [78]), and apply DIFUZZRTL's register coverage passes. However, we observed several issues during this conversion due to the limited support for SystemVerilog to FIRTTL conversion and thus failed to instrument BlackParrot. In our experiments, we used DIFUZZRTL as the sole comparison point since it shows clear benefits over previous processor fuzzing frameworks as well as its open-source nature. Also, for each setting, we reported Time-to-Exposures (TTE) which is defined as the total elapsed time from the starting of the fuzzing session until the bug is exposed.

4.1.4 Infrastructure. All the experiments based on ISA and the RTL simulations were conducted on server nodes with Intel ${}^{\circledR }{\text{Xeon}}^{\circledR }$ E5- 2670 CPUs and CentOS Linux 7 as the operating system. We fuzzed each processor design 10 times for each setting and allocated 48 hours (2 days) of time limit for each fuzzing instance. For each fuzzing instance, we dedicated two cores and 8GB of memory. In total, it took 4320 CPU hours to conduct all the experiments detailed in the following sections.

### 4.2 Ground-truth Bugs

As discussed by many prior works [39, 48], the bug-finding capability of a fuzzer is the ultimate litmus test for a fuzzer. While there exist several fuzzing benchmarks for software programs $\left\lbrack  {{30},{45}}\right\rbrack$ , this is not the case for processors. Therefore, we relied on a set of bugs (in total six bugs) previously reported by DIFUZZRTL for BOOM processor to evaluate the bug-finding capability of Proces-sorFuzz and perform a head-to-head comparison with DIFUZZRTL. Overall, our evaluation aims to demonstrate that ProcessorFuzz can guide the fuzzer efficiently to discover ground-truth bugs thanks to the CSR-transition feedback obtained using the ISA simulation. In summary, ProcessorFuzz was able to discover all the ground truth bugs and achieved lower TTE compared to DIFUZZRTL.

In Table 1, we report the TTE of bugs in seconds for three different settings in 2nd-4th columns; no-cov-difuzzrt1, reg-cov- -difuzzrt1, and ProcessorFuzz for the BOOM processor core. We also provide the achieved speedups by ProcessorFuzz over no-cov- -difuzzrtl, and reg-cov-difuzzrtl. For ProcessorFuzz, we provide results for three different configurations; selected, fp-csr, and all-csr. These configurations differ in the CSRs that Proces-sorFuzz monitors during fuzzing. Specifically, selected configuration of ProcessorFuzz uses the CSRs in Table 5 in Appendix for transition extraction based on the criteria that we detailed in Section 3.3.3. all-csr configuration monitors all implemented CSRs in the BOOM core. Here, by using all-csr configuration, we aim to present that ProcessorFuzz can be effectively guided towards bugs by eliminating certain CSRs that do not assist fuzzing towards exploring bugs (e.g., minstret that repeatedly changes after an instruction retires). Finally, fp-csr configuration uses only the floating-point CSRs (unprivileged CSRs in Table 5 in Appendix). The aim of this experiment is to show that ProcessorFuzz can focus on certain parts of processors by selecting a subset of CSRs (e.g., floating point unit). Overall, ProcessorFuzz selected configuration and DIFUZZRTL discovered five out of six bugs reported in the DI-FUZZRTL within the fuzzing time limit in our experiments. Unfortunately, we could not detect #504 with any of the settings. In summary, ProcessorFuzz (selected) achieved, on average, 1.21 (up to ${2.1} \times  )$ and ${1.23} \times$ (up to ${2.32} \times$ ) speedups over no-cov-difuzzrtl and reg-cov-difuzzrtl, respectively. no-cov-difuzzrtl performed slightly better than regcov-difuzzrtl.

We included fp-csr configuration to demonstrate the Proces-sorFuzz's ability to change the scope of verification by changing the CSR selection. fp-csr detected the bugs in the floating-point unit (issues #492, #493 and #503) x2.08 times faster compared to the selected configuration while showing a slowdown in detecting other bugs.

![0196f2be-ea78-7594-b960-28a9b8cd3eda_8_238_241_546_888_0.jpg](images/0196f2be-ea78-7594-b960-28a9b8cd3eda_8_238_241_546_888_0.jpg)

Figure 7: Coverage details for different settings.

We also show the effect of CSR selection on TTE of the bugs through all-csr configuration. all-csr configuration failed to detect two of the bugs within the allocated fuzzing time. Moreover, all-csr is significantly slower (i.e., ${0.06} \times$ on average) than selected in detecting bugs.

To understand the performance of ProcessorFuzz and DIFUZ-ZRTL for different bugs, we further study the relationship among register coverage, CSR-transition coverage, and bug-finding times. Specifically, in Figure 7a, we show the measured register coverage progress for different settings of DIFUZZRTL and ProcessorFuzz. Although ProcessorFuzz covers less number of states (i.e., achieves lower register coverage) during fuzzing, it was still able to discover bugs faster. For instance, ProcessorFuzz triggered the most challenging bug based on the TTE (i.e., #454) after exploring ${303}\mathrm{\;K}$ states while no-cov-difuzzrt1 and reg-cov-difuzzrt1 triggered that bug after exploring ${364}\mathrm{\;K}$ and ${354}\mathrm{\;K}$ states, respectively. This particular bug shows that higher register state coverage does not necessarily translate to a faster bug discovery. Indeed, an increase in coverage due to value changes in datapath registers can mislead the fuzzer since inputs with similar characteristics (see the multiplication example in Section 2.3) are repeatedly used by the fuzzer to generate a new set of inputs.

In Figure 7b, we also show the total number of test inputs that lead to a coverage increase, i.e. 'interesting test inputs', and the total number of inputs generated by the mutation engine for the two settings of DIFUZZRTL and ProcessorFuzz. For no-cov-difuzzrtl and reg-cov-difuzzrtl, we use the register coverage metric, same as that used in the DIFUZZRTL work, to realize if a test input increases coverage. For ProcessorFuzz, we use the CSR-transition coverage metric to detect inputs that resulted in a coverage increase. The results provide an important takeaway. Although Processor-Fuzz generates significantly more inputs than other approaches, it is very selective when categorizing a test input as an 'interesting' input. Consequently, ProcessorFuzz identified only 33% of the generated test inputs as interesting (i.e., caused a unique CSR transition). Moreover, ProcessorFuzz could expose the bugs faster although it used the least number of test inputs for RTL simulation. Note that ProcessorFuzz launched the RTL simulation only with interesting inputs (i.e., curved dotted red line) and discarded any other generated input. Using the fast ISA simulation enabled ProcessorFuzz to quickly eliminate inputs that do not result in a new FSM state and spend more time on inputs that explore new FSM states.

In Figure 8, we show how ProcessorFuzz performs in terms of industry standard RTL coverage metrics (i.e., line, toggle, FSM, and branch coverage) for BOOM core. Here, our goal is to present the effectiveness of ProcessorFuzz based on the widely-used coverage metrics. In particular, we aim to explore how well the CSR-transition coverage metric is able to result in test cases that cover different lines, toggles, FSMs, and branches in the processor RTL. We compare ProcessorFuzz's overall coverage based on these four metrics against DIFUZZRTL. Line coverage represents the percentage of RTL code lines that got exercised during the simulation. Toggle coverage indicates the percentage of bits in wires and registers that toggled during the simulation. FSM coverage represents the percentage of FSM states reached during the simulation. Branch coverage represents the percentage of different branches that was taken during the simulation against the total branches in the design. In particular, we obtained all the seeds generated by each approach during fuzzing and feed them to the Synopsys VCS tool one by one and reported coverage. Note that Synopsys VCS tool is significantly slower when collecting coverage, and therefore, we could report coverage progress for the first 12 hours of fuzzing although we run VCS tool for a week. The main takeaway from Figure 8 is that, even though ProcessorFuzz uses CSR-transition coverage extracted from ISA simulation, ProcessorFuzz performs as well as DIFUZZRTL in terms of standard RTL coverage metrics and is able to cover different RTL regions based on different metrics.

### 4.3 Newly Discovered Bugs

In Table 2, we document the various new bugs discovered by Proces-sorFuzz in the selected processors mentioned earlier and in the ISA simulator used as a reference model. In the following subsections, we describe and highlight the significance of each bug.

4.3.1 Bug Descriptions. Bug 1. When multiple floating-point precisions are supported by a floating-point unit in a processor, valid lower precision values are expected to be NaN-boxed (i.e., remaining upper bits set to 1 's). Otherwise, lower precision values are expected to be interpreted as NaNs. BlackParrot does not interpret non-boxed floats as NaNs, which leads to functionally incorrect computations on non-boxed floats, thus violating the ISA specification. Incorrect computations in security-critical functions (e.g., in cryptographic applications) can compromise the security of a processor (CWE-1201 [52]).

Table 1: The speedup achieved by selected ProcessorFuzz configuration over no-cov-difuzzrtl, and reg-cov-difuzzrtl for the ground-truth bugs in the BOOM processor. We also report speedup of fp-csr and all-csr ProcessorFuzz configurations over selected ProcessorFuzz configuration. In the table, we state the maximum allowed runtime of 48 hours (172800 seconds) for bugs that could not be found.

<table><tr><td/><td>no-cov- difuzzrtl</td><td colspan="2">reg-cov- difuzzrtl</td><td colspan="3">ProcessorFuzz (selected)</td><td colspan="2">ProcessorFuzz (fp-csr)</td><td colspan="2">ProcessorFuzz (all-csr)</td></tr><tr><td>Issue No</td><td>Time (s)</td><td>Time (s)</td><td>Speedup (over no-cov)</td><td>Time (s)</td><td>Speedup (over no-cov)</td><td>Speedup (over reg-cov)</td><td>Time (s)</td><td>Speedup (over selected)</td><td>Time (s)</td><td>Speedup (over selected)</td></tr><tr><td>#458</td><td>104.3</td><td>70.3</td><td>1.48</td><td>54</td><td>1.93</td><td>1.3</td><td>151324.8</td><td>0.0</td><td>172800</td><td>NA</td></tr><tr><td>#454</td><td>32883.3</td><td>45322</td><td>0.73</td><td>25020</td><td>1.31</td><td>1.81</td><td>119886.2</td><td>0.2</td><td>39523.3</td><td>0.63</td></tr><tr><td>#492</td><td>2047.2</td><td>4238.9</td><td>0.48</td><td>1821.2</td><td>1.12</td><td>2.32</td><td>1221.8</td><td>1.49</td><td>172800</td><td>NA</td></tr><tr><td>#493</td><td>585.4</td><td>494.9</td><td>1.18</td><td>278.7</td><td>2.1</td><td>1.77</td><td>170.1</td><td>1.63</td><td>526.6</td><td>0.52</td></tr><tr><td>#503</td><td>1463.7</td><td>1011.1</td><td>1.44</td><td>2795.9</td><td>0.52</td><td>0.36</td><td>757.6</td><td>3.69</td><td>62246.8</td><td>0.04</td></tr><tr><td>#504</td><td>172800</td><td>172800</td><td>NA</td><td>172800</td><td>NA</td><td>NA</td><td>172800</td><td>NA</td><td>172800</td><td>NA</td></tr><tr><td>Geo.</td><td>3182.9</td><td>3225.9</td><td>0.98</td><td>2630.7</td><td>1.21</td><td>1.23</td><td>8890.2</td><td>0.23</td><td>43402.2</td><td>0.06</td></tr></table>

Table 2: Brief description of bugs discovered by ProcessorFuzz, and their current status, in various processor cores.

<table><tr><td>Bug</td><td>Core / Simulator</td><td>Brief description of the bug</td><td>Status</td></tr><tr><td>1</td><td>BlackParrot</td><td>Non-boxed single-precision floating point values are not interpreted as NaNs</td><td>Confirmed; not fixed</td></tr><tr><td>2</td><td>BlackParrot</td><td>Read-after-Write dependencies on fcsr. fflags are not satisfied.</td><td>Fixed</td></tr><tr><td>3</td><td>BlackParrot</td><td>When mstatus. FS is not set and the fcsr is written, FS is unexpectedly updated.</td><td>Fixed</td></tr><tr><td>4</td><td>BlackParrot</td><td>The 2 low-bits of sepc CSR are not write-insensitive.</td><td>Fixed</td></tr><tr><td>5</td><td>BlackParrot</td><td>No exception raised when writing certain read-only CSRs.</td><td>Fixed</td></tr><tr><td>6</td><td>BlackParrot</td><td>Reading zero register, following specific instruction sequences, return unexpected non- zero values</td><td>Fixed</td></tr><tr><td>7</td><td>BlackParrot</td><td>Unexpected store access-fault on properly aligned, unpaired sc.d instruction.</td><td>Reported</td></tr><tr><td>8</td><td>Dromajo</td><td>PMP checks are performed, and raise exceptions upon encountering violations, even with no PMP entries set.</td><td>Confirmed; not fixed.</td></tr><tr><td>9</td><td>Rocket & BOOM</td><td>Instruction page fault not raised when accessing non-leaf PTEs with certain unspecified page attributes.</td><td>Fixed</td></tr><tr><td>10</td><td>BOOM</td><td>mstatus. FS is gratuitously set to dirty.</td><td>Confirmed; not fixed</td></tr></table>

![0196f2be-ea78-7594-b960-28a9b8cd3eda_9_159_1399_698_571_0.jpg](images/0196f2be-ea78-7594-b960-28a9b8cd3eda_9_159_1399_698_571_0.jpg)

Figure 8: The progress of industry-standard RTL coverage metrics for ProcessorFuzz and DIFUZZRTL for BOOM core.

Bug 2. Certain floating-point instructions update fflags CSR which holds events like floating-point overflow, division by zero, etc. A read-after-write (RAW) hazard occurs in a pipelined processor when a floating-point instruction that writes to fflags is followed by an instruction that reads fflags.ProcessorFuzz detected that this particular hazard is not handled by the BlackParrot processor, causing the software to read an outdated fflags value.The RISC-V ISA requires explicit checks of the fflags CSR in software to identify floating-point overflows, invalid operations, etc. Therefore, failure to detect an overflow can lead to security issues in software such as buffer overflows (e.g., CVE-2020-10029 [17]). This bug falls under core and compute hardware weakness (CWE-1201 [52]). ProcessorFuzz was able to detect this bug since it monitors the transitions in fflags.

Bug 3. When the FS field of mstatus CSR is set to 0 , it indicates that floating-point extension is turned off. In such cases, accessing fcsr is expected to raise an illegal instruction exception without altering the FS field of mstatus CSR. However, BlackParrot wrongly sets the FS field to dirty, instead of keeping it unchanged.

Bug 4. The least significant two bits of sepc CSR must be always hardwired to 0 on implementations that only support 32-bit instruction alignment. Any write from software to the least significant two bits of the sepc CSR must be discarded. However, BlackParrot updated the low bits when a test input attempted to modify them. Further analysis showed that this issue exists for mepc CSR as well.

Bug 5. Any attempt to modify a read-only register is supposed to trap with an illegal instruction exception. ProcessorFuzz discovered that BlackParrot does not raise an illegal instruction exception if a test input updates a read-only register, specifically mhartid. ProcessorFuzz monitors mcause and scause CSRs that are in charge of exception handling and was, therefore, able to expose this bug.

Bug 6. Any write attempt to the zero register (i.e., x0) must be ignored according to the RISC-V ISA. However, in the BlackParrot processor, we detected that the $\times  0$ register is read as a non-zero value one of the preceding division instructions that writes to $\times  0$ is still in the pipeline. Further analysis revealed that this discrepancy is due to bypassing the result of division operation to the following instruction even when the destination register of a division operation is $\times  0$ . ProcessorFuzz was able to identify this bug because a test input that has this scenario resulted in a CSR transition in fflags due to division by zero.

Bug 7. A store-conditional instruction, if properly aligned to the appropriate word boundary, should not raise a store-access fault. However, BlackParrot raises a store access fault when executing an unpaired, but properly aligned sc.d instruction. We reported the issue to the BlackParrot designers and are currently waiting for their response.

Bug 8. According to RISC-V privileged specification, the effective privilege mode for implicit page table accesses should be supervisor mode. However, we observed that Dromajo accesses page tables in user mode privilege level when executing user-mode programs. Further analysis revealed that Dromajo also carries out Physical Memory Protection (PMP) checks in user mode when no PMP entries are set, violating the RISC-V ISA privileged specification in two counts.

Bug 9. In a multi-level page table implementation, the accessed (A), dirty (D), and user-mode (U) bits of a non-leaf page table entry (PTE) are reserved for future use and should be cleared. If these bits are set in a non-leaf PTE, the processor must raise an instruction page fault when accessing the PTE according to RISC-V ISA. We discovered that Rocket and BOOM cores do not raise instruction page fault when software attempts to access a PTE with any of A, D, or U bits set. This bug is similar to CWE-1209[52] where failure to disable reserved bits allows attackers to compromise the hardware state.

Bug 10. The FS field in the mstatus CSR in RISC-V ISA is used to check whether save and restore of floating-point registers are required when there is a context switch. ProcessorFuzz detected that BOOM set the FS field to dirty for any write to fcsr register, even when the value of fcsr is zero and unchanged by the write operation. This scenario is not a violation of RISC-V ISA due to the flexibility allowed by the ISA for maintaining FS field. Nevertheless, setting FS field when the floating-point unit state is unchanged degrades the performance as the processor unnecessarily saves and restores floating-point registers.

4.3.2 Timing Results. In Table 3, we provide the TTEs for six newly identified and confirmed bugs (Bug 1-6) and one newly identified but currently waiting confirmation bug (Bug 7) in BlackParrot core. We did not include Bug 8-10 since they were easily detected in all the settings that we used in our evaluation. For this evaluation, we were only able to compare ProcessorFuzz with no-cov-difuzzrtl. As detailed in Section 4.1.3, we could not instrument BlackPar-rot with register coverage since DIFUZZRTL lacks support for SystemVerilog (detailed in Section 4.1.3). ProcessorFuzz does not require any instrumentation on the RTL design, therefore, could successfully guide the fuzzer with CSR-transition coverage to expose bugs. Overall, ProcessorFuzz achieved ${1.6} \times$ speedup, on average, over no-cov-difuzzrtl. Note that only ProcessorFuzz was able to detect Bug 6 from Table 2. Similar to the experiment that we conducted in the BOOM processor using the ground-truth bugs, all-csr configuration of ProcessorFuzz performed poorly compared to selected configuration (i.e., ${0.47} \times$ slow-down). Moreover, fp-csr configuration identified floating-point related bugs fairly faster (e.g., Bug 2) compared to other type of bugs (e.g., Bug 4 that focuses on sepc CSR).

## 5 RELATED WORK

We divide the related work into three different categories. First, we present traditional methods in hardware verification such as random instruction generation, coverage-directed test generation, and formal verification. Then, we present fuzzing-based hardware verification approaches and how ProcessorFuzz differs from existing fuzzing works. Finally, we discuss the usage of differential testing in the software domain.

### 5.1 Traditional Hardware Verification

Random instruction generators $\left\lbrack  {{22},{27},{29},{31},{43}}\right\rbrack$ have been commonly used in processor verification since they require limited human expertise and are scalable to large RTL designs. These tools produce random assembly programs based on a set of constraints such as the instruction mix, frequencies, etc., to identify functional bugs in processors. The lack of coverage guidance in these tools leads to the generation of the repetitive inputs that test the same processor functionalities, thereby decreasing the chances of finding bugs $\left\lbrack  {{32},{42}}\right\rbrack$ .

A verification engineer can target the uncovered RTL regions by adjusting the constraints that control the random test generator. For instance, if coverage is maximized in the branch prediction unit but not in the load-store unit, the verification engineer can increase the ratio of load and store instructions. However, this method significantly increases engineering effort, and therefore, slows down the verification process. To overcome this problem, researchers proposed several coverage-directed test generation mechanisms $\left\lbrack  {5,{20},{23},{56},{69},{70},{75}}\right\rbrack$ that automatically direct the next round of test generation that targets the uncovered parts of RTL. These works use RTL simulators to dynamically monitor the behavior of an RTL design and adjust the test generator constraints towards producing tests inputs that target uncovered RTL regions. Unfortunately, these works are generally DUT-specific which hinders their general applicability.

Table 3: The speedup achieved by ProcessorFuzz over no-cov-difuzzrtl, and reg-cov-difuzzrtl for the ground-truth bugs in the BOOM processor. We also report speedup of fp-csr and all-csr ProcessorFuzz configurations over selected ProcessorFuzz configuration. In the table, we state the maximum allowed runtime of 48 hours (172800 seconds) for bugs that could not be found.

<table><tr><td/><td>no-cov-difuzzrtl</td><td colspan="2">ProcessorFuzz (selected)</td><td colspan="3">ProcessorFuzz (all-csr)</td><td colspan="3">ProcessorFuzz (fp-csr)</td></tr><tr><td>Bug</td><td>Time (s)</td><td>Time (s)</td><td>Speedup (over no-cov)</td><td>Time (s)</td><td>Speedup (over no-cov)</td><td>Speedup (over selected)</td><td>Time (s)</td><td>Speedup (over no-cov)</td><td>Speedup (over selected)</td></tr><tr><td>1</td><td>464.9</td><td>230.2</td><td>2.02</td><td>430.2</td><td>1.08</td><td>0.54</td><td>1608.7</td><td>0.29</td><td>0.14</td></tr><tr><td>2</td><td>95695.0</td><td>57441.3</td><td>1.67</td><td>100804.9</td><td>0.95</td><td>0.57</td><td>122076.0</td><td>0.78</td><td>0.47</td></tr><tr><td>3</td><td>1520.1</td><td>1474.5</td><td>1.03</td><td>921.8</td><td>1.65</td><td>1.60</td><td>172800</td><td>0.01</td><td>0.01</td></tr><tr><td>4</td><td>585.3</td><td>308.0</td><td>1.90</td><td>558.8</td><td>1.05</td><td>0.55</td><td>13560.4</td><td>0.04</td><td>0.02</td></tr><tr><td>5</td><td>476.1</td><td>242.1</td><td>1.97</td><td>239.7</td><td>1.99</td><td>1.01</td><td>39150.9</td><td>0.01</td><td>0.01</td></tr><tr><td>6</td><td>172800</td><td>147942.3</td><td>1.17</td><td>148655.0</td><td>1.16</td><td>1.00</td><td>172800</td><td>1.00</td><td>0.86</td></tr><tr><td>7</td><td>5192.6</td><td>3264.6</td><td>1.59</td><td>172800</td><td>0.03</td><td>0.02</td><td>172800</td><td>0.03</td><td>0.02</td></tr><tr><td>Geo.</td><td>4018.1</td><td>2550.5</td><td>1.58</td><td>5420.9</td><td>0.74</td><td>0.47</td><td>47404.8</td><td>0.08</td><td>0.05</td></tr></table>

Formal verification methods (e.g., symbolic execution, model checking) are also widely used in hardware verification $\left\lbrack  {7,{12},{55}}\right\rbrack$ . These methods use mathematical reasoning to prove that a hardware design conforms to its specification. Unfortunately, formal verification methods have a well-known state explosion problem, and therefore, do not scale well for complex RTL designs such as a processor [18]. Indeed, a prior work [18] clearly presented that many processor bugs cannot be identified with these tools due to the space explosion issues and emphasised the necessity of complementary approaches to existing formal verificaiton tools.

### 5.2 Hardware Fuzzing

Over the past few years, fuzzing has gained traction in RTL verification due to its bug-finding success in the software domain [28]. In Table 4, we provide a high-level overview of all fuzzing-based RTL verification approaches. For each approach, we include the input format, the coverage metric used to guide the fuzzer, and the method to identify bugs.

RFUZZ [42] defines a simple input format (i.e., a series of bits) to increase the portability of hardware fuzzing to a wide range of RTL designs. Unfortunately, this input format is not effective when fuzzing processors since a processor requires instructions defined by an ISA. RFUZZ also proposes a new coverage metric, the multiplexer toggle coverage. RFUZZ monitors all the multiplexers in the RTL design. It retains an input for further mutations if the input toggles a previously uncovered multiplexer selection signal. A follow-up work by Li et al. [44] enhances RFUZZ with symbolic simulation and defines a full multiplexer toggle coverage metric that counts a multiplexer signal as covered for either 1-0-1 or 0-1-0 toggles. Both RFUZZ and Li et al. are highly coupled to Chisel HDL which limits the applicability of the approach [63]. Additionally, monitoring multiplexers in complex designs introduces excessive performance overhead [32]. ProcessorFuzz is agnostic to HDL and also does not require any instrumentation in the HDL code, which makes it both practical and efficient.

DIFUZZRTL monitors registers that directly or indirectly control multiplexer selection signals. This design choice makes it more efficient than RFUZZ since the total number of bits in the identified registers is significantly less than multiplexers. Moreover, DIFUZZRTL shows that RFUZZ's coverage metric does not precisely capture the FSM states. To mitigate this issue, DIFUZZRTL monitors value changes in the identified registers for each cycle. Unfortunately, DIFUZZRTL monitors many registers in the datapath as well, thereby misguiding the fuzzer as detailed in Section 2.3.

Trippel et al. [72] translate hardware designs to software models and fuzzes those models. This way, available coverage metrics used by software fuzzers (e.g., basic block, edge) can be used for fuzzing hardware as well. However, this method of converting hardware designs to software models introduces additional challenges such as proving the equivalency between hardware design and software model [63].

A recent processor fuzzer, TheHuzz [73], relies on a variety of coverage metrics extracted using industrial-standard tools such as Cadence [8] and ModelSim [65]. TheHuzz proposes an optimization strategy to increase the effectiveness of fuzzing in discovering bugs. Specifically, TheHuzz profiles individual instructions and determines optimum instruction and mutation pairs while generating new set of inputs. This way, TheHuzz associates individual instructions with relevant mutation strategies and aims to guide fuzzing towards buggy processor states. Unlike DIFUZZRTL or ProcessorFuzz, TheHuzz does not propose a new coverage metric . TheHuzz relies on several coverage metrics used in software testing (i.e., statement, branch, line, expression). As discussed by prior works $\left\lbrack  {{32},{70}}\right\rbrack$ , these metrics are not sufficient metrics to verify a processor. Besides, D-flip flop (DFF) toggle coverage misses certain states as detailed by DIFUZZRTL. Finally, it is not clear how registers that control FSM coverage are identified as the industrial-tools are not open-sourced. Moreover, the runtime overhead of TheHuzz is higher (71%) than ProcessorFuzz due to the instrumentation applied by industrial tools and profiling coverage. We could not quantatively compare ProcessorFuzz with TheHuzz as TheHuzz is not open sourced.

The common goal of the aforementioned fuzzing works is to maximize coverage of an RTL design, thereby discovering bugs across the entire RTL design. Researchers have also proposed fuzzing frameworks for achieving alternate verification goals. For instance,

Table 4: Existing RTL Fuzzers.

<table><tr><td/><td>Input Format</td><td>Coverage Metric</td><td>Evaluated RTL Designs</td><td>Bug Discovery Method</td></tr><tr><td>RFUZZ [42]</td><td>A Series of Bits</td><td>Mux Toggle</td><td>Peripherals, RISC-V Processors (Sodor 1-3-5)</td><td>Assertion</td></tr><tr><td>Li et. al [44]</td><td>A Series of Bits</td><td>Full Mux Toggle</td><td>Custom RISC-V Processor, OpenCore 1200</td><td>Assertion</td></tr><tr><td>DIFUZZRTL [32]</td><td>Assembly</td><td>Register Coverage</td><td>RISC-V Processors (BOOM, Mork1x, Rocket Chip)</td><td>Golden Model</td></tr><tr><td>DirectFuzz [9]</td><td>A Series of Bits</td><td>Mux Toggle</td><td>Same as RFUZZ</td><td>Assertion</td></tr><tr><td>Trippel et al. [72]</td><td>Byte Sequence</td><td>Edge Coverage</td><td>RISC-V IP Cores (AES, HMAC, KMAC, Timer)</td><td>Golden Model Assertion</td></tr><tr><td>TheHuzz [73]</td><td>Assembly</td><td>Branch, Line, Statement, Expression, DFF Toggle, FSM</td><td>RISC-V Processors ( Rocket Chip, CVA6), mor1kx, OpenCore 1200</td><td>Golden Model</td></tr><tr><td>HYPERFUZZER [54]</td><td>A Series of Bits</td><td>High-Level</td><td>Custom SoC</td><td>Property Check</td></tr><tr><td>Logic Fuzzer [38]</td><td>A Series of Bits, Random Data</td><td>N/A</td><td>RISC-V Processors (BlackParrot, BOOM, CVA6)</td><td>Golden Model</td></tr><tr><td>ProcessorFuzz (this work)</td><td>Assembly</td><td>Control Path Register, ISA-Sim Transition</td><td>RISC-V processors (BOOM, BlackParrot, Rocket Chip)</td><td>Golden Model</td></tr></table>

DirectFuzz [9] adapts the notion of directed greybox fuzzing and applies it to the RTL verification. Contrary to the aforementioned common goal of fuzzing, the goal of DirectFuzz is to cover certain specific RTL regions with a targeted fuzzing approach. Here, the motivation is to dedicate more fuzzing time to the RTL components that need to undergo thorough testing. HYPERFUZZER [54] introduces a new grammar that represents the hardware security properties. During fuzzing, HYPERFUZZER checks if any of the fuzzer-generated inputs violates a security property. Defining properties requires human expertise which is error-prone. Logic Fuzzer [38] randomizes control signals and states of a DUT without compromising the functional correctness of the DUT. Logic Fuzzer needs to be provided with fuzzing targets (e.g., congestible points in an RTL design), and therefore requires domain expertise. INTROSPEC-TRE [24] and Osiris [76] use blackbox fuzzing approach to discover microarchitectural side channels (i.e., Meltdown [46] and Spectre [40]) in processors.

### 5.3 Differential Testing in Software Domain

Differential testing is commonly used in the software domain to discover inconsistencies (e.g., semantic bugs, side-channels, consensus bugs) across multiple programs with similar functionalities. One use case of differential testing in the software domain is to identify discrepancies between emulators and real hardware. Prior works $\left\lbrack  {{49},{59},{64}}\right\rbrack$ aim to eliminate the source of discrepancies in emulation environments since adversaries use discrepancies to infer the execution environment and bypass malware analysis. ProcessorFuzz differs from these works in two ways. First, Proces-sorFuzz's test input generation is coverage-guided whereas these works employ blackbox fuzzing. Second, these works aim to identify discrepancies that may or may not necessarily translate to actual bugs. Besides emulators, differential testing has been used to test different types of software including Web application firewalls [2], SSL/TLS libraries [6, 15, 62, 66], compilers [79], cryptocurrency protocols $\left\lbrack  {{21},{41},{80}}\right\rbrack$ , deep learning systems $\left\lbrack  {{58},{60}}\right\rbrack$ , Java Virtual Machines [13, 14], PDF viewers [62], mobile applications [37], file systems [51], and Java programs [57, 58].

## 6 DISCUSSION AND LIMITATIONS

Other ISAs. In this work, we demonstrated the capability of Proces-sorFuzz using the RISC-V ISA. However, CSRs are not only specific to the RISC-V architecture and defined as part of many other ISAs including x86. Therefore, ProcessorFuzz is not limited to the RISC-V-based processors and can be used in processors based on other ISAs.

Unintended RTL Transitions. ProcessorFuzz uses ISA simulation as part of a feedback mechanism since it is faster and agnostic to the HDL. ProcessorFuzz does not use an input for RTL simulation if the input lacks of a unique transition in its ISA simulation trace. One limitation of this design choice is that ProcessorFuzz can potentially miss certain bugs that follow the given scenario. If a test input would result in an unintended transition in RTL simulation but the same test input does not cause any unique transition in ISA simulation, such a test input will be discarded. Hence, the bug will not be identified.

No RTL Coverage. ProcessorFuzz does not collect feedback (i.e., CSR transitions) from the RTL design during fuzzing. However, we can extend its design and collect coverage from the RTL during RTL simulation to further aid fuzzing. Note that ProcessorFuzz is able to discover all bugs found by prior work in its current form without using any feedback from the RTL design during fuzzing.

## 7 CONCLUSION

This work presents ProcessorFuzz, a processor fuzzer guided by a novel CSR-transition coverage feedback obtained from ISA simulation. ProcessorFuzz demonstrates that monitoring CSR transitions can effectively guide fuzzing towards buggy processor states. Moreover, using ISA simulation instead of RTL simulation can quickly eliminate inputs that result in the same coverage, thereby helping the fuzzer to test as many qualitatively different inputs as possible. Our experimental results discovered eight new bugs in established, real-world, RISC-V processors, and one new bug in a reference model.

## REFERENCES

[1] Vineeth V Acharya, Sharad Bagri, and Michael S Hsiao. 2015. Branch guided functional test generation at the RTL. In 2015 20th IEEE European Test Symposium (ETS). IEEE, 1-6.

[2] George Argyros, Ioannis Stais, Suman Jana, Angelos D Keromytis, and Agge-los Kiayias. 2016. Sfadiff: Automated evasion attacks and fingerprinting using black-box differential automata learning. In Proceedings of the 2016 ACM SIGSAC conference on computer and communications security. 1690-1701.

[3] Krste Asanović, Rimas Avizienis, Jonathan Bachrach, Scott Beamer, David Bian-colin, Christopher Celio, Henry Cook, Daniel Dabbelt, John Hauser, Adam Izraele-vitz, Sagar Karandikar, Ben Keller, Donggyu Kim, John Koenig, Yunsup Lee, Eric Love, Martin Maas, Albert Magyar, Howard Mao, Miquel Moreto, Albert Ou, David A. Patterson, Brian Richards, Colin Schmidt, Stephen Twigg, Huy Vo, and Andrew Waterman. 2016. The Rocket Chip Generator. Technical Report UCB/EECS-2016-17. EECS Department, University of California, Berkeley. http://www2.eecs.berkeley.edu/Pubs/TechRpts/2016/EECS-2016-17.html

[4] J. Bachrach, H. Vo, B. Richards, Y. Lee, A. Waterman, R Avižienis, J. Wawrzynek, and K. Asanović. 2012. Chisel: Constructing hardware in a Scala embedded language. In DAC Design Automation Conference 2012. 1212-1221.

[5] Mrinal Bose, Jongshin Shin, Elizabeth M Rudnick, Todd Dukes, and Magdy Abadir. 2001. A genetic approach to automatic bias generation for biased random instruction generation. In Proceedings of the 2001 Congress on Evolutionary Computation (IEEE Cat. No. 01TH8546), Vol. 1. IEEE, 442-448.

[6] Chad Brubaker, Suman Jana, Baishakhi Ray, Sarfraz Khurshid, and Vitaly Shmatikov. 2014. Using frankencerts for automated adversarial testing of certificate validation in SSL/TLS implementations. In 2014 IEEE Symposium on Security and Privacy. IEEE, 114-129.

[7] Cadence. 2019. JasperGold Formal Verification Platform.

[8] Cadence. 2022. Circuit Simulation. https://www.cadence.com/en_US/home/tools /custom-ic-analog-rf-design/circuit-simulation.html.

[9] Sadullah Canakci, Leila Delshadtehrani, Furkan Eris, Michael Bedford Taylor, Manuel Egele, and Ajay Joshi. 2021. Directfuzz: Automated test generation for rtl designs using directed graybox fuzzing. In Proceedings of the 58th annual Design Automation Conference.

[10] Christopher Celio, David A. Patterson, and Krste Asanović. 2015. The Berkeley Out-of-Order Machine (BOOM): An Industry-Competitive, Synthesizable, Parameterized RISC-V Processor. Technical Report UCB/EECS-2015-167. EECS Department, University of California, Berkeley. http://www2.eecs.berkeley.edu/Pubs/TechRp ts/2015/EECS-2015-167.html

[11] C. P. Celio. 2017. A Highly Productive Implementation of an Out-of-Order Processor Generator. Ph. D. Dissertation. UC Berkeley.

[12] Mingsong Chen and Prabhat Mishra. 2011. Property learning techniques for efficient generation of directed tests. IEEE Trans. Comput. 60, 6 (2011), 852-864.

[13] Yuting Chen, Ting Su, and Zhendong Su. 2019. Deep differential testing of JVM implementations. In 2019 IEEE/ACM 41st International Conference on Software Engineering (ICSE). IEEE, 1257-1268.

[14] Yuting Chen, Ting Su, Chengnian Sun, Zhendong Su, and Jianjun Zhao. 2016. Coverage-directed differential testing of JVM implementations. In proceedings of the 37th ACM SIGPLAN Conference on Programming Language Design and Implementation. 85-99.

[15] Yuting Chen and Zhendong Su. 2015. Guided differential testing of certificate validation in SSL/TLS implementations. In Proceedings of the 2015 10th Joint Meeting on Foundations of Software Engineering. 793-804.

[16] Robert R. Collins. 1998. The Pentium FOOF bug. Dr. Dobb's Journal. https: //www.drdobbs.com/embedded-systems/the-pentium-f00f-bug/184410555.

[17] National Vulnerability Database. 2020. CVE-2020-10029 Detail. https://nvd.nist.g ov/vuln/detail/CVE-2020-10029.

[18] Ghada Dessouky, David Gens, Patrick Haney, Garrett Persyn, Arun Kanuparthi, Hareesh Khattri, Jason M Fung, Ahmad-Reza Sadeghi, and Jeyavijayan Rajen-dran. 2019. Hardfails: Insights into software-exploitable hardware bugs. In 28th \{USENIX\} Security Symposium (\{USENIX\} Security 19). 213-230.

[19] Alan Edelman. 1997. The Mathematics of the Pentium Division Bug. SIAM Rev. 39, 1 (1997), 54-67. http://www.jstor.org/stable/2133004

[20] Shai Fine and Avi Ziv. 2003. Coverage directed test generation for functional verification using bayesian networks. In Proceedings of the 40th annual Design Automation Conference. 286-291.

[21] Ying Fu, Meng Ren, Fuchen Ma, Heyuan Shi, Xin Yang, Yu Jiang, Huizhong Li, and Xiang Shi. 2019. Evmfuzzer: detect evm vulnerabilities via fuzz testing. In Proceedings of the 2019 27th ACM Joint Meeting on European Software Engineering Conference and Symposium on the Foundations of Software Engineering. 1110- 1114.

[22] Inc Futurewei Technologies. 2020. force-riscv. https://github.com/openhwgroup /force-riscv.

[23] Raviv Gal, Eldad Haber, Wesam Ibraheem, Brian Irwin, Ziv Nevo, and Avi Ziv. 2021. Automatic Scalable System for the Coverage-Directed Generation (CDG) Problem. In 2021 Design, Automation & Test in Europe Conference & Exhibition (DATE). IEEE, 206-211.

[24] Moein Ghaniyoun, Kristin Barber, Yinqian Zhang, and Radu Teodorescu. 2021. INTROSPECTRE: A Pre-Silicon Framework for Discovery and Analysis of Transient Execution Vulnerabilities. In 2021 ACM/IEEE 48th Annual International Symposium on Computer Architecture (ISCA). IEEE, 874-887.

[25] Google. 2016. OSS-Fuzz: Continuous Fuzzing for Open Source Software. https: //github.com/google/oss-fuzz.

[26] Google. 2017. American Fuzzy Lop. https://github.com/google/AFL.

[27] Google. 2021. RISCV-DV. https://github.com/google/riscv-dv.

[28] Google. 2022. OSS-Fuzz. https://google.github.io/oss-fuzz/#trophies.

[29] Shakti Group. 2018. Shakti AAPG. https://gitlab.com/shaktiproject/tools/aapg/- /wikis/Wiki.

[30] Ahmad Hazimeh, Adrian Herrera, and Mathias Payer. 2020. Magma: A Ground-Truth Fuzzing Benchmark. ACM POMACS 4, 3 (2020), 1-29.

[31] Vladimir Herdt, Daniel Große, Eyck Jentzsch, and Rolf Drechsler. 2020. Efficient cross-level testing for processor verification: A RISC-V case-study. In 2020 Forum for Specification and Design Languages (FDL). IEEE, 1-7.

[32] J. Hur, S. Song, D. Kwon, E. Baek, J. Kim, and B. Lee. 2021. DiFuzzRTL: Differential Fuzz Testing to Find CPU Bugs. In 2021 2021 IEEE Symposium on Security and Privacy (SP). IEEE Computer Society, Los Alamitos, CA, USA, 1286-1303. https: //doi.org/10.1109/SP40001.2021.00103

[33] Intel. 1994. Intel Annual Report. https://www.intel.com/content/www/us/en/hi story/history-1994-annual-report.html.

[34] Intel. 2022. Machine Check Error Avoidance on Page Size Change. https://www.intel.com/content/www/us/en/developer/articles/troubleshooting/software-security-guidance/technical-documentation/machine-check-error-avoidance-page-size-change.html.

[35] A. Izraelevitz, J. Koenig, P. Li, R. Lin, A. Wang, A. Magyar, D. Kim, C. Schmidt, C. Markley, J. Lawson, and J. Bachrach. 2017. Reusability is FIRRTL ground: Hardware construction languages, compiler frameworks, and transformations. In 2017 IEEE/ACM International Conference on Computer-Aided Design (ICCAD). 209-216.

[36] Yeongjin Jang, Sangho Lee, and Taesoo Kim. 2016. Breaking Kernel Address Space Layout Randomization with Intel TSX. In Proceedings of the 2016 ACM SIGSAC Conference on Computer and Communications Security. 380-392.

[37] Jaeyeon Jung, Anmol Sheth, Ben Greenstein, David Wetherall, Gabriel Maganis, and Tadayoshi Kohno. 2008. Privacy oracle: a system for finding application leaks with black box differential testing. In Proceedings of the 15th ACM conference on Computer and communications security. 279-288.

[38] Nursultan Kabylkas, Tommy Thorn, Shreesha Srinath, Polychronis Xekalakis, and Jose Renau. 2021. Effective Processor Verification with Logic Fuzzer Enhanced Co-simulation. In MICRO-54: 54th Annual IEEE/ACM International Symposium on Microarchitecture. 667-678.

[39] George Klees, Andrew Ruef, Benji Cooper, Shiyi Wei, and Michael Hicks. 2018. Evaluating fuzz testing. In ACM SIGSAC CCS. 2123-2138.

[40] Paul Kocher, Jann Horn, Anders Fogh, Daniel Genkin, Daniel Gruss, Werner Haas, Mike Hamburg, Moritz Lipp, Stefan Mangard, Thomas Prescher, Michael Schwarz, and Yuval Yarom. 2019. Spectre Attacks: Exploiting Speculative Execution. In 2019 IEEE Symposium on Security and Privacy (SP). 1-19.

[41] EVM lab. 2021. EVM lab utilities: Utilities for inter- acting with the Ethereum virtual machine. https://github.com/ethereum/evmlab.

[42] Kevin Laeufer, Jack Koenig, Donggyu Kim, Jonathan Bachrach, and Koushik Sen. 2018. RFUZZ: Coverage-Directed Fuzz Testing of RTL on FPGAs. In 2018 IEEE/ACM International Conference on Computer-Aided Design (ICCAD). 1-8. https://doi.org/10.1145/3240765.3240842

[43] Yunsup Lee and Henry Cook. 2015. riscv-torture. https://github.com/ucb-bar/riscv-torture.

[44] Tun Li, Hongji Zou, Dan Luo, and Wanxia Qu. 2021. Symbolic Simulation Enhanced Coverage-Directed Fuzz Testing of RTL Design. In 2021 IEEE International Symposium on Circuits and Systems (ISCAS). IEEE, 1-5.

[45] Yuwei Li, Shouling Ji, Yuan Chen, Sizhuang Liang, Wei-Han Lee, Yueyao Chen, Chenyang Lyu, Chunming Wu, Raheem Beyah, and Peng Cheng. 2021. Unifuzz: A holistic and pragmatic metrics-driven platform for evaluating fuzzers. In USENIX Security.

[46] Moritz Lipp, Michael Schwarz, Daniel Gruss, Thomas Prescher, Werner Haas, Anders Fogh, Jann Horn, Stefan Mangard, Paul Kocher, Daniel Genkin, Yuval Yarom, and Mike Hamburg. 2018. Meltdown: Reading Kernel Memory from User Space. In 27th USENIX Security Symposium (USENIX Security 18). 973-990.

[47] LLVM. 2021. libFuzzer. https://llvm.org/docs/LibFuzzer.html#corpus.

[48] Valentin Jean Marie Manès, HyungSeok Han, Choongwoo Han, Sang Kil Cha, Manuel Egele, Edward J Schwartz, and Maverick Woo. 2019. The art, science, and engineering of fuzzing: A survey. IEEE Transactions on Software Engineering (2019).

[49] Lorenzo Martignoni, Roberto Paleari, Giampaolo Fresi Roglia, and Danilo Bruschi. 2009. Testing CPU emulators. In Proceedings of the eighteenth international symposium on Software testing and analysis. 261-272.

[50] Microsoft. 2020. onefuzz. https://github.com/microsoft/onefuzz.

[51] Changwoo Min, Sanidhya Kashyap, Byoungyoung Lee, Chengyu Song, and Tae-soo Kim. 2015. Cross-checking semantic correctness: The case of finding file system bugs. In Proceedings of the 25th Symposium on Operating Systems Principles.

[52] MITRE. 2019. Hardware design CWEs. https://cwe.mitre.org/data/definitions/1 194.html.

[53] Dinos Moundanos, Jacob A Abraham, and Yatin Vasant Hoskote. 1998. Abstraction techniques for validation coverage analysis and test generation. IEEE Trans. Comput. 47, 1 (1998), 2-14.

[54] Sujit Kumar Muduli, Gourav Takhar, and Pramod Subramanyan. 2020. Hy-perfuzzing for SoC security validation. In Proceedings of the 39th International Conference on Computer-Aided Design. 1-9.

[55] Rajdeep Mukherjee, Daniel Kroening, and Tom Melham. 2015. Hardware verification using software analyzers. In 2015 IEEE Computer Society Annual Symposium on VLSI. IEEE, 7-12.

[56] Gilly Nativ, S Mittennaier, Shmuel Ur, and Avi Ziv. 2001. Cost evaluation of coverage directed test generation for the IBM mainframe. In Proceedings International Test Conference 2001 (Cat. No. 01CH37260). IEEE, 793-802.

[57] Shirin Nilizadeh, Yannic Noller, and Corina S Pasareanu. 2019. DifFuzz: differential fuzzing for side-channel analysis. In 2019 IEEE/ACM 41st International Conference on Software Engineering (ICSE). IEEE, 176-187.

[58] Yannic Noller, Corina S Păsăreanu, Marcel Böhme, Youcheng Sun, Hoang Lam Nguyen, and Lars Grunske. 2020. HyDiff: Hybrid differential software analysis. In 2020 IEEE/ACM 42nd International Conference on Software Engineering (ICSE). IEEE, 1273-1285.

[59] Roberto Paleari, Lorenzo Martignoni, Giampaolo Fresi Roglia, and Danilo Bruschi. 2009. A fistful of red-pills: How to automatically generate procedures to detect CPU emulators. In Proceedings of the USENIX Workshop on Offensive Technologies (WOOT), Vol. 41. 86.

[60] Kexin Pei, Yinzhi Cao, Junfeng Yang, and Suman Jana. 2017. Deepxplore: Automated whitebox testing of deep learning systems. In proceedings of the 26th Symposium on Operating Systems Principles. 1-18.

[61] Daniel Petrisko, Farzam Gilani, Mark Wyse, Dai Cheol Jung, Scott Davidson, Paul Gao, Chun Zhao, Zahra Azad, Sadullah Canakci, Bandhav Veluri, Tavio Guarino, Ajay Joshi, Mark Oskin, and Michael Bedford Taylor. 2020. BlackParrot: An Agile Open-Source RISC-V Multicore for Accelerator SoCs. IEEE Micro 40, 4 (2020), 93-102.

[62] Theofilos Petsios, Adrian Tang, Salvatore Stolfo, Angelos D Keromytis, and Suman Jana. 2017. Nezha: Efficient domain-independent differential testing. In 2017 IEEE Symposium on security and privacy (SP). IEEE, 615-632.

[63] Ahmad-Reza Sadeghi, Jeyavijayan Rajendran, and Rahul Kande. 2021. Organizing The World's Largest Hardware Security Competition: Challenges, Opportunities, and Lessons Learned. In Proceedings of the 2021 on Great Lakes Symposium on VLSI. 95-100.

[64] Onur Sahin, Ayse K Coskun, and Manuel Egele. 2018. Proteus: Detecting android emulators from instruction-level profiles. In International Symposium on Research in Attacks, Intrusions, and Defenses. Springer, 3-24.

[65] SIEMENS. 2022. ModelSim. https://eda.sw.siemens.com/en-US/ic/modelsim/.

[66] Suphannee Sivakorn, George Argyros, Kexin Pei, Angelos D Keromytis, and Suman Jana. 2017. HVLearn: Automated black-box analysis of hostname verification in SSL/TLS implementations. In 2017 IEEE Symposium on Security and Privacy (SP). IEEE, 521-538.

[67] Wilson Snyder. 2018. Verilator, a Verilog/Systemverilog simulator and compiler. https://www.veripool.org/verilator/.

[68] RISC-V Software. 2019. Spike RISC-V ISA Simulator. https://github.com/riscv-software-src/riscv-isa-sim.

[69] Giovanni Squillero. 2005. MicroGP-an evolutionary assembly program generator. Genetic Programming and Evolvable Machines 6, 3 (2005), 247-263.

[70] Serdar Tasiran, Farzan Fallah, David G Chinnery, Scott J Weber, and Kurt Keutzer. 2001. A functional validation technique: biased-random simulation guided by observability-based coverage. In Proceedings 2001 IEEE International Conference on Computer Design: VLSI in Computers and Processors. ICCD 2001. IEEE, 82-88.

[71] Esperanto Technologies. 2019. Dromajo - Esperanto Technology's RISC-V Reference Model. https://github.com/chipsalliance/dromajo.

[72] Timothy Trippel, Kang G Shin, Alex Chernyakhovsky, Garret Kelly, Dominic Rizzo, and Matthew Hicks. 2021. Fuzzing Hardware Like Software. arXiv preprint arXiv:2102.02308 (2021).

[73] Aakash Tyagi, Addison Crump, Ahmad-Reza Sadeghi, Garrett Persyn, Jeyavijayan Rajendran, Patrick Jauernig, and Rahul Kande. 2022. TheHuzz: Instruction Fuzzing of Processors Using Golden-Reference Models for Finding Software-Exploitable Vulnerabilities. arXiv preprint arXiv:2201.09941 (2022).

[74] Jo Van Bulck, Marina Minkin, Ofir Weisse, Daniel Genkin, Baris Kasikci, Frank Piessens, Mark Silberstein, Thomas F. Wenisch, Yuval Yarom, and Raoul Strackx. 2018. Foreshadow: Extracting the Keys to the Intel SGX Kingdom with Transient Out-of-Order Execution. In Proceedings of the 27th USENIX Security Symposium.

[75] Ilya Wagner, Valeria Bertacco, and Todd Austin. 2005. StressTest: an automatic approach to test generation via activity monitors. In Proceedings of the 42nd annual Design Automation Conference. 783-788.

[76] Daniel Weber, Ahmad Ibrahim, Hamed Nemati, Michael Schwarz, and Christian Rossow. 2021. Osiris: Automated Discovery of Microarchitectural Side Channels. In 30th USENIX Security Symposium (USENIX Security 21). USENIX Association, 1415-1432. https://www.usenix.org/conference/usenixsecurity21/presentation/ weber

[77] Rafal Wojtczuk. 2012. PV Privilege Escalation. https://lists.xen.org/archives/htm l/xen-announce/2012-06/msg00001.html.

[78] Clifford Wolf. 2014. Yosys Open SYnthesis Suite. https://yosyshq.net/yosys/.

[79] Xuejun Yang, Yang Chen, Eric Eide, and John Regehr. 2011. Finding and understanding bugs in C compilers. In Proceedings of the 32nd ACM SIGPLAN conference on Programming language design and implementation. 283-294.

[80] Youngseok Yang, Taesoo Kim, and Byung-Gon Chun. 2021. Finding Consensus Bugs in Ethereum via Multi-transaction Differential Fuzzing. In 15th \{USENIX\} Symposium on Operating Systems Design and Implementation ( $\{$ OSDI $\} {21}$ ). 349- 365.

## A SELECTED AND EXCLUDED CSRS

In this section, we provide the reasoning for the CSR selection in the ProcessorFuzz implementation for RISC-V ISA. We used the criteria mentioned in 3.3.3 to select the CSRs. In general, we selected status CSRs under first criteria (C1) and CSRs that change the configuration of the processor under the second criteria (C2). In Table 5, we show the selected CSRs along with the criteria that was used to select them.

We also provide a list of the CSRs that are implemented in the RISC-V cores, but excluded from the selection in Table 6. Exclusion of CSRs is done based on three intuitive reasons. First, we exclude any CSR that holds the same value throughout all tests. For example, we maintain the same physical memory protection (PMP) configuration for all tests. Therefore, we exclude the CSRs that configure PMP (pmpcfg and pmpaddr) because they are not expected to cause CSR transitions. We exclude misa, mhartid, mtvec, satp and stvec CSRs with the same reasoning.

Second, we exclude any CSRs that are not supported by the testing infrastructure. For instance, RISC-V ISA vector extension is not supported in our current instruction generator. Intuitively, vector extension CSRs (vstart, vxsat and vxrm) are not expected to cause any transitions when the vector instructions are not present. Hence, we exclude any CSRs from vector extension. Similarly, debug extension CSRs and CSRs related to handling interrupts are excluded.

Third, we exclude CSRs that does not directly represent the architectural state of the processor. These registers contain information to assist designers during analysis of a hardware bug rather than revealing the fundamental issue. For example, hardware performance-monitoring counters (HPCs) provide information for hardware to assist several debugging use cases including performance bottlenecks. Similarly, CSRs that assist in context switching and trap handling (tval, scratch and epc CSRs) are excluded because they similarly do not reveal the origin of a bug. Also, note that we already monitor the trap cause CSRs (mcause, scause) and mstatus CSR to capture any changes in the architectural state due to exceptions and context switches.

Four, we exclude CSR that is already a subset of a CSR that we are already monitoring. For example, sstatus CSR is excluded because it is a subset of mstatus CSR.

Table 5: CSR selection for RISC-V ISA implementation of ProcessorFuzz along with the criteria that was used to select them. Here, C1 and C2 correspond to two criteria that we describe in Section 3.3.3.

<table><tr><td>CSR Group</td><td>CSR</td><td>Description</td><td>Criteria</td></tr><tr><td rowspan="18">Privileged</td><td>mstatus.xIE</td><td>Controls the global interrupt enable bit for privilege $\mathrm{x},\mathrm{x} = \{ \mathrm{M},\mathrm{S},\mathrm{U}\}$</td><td>C2</td></tr><tr><td>mstatus.xPIE</td><td>Holds the value of interrupt-enable bit active prior to the trap for privilege mode x</td><td>C1</td></tr><tr><td>mstatus.xPP</td><td>Holds the previous privilege mode active prior to a trap taken to privilege mode x</td><td>C1</td></tr><tr><td>mstatus.XS</td><td>Contains the state of any additional user-mode extensions</td><td>C1</td></tr><tr><td>mstatus.FS</td><td>Contains the state of the floating-point unit</td><td>C1</td></tr><tr><td>mstatus.MPRV</td><td>Controls the privilege mode in which the memory operations are performed</td><td>C2</td></tr><tr><td>mstatus.SUM</td><td>Controls the permission for accessing user memory from supervisor mode</td><td>C2</td></tr><tr><td>mstatus.MXR</td><td>Controls the privilege with which loads access virtual memory</td><td>C2</td></tr><tr><td>mstatus.TVM</td><td>Controls the ability to edit virtual-memory configuration from supervisor mode</td><td>C2</td></tr><tr><td>mstatus.TW</td><td>Controls the privilege modes that wait for interrupt (WFI) is allowed to execute</td><td>C2</td></tr><tr><td>mstatus.TSR</td><td>Provides the ability to trigger a trap when SRET instruction is executed in supervisor mode</td><td>C2</td></tr><tr><td>mstatus.xXL</td><td>Controls the width of an integer register for privilege mode $\mathrm{x},\mathrm{x} = \{ \mathrm{S},\mathrm{U}\}$</td><td>C2</td></tr><tr><td>mstatus.SD</td><td>Indicate the combined state of mstatus.FS and mstatus.XS for context switches</td><td>C1</td></tr><tr><td>meause</td><td>Contains the trap cause when a trap is taken in to machine mode</td><td>C1</td></tr><tr><td>scause</td><td>Contains the trap cause when a trap is taken in to supervisor mode</td><td>C1</td></tr><tr><td>medeleg</td><td>Decides what type of exceptions are delegated to supervisor mode from machine mode</td><td>C2</td></tr><tr><td>mcounteren</td><td>Controls the availability of the hardware performance-monitoring counters for supervisor mode</td><td>C2</td></tr><tr><td>scounteren</td><td>Controls the availability of the hardware performance-monitoring counters for user mode</td><td>C2</td></tr><tr><td rowspan="2">Unprivileged</td><td>frm</td><td>Controls the dynamic rounding mode for floating-point operations</td><td>C2</td></tr><tr><td>fflags</td><td>Holds the accrued exceptions from the floating-point operations</td><td>C1</td></tr></table>

Table 6: CSRs not monitored by ProcessorFuzz along with the reason for exclusion.

<table><tr><td>Category</td><td>CSR</td><td>Description</td><td>Reason for Exclusion</td></tr><tr><td rowspan="6">Privileged</td><td>sstatus</td><td>Holds the supervisor mode operating status of the processor</td><td>Subset of mstatus</td></tr><tr><td>misa</td><td>Reports the CPU capabilities of a hart</td><td rowspan="7">Holds a constant value during testing</td></tr><tr><td>mhartid</td><td>Contains the integer ID of the hardware thread running the code</td></tr><tr><td>mtvec</td><td>Contains the trap handler base address and vector configuration for machine mode</td></tr><tr><td>satp</td><td>Controls supervisor-mode address translation and protection</td></tr><tr><td>stvec</td><td>Contains the trap handler base address and vector configuration for supervisor mode</td></tr><tr><td rowspan="2">PMP</td><td>pmpcfg</td><td>Contains the physical memory protection configuration</td></tr><tr><td>pmpaddr</td><td>Contains the physical memory protection addresses</td></tr><tr><td rowspan="5">Interrupt</td><td>mip</td><td>Reports pendng interrupts in machine mode</td><td rowspan="13">Not supported by the testing infrastructure</td></tr><tr><td>mie</td><td>Control what interrupts are enabled in machine mode</td></tr><tr><td>mideleg</td><td>Decides what type of interrupts are delegated from machine mode to supervisor mode</td></tr><tr><td>sie</td><td>Reports pendng interrupts in supervisor mode</td></tr><tr><td>sip</td><td>Control what interrupts are enabled in supervisor mode</td></tr><tr><td rowspan="5">Debug Extension</td><td>dcsr</td><td>Contains the configuration and status of debug extension</td></tr><tr><td>dpc</td><td>Holds the program counter of the next instruction to be executed before entering debug mode</td></tr><tr><td>dscratch</td><td>Optional scratch register that holds temporary values</td></tr><tr><td>tselect</td><td>Control which trigger is accessible through the other trigger registers</td></tr><tr><td>tdata1-3</td><td>Holds trigger-specific data</td></tr><tr><td rowspan="3">Vector Extension</td><td>vstart</td><td>Holds the index of the first element to be executed by a vector instruction</td></tr><tr><td>vxsat</td><td>Holds the saturation flag for fixed-point operations</td></tr><tr><td>vxrm</td><td>Controls the rounding mode used in the vector extension</td></tr><tr><td rowspan="5">HPC</td><td>mcountinhibit</td><td>Controls which hardware performance-monitoring counters are allowed to increment</td><td rowspan="11">Does not directly reveal the origin of a potential bug</td></tr><tr><td>cycle</td><td>Holds the elapsed cycle count of the CPU</td></tr><tr><td>instret</td><td>Holds the number of retired instruction count</td></tr><tr><td>hpmevent</td><td>Hardware performance-monitoring event selector</td></tr><tr><td>hpmcounter</td><td>Performance-monitoring counter of the event selected by hpmevent</td></tr><tr><td rowspan="6">Privileged (assisting trap handling and context switches)</td><td>mtval</td><td>Hold the exception-specific information when a trap is taken to machine mode</td></tr><tr><td>mscratch</td><td>Holds a pointer to the machine mode context space while the hart executes in lower privilege</td></tr><tr><td>mepc</td><td>Contains the program counter of an instruction that caused an exception in machine mode</td></tr><tr><td>stval</td><td>Hold the exception-specific information when a trap is taken to supervisor mode</td></tr><tr><td>sscratch</td><td>Holds a pointer to the supervisor mode context space while the hart executes in user mode</td></tr><tr><td>sepc</td><td>Contains the program counter of an instruction that caused an exception in supervisor mode</td></tr></table>

