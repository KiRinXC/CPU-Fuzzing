# The Emergence of Hardware Fuzzing: A Critical Review of its Significance

Raghul Saravanan and Sai Manoj Pudukotai Dinakarrao

Department of Electrical and Computer Engineering, George Mason University, Fairfax, VA, USA \{rsaravan, spudukot\}@gmu.edu

${Abstract}$ -In recent years, there has been a notable surge in attention towards hardware security, driven by the increasing complexity and integration of processors, SoCs, and third-party IPs aimed at delivering advanced solutions. However, this complexity also introduces vulnerabilities and bugs into hardware systems, necessitating early detection during the IC design cycle to uphold system integrity and mitigate re-engineering costs. While the Design Verification (DV) community employs dynamic and formal verification strategies, they encounter challenges such as scalability for intricate designs and significant human intervention, leading to prolonged verification durations. As an alternative approach, hardware fuzzing, inspired by software testing methodologies, has gained prominence for its efficacy in identifying bugs within complex hardware designs. Despite the introduction of various hardware fuzzing techniques, obstacles such as inefficient conversion of hardware modules into software models impede their effectiveness. This Systematization of Knowledge (SoK) initiative delves into the fundamental principles of existing hardware fuzzing, methodologies, and their applicability across diverse hardware designs. Additionally, it evaluates factors such as the utilization of golden reference models (GRMs), coverage metrics, and toolchains to gauge their potential for broader adoption, akin to traditional formal verification methods. Furthermore, this work examines the reliability of existing hardware fuzzing techniques in identifying vulnerabilities and identifies research gaps for future advancements in design verification techniques.

## I. INTRODUCTION

Addressing the intricacies of the global semiconductor supply chain demands collaborative efforts from integrated circuit (IC) designers and vendors [1]. Various IC design companies or teams actively participate and contribute across the spectrum of IC design cycle, encompassing design, testing, fabrication, packaging, and integration [2]. This collaborative engagement is integral to achieving comprehensive solutions that meet the dynamic demands of the semiconductor industry. For example, in the design and evaluation of the Apple ${}^{ \otimes  }{15}$ chip, more than 11 third-party entities have been involved in delivering sophisticated solutions for the device [3].

Such complex modern IC design flow is vulnerable to trust issues, including the insertion of bugs and the existence of vulnerabilities due to opaqueness between different entities. As the IC designs are increasingly complex with heterogeneous intellectual properties (IPs) from third-party vendors, the feasibility of such bugs and vulnerabilities are growing at a rapid pace [4]-[7]. For instance, in the year 2021 the number of identified common vulnerability enumeration (CVEs) recorded is close to 18,439 [8]-[10] increased by 184% compared to 2015 [11].

With the complexity of system-on-chips (SoCs) and IC designs, such increasing vulnerabilities stem from the orchestration of software and hardware components of the system. [12]-[23]. For instance, Intel's machine check bug [24], [25] facilitates adversaries to launch a denial-of-service attack with machine check as a trigger, leading to system freeze and hanging of the processor. Re-engineering the ICs cost nearly half a Billion Dollars, along with putting Intel's reputation at stake. Unlike software, hardware patches are expensive and permanent. It is crucial to rectify the CVEs and common weakness enumerations (CWEs) vulnerabilities before fabrication (i.e., pre-silicon) and release them into the market to prevent reengineering and manufacturing costs.

To nullify the effects of the vulnerabilities and bugs in the SoCs and ICs, the design verification (DV) community both from academics and industry, has proposed numerous pre-silicon DV techniques [26]-[35] and Electronic Design Automation (EDA) tools [36]-[42]. It is estimated that up to ${70}\%$ of the time and effort of the IC development cycle is spent on the verification activities [43]-[47], which highlights the prominence of verification. The two popular verification methods are the dynamic [29], [30], [32], [48] and formal verification [28], [36], [49]-[56] methods. Dynamic and formal verification methods have been used over the years for design verification and is also embedded in current-generation EDA tools [36], [37], [51], [57].

However, both dynamic and formal verification [58]-[64] techniques have failed to match the pace of ever-increasingly complex IC and SoC and are less efficient in detecting bugs [65]-[70]. These techniques require increased human effort, lack scalability, have limited design coverage, and the verification time exponentially increases with the design complexity [65], [68], [71]. Hence, there is a need for efficient and automated verification methodologies that can detect bugs compatible with the current IC design and verification flow.

To address the shortcomings of formal and dynamic verification, hardware fuzzing has been introduced in recent years [65]-[67], [72]-[74]. Fuzzing is a widely used software testing methodology for bug detection in software applications. Fuzzing, in simple terms, is bombarding the software with test cases and analyzing for any invalid targets, such as memory crashes. Industry-based fuzzing platforms such as Google's OSS-Fuzz [75] and Microsoft's Security Risk Detection [76] have proved their efficacy in identifying a plethora of security vulnerabilities. In an attempt to detect vulnerabilities, Google's OSS-Fuzz has identified more than 30,000 bugs in 500 open-source projects [75]. The popular software fuzzer, American Fuzzy Lop (AFL) [77], is predominantly used in software fuzzing. Inspired by software fuzzing [78]-[81], hardware community researchers adopted fuzzing for hardware designs [68], [69], [71], [78], [79].

RFuzz [72] is one of the first hardware fuzzing frameworks proposed in the literature. The hardware-fuzzing frameworks aim at fuzzing CPU designs for vulnerability detection. Later, some of the works proposed translating the hardware as software for adaption of software fuzzers [65]-[67], whereas some works proposed fuzzing hardware as hardware [68], [69], [74] which will be discussed in Section 3.

The fuzzing entities include state-of-the-art famous RISC-V processors such as Ariane (CVA6) [82], RocketBoom [83], IBEX core [84], and peripheral components such as Advanced Encryption Standard (AES), KMAC, and RISC-V time core inspired by Google's OpenTitan framework [85]. The fuzzing frameworks have detected various hardware vulnerabilities across these platforms. In addition, some of the works have achieved increased code coverage, paving the way for deeper testing of the IC and SoC designs.

Though the hardware fuzzing has garnered interest from researchers, multiple challenges and ambiguities still lie forward that hamper its adaptability in EDA tool flows. For example, the key parameters that are employed for fuzzing are the coverage metric, fuzzing engine, and evaluation metrics. However, ambiguities exist with the defined parameters and their validation. To clarify the uncertainties and pave a clear path forward for enriched hardware fuzzing technique development, systemization of knowledge (SoK) is pivotal. In this SoK work, we lay out and analyze the existing techniques in terms of their pros, cons, and ability to meet the golden standards required for their wider adaptation and standardization. The cardinal contributions of this SoK are:

- We lay out the fundamental principle of the Design Verification (DV) techniques and its limitations.

- The overlooked challenges and fundamental issues with deploying existing tool flows for hardware fuzzing is discussed.

- The leniency with employing traditional verification metrics and inability to exploit full potential of fuzzers when employed for hardware fuzzing is discussed.

- The delusion of hardware fuzzing in the hardware domain is outlined.

## II. BACKGROUND ON DESIGN VERIFICATION

Hardware verification techniques for vulnerability detection can be broadly classified into 1) Formal Verification, 2) Information Flow Tracking, 3) Dynamic Verification, and 4) Recently introduced Hardware Fuzzing inspired by software testing as shown in Figure 1

## A. Formal Verification

Formal verification is one of the conventional methods in hardware verification techniques that ensure the correctness and reliability of complex integrated circuits and electronic systems [36], [48], [53], [54], [86]-[93]. It encompasses a diverse range of mathematical theorems and logical techniques to prove the correctness of hardware designs against specified properties. Formal verification is supported by commercial EDA tools such as Cadence Jasper Gold and Synopsys VCS Formal [37], [51], [57]. Formal verification has gained significant traction due to its ability to mitigate deep flaws in hardware design. However, the existing formal verification techniques are crippling due to their inability to cope with increasingly large complex modern processor designs and the need for expert knowledge [28], [68].

## B. Information Flow Tracking (IFT)

Information-flow tracking (IFT) is a method that tracks and controls the flow of information within the hardware system for verification. It is often applied to identify and prevent unauthorized or unintended information flows, such as sensitive data leaks [59]-[61]. Several secure RTL programming languages [61]-[63], [86], have been created to assess noninterference properties, primarily based on IFT [86], [88]. In this technique, the input signals are labeled, and the labels are propagated throughout the design to detect and identify design vulnerabilities, including information leakage, data manipulation, or design caveats [87]. With the growing complexity and the size of current designs (with several thousands of lines of code), the labels often get polluted or lost, making the IFT impractical [65], [69], [72].

## C. Dynamic Verification

Dynamic Verification (DV), a.k.a runtime bug detection, is a popular verification technique. Dynamic hardware verification techniques simulate or emulate the design-under-test (DUT) with input seeds to obtain the outputs and analyze for bugs and vulnerabilities. The three cardinal steps in DV are 1) Test Generation 2) DUT Simulation 3) Evaluation as shown in Figure 2

a) Test Generation: DV engineers craft the input test vectors during the test generation phase to simulate the DUT. The most popular test generation techniques are the Constrained Random Verification (CRV) [94]-[96] and Coverage Directed Test Generation (CDG) [65].

CRV techniques, though require human intervention, are widely practiced in DV techniques such as Universal Verification Methodology (UVM) [94], [97]. To evade the shortcomings of CRV, CDG is proposed [98], [99], where the input sequences are mutated based on feedback code coverage metrics to explore the non-executed regions of the design. But CDG suffers from deployment bottlenecks (i.e., inefficiency of coverage tracing) for a complex DUT [65].

b) DUT Simulation: Once the test vectors are generated, the HDL of the design can be simulated through various commercially available and open-source EDA tools [39], [57], [100] for extracting the output responses. The simulation tools translate the HDL design to a $\mathrm{C}$ or a $\mathrm{C} +  +$ program, i.e., an equivalent software model. The compiler compiles the translated software model with the testbench to provide a binary executable file (.exe) called Hardware Simulation Binary (HSB). Post compilation, the simulation engine executes the HSB with the inputs from the testbench to obtain the outputs.

![0196f2cf-9266-7b22-9c79-5d63c76a896e_2_246_137_1303_231_0.jpg](images/0196f2cf-9266-7b22-9c79-5d63c76a896e_2_246_137_1303_231_0.jpg)

Fig. 1. Design Verification Classification

![0196f2cf-9266-7b22-9c79-5d63c76a896e_2_134_420_739_411_0.jpg](images/0196f2cf-9266-7b22-9c79-5d63c76a896e_2_134_420_739_411_0.jpg)

Fig. 2. Dynamic Verification

c) Evaluation: Test evaluation is performed by monitoring the outputs of the hardware for a given input stimuli. The common methods for evaluation are to make use of the Golden Reference Model (GRM) or System Verilog Assertions (SVA). The model is examined for any violation against the defined assertion properties or the behavior of the GRMs.

## D. Hardware Fuzzing

To surpass the limitations in the existing DV frameworks, hardware fuzzing [65], [66], [68], [72] has gained traction due to its popularity in the software testing community [101]- [103]. The concept of fuzzing primarily involves: a) test generation; b) monitoring the DUT/program-under-test (PUT); and c) analyzing for bugs or errors as shown in Figure 3.

![0196f2cf-9266-7b22-9c79-5d63c76a896e_2_181_1477_641_215_0.jpg](images/0196f2cf-9266-7b22-9c79-5d63c76a896e_2_181_1477_641_215_0.jpg)

Fig. 3. Overview of Software Fuzzing

The core concept behind traditional fuzzing begins with the generation of acceptable test case generations (i.e., input stimuli). The input stimuli are then fed to the DUT/PUT for monitoring the running status of the program and is subjected to record any crash during the period of fuzzing. The monitored output is analyzed for bugs. The DUT/PUT is bombarded with test input until a crash is recorded. Software fuzzers analyze crashes for bug detection, whereas in hardware fuzzing, the expected outcome from DUT is verified against assertions or expected output from the GRM. Fuzzing incurs low deployment costs and a reduced verification period for testing complex designs.

Depending on the available information regarding the DUT, fuzzing techniques can be broadly classified into three types: i) Blackbox fuzzing, ii) Greybox fuzzing, and iii) Whitebox fuzzing. In black box fuzzing [104], the fuzzer does not have access to the internal details or the source code of the DUT/PUT being tested. However, the bug detection capabilities are constrained due to limited code coverage, inefficient input generation, lack of knowledge of the system's behavior, and limited feedback for debugging [104]. In contrast, white box fuzzing depends upon the target code access. In Greybox Fuzzing (GF) such as AFL [77], the fuzzer has limited knowledge about the DUT while extracting the peripheral information about the data flow, control flow, data format, protocols, and high-level architecture. These fuzzers discard the primary source code after the necessary instrumentation for the program is added. The instrumented binary is subjected to monitoring based on the coverage reports.

![0196f2cf-9266-7b22-9c79-5d63c76a896e_2_922_1122_681_423_0.jpg](images/0196f2cf-9266-7b22-9c79-5d63c76a896e_2_922_1122_681_423_0.jpg)

Fig. 4. Coverage Greybox Fuzzing

Based on the code exploration strategies, fuzzers can be further classified as Coverage-based Greybox Fuzzing (CGF) and Directed Greybox Fuzzing (DGF).

Coverage-based Greybox Fuzzing: In CGF, the fuzzing is aimed at achieving maximum code coverage through feedback engines (i.e., coverage metrics). In hardware, the coverage metrics that can be used are Finite State Machine (FSM), line, conditional, and MUX toggle. As shown in Figure 4, a set of input seeds is stored and passed to the mutation engine that performs mutation operations to generate multiple input seeds. During the runtime, the coverage reports are extracted based on the input provided to the DUT and given as feedback to the mutation engine. Based on the coverage feedback, the mutation engine further mutates the interesting input seed to generate a new set of seeds. The uninteresting input seeds are discarded from the input pool. The HSB is executed with these seed inputs, and any potential crashes are saved for analyzing the vulnerabilities.

Directed Greybox Fuzzing: Hardware and Software designs often undergo revisions for updating a component for better performance (i.e., incremental designs). To fuzz such incremental designs, DGF is used for fuzzing at a particular region rather than the whole DUT/PUT, resulting in reduced verification time.

## III. STATE-OF-THE-ART ON HARDWARE FUZZING

This Section briefly discusses recent hardware fuzzing works as shown in Table 1.

## A. Direct Adoption of Software Fuzzer for Hardware

As a solution to the CDG shortcomings such as deployment bottleneck (i.e., inefficiency of coverage tracing, RFuzz [72] is the first FPGA emulation-based hardware fuzzing technique introduced as shown in Figure 5.

The RFuzz translates the target HDL to FIRRTL [105] such that instrumentation is leveraged through compiler passes, enabling the test harness generator to design a wrapper for the RTL design. The input pins are concatenated to form a bit vector and are mapped to a series of bytes representing the input values. To ensure the tests are deterministic and repeatable, the RTL should be reset to a known test before each test execution. MetaReset (for resetting the registers to zero for enabling DUT reset) and Spare memories (to reset the memory locations that have been written in previous test execution) are deployed to enable quick RTL reset without modifying the DUT characteristics. Furthermore, the instrumentation is appended to enable MUX control coverage in hardware, ensuring FPGA-synthesizable coverage extraction. RFuzz utilizes the AFL-based mutation functions. The AFL fuzzer efficiently communicates with the FPGA through high-speed Direct Memory access (DMA), as shown in Figure 5

## B. Fuzzing Hardware as a Software

In contrast to RFuzz, Trippel et al. [65] proposed fuzzing hardware-like software rather than porting software fuzzers directly on the hardware designs. The cardinal aspect of fuzz hardware like software is driven by the executable software version of the hardware provided by the hardware simulation tools. The translated software version of the hardware is compiled, instrumented and fuzzed using AFL for bug detection.

In [65], the authors make use of Verilator to translate the hardware to an equivalent software model as shown in Figure 5. Verilator [100] is a popular open-source tool that leverages the equivalent software model in $\mathrm{C} +  +$ for the given System Verilog of the RTL design. The C++ program can be instantiated with a $\mathrm{C} +  +$ wrapper acting as a testbench to output HSB and simulate the device. To trace the hardware coverage in the software domain, it exploits the seamless binary instrumentation provided by the AFL fuzzer to trail code coverage.

In addition, the verilator generates $\mathrm{C} +  +$ code for both blocking and non-blocking statements of System Verilog and interpolates assertions and coverage points, ensuring functional coverage. With edge coverage from AFL, both code and functional coverage of the hardware in a software domain can be monitored. The fuzzing cores are monitored for crashes and are evaluated against SVAs. This work monitors for crashes such as improper memory mapping, errors in registers, buffer overflow, and floating-point computation for vulnerability detection, as the hardware is modeled as software.

## C. Fuzzing Hardware as Hardware

Recent advancements in hardware fuzzing have embraced a domain-specific approach to eliminate the need for translating hardware to software, along with the associated preliminary tasks and equivalence checks. TheHuzz [68], DiFuzz [74], and ProcessorFuzz [69] adeptly integrate hardware fuzzing into conventional industry-standard hardware design and verification workflows.

a) TheHuzz: The three pivotal components of TheHuzz include the Seed Generator, Stimulus Generator, and Bug Detector, as illustrated in Figure 5 Operating at the instruction set architecture (ISA) abstraction level, the Seed Generator furnishes the input seed in the form of instruction sequences. The Stimulus Generator then mutates the instruction at the binary level, ensuring that all bits of the instruction undergo mutation to test the processor with illegal instructions. This includes mutating opcode bits and data bits to unveil unexplored datapaths.

The RTL design of the processor [82], [84], [106], is simulated with the binary format of the instruction using Synopsys VCS, a tool entrenched in the semiconductor industry for several decades. Synopsys VCS [57] trace code coverage through various metrics, including branch, condition, toggle, FSM, and functional coverage. Based on these coverage metrics (the feedback engine), optimal weights are assigned to each instruction-mutation pair, and uninteresting pairs are discarded. Bugs are identified by simulating the ISA emulator [107] (GRM), and comparing the behavior of the RTL design against the GRM behavior. Seamless compatibility with traditional EDA tools and requiring minimal additional effort are some of the unique aspects of this work.

b) ProcessorFuzz and DiFuzzRTL: Other prominent works associated with hardware-specific fuzzing to detect CPU bugs are the ProcessorFuzz as shown in Figure 5 [69], and DiFuzzRTL [74]. At an elevated stratum of hardware abstraction, a Central Processing Unit (CPU) manifests as an intricately designed FSM. It is indispensable to monitor these transitions of CPU states for potential bug detection for a given assembly program. ISA simulators are used to simulate the functional behaviors of the CPU, serving as a reference model for the existing fuzzing frameworks. Rather than using coverage metrics from the RTL simulation, ProcessorFuzz exploits CSRs from the ISA simulator as coverage metrics. The CSR values and the transitions in them represent the changes in the architectural state flow of the CPU.

The input pool for the ProcessorFuzz encompasses a pool of instruction at the assembly level adhering to the target ISA upon which mutation operations are performed. Rather than simulating the RTL with the mutated input, ISA simulator simulates the targeted CPU with the mutated input sequences. During the simulation, the simulator initiates trace logs from which CSR transitions, program counter, and disassembled instructions can be obtained. The trace logs of the RTL and ISA emulators are compared to detect CPU vulnerabilities. In contrast, DiFuzzRTL captures the FSM state transition of the RTL design through static analysis of the small group of registers, which plays a vital role in controlling the states of the CPU. The feedback is given through the register coverage in RTL and is compared against the ISA for evaluation of vulnerability detection.

![0196f2cf-9266-7b22-9c79-5d63c76a896e_4_200_135_1386_808_0.jpg](images/0196f2cf-9266-7b22-9c79-5d63c76a896e_4_200_135_1386_808_0.jpg)

Fig. 5. Hardware Fuzzing Frameworks

### D.SoC Fuzzers

In contrast to the fuzzing techniques discussed above, works such as [108] focus on fuzzing SoCs. It employs Hyperprop-erties [108], which are higher-level properties that describe security policies by comparing the behavior of instances of a system. The concept of hyperproperties is commonly employed in analyzing security protocols and detecting bugs in a system. These hyperproperties can be used for laying out the SoC security specifications. Hyperfuzzing, as shown in Figure 5 [66], encompasses defining SoC security properties that adhere to confidentiality, integrity, and noninterference expressed in HyperPLTL (Hyper Past-time Linear Temporal Logic). Hyperfuzzer complies with CGF with High-Level coverage metrics. The famous Verilator is used for SoC RTL simulation, which is further instrumented to collect coverage metrics. AFL fuzzes the instrumented code to find potential bugs and is evaluated using a property checker by comparing it against the security property specifications (i.e., hypeprop-erties) in HyperPLTL.

### IV.Is HARDWARE FUZZING A DELUSION?

The state-of-the-art fuzzing-based DV frameworks exhibit proficiency in fuzzing various IC/IP entities, as shown in Table I. Nonetheless, these cutting-edge frameworks are not without certain limitations. In this section, we shed light on the often underestimated drawbacks within these domains, posited as potential impediments to the optimal efficiency of fuzzing frameworks.

## A. Tool Dependency

Existing works on hardware fuzzing have chosen a wide range of tools, such as Verilator [100] and AFL [77], for fuzzing as outlined in Table 1. The limitations associated with these tools are deconstructed below.

1) Equivalency of Hardware Bugs: The distinction between hardware and software lies not only in their design but also in their behavioral exhibits. Employing software fuzzers directly onto the RTL poses its own set of challenges. In the software domain, a common definition for a bug is an outcome marked by crashes, signifying an improper functioning of the operating system, which in turn crashes the fuzz engine. However, such scenarios do not exist in the hardware realm.

Limitation A.1.1: AFL fails to report hardware bugs, as crash or hang doesn't exist in hardware. Hardware inherently does not crash; instead, it produces erroneous outputs.

For instance, a bug in a software application that causes it to access invalid memory locations, such as CVE-2023- 3953 [110], can be detected as a crash by AFL. On the other hand, hardware, instead of crashing, issues often manifest as malfunctions or failures in the form of faulty outputs. Because of this limitation, techniques such as RFuzz [72], which relies on employing AFL directly on the hardware, fail to detect any hardware vulnerabilities.

TABLE I

EXISTING FUZZING FRAMEWORK

<table><tr><td>Framework</td><td>Fuzzer</td><td>Input</td><td>Simulator</td><td>Coverage Metric</td><td>Target Design</td><td>Comparison</td></tr><tr><td>RFUZZ [72</td><td>HW Fuzzer</td><td>Series of bits</td><td>Any</td><td>Mux Toggle</td><td>Peripherals, RISC-V</td><td>Assertion</td></tr><tr><td> <img src="https://cdn.noedgeai.com/0196f2cf-9266-7b22-9c79-5d63c76a896e_5.jpg?x=224&y=316&w=125&h=42&r=0"/> </td><td>HW Fuzzer</td><td>Series of bits</td><td>PyRTL</td><td>Mux Toggle</td><td>RISC-V Opencore 1200</td><td>Assertion</td></tr><tr><td>DifuzzRTL [74]</td><td>HW Fuzzer</td><td>Assembly</td><td>Any</td><td>Register Coverage</td><td>RISC-V CPU</td><td>GRM</td></tr><tr><td>Trippel et al [65]</td><td>SW AFL Fuzzer</td><td>Byte Sequence</td><td>Verilator</td><td>Edge Coverage</td><td>AES, HMAC KMAC, Timer</td><td>SW crashes</td></tr><tr><td>HyperFuzzer 108</td><td>SW AFL Fuzzer</td><td>Series of bits</td><td>Verilator</td><td>High-level</td><td>SoC</td><td>Assertion</td></tr><tr><td>DirectFuzz [109]</td><td>SW AFL Fuzzer</td><td>Series of bits</td><td>Verilator</td><td>MUX</td><td>Peripherals, RISC-V</td><td>Assertion</td></tr><tr><td>TheHuzz [68]</td><td>HW Fuzzer</td><td>Assembly</td><td>Synopsys VCS</td><td>FSM, Branch, toggle, conditional</td><td>RISC-V</td><td>GRM</td></tr><tr><td>Processor Fuzz [69</td><td>HW Fuzzer</td><td>Assembly</td><td>Verilator</td><td>Control path register, ISA-tranistion</td><td>RISC-V</td><td>GRM</td></tr><tr><td/><td>HW Fuzzer</td><td>Byte Sequence</td><td>Xilinx ISA</td><td>Randomness, target output, input coverage</td><td>SoC</td><td>Database</td></tr></table>

2) Dependency on HDL and its Translation: Fuzzing techniques such as Rfuzz [72] translate the given RTL to FIRRTL [105], casting dubitable shadows over their equivalence. In addition, RFuzz is based on Midas, lacking language compatibility with FIRRTL. This support is currently confined to Chisel, Verilog, and specific segments of System Verilog [111]. These factors oblige the users to have the expertise and prior knowledge of the languages, fostering fuzzing difficulties.

Limitation A.2.1: FIRRTL is limited to only certain languages that require expertise and prior knowledge.

To exhibit the shortcomings of deploying software fuzzers directly on RTL designs, hardware designs are translated to software models such as binary executables on which AFL is deployed. Such translation introduces additional challenges. Verilator [100] is a popular open-source tool that leverages the equivalent software model in C++ for the given System Verilog of the RTL design. The RTL design is subjected to parsing and lexical analysis to format the Abstract Syntax Tree (AST) [112], [113] from which the verilator generates an equivalent C++ of the hardware.

However, there arises a question on the equivalency between the hardware and the translated hardware. The ver-ilator fails to capture inherent hardware behaviors such as signal transitions, FSMs, and floating wires described in HDL languages to software constructs. In addition, the verilator requires linking auxiliary libraries to simulate the translated hardware, generating an emulated model rather than a simulated model. A translated hardware design needs to consider the fundamental hardware behaviors, including bus transactions, register computation, and controllers. These aspects constitute the foundation for system operations. Limitation A.1.1, along with A.2.2, is well proven by the results of employing AFL directly on RTL [72] and translating hardware as a software [65], which failed to report hardware bugs, whereas the later works [74] reported the bugs.

Limitation A.2.2: Tools such as Verilator fail to capture inherent hardware behaviors such as signal transitions, FSMs, and floating wires described in HDL languages to software constructs.

3) FPGA Overheads: FPGA-based fuzzing has been introduced in [72] to accelerate the fuzzing and minimize the software conversions. However, FPGA-based fuzzing also requires traditional software fuzzers such as AFL, which introduces limitations such as A.1.1. In addition, FPGA for accelerating fuzzing incur fuzzing costs, and complex designs may not map to FPGA fabric structures, promoting scalability constraints. Fuzz testing on FPGAs can be computationally intensive and time-consuming [114]. Hardware encapsulates the physical components of a computing system, encompassing tangible entities such as processors, memory modules, and peripheral devices. Simulating these complex, intricate hardware blocks requires a substantial amount of time than the translated hardware model (i.e., the software model of the hardware). For instance, fuzzing of Ariane CPU, which is capable of hosting LINUX OS, has 20,968 lines of code (LoC) [115]. Generating a large number of random test vectors for complex CPU designs leads to memory and resource overheads. Fuzzing FPGAs may not be easily portable to different designs or platforms, as they are often customized for the target board(s) [116].

Limitation A.3.1: Fuzz testing on FPGAs can be computationally intensive and time-consuming.

4) Instrumentation Overheads: Obtaining coverage metrics is crucial in fuzzing, for which instrumentation is performed. As the majority of the fuzzers, including AFL, are inherently software fuzzers, the instrumentation supports only software models, which limits to deployment of AFL directly for hardware designs.

Limitation A.4.1: Instrumentation provided by AFL does not support HDL constructs.

Even though efforts such as translating the hardware to software were taken to circumvent these challenges [65], Limitation A.2.2 still persists and lacks code coverage, scalability for complex CPU designs, and instrumentation overheads. For example, instrumenting an Ariane Processor (20,968 LoC) leads to large instrumentation overheads corresponding to (CWE-400) [68], [114]. Besides these, the primary difference between hardware and software is in the definition of input arguments. The input for the software fuzzing is in the form of bytes, whereas for the hardware, it is in the notion of input pins accepting values at each clock cycle. The input format for a translated hardware design should be well crafted in order to perform meaningful mutations and effective fuzzing.

Limitation A.4.2: Unoptimized instrumentation leads to significant instrumentation overheads - CWE-400.

## B. Reliance on Coverage Metrics

The existing works on hardware fuzzing and hardware verification have chosen a wide range of coverage metrics to detect functional and security vulnerabilities, as outlined in Table II

1) Limitation of Verification Capability due to Chosen Coverage Metric(s): The capability of hardware to detect a vulnerability highly depends on the considered coverage metrics. For instance, RFuzz employs MUX toggling as the coverage metric to determine whether an input seed is interesting or not. This captivates the RFuzz to capture the RTL design implementations of MUX expressed in combinational logic impending to account for the coverage point [68]. Thus, RFUZZ will not cover any combinational logic that does not drive the select signals of the MUXes.

Limitation B.1.1: Selection of coverage metrics is prone to human bias, limiting verification capability.

ProcessorFuzz [69] utilizes CSR as a coverage metric to detect bugs. Use of CSR-based metrics leads to the detection of Read-after-Write dependencies vulnerabilities, which may not be detected using MUXes. Furthermore, if semantic correctness and timing information are considered, then the possibility of detecting such attacks is viable. Hence, meticulous planning is required for selecting the coverage metrics as it is influenced by humans, which is often prone to biased selection.

In addition to the manual selection of metrics, fuzzers such as AFL utilizes the in-house code and edge coverage metrics to trace the fuzzing efficacy on the translated hardware design [65], [77]. However, extracting actual hardware coverage metrics from the translated design is questionable as there exists a fundamental difference between hardware and software coverage metrics. The software coverage metrics line and edge coverage is comparable in some hardware instances, however, other coverage metrics, such as FSM or MUX Toggle are not transferrable. These coverage metrics are required for detecting vulnerabilities associated with hardware [68].

First of all, these software coverage metrics are valid if and only if the limitation A.1.1 and A.2.2 is resolved. As there is a disparity between the RTL design and the translated hardware design, usage of inbuilt AFL coverage metrics is not viable and inefficient failing to detect CWE-705 [117] as well.

Limitation B.1.2: AFL coverage metrics doesn't imply to hardware and fails to detect CWE-705.

2) Robustness of Coverage Metrics: To bypass the limitations existing with the software coverage metrics and to account for hardware coverage metrics, as mentioned earlier, some of the hardware fuzzing frameworks (i.e., fuzzing hardware as hardware) utilize the traditional hardware coverage metrics provided by the EDA tools. These frameworks extract the line, code, FSM, toggle, and branch coverage to capture the behaviors of the RTL design. However, RTL simulations for CPU designs are slower compared to ISA simulations. ISA simulations are ${75} \times$ faster than RTL simulations [69]. With these advantages, some works use the CPU's Control Status Registers (CSRs) to steer the fuzzing. However, multiple robustness challenges exist while capturing the coverage metrics in a reliable manner. For instance, the reliability challenges associated with CSR are outlined below.

Limitation B.2.1: Reliable extraction of coverage metrics is challenging.

a) Limited Granuality: CSRs often provide coarse-grained coverage information. They may not offer fine-grained details about which specific portions of the hardware design have been tested or remain untested. This lack of granularity can make it challenging to pinpoint precise areas that need additional testing or improvement.

b) Difficulty in Identifying Root Causes: CSR-based coverage metrics may not assist in identifying the root causes of issues. They can indicate whether certain control paths have been exercised but might not provide insights into why failures occurred or how to address them effectively. These fail to identify side-channel attack CVEs on the processors [69].

c) Architecture-Specific: CSRs are architecture-specific. Each processor or system may have its own set of CSRs with unique features and behavior. This can make it challenging to write code that is portable across different platforms. For example x86 processors have a variety of control registers, such as CR0, CR2, CR3, and CR4 [118], whereas RISC-V uses mstatus, misa, mip [119]

d) Complexity and Overhead: Implementing CSR-based coverage tracking can add complexity and potential overhead to the testing process. This may result in increased computational resource usage and potentially slower testing.

Limitation B.3.1: The majority of the coverage metrics primarily focus on functional checks but not parametric behavior (such as temporal behavior).

3) Can Coverage Metrics Capture Hardware Bugs or Vulnerabilities?: The existing works on hardware fuzzing as outlined in Table I have chosen a wide range of coverage metrics to detect functional and security vulnerabilities. Although usage of these metrics can detect functional bugs, the security vulnerabilities such as buffer overload CVE-2023- 29856 [120], data leakage CVE-2017-5927 [121], CVE-2023- 32342 [122] can still be left undetected through these metrics. For instance, though the width of input ports match the incoming signals, timing violation can lead to erroneous inputs and outputs, which is analogous to the CVE-2023-29856 bug in the D-Link hardware [120].

Consideration of these non-parametric metrics will lead to the detection of exploits including [121]-[124]. Though Di-Fuzz [74] facilitates the identification of side-channel vulnerabilities, it requires prior knowledge of the details of microar-chitectural features. Furthermore, it requires static analysis to monitor a small group of registers in the RTL design. A CPU is a complex system encompassing thousands of memory elements. It is tedious and error-prone to manually compare the registers in the RTL designs against the registers in the ISA emulator owing to monitoring and analyzing overheads.

Limitation B.3.2: Detection of security vulnerabilities is often challenging due to limited information captured through coverage metrics.

## C. Analysis and Verification of Vulnerabilities in Hardware Fuzzing

The pivotal component of sophisticated fuzzing frameworks lies in the meticulous evaluation criteria employed, where the fuzzer scrutinizes and contrasts the behavioral aspects of the hardware. The fuzzing frameworks exploit the usage of assertions, Golden Reference Models (GRMs) (i.e., physical hardware or GRMs through ISA), and a database of vulnerabilities as outlined in Figure 5 and Table 1. Though these metrics evaluate the presence of bugs in the CPU, many alarming concerns are disregarded during the evaluation process.

Limitation C.1.1: Insertion of assertion requires manual instrumentation, which may not suffice for large complex CPU designs as it lead to instrumentation overheads.

1) Reliance on Assertions: DV engineers inject System Verilog assertion (SVA), treating them as equivalent versions of software crashes to monitor any potential bugs in the code. However, to define the assertion it requires manual instrumentation, which may not suffice for large complex CPU designs as it may lead to instrumentation overheads. As mentioned earlier, hardware does not inherently crash, which fosters the question of equivalency between the SVAs and a software crash. Instead commercially available tools such as JasperGold [37] or Synopsys VCS [51] can be used to fuzz in the same manner with minimal instrumentation overheads and less preliminary setups. In contrast to software, as hardware does not offer any hang, crash, or termination, the fuzzer may not capture any potential vulnerabilities.

Limitation C.1.2: System Verilog Assertions (SVA) and software crashes are not equal.

Moreover, verification engineers predetermine all assertions, directing the fuzzers to specifically examine predefined conditions rather than uncovering unknown vulnerabilities. As the verification engineers are responsible for crafting the assertion, these predefined conditions are prone to human biases. Moreover, complex CPU designs encompassing several thousands of LoC require increased human intervention and instrumental overheads. Even though the fuzzer engine alerts against the predefined assertions, it does not analyze the underlying reason for the caused bug. These are the two cardinal underlying reasons why neither adoption of software fuzzers on hardware nor fuzzing hardware-like software has reported any potential crashes or bugs [65], [72]. Unlike software, CPUs are complex multi-cycle systems that have an impact on the sequence of states of instructions. It may take several clock cycles to execute, decode, and commit the changes at the architectural level impending the assertions to capture the cycle of events. All these factors limit the scalability of the assertions as an evaluation metric upon which the fuzzer can rely.

Limitation C.1.3: System Verilog Assertions (SVA) are prone to human bias and can only examine pre-defined conditions rather than uncover unknown vulnerabilities.

2) Reliance on GRMs: Golden Reference Models (GRMs) in hardware refer to meticulously crafted and validated models that serve as benchmarks or standards for the expected behavior of a hardware design. These models are considered the golden or authoritative representations against which the actual hardware implementation is compared during verification and testing processes. Predominantly, most of the fuzzers [68], [69] rely on GRMs to trace the bug in the RTL design. The possible GRMs can be dissected into a physical hardware model or a GRM generated by curated software simulators, which poses additional reliance challenges, as discussed in Section 3.

Limitation C.2.1: The availability of third-party GRMs and access to the source code and internal signals are limited.

The incorporation of third-party IPs along with CPU is a common practice in modern IC design methodologies, delivering exceptional real-world solutions. However, with the current generation ICs becoming complex and adopting third-party IPs [16]-[18], the availability of GRMs and access to the source code and internal signals are limited. This impedes the efficiency and efficacy of hardware fuzzing.

What is more, although GRMs can be used to detect functional errors, the detection of security exploits is error-prone as they can be embedded unintentionally or can even be a side effect. The unavailability of GRMs should not impede the fuzzing, as verification of the hardware is pivotal from security and reengineering cost perspectives. For instance, CPUs have an ISA simulator [83], [107], but a standalone IP block/peripheral unit does not have any equivalent entity to identify as a GRM. The verification engineers rely upon the RTL code, which might also have vulnerabilities. This hardware might have injected trojans, infecting the entity to which it is connected. Fuzzing based on a buggy GRM as an evaluation criterion may lead to poor fuzzing standards and mislead the verification community with false positives.

Limitation C.3.1: The disparities in ISA simulation tools might result in faulty evaluations against the RTL design, limiting the bug detection capabilities.

3) Software Models: The other way to depend on the GRMs is through software models. The CPUs can be simulated in ISA simulators (i.e., software simulations) [83], [107] to verify the functionality of the processors. For instance, the RISC-V community has crafted ISA simulators for RISC-V processors. However, these software are designed by developers and are still prone to human errors and bias.

Consider a simple case study shown in Figure 6, which demonstrates the disparity in the encoding of the BNE instruction in MIPS simulator [125]-[127]. We consider two popular ISA simulation tools, QtSpim [127] and MARS [126] for ISA GRM generation.

There is inconsistency with the branch distance calculation between the outputs of QtSpim [127] and MARS [126] simulators, specifically related to the encoding of the BNE (branch not equal) instruction (Line 9). The above code loads values in register location $\$ 6$ and $\$ 7$ , and if the register values are not equal, it jumps to branch L4 (Line 14) using the BNE instruction.

The BNE instruction is of type I (immediate), and the last 16 bits are the branch distance (in words) computed from where the PC is, which is pointing to L1 after fetching the BNE instruction. QtSpim encodes it as $\mathbf{0.{14c70002}}$ as shown in Figure 6, indicating " 2 words ahead" or "2 instructions ahead" from the current instruction. However, according to the standard interpretation, the branch distance should be calculated from the current instruction's location, considering it as 0 instructions ahead. Therefore, the correct branch distance from the current instruction should be 1 , not 2 .

In contrast, MARS encodes it correctly as 0x14c70001 as shown in Figure 6. This deviation in the branch distance calculation may suggest a potential inconsistency or discrepancy in the QtSpim simulator's handling of BNE instruction. Some of the hardware fuzzing works such as TheHuzz [68] solely depend on the instruction encoding to generate input seeds and mutate seeds. These disparities in ISA simulation tools might result in faulty evaluations against the RTL design, limiting the bug detection capabilities. Additionally, such disparities may lead to false alarms in bug detection and incur re-verification costs and delays.

Even though the GRMs are carefully curated to eliminate bugs, a research group recently found a bug associated with incorrect memory accessing in the RISC-V cores [128].The existing hardware frameworks TheHuzz and ProcessorFuzz solely depends on the ISA simulator for the evaluation process. In addition, there exist bugs in the x86 ISA as well [129]. However, the open-source development community [82] is consistently committed to rectifying the reported faults to design robust architectures for future computing needs.

4) Portability Challenges due to Software Limitations: The ISA tools are instrumented to obtain coverage metrics from CSRs. However, the instrumented ISA tool is not compatible with other processor designs following different instruction sets. For instance, the instrumented RISC-V ISA tool is incompatible with x86 ISA processors [129]. Though the existing frameworks claim that they can be ported to other ISA targets, these claims are not proven due to the differences in their instruction sets and rules [82], [129]. With such discrepancies associated with ISA as an evaluation benchmark, it is not feasible to port the ISA simulator to other architectures.

Limitation C.4.1: The ISA tools can be used only for the processors of the same ISA target.

## D. Fuzzing Entitites

The current fuzzing frameworks are limited to fuzz either a CPU or a standalone peripheral IP or a SoC. None of the fuzzing frameworks provide flexibility to be adaptable across the spectrum of IC designs. For instance, Tripple et al. work [65] is limited to fuzz-only OpenTitanCores, whereas The-Huzz, ProcessorFuzz, and Diffuz are limited to fuzz processor designs. Furthermore, within the processor fuzzers, there exists a limit to fuzz only specific ISA associated with the proposed frameworks. TheHuzz evaluates only the functional behavior of the CPU but does not monitor temporal behaviors. This might lead to the inability to detect side-channel vulnerabilities such as specter and meltdown attacks in the processor designs. It is indeed to evaluate the peripherals interfaced to the CPU as they may pose a huge threat as well [130]-[132]. There is a need for a unanimous fuzzing framework to fuzz all the entities associated with hardware design.

Limitation D.1.1: Unanimous fuzzing framework for RTL/CPU Designs does not exist.

## V. CONCLUSION

This SoK paper provides insights into hardware fuzzing techniques for identifying bugs and vulnerabilities. We comprehensively reviewed and explained the fundamental principles of existing hardware fuzzing, the methodologies involved, and the diverse hardware designs in which it can be employed. We have laid out the challenges/limitations existing with the current frameworks. We have given insights on how the fuzzing frameworks have been assumed and pointed out its pros and cons. The overlooked challenges and fundamental issues with deploying existing tool flows for hardware fuzzing is discussed. We also provide future possible directions for further advancement of design verification techniques.

<table><tr><td>Address</td><td>CodeBasic</td><td>Source</td></tr><tr><td>0x00400000</td><td>0x2406000a addiu \$6,\$0,0x00000...10:</td><td>li \$6, 10 # Load a value into register \$6</td></tr><tr><td>$0 \times  {0.0400004}$</td><td>0x24070014 addiu \$7_\$0.0x00000_11 :</td><td>li \$7, 20#Load a different value into register \$7</td></tr><tr><td>0x00400008</td><td>0x14c70001 bne \$6,\$7,0x0000000113:</td><td>L1 # Branch to L4 if $\$ 6$ and $\$ 7$ are not equal</td></tr><tr><td>$0 \times  {0040000}\mathrm{c}$</td><td>0x2408001e addiu \$8,\$0,0x00000...116:</td><td>li \$8, 30#Load a value into register $\$ 8$ if $\$ 6$ and $\$ 7$ are equal</td></tr><tr><td>0x00400010</td><td>0x24090028 addiu \$9,\$0,0x00000...21 :</td><td>li \$9, 40#Load a value into register $\$ 9$ if $\$ 6$ and $\$ 7$ are not equal</td></tr><tr><td>0x00400014</td><td>0x2402000a addiu \$2,\$0,0x00000...;28:</td><td>li \$v0, 10#syscall code for exit</td></tr><tr><td>0x00400018</td><td>29:</td><td>svscall</td></tr></table>

![0196f2cf-9266-7b22-9c79-5d63c76a896e_9_271_368_1150_342_0.jpg](images/0196f2cf-9266-7b22-9c79-5d63c76a896e_9_271_368_1150_342_0.jpg)

Fig. 6. BNE encoding in a) MARS , b) QtSpim

## REFERENCES

[1] J. C. Chen, H. Rau, C.-J. Sun, H.-W. Stzeng, and C.-H. Chen, "Work-flow design and management for IC supply chain," in International Conference on Networking, Sensing and Control, 2009, pp. 697-701.

[2] A.Yeh, "Trends in the global IC design service market," last accessed : 11/19/2023. [Online]. Available: https://www.digitimes.com/news/ a20120313RS400.html&chid=2

[3] TechInsights, "Apple iPhone 15 Pro Teardown," last accessed 11/19/2023. [Online]. Available: https://www.techinsights.com/blog/ apple-iphone-15-pro-teardown

[4] N. Artenstein, "Broadpwn: Remotely compromising android and iOS via a bug in broadcom's Wi-Fi chipsets," in BlackHat USA, 2017.

[5] Y. Liu, Y. Jin, A. Nosratinia, and Y. Makris, "Silicon demonstration of hardware trojan design and detection in wireless cryptographic ICs," IEEE Transactions on Very Large Scale Integration (VLSI) Systems, vol. 25, no. 4, pp. 1506-1519, 2017.

[6] M. Lipp, M. Schwarz, D. Gruss, T. Prescher, W. Haas, A. Fogh, J. Horn, S. Mangard, P. Kocher, D. Genkin, Y. Yarom, and M. Hamburg, "Meltdown: Reading kernel memory from user space," in USENIX Security Symposium, 2018.

[7] P. Kocher, J. Horn, A. Fogh, , D. Genkin, D. Gruss, W. Haas, M. Hamburg, M. Lipp, S. Mangard, T. Prescher, M. Schwarz, and Y. Yarom, "Spectre attacks: Exploiting speculative execution," in IEEE Symposium on Security and Privacy (S&P'19), 2019.

[8] N. N. V. Database, last accessed 11/19/2023. [Online]. Available: https://nvd.nist.gov/

[9] REDSCAN, "2021 has officially been a record-breaking year for vulnerabilities." last accessed 11/19/2023. [Online]. Available: https://www.redscan.com/news/nist-nvd-analysis-2021- record-vulnerabilities/#: `:text=2021%20was%20an%20especially% 20difficult,50%20CVEs%20logged%20each%20day

[10] MITRE, "Cve database," last accessed 11/19/2023. [Online]. Available: https://cveform.mitre.org/

[11] S. Engineering, "A glossary for chip and semiconductor ip security and trust," last accessed 11/19/2023. [Online]. Available: https://semiengineering.com/a-glossary-for-chip-and-semiconductor-ip-security-and-trust/

[12] J. V. Bulck, M. Minkin, O. Weisse, D. Genkin, B. Kasikci, F. Piessens, M. Silberstein, T. F. Wenisch, Y. Yarom, and R. Strackx, "Foreshadow: Extracting the keys to the intel SGX kingdom with transient Out-of-Order execution," in USENIX Security Symposium, 2018.

[13] M. Schwarz, M. Lipp, D. Moghimi, J. Van Bulck, J. Stecklina, T. Prescher, and D. Gruss, "Zombieload: Cross-privilege-boundary data sampling," in ACM SIGSAC Conference on Computer and Communications Security, 2019.

[14] Y. Jang, S. Lee, and T. Kim, "Breaking kernel address space layout randomization with intel TSX," in ACM SIGSAC Conference on Computer and Communications Security, 2016.

[15] R. Wojtczuk, "Xen security advisory 7 (CVE-2012-0217) - PV privilege escalation," last accessed: 11/18/2023. [Online]. Available: https: //lists.xen.org/archives/html/xen-announce/2012-06/msg00001.html

[16] T. Feng, H. Pei, Z. Jin, and X. Wu, "A survey and perspective on electronic design automation tools for ensuring soc security," in International SoC Design Conference (ISOCC), 2022.

[17] P. Ghosh, V. N. Dwaraka Mai, A. Chopra, and B. Sood, "Self-checking performance verification methodology for complex SoCs," in International Symposium on Quality Electronic Design (ISQED), 2023.

[18] K. Kang, S. Park, B. Bae, J. Choi, S. Lee, B. Lee, and J.-B. Lee, "Seamless SoC verification using virtual platforms: An industrial case study," in Design, Automation & Test in Europe Conference & Exhibition (DATE), 2019.

[19] K. Alatoun, B. Shankaranarayanan, S. M. Achyutha, and R. Vemuri, "SoC trust validation using assertion-based security monitors," in International Symposium on Quality Electronic Design (ISQED), 2021.

[20] S. Tang, X. Wang, Y. Gao, and W. Hu, "Accelerating SoC security verification and vulnerability detection through symbolic execution," in International SoC Design Conference (ISOCC), 2022.

[21] P. Mohandoss and A. Rengaraj, "Pre-silicon DFT verification on SoC slim model," in International Workshop on Microprocessor and SOC Test and Verification (MTV), 2018.

[22] S. R. Vangal, J. Howard, G. Ruhl, S. Dighe, H. Wilson, J. Tschanz, D. Finan, A. Singh, T. Jacob, S. Jain, V. Erraguntla, C. Roberts, Y. Hoskote, N. Borkar, and S. Borkar, "An 80-tile sub-100-w ter-aFLOPS processor in 65-nm CMOS," IEEE Journal of Solid-State Circuits, vol. 43, no. 1, pp. 29-41, 2008.

[23] R. Saravanan, S. Bavikadi, S. Rai, A. Kumar, and S. M. Puduko-tai Dinakarrao, "Reconfigurable fet approximate computing-based accelerator for deep learning applications," in 2023 IEEE International Symposium on Circuits and Systems (ISCAS), 2023, pp. 1-5.

[24] Intel, "Machine check error avoidance on page size change/CVE- 2018-12207," 2019, last accessed: 11/18/2023. [Online]. Available: https://www.intel.com/content/www/us/en/developer/ articles/troubleshooting/software-security-guidance/advisory-guidance/machine-check-error-avoidance-page-size-change.html

[25] MITRE, "Common weakness enumeration," last accessed: 11/18/2023. [Online]. Available: https://cwe.mitre.org/data/index.html

[26] M. Chen and P. Mishra, "Property learning techniques for efficient generation of directed tests," IEEE Transactions on Computers, vol. 60, no. 6, pp. 852-864, Feb 2011.

[27] R. Mukherjee, D. Kroening, and T. Melham, "Hardware verification using software analyzers," in IEEE Computer Society Annual Symposium on VLSI, 2015.

[28] G. Dessouky, D. Gens, P. Haney, G. Persyn, A. Kanuparthi, H. Khattri, J. M. Fung, A.-R. Sadeghi, and J. Rajendran, "Hardfails: Insights into software-exploitable hardware bugs," in USENIX Conference on Security Symposium, 2019.

[29] S. R. Sarangi, A. Tiwari, and J. Torrellas, "Phoenix: Detecting and recovering from permanent processor design bugs with programmable hardware," in IEEE/ACM International Symposium on Microarchitecture, 2006.

[30] I. Wagner and V. Bertacco, "Engineering trust with semantic guardians," in Design, Automation & Test in Europe Conference & Exhibition, 2007.

[31] J. Yang and A. Puder, "Tightly integrate dynamic verification with formal verification: a GSTE based approach," in Asia and South Pacific Design Automation Conference, 2005.

[32] S. Gogri, A. Tyagi, M. Quinn, and J. Hu, "Transaction level stimulus optimization in functional verification using machine learning predictors," in International Symposium on Quality Electronic Design (ISQED), 2022.

[33] O. Guzey, L.-C. Wang, J. R. Levitt, and H. Foster, "Increasing the efficiency of simulation-based functional verification through unsupervised support vector analysis," IEEE Transactions on Computer-Aided Design of Integrated Circuits and Systems, vol. 29, no. 1, pp. 138-148, 2010.

[34] S. Kasarapu, S. Shukla, and S. M. Pudukotai Dinakarrao, "Resource-and workload-aware model parallelism-inspired novel malware detection for iot devices," IEEE Transactions on Computer-Aided Design of Integrated Circuits and Systems, vol. 42, no. 12, pp. 4618-4628, 2023.

[35] S. Kasarapu, S. Shukla, R. Hassan, A. Sasan, H. Homayoun, and S. M. PD, "Cad-fsl: Code-aware data generation based few-shot learning for efficient malware detection," in Proceedings of the Great Lakes Symposium on VLSI 2022, ser. GLSVLSI '22, 2022, p. 507-512.

[36] Cadence, "Cadence webpage," last accessed: 11/20/2023. [Online]. Available: https://www.cadence.com/en_US/home.html

[37] ——, “Jaspergold formal verification platform,” last accessed: 11/18/2023. [Online]. Available: https://www.cadence.com/ enUS/home/tools/system-design-and-verification/formal-and-static-verification/jasper-gold-verification-platform.html

[38] Siemens, "Questa advanced verification," last accessed: 11/18/2023. [Online]. Available: https://eda.sw.siemens.com/en-US/ic/questa/

[39] Cadence, "Fastest simulator to achieve verification closure for IP and SoC designs," last accessed: 11/20/2023. [Online]. Available: https://www.cadence.com/en_US/home/tools/system-design-and-verification/simulation-and-testbench-verification/xcelium-simulator.html

[40] Aldec, "Riviera-PRO: Advanced Verification Platform," last accessed: 11/20/2023. [Online]. Available: https://www.aldec.com/en/products/ functional_verification/riviera-pro

[41] Yosys, "Symbiyosys documentation," last accessed: 11/18/2023. [Online]. Available: https://symbiyosys.readthedocs.io/en/latest/

[42] U. of California Berkley, "ABC," last accessed: 11/20/2023. [Online]. Available: https://people.eecs.berkeley.edu/~alanmi/abc/

[43] M. Mintz and R. Ekendahl, Hardware Verification with System Verilog. Springer Link, 2007.

[44] C. Spear and G. Tumbush, SystemVerilog for Verification: A Guide to Learning the Testbench Language Features. Springer Link, 2012.

[45] S. Fine and A. Ziv, "Coverage directed test generation for functional verification using bayesian networks," in Design Automation Conference, 2003.

[46] L.-T. Wang, Y.-W. Chang, and K.-T. Cheng, Electronic Design Automation: Synthesis, Verification, and Test. Elsevier, 2009.

[47] A. Olofsson, "Intelligent design of electronic assets (idea) & posh open source hardware (posh)," last accessed: 11/18/2023. [Online]. Available: https://www.darpa.mil/attachments/eri_design_proposers_day.pdf

[48] M. Hicks, C. Sturton, S. T. King, and J. M. Smith, "SPECS: A lightweight runtime mechanism for protecting software from security-critical processor bugs," SIGPLAN Not., vol. 50, no. 4, p. 517-529, Mar 2015.

[49] Averant, "Averant solidify," last accessed: 11/20/2023. [Online]. Available: http://www.averant.com/storage/documents/Solidify.pdf

[50] Onespin, "Onespin website," last accessed: 11/18/2023. [Online]. Available: https://www.onespin.com/

[51] Synopsys, "Synopsys webpage," last accessed: 11/18/2023. [Online]. Available: https://www.synopsys.com/

[52] Siemens, "Modelsim," last accessed: 11/18/2023. [Online]. Available: https://eda.sw.siemens.com/en-US/ic/modelsim/

[53] B. Wile, J. Goss, and W. Roesner, Comprehensive Functional Verification: The Complete Industry Cycle. Morgan Kaufmann Publishers Inc., 2005.

[54] E. M. Clarke, W. Klieber, M. Nováček, and P. Zuliani, Model Checking and the State Explosion Problem. Springer, 2012, pp. 1-30.

[55] P. D. Sai Manoj, H. Yu, C. Gu, and C. Zhuo, "A zonotoped macromod-eling for reachability verification of eye-diagram in high-speed i/o links with jitter," in 2014 IEEE/ACM International Conference on Computer-Aided Design (ICCAD), 2014, pp. 696-701.

[56] L. Ni, S. Manoj P. D., Y. Song, C. Gu, and H. Yu, "A zonotoped macromodeling for eye-diagram verification of high-speed i/o links with jitter and parameter variations," IEEE Transactions on Computer-Aided Design of Integrated Circuits and Systems, vol. 35, no. 6, pp. 1040-1051, 2016.

[57] Synopsys, "VCS: The industry's highest performance simulation solutions," last accessed: 11/18/2023. [Online]. Available: https: //www.synopsys.com/verification/simulation/vcs.html

[58] F. Wang, H. Zhu, P. Popli, Y. Xiao, P. Bodgan, and S. Nazarian, "Accelerating coverage directed test generation for functional verification: A neural network-based framework," in Great Lakes Symposium on VLSI, 2018.

[59] M. Tiwari, J. K. Oberg, X. Li, J. Valamehr, T. Levin, B. Hardekopf, R. Kastner, F. T. Chong, and T. Sherwood, "Crafting a usable micro-kernel, processor, and I/O system with strict and provable information flow security," in International Symposium on Computer Architecture (ISCA), 2011.

[60] A. Ardeshiricham, W. Hu, J. Marxen, and R. Kastner, "Register transfer level information flow tracking for provably secure hardware design," in Design, Automation & Test in Europe Conference & Exhibition (DATE), 2017.

[61] X. Li, M. Tiwari, J. K. Oberg, V. Kashyap, F. T. Chong, T. Sherwood, and B. Hardekopf, "Caisson: A hardware description language for secure information flow," SIGPLAN Not., vol. 46, no. 6, p. 109-120, June 2011.

[62] X. Li, V. Kashyap, J. K. Oberg, M. Tiwari, V. R. Rajarathinam, R. Kastner, T. Sherwood, B. Hardekopf, and F. T. Chong, "Sapper: A language for hardware-level security policy enforcement," in International Conference on Architectural Support for Programming Languages and Operating Systems, 2014.

[63] D. Zhang, Y. Wang, G. E. Suh, and A. C. Myers, "A hardware design language for timing-sensitive information-flow security," in International Conference on Architectural Support for Programming Languages and Operating Systems, 2015.

[64] X. Meng, S. Kundu, A. K. Kanuparthi, and K. Basu, "RTL-contest: Concolic testing on RTL for detecting security vulnerabilities," IEEE Transactions on Computer-Aided Design of Integrated Circuits and Systems, vol. 41, no. 3, pp. 466-477, 2022.

[65] T. Trippel, K. G. Shin, A. Chernyakhovsky, G. Kelly, D. Rizzo, and M. Hicks, "Fuzzing hardware like software," in USENIX Security Symposium (USENIX Security), 2022.

[66] S. K. Muduli, G. Takhar, and P. Subramanyan, "Hyperfuzzing for SoC security validation," in IEEE/ACM International Conference On Computer Aided Design (ICCAD), 2020.

[67] N. Kabylkas, T. Thorn, S. Srinath, P. Xekalakis, and J. Renau, "Effective processor verification with logic fuzzer enhanced co-simulation," in IEEE/ACM International Symposium on Microarchitecture, 2021.

[68] R. Kande, A. Crump, G. Persyn, P. Jauernig, A.-R. Sadeghi, A. Tyagi, and J. Rajendran, "TheHuzz: Instruction fuzzing of processors using Golden-Reference models for finding Software-Exploitable vulnerabilities," in USENIX Security Symposium (USENIX Security), 2022.

[69] S. Canakci, C. Rajapaksha, L. Delshadtehrani, A. Nataraja, M. Taylor, M. Egele, and A. Joshi, "Processorfuzz: Processor fuzzing with control and status registers guidance," in IEEE International Symposium on Hardware Oriented Security and Trust (HOST), 2023.

[70] X. Qin and P. Mishra, "Scalable test generation by interleaving concrete and symbolic execution," in International Conference on VLSI Design, 2014.

[71] M. M. Hossain, A. Vafaei, K. Z. Azar, F. Rahman, F. Farahmandi, and M. Tehranipoor, "SoCFuzzer: SoC vulnerability detection using cost function enabled fuzz testing," in Design, Automation & Test in Europe Conference & Exhibition (DATE), 2023.

[72] K. Laeufer, J. Koenig, D. Kim, J. Bachrach, and K. Sen, "RFUZZ: Coverage-directed fuzz testing of rtl on fpgas," in IEEE/ACM International Conference on Computer-Aided Design (ICCAD), 2018.

[73] T. Li, H. Zou, L. D, and Q. W, "Symbolic simulation enhanced coverage-directed fuzz testing of RTL design," in IEEE International Symposium on Circuits and Systems (ISCAS), 2021.

[74] J. Hur, S. Song, D. Kwon, E. Baek, J. Kim, and B. Lee, "DifuzzRTL: Differential fuzz testing to find CPU bugs," in IEEE Symposium on Security and Privacy (SP), 2021.

[75] K. Serebryany, "OSS-Fuzz - google's continuous fuzzing service for open source software," in USENIX Security Symposium, 2017.

[76] Microsoft, "Microsoft security risk detection," last accessed 11/19/2023. [Online]. Available: https://www.microsoft.com/en-us/research/project/project-springfield/

[77] Google, "Americal fuzzy loop," last accessed: 11/20/2023. [Online]. Available: https://github.com/google/AFL

[78] M. Sutton, A. Greene, and P. Amini, Fuzzing: Brute Force Vulnerability Discovery. Addison-Wesley Professional, 2007.

[79] E. Bounimova, P. Godefroid, and D. Molnar, "Billions and billions of constraints: Whitebox fuzz testing in production," in International Conference on Software Engineering (ICSE), 2013.

[80] S. Nagy and M. Hicks, "Full-speed fuzzing: Reducing fuzzing overhead through coverage-guided tracing," in IEEE Symposium on Security and Privacy (SP), 2019.

[81] Kitplot, "Dharma - a generation-based, context-free grammar fuzzer," last accessed: 11/18/2023. [Online]. Available: https://www.kitploit.com/2015/07/dharma-generation-based-context-free.html

[82] R.-V. W. Group, "Risc-v," last accessed: 11/18/2023. [Online]. Available: https://riscv.org/

[83] O. W. Group, "Openrisc," last accessed: 11/18/2023. [Online]. Available: https://openrisc.io/

[84] lowRISC, "Ibex," last accessed 11/19/2023. [Online]. Available: https://lowrisc.org/

[85] Google, "Opentitan," last accessed 11/19/2023. [Online]. Available: https://opentitan.org/

[86] X. Guo, R. G. Dutta, P. Mishra, and Y. Jin, "Automatic code converter enhanced pch framework for soc trust verification," IEEE Transactions on Very Large Scale Integration (VLSI) Systems, vol. 25, no. 12, pp. 3390-3400, 2017.

[87] X. Guo, R. G. Dutta, J. He, and Y. Jin, "Pch framework for ip runtime security verification," in 2017 Asian Hardware Oriented Security and Trust Symposium (AsianHOST), 2017, pp. 79-84.

[88] X. Guo, R. G. Dutta, and Y. Jin, "Eliminating the hardware-software boundary: A proof-carrying approach for trust evaluation on computer systems," IEEE Transactions on Information Forensics and Security, vol. 12, no. 2, pp. 405-417, 2017.

[89] F. Farahmandi, Y. Huang, and P. Mishra, "Trojan localization using symbolic algebra," in 2017 22nd Asia and South Pacific Design Automation Conference (ASP-DAC), 2017, pp. 591-597.

[90] A. Biere, A. Cimatti, E. M. Clarke, O. Strichman, and Y. Zhu, "Bounded model checking," ser. Advances in Computers. Elsevier, 2003, pp. 117-148.

[91] S. Drzevitzky, "Proof-carrying hardware: Runtime formal verification for secure dynamic reconfiguration," in 2010 International Conference on Field Programmable Logic and Applications, 2010, pp. 255-258.

[92] L. De Moura and N. Bjørner, "Z3: An Efficient SMT Solver," in Proceedings of the Theory and Practice of Software, 14th International Conference on Tools and Algorithms for the Construction and Analysis of Systems. Springer-Verlag, 2008, p. 337-340.

[93] INRIA, "The coq proof assistant," last accessed: 11/20/2023. [Online]. Available: https://coq.inria.fr/

[94] Accellera, "Universal Verification Methodology (UVM)," last accessed 11/19/2023. [Online]. Available: https://www.accellera.org/

[95] C. Ioannides, G. Barrett, and K. Eder, "Introducing xcs to coverage directed test generation," in 2011 IEEE International High Level Design Validation and Test Workshop, 2011, pp. 57-64.

[96] J. Yuan, C. Pixley, A. Aziz, and K. Albin, "A framework for constrained functional verification," in ICCAD-2003. International Conference on Computer Aided Design (IEEE Cat. No.03CH37486), 2003, pp. 142- 145.

[97] Cocotb, "Use cocotb to test and verify chip designs in python. productive, and with a smile." last accessed: 11/20/2023. [Online]. Available: https://www.cocotb.org/

[98] M. Teplitsky, A. Metodi, and R. Azaria, "Coverage driven distribution of constrained random stimuli," 2015.

[99] O. Guzey and L.-C. Wang, "Coverage-directed test generation through automatic constraint extraction," in 2007 IEEE International High Level Design Validation and Test Workshop, 2007, pp. 151-158.

[100] Verilator, "Welcome to verilator," last accessed: 11/18/2023. [Online]. Available: https://www.veripool.org/verilator/

[101] J. Wang, B. Chen, L. Wei, and Y. Liu, "Skyfire: Data-driven seed generation for fuzzing," in IEEE Symposium on Security and Privacy (SP), 2017.

[102] M. Böhme, V.-T. Pham, M.-D. Nguyen, and A. Roychoudhury, "Directed greybox fuzzing," in ACM SIGSAC Conference on Computer and Communications Security, 2017.

[103] Google, "Honggfuzz," last accessed: 11/18/2023. [Online]. Available: https://github.com/google/honggfuzz

[104] A. Geralis, "Libfuzzer," last accessed: 11/18/2023. [Online]. Available: https://github.com/planetis-m/libfuzzer

[105] Chisel, "Chisel/FIRRTL hardware compiler framework," last accessed: 11/20/2023. [Online]. Available: https://www.chisel-lang.org/

[106] OpenRISC, "morlkx - an openrisc processor IP core," last accessed: 11/18/2023. [Online]. Available: https://github.com/openrisc/mor1kx

[107] R.-V. W. Group, "Spike ISA simulator," last accessed: 11/18/2023. [Online]. Available: https://github.com/riscv-software-src/riscv-isa-sim

[108] M. R. Clarkson and F. B. Schneider, "Hyperproperties," in 2008 21st IEEE Computer Security Foundations Symposium, 2008, pp. 51-65.

[109] S. Canakci, L. Delshadtehrani, F. Eris, M. B. Taylor, M. Egele, and A. Joshi, "Directfuzz: Automated test generation for rtl designs using directed graybox fuzzing," in 2021 58th ACM/IEEE Design Automation Conference (DAC), 2021, pp. 529-534.

[110] MITRE Corportation, "CVE-2023-3953 improper restriction of operations," last accessed: 11/20/2023. [Online]. Available: https: //cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-3953

[111] C.Alliance, "Flexible internal representation for rtl," last accessed: 11/20/2023. [Online]. Available: https://github.com/chipsalliance/firrtl

[112] GNU, "The GNU assembler," last accessed: 11/18/2023. [Online]. Available: https://ftp.gnu.org/old-gnu/Manuals/gas-2.9.1/ html node/as 3.html

[113] J. M. Vern Paxson, Will Estes, "Lexical analysis with flex, for flex 2.6.2," last accessed: 11/20/2023. [Online]. Available: https://westes.github.io/flex/manual/

[114] MITRE Corportation, "Uncontrolled resource consumption," last accessed: 11/20/2023. [Online]. Available: https://cwe.mitre.org/data/ definitions/400.html

[115] Openhwgroup, last accessed 11/19/2023. [Online]. Available: https: //github.com/openhwgroup/cva6

[116] H. Wong, V. Betz, and J. Rose, "Quantifying the gap between fpga and custom cmos to aid microarchitectural design," IEEE Transactions on Very Large Scale Integration (VLSI) Systems, vol. 22, no. 10, pp. 2067-2080, 2014.

[117] MITRE Corportation, "Incorrect control flow scoping," last accessed: 11/20/2023. [Online]. Available: https://cwe.mitre.org/data/definitions/ 705.html

[118] OSDev, "Cpu registers x86," last accessed: 11/20/2023. [Online]. Available: https://wiki.osdev.org/CPU_Registers_x86#Control_Registers

[119] Five EmbedDev, "Control and status registers (csrs)," last accessed: 11/20/2023. [Online]. Available: https://five-embeddev.com/quickref/ csrs.html

[120] MITRE Corportation, "CVE-2023-29856 in D-link," last accessed: 11/20/2023. [Online]. Available: https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-29856

[121] National Vulnerability Database - ARM, "CVE-2017-5927 in ARM processors," last accessed: 11/20/2023. [Online]. Available: https://nvd.nist.gov/vuln/detail/CVE-2017-5927

[122] MITRE Corportation, "CVE-2023-32342 in IBM X-Force," last accessed: 11/20/2023. [Online]. Available: https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2023-32342

[123] NIST National Vulnerability Database - HP, "CVE-2004-2439 in HP printers," last accessed: 11/18/2023. [Online]. Available: https://nvd.nist.gov/vuln/detail/CVE-2004-2439

[124] NVIDIA, "CVE-2021-1088 in NVIDIA GPU and Tegra hardware," last accessed: 11/20/2023. [Online]. Available: https://nvidia.custhelp.com/app/answers/detail/a_id/5263

[125] MIPS Technologies, "MIPS," last accessed: 11/20/2023. [Online]. Available: https://mips.com/

[126] Ken Vollmar, "Mars," last accessed: 11/20/2023. [Online]. Available: https://courses.missouristate.edu/kenvollmar/mars/

[127] James Larus, "SPIM: A MIPS32 Simulator," last accessed: 11/20/2023. [Online]. Available: https://spimsimulator.sourceforge.net/

[128] C. Trippel, Y. A. Manerkar, D. Lustig, M. Pellauer, and M. Martonosi, "Tricheck: Memory model verification at the trisection of software, hardware, and isa," Proceedings of the Twenty-Second International Conference on Architectural Support for Programming Languages and Operating Systems, 2016.

[129] C. Domas, "Breaking the x86 isa," 2017.

[130] M. Bartley and Chakravarthi, "Shortage of verification resources in the semiconductor industry," last accessed: 11/20/2023. [Online]. Available: https://www.design-reuse.com/articles/28108/shortage-of-verification-resources-in-the-semiconductor-industry.html

[131] Zippa, "Verification engineer projected growth in the united states," last accessed: 11/18/2023.

[132] Google, "Hardware product sprint," last accessed: 11/18/2023. [Online]. Available: https://buildyourfuture.withgoogle.com/programs/ hps