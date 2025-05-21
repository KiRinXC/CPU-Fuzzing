# RTL-ConTest: Concolic Testing on RTL for Detecting Security Vulnerabilities

Xingyu Meng ${}^{ \oplus  }$ , Student Member, IEEE, Shamik Kundu ${}^{ \oplus  }$ , Student Member, IEEE, Arun K. Kanuparthi, Member, IEEE, and Kanad Basu ${}^{\square }$ , Senior Member, IEEE

Abstract-This article presents RTL-ConTest, a register transfer-level (RTL) security vulnerability detection algorithm, that extracts critical process flows from a RTL design and executes RTL-level concolic testing to generate security test cases for identifying critical exploits manifested in a System on Chip (SoC). The efficiency of the proposed approach is evaluated on opensource RISC-V-based SoCs. Our technique is successful in detecting the security vulnerabilities manifested in the processor core as well as in the rest of the SoC, e.g., debug modules, peripherals, etc., thereby providing a thorough vulnerability check on the entire hardware design. As demonstrated by our experimental results, in circumstances where conventional security verification tools are limited, RTL-ConTest furnishes significantly improved efficiency in detecting SoC security vulnerabilities.

Index Terms-Concolic testing, register transfer-level (RTL) design security, security property, security verification.

## I. INTRODUCTION

SYSTEM-ON-CHIPS (SoCs) is used in modern comput- ing systems, which are usually integrated with multiple intellectual property (IP) cores [1]. While these IPs potentially perform their unique functionalities and tasks, they also bring forth their distinctive security challenges when integrated into an SoC and while interacting with other IPs [2]. A vulnerable design is the result of a security bug that escapes the verification phase and later turns out to be an exploit. MITRE recently released the common weakness enumeration (CWE) for hardware; a view which organizes security weaknesses that are frequently encountered in hardware designs. These include issues in debug and test, peripherals and on-chip fabric, core and compute, etc. [3]. While most hardware security vulnerabilities can be addressed using software or firmware patches, some cannot be fixed and will end up as product recall. This results in significant revenue loss and adversely impacts the brand value. Hence, it is imperative to perform thorough security assurance before a product is shipped.

To ensure security robustness, semiconductor companies practice a very rigorous security development lifecycle (SDL) process [4], [5], concurrently with the conventional hardware development cycle [6]. When the hardware architectural features are finalized, a comprehensive security assessment in SDL is performed, as shown in Fig. 1. SDL prescribes testing methodologies for both presilicon (RTL) and post-silicon testing. Unfortunately, since most of the commercial electronic design automation (EDA) tools are predominantly developed to ensure the functional correctness of the design, it is difficult to validate complicated security properties during the verification phase. Security properties, compared to functional properties, require far more complete and detailed description in terms of prohibited and permitted behaviors. Regular functional verification can be applied to a module to detect functional bugs. A functional bug, manifested due to deviation from the functional specification, has a functional impact on the SoC. But, it might not necessarily have a security impact. On the contrary, a security vulnerability might not have any user-visible functional impact, but will have a security impact (compromise of confidentiality, integrity, or availability). Therefore, detailed security verification approaches are required to ensure that there is no vulnerability in the SoC. When traditional techniques do not work effectively, manual register transfer-level (RTL) reviews are conducted, which are not scalable. In spite of such rigorous testing, no chip is guaranteed to be totally secure, and escapes can happen, significantly impacting the design house.

![0196f2c5-9f7d-74cc-ab1e-9ee3b8fb88ae_0_929_543_705_265_0.jpg](images/0196f2c5-9f7d-74cc-ab1e-9ee3b8fb88ae_0_929_543_705_265_0.jpg)

Fig. 1. SDL process followed by the semiconductor companies.

The wake of microarchitectural and hardware attacks necessitates a paradigm shift in existing hardware security assurance tools and methodologies [2]. Similar to functional validation, it is desirable to have directed test cases to detect security vulnerabilities in a design. Directed test generation using formal methods like bounded model checking on such RTL designs suffer from state space explosion [7]. On the other hand, concolic testing addresses this issue by combining symbolic execution with concrete simulation [8]. It is a software verification technique, which considers a variable in the program as a symbolic variable and then performs concrete execution on a specific path to generate input patterns. Recently, a concolic testing-based technique, Coppelia has been proposed, which converts the entire processor RTL to a high-level software equivalent using tools such as Verilator [9], to detect potential security vulnerabilities at the RTL level [10].

---

Manuscript received October 28, 2020; revised January 16, 2021; accepted February 23, 2021. Date of publication March 17, 2021; date of current version February 21, 2022. This article was recommended by Associate Editor S. Ghosh. (Corresponding author: Xingyu Meng.)

Xingyu Meng, Shamik Kundu, and Kanad Basu are with the Department of Electrical and Computer Engineering, University of Texas at Dallas, Richardson, TX 75080 USA (e-mail: xxm150930@utdallas.edu).

Arun K. Kanuparthi is with the Product Assurance and Security (IPAS), Intel Corporation, Hillsboro, OR 97124 USA.

Digital Object Identifier 10.1109/TCAD.2021.3066560

---

Although Coppelia could detect core-level security vulnerabilities, it has not been demonstrated to perform security analysis outside the processor core of a complete SoC design. Verilator can treat core and peripherals of an SoC separately, and it is possible to generate test cases with the software representation of the RTL obtained from this tool. However, it flattens the design and creates a single cycle-accurate process model, thus ignoring all the intracycle delays during this process. Typically, the peripherals of an SoC have multiple clock cycles, which handle multiple process flows assigned to them. Furthermore, generating the test cases for them requires multiple flow graphs to represent the concurrent processes. This poses a major challenge in detecting critical security vulnerabilities across different modules in the SoC [2].

In this article, we present RTL-ConTest, a comprehensive framework, which can be used to efficiently validate the security properties and detect security vulnerabilities of an SoC. In our proposed technique, we extract the process flow for symbolic execution by generating the critical flows of target RTL as control flow graphs (CFGs), and then implement RTL-level concolic testing to generate security test cases. Prior research has demonstrated the application of concolic testing to generate directed test cases in both hardware and software designs [8], [11], [12]. In the software domain, concolic testing uses the structural information provided by CFG to efficiently guide the concrete path toward coverage target. However, since the software-based test generation technique deals with only one CFG, it cannot handle multiple concurrent CFGs, which is required for directed test generation in hardware. Furthermore, it falls short in dealing with the complexity of unrolling CFGs across multiple clock cycles [13]. Typical software executions use sequential processing, which is significantly different from conventional RTL process flow. Since the process flows are not executed simultaneously, the value of a variable in software execution will differ from the hardware execution. Furthermore, the clock cycles required to complete the function will increase, when compared to the hardware process. Hence, it is difficult to collect the correct value of each variable and the period of clock cycle through software simulation. Therefore, to alleviate this setback, we choose to perform RTL-level concolic testing to detect security vulnerabilities manifested in the hardware design of the SoC. As demonstrated in Section V, RTL-ConTest is most efficient in detecting hardware vulnerabilities categorized in Debug and Test (CWE-1207). As an example of a debug design vulnerability, a 32-bit password check function in JTAG module does not check the last bit of input password. Such a security vulnerability is difficult to detect using standard taint tracking tools, since the password check function is still included in the security flow. However, RTL-ConTest can generate test cases to observe the user input of the password, and detect the faulty function of password check. The generated test cases are the input values that are recorded for each clock cycle until the execution flow reaches the end of the process. The key contributions of this article are as follows.

1) We propose a security-specific RTL-based concolic testing algorithm, RTL-ConTest. This scalable security vulnerability detection algorithm operates directly on the SoC RTL, instead of a transformed software equivalent, thereby efficiently identifying potential security exploits in the design based on predetermined security properties.

2) We develop a novel ${CFG}$ generator to extract the critical process flow from the selected RTL, which drives the ensuing concrete path specification and concolic testing algorithms to generate RTL-level security test cases. These test cases are validated against preestablished security properties to detect critical violations in the SoC RTL.

3) The proposed RTL-ConTest algorithm is evaluated on two opensource RISC-V-based SoCs. The algorithm is successful in detecting 17 security vulnerabilities in PULPissimo SoC-4 in the core and 13 in the rest of the SoC, and seven security vulnerabilities in Ariane SoC-3 in the core and 4 in the rest of the SoC.

The remainder of this article is organized as follows. Section II describes the related work in the domain of vulnerability detection on hardware designs. Section III provides an overview of the proposed algorithm. Section IV briefly discusses the SoC architectures, security objectives and threat model, and security features added to support the security objectives. Section V evaluates the performance of RTL-ConTest in detecting the security vulnerabilities manifested in RISC-V-based SoCs. Finally, Section VI concludes this article.

## II. RELATED WORK

The software industry has several tools and methodologies for evaluating security robustness of software, either for source code or for binaries. However, there are very few EDA tools that are specific to RTL security. A paradigm shift is necessary in the development of tools and methodologies for hardware security assurance. We will describe a few existing methodologies for ensuring RTL security in this section.

Information Flow Tracking (IFT): Dynamic IFT prevents malicious software attacks by identifying counterfeit input channels, restricting the usage of spurious information, and detecting hardware timing channels, thereby bolstering the security of a hardware architecture [14]-[19]. Although these techniques demonstrate their efficiency in tracking malicious information, they lack in performance on systematic evaluation on complex RTL designs.

Assertion-Based Verification: Assertion-based security verification approaches utilize simulation or formal analysis to observe violations in the included assertions, thereby assisting in the generation of security properties in an SoC [20], [21]. Assertion-based security verification for an SoC core has been proposed by [1], [22]. However, none of the authors investigated security violations in the noncore components of the SoC like the peripherals.

Property-Specific Information Flow: Property specification creates information flow models are tailored to the properties to be verified by performing a property-specific search to identify security critical paths [23]. This technique helps to find suspicious signals that require closer inspection and quickly eliminates portions of the design that are free of security violations.

![0196f2c5-9f7d-74cc-ab1e-9ee3b8fb88ae_2_183_146_659_253_0.jpg](images/0196f2c5-9f7d-74cc-ab1e-9ee3b8fb88ae_2_183_146_659_253_0.jpg)

Fig. 2. RTL-ConTest framework.

Symbolic Testing: Directed test generation frameworks have been developed with the concept of symbolic testing [24], which resolve the inherent problem of state space explosion in the aforementioned formal approaches. Recent research in the direction of deploying concolic testing directly on RTL models is presented in [13] and [25], to ensure the functional correctness of the designed hardware. However, these approaches tend to overlook the detection of security vulnerabilities that may inhibit in the RTL design. Ahmed et al. [13] did not apply security properties in the algorithm and, hence, it could not detect security vulnerabilities in the design, leading to runtime violation. Lyu and Mishra [25] proposed a technique to apply security assertions in the original RTL for detecting manifested Trojans. However, this approach will incur massive area overhead on the design if it requires the assertions to cover all the security properties. Coppelia, a symbolic testing-based directed test generation framework, has been proposed in [10], which converts the hardware RTL design into a high-level software format by utilizing Verilator [9]. KLEE symbolic testing framework [26] executes the vulnerability check on the translated software design. Coppelia has only demonstrated to detect bugs only in the CPU core of the SoC. Furthermore, it can be assumed without loss of generality that it can detect bugs in the rest of the SoC that are accessible over memory-mapped input/output (MMIO), e.g., bugs that require physical presence of an adversary to exploit. However, it cannot test SoC components that are not MMIO accessible.

Symbolic testing with test generation to detect inserted hardware Trojans by exploiting abnormal test patterns has been proposed by Shen et al. [27]. Although this method can detect the presence of maliciously introduced hardware Trojans, it is not capable of verifying security vulnerabilities caused by faulty designs in a full-scale SoC. Symbolic testing techniques have also been proposed for RTL functional validation [28]- [32]. However, these techniques are not capable of detecting RTL-level security vulnerabilities.

To the best of our knowledge, this is the first work that performs an extensive intermodular security vulnerability check directly on the entire RTL design of the SoC, including the core, by generating security test patterns with the help of RTL-level concolic testing to validate the crucial security properties of the design.

![0196f2c5-9f7d-74cc-ab1e-9ee3b8fb88ae_2_915_142_735_939_0.jpg](images/0196f2c5-9f7d-74cc-ab1e-9ee3b8fb88ae_2_915_142_735_939_0.jpg)

Fig. 3. Example Verilog code with multiple processes. (a) RTL of Process I. (b) ${CFG}$ of Process I. (c) RTL of Process II. (d) ${CFG}$ of Process II. Each block represents the node number; line number, and flow value. For example, ${n}_{1};{L}_{2},0$ represents node ${n}_{1}$ with execution line ${L}_{2}$ , with a flow value of 0 .

## III. RTL-CONTEST

In this section, we describe the RTL-ConTest framework. Fig. 2 demonstrates a high-level representation of the proposed approach. The threat model and security objectives pertinent to an $\mathrm{{SoC}}$ are leveraged to obtain the security properties. The security properties and the target RTL are provided as input to RTL-ConTest, which has three phases as described in detail in this section.

## A. Control Flow Graph Generation

In RTL-ConTest, we develop an independent approach to extract the process flows from the RTL using a ${CFG}$ , as outlined in Algorithm 1. First, we initialize the ${CFG}$ with an empty list, and search for all the if-else conditions in the RTL design. An entire set of interlinked if-else conditions is regarded as an if_chain. We isolate each condition and its assignment and proceed to establish their flow order in the ${CFG}$ . A sub ${CFG}$ array is defined as the connection between a condition and its assignment. Next, we define a variable flow to represent the order of the if_chain that decides the succeeding condition. The value of flow is incremented for each if statement. For example, in Fig. 3(a), ${L}_{2}$ and ${L}_{3}$ are contained in one subCFG, and marked as nodes ${n}_{1}$ and ${n}_{2}$ , respectively, in the if_chain. Node ${n}_{1}$ is part of flow 0 and ${n}_{2}$ is part of flow 1 . Fig. 3 shows the ${CFG}$ for two interconnected processes. The process in Fig. 3(a) demonstrates a set of if-else statements in Verilog. It assigns the value of variable state, which is the trigger variable for the case statements in Fig. 3(c). When the input of the module changes, different values will be assigned to the state, which will impact the output of the processes. This process can be transformed into a ${CFG}$ as shown in Fig. 3(b). Once an if_chain is added into the ${CFG}$ , the algorithm starts searching for the next if-else condition.

The algorithm is further extended to analyze the case statements in the RTL, which share identical characteristics as the if-else conditions. For example, in Fig. 3(c), the case state in line ${L}_{12}$ represents that if state $=  = A$ , the lines from ${L}_{14}$ to ${L}_{15}$ will be executed. This format can be transformed into an if condition: if state $=  = A$ is True, then execute lines ${L}_{14}$ and ${L}_{15}$ . With this transformation, the process II shown in Fig. 3(c) can be converted into a subCFG shown in Fig. 3(d). If a critical flow of if-else condition or case statement is declared in a loop, RTL-ConTest first extracts all the lines that are in the loop, along with the loop variable and the number of iterations in the loop. Algorithm 1 generates sub- ${CFG}$ s for all the if-else conditions and case statements within a for_loop. For example, a for_loop with ten iterations will generate ten different subCFGs, all of which will be included in the concluding ${CFG}$ .

## B. Path Specification

In order to generate test cases for a specific property in RTL, we need to provide the symbolic execution engine a particular execution path, which is termed a concrete path. The path specification algorithm extracts this concrete path from the generated ${CFG}$ , as shown in Algorithm 2. The concrete path usually starts from the final assignment of a process to test whether the function is operating correctly. These assignments are selected as target nodes, e.g., nodes ${n}_{2},{n}_{4},{n}_{6}$ , and ${n}_{7}$ shown in Fig. 3(b) and nodes ${n}_{10},{n}_{13},{n}_{14},{n}_{16}$ , and ${n}_{17}$ , shown in Fig. 3(d). The assignments usually have a guard_condition before reaching the node. The guard_condition is a Boolean condition check that guards the node, e.g., node ${n}_{3}$ in Fig. 3(b) is the guard_condition when the target is ${n}_{4}$ . The variable that guards the assignment is set as guard_variable and the value that satisfies the condition is set as guard_value. For example, in a guard_condition represented by ${n}_{3}$ in Fig. 3(b), the guard_variable is input and its guard_value is 1 . The function expand_guard_condition searches for the target node in the CFG and returns the guard_variable and the guard_value of the target node.

The function set_of_strict_value searches the process flow for any assignment to the guard_variable with the guard_value. If there is such an assignment, the guard_value of the assignment is marked as strict_value, and the node containing the assignment is added as the predecessor of the target node and the new target in the path. The algorithm then continues with the new target. For example, in case of selecting node ${n}_{16}$ in Fig. 3(d) as the first target, its guard_condition is state $=  = C$ . Node ${n}_{6}$ in Fig. 3(b) assigns the guard_value to the guard_variable. Hence, ${n}_{6}$ becomes the predecessor of the path and the new target. If there is no such assignment, the guard_condition is added as the predecessor of the path, and is set as the new target. The algorithm terminates the path specification when there is no guard_condition of the current target. Once we obtain the concrete path, path_evaluation is performed. This function furnishes with a reference of the node that has the most critical information. The evaluation initializes the priority in the concrete path from the final target node to the first predecessor in order of the highest to the lowest and sets the rest of the nodes as no priority.

Algorithm 1 CFG Generation

---

Input: RTL, Security Property

Output: ${CFG}$

	Initialize ${CFG} \leftarrow$ empty list

	for all lines in RTL file do

		for all if_chain do

			Initialize subCFG $\leftarrow$ empty Array

			Initialize flow $\leftarrow  0$

			while if_chain is not complete do

				subCFG $\leftarrow$ [condition, flow]

				flow ++

				subCFG append [assignments, flow]

				CFG append subCFG

				if else-if or else in if_chain then

					goto 7

				end if

			end while

		end for

		for all case statement do

			if_chain $\leftarrow$ transform(case statement),

			goto 6

		end for

		for all for_loop in the line do

			extract lines in for_loop, loop_var of for_loop

			Count number of loops

			if if_chain in lines then

				goto 3 ,

				get subCFG of if_chain

				for $i$ in range of(number of loops) do

					subCFG[i] replace loop_var with $i$

					CFG append subCFG[i]

				end for

			end if

			if case statement in lines then

				if_chain $\leftarrow$ transform(case statement)

				goto 23

			end if

		end for

	end for

---

## C. Test Generation

In the next step, we proceed to use concolic testing for generating test cases to detect security vulnerabilities, as represented in Algorithm 3. We develop a symbolic simulator and add security properties to reinforce its abilities to detect potential security vulnerabilities during each simulation cycle. The algorithm starts with assigning random values to the first round inputs. If the target_node is not in the execution path, the simulator checks which nodes in the concrete path have been reached, and reduces the priority of these nodes. The constraint_solver developed by us is used to solve the guard_condition of the node with the highest priority. The constraints in the RTL are translated to software representations during the CFG generation by transforming the Boolean conditions into equivalence conditions, e.g., if (~variable) in RTL is translated into if (variable $=  = 0$ ). From these constraints, the constraint_solver computes the guard_values, following which it searches for guard_variables that are provided as inputs or are assigned by the inputs. If a satisfiable input vector is found, the simulator assigns this input with the guard_value for the next round of simulation. The security restrictions added to Algorithm 3 are as follows.

## Algorithm 2 Path Specification

---

Input: ${CFG}$

Output: Path

Select target_node

update_edge(target_node)

	Path append target_node

	${n}_{g},{n}_{v} \leftarrow$ expand_guard_condition (target_node, CFG)

	${n}_{s} \leftarrow$ set_of_strict_value $\left( {{n}_{g},{n}_{v}}\right)$

xpand_ guard_condition(target_node, CFG)

	search for assignment $\in$ subCFG, target_node $\in$ assignment

	get guard_condition of assignment

	${n}_{g} \leftarrow$ guard_variable $\in$ guard_condition

	$: {n}_{v} \leftarrow$ guard_value $\in$ guard_condition

	of_of_strict_value $\left( {{n}_{g},{n}_{v}}\right)$

	search for assignment $\in  {CFG},{n}_{g} \in  {subCFG}$

	if ${n}_{g}$ in assignment then

		if ${n}_{v} =$ assignment.value then

			node $\leftarrow$ assignment.node

			target_node $\leftarrow$ node

			Path append node of assignment

			update_edge(node)

		end if

	end if

	else Path append predecessors of target_node

path_evaluation(CFG, Path)

	all node.priority in ${CFG} \leftarrow  \infty$

	for all node in Path do

		node.priority $\leftarrow$ length(Path) - index(node)

	end for

---

Algorithm 3 Test Generation

---

Input: Path, CFG, Security_ Properties

Output: Simulation Vector

1: Restricts $\leftarrow$ Security_ Properties

	Initialize Input $\leftarrow$ randombits(   )

	Executed Path, Invalid $\leftarrow$ Simulate(Input, Restricts)

	if target_node $\in$ path then

		return Input

	end if

	while target_node $\notin$ path do

		guiding_node $\leftarrow$ adjust_priority(Path)

		Input $\leftarrow$ constraint_solver(guiding_node)

		Testcase append(Input)

		Execute Path, Invalid $\leftarrow$ Simulate(Input, Restricts)

		if target_node $\in$ Path then

			return Testcase

		end if

		if Invaild not empty then

			return Invalid

		end if

	end while

	nulate(Input, Restricts)

	Initialize Invalid

	for all node in ${CFG}$ do

		Variables $\leftarrow$ Input

		values $\leftarrow$ execute node

		Input append values

		check_invalid(Restricts, Input)

	end for

---

1) Variable Boundary: A variable boundary sets a restriction on the value of a particular variable. The security property defines a specific boundary for the security-critical variable. This property helps to detect threats, which causes incorrect assignments or faulty logic in the process.

2) Condition Check: A condition check creates a chain between two variables, e.g., input $=  = 1$ and state $=  = \mathrm{B}$ both are required to be True simultaneously. Thus, the algorithm detects whether there is any incorrect statement in the simulation process.

3) Test Case Record: The test case records the inputs, while the algorithm is reaching the target node. It analyzes these to verify whether the generated test cases match the specified security properties, violation of which identifies the existence of threats, such as incorrect timing and under-specific logic.

The symbolic simulator uses the CFG generated from Algorithm 1 to simulate the flow of the RTL design. All the assignments and parameters in RTL are incorporated in the simulator to enable it with the exact information flow. After each subCFG is executed, it updates the simulation flow with the node that has been executed in the process flow. In the next step, all the variables in the ${CFG}$ are updated and checked with respect to the security restrictions. If there is a mismatch, the simulator returns the test cases and mismatch information and terminates the simulation, displaying an invalidation message. If the simulation reaches the target node, the algorithm terminates simulation and returns all inputs that have been assigned, validating the security property.

### IV.SoC SECURITY OBJECTIVES, THREAT MODEL, AND SECURITY ARCHITECTURE

We evaluate RTL-ConTest on open source RISC-V-based SoCs, which were used in the hardware hacking contest, Hack@DAC 2018 [33] and Hack@DAC 2020 [34], the flagship hardware hacking competition co-located with the Design Automation Conference. The nontrivial SoCs provide substantial design complexity with relevant security features. They consist of an adequate blend of manifested security-critical vulnerabilities across multiple hardware CWE categories that are commonly exploited by the potential adversaries. In this section, we briefly describe the overall architectures of the SoCs, the adversary and threat model, security objectives of the SoCs, and the security features that were implemented to support the security objectives. We then list out the security properties that are validated using RTL-ConTest.

### A.SoC Architecture

Fig. 4 represents the high-level architecture of the open source PULpissimo SoC (PULPino Gen 2) [33] used in the hardware hacking competition, Hack@DAC 2018 [2]. The entire SoC contains around 500 Verilog files with 26000 lines of code. It consists of a 32-bit RISC-V core tightly coupled with a memory subsystem. The core can support privilege separation between supervisor and user. The SoC boots out of the boot ROM. It also has an I/O subsystem through which the SoC can connect to interfaces such as UART, SPI, I2C, etc. A hardware processing engine (HWPE) implements the cryptographic accelerators for AES, MD5, and media access control (MAC). Peripherals, such as the real-time clock (RTC), temperature sensors, and general-purpose inputs/outputs (GPIOs) are connected using the advanced extensible interface (AXI) and advanced peripheral bus (APB) interfaces. The maximum depth of Pulpissimo is 5, which is the core process of the SoC. The SoC can be debugged through the JTAG interface, which is connected to the advanced debug unit. The entire $\mathrm{{SoC}}$ is memory mapped and the peripheral registers can be accessed over MMIO.

![0196f2c5-9f7d-74cc-ab1e-9ee3b8fb88ae_5_140_142_740_418_0.jpg](images/0196f2c5-9f7d-74cc-ab1e-9ee3b8fb88ae_5_140_142_740_418_0.jpg)

Fig. 4. PULpissimo SoC architecture.

Fig. 5 represents the high-level architecture of the open source Ariane SoC, used in the hardware hacking competition, Hack@DAC 2020 [34]. The SoC consists around 460 Verilog files with 20000 lines of code [34]. The core supports three privilege levels (Master, Slave, and User), and it consists of a 64-bit address space. It has a 6-stage pipeline implementation of PC generation, instruction fetch, instruction decode, issue handler, execute stage, and commit stage. The maximum depth of Ariane is 7, which is the AXI process of the SoC. The IO system is implemented with AXI interface interconnected with core-peripherals such as SPI, UART, JTAG, CLINT (Core-local Interrupt Controller), and FUSE. The cryptographic modules are implemented with AES, SHA256, and HMAC-256.

## B. Adversary and Threat Model

We utilize the security threat model used in the two competitions. We consider an unprivileged software adversary and a simple hardware adversary to be in scope. An unprivileged software adversary executes code on the core in user mode. She has complete control over user-space processes and can issue unprivileged instructions and system calls. A simple hardware adversary has physical access to the device and can add or remove components from the platform and has the skills of a JTAG debugger. The goal of the adversary is to exploit vulnerabilities in the system to bypass security objectives and invoke unintended functionality.

## C. Security Objectives and Features

The following are the security objectives of the SoC as specified in the competition.

1) SOI: Prevent unprivileged code running in the core from compromising the privilege level.

2) SO2: Protect debug interface from malicious debugger.

3) SO3: Protect device integrity from software adversary.

![0196f2c5-9f7d-74cc-ab1e-9ee3b8fb88ae_5_909_145_751_277_0.jpg](images/0196f2c5-9f7d-74cc-ab1e-9ee3b8fb88ae_5_909_145_751_277_0.jpg)

Fig. 5. Ariane SoC architecture.

Several security features are added to support the above-listed security objectives. These include: privilege-based access control to peripherals and write locks on security-sensitive registers to prevent unprivileged code from compromising privilege level. For example, a password-based unlock mechanism is implemented to access the debug interface. The password is stored in the JTAG module in Pulpissimo SoC, and it is generated in the hash message authentication code (HMAC) module and passed to the JTAG interface in Ariane SoC. Write protection is also enabled on temperature configuration registers to prevent persistent denial of service. There are other ways to prevent denial-of-service attacks besides the write protection, e.g., adding additional sensors to monitor the temperature read. However, these methods usually require large hardware overhead and they are typically not considered when write protection is valid.

To perform security verification, we define security properties based on the threat model and security objectives. Since RTL descriptions of the SoCs are available to us, we manually draft the security properties from the security specifications of the SoC. These security properties span all the security features that were added to the SoC-register locks, debug infrastructure, cryptographic accelerators, fabric address decode and access control, peripherals, and the core.

## V. EXPERIMENTAL RESULTS

## A. Experimental Setup

We implemented the RTL-ConTest algorithm in Python 3.7. All the security properties used for evaluating RTL-ConTest are outlined and categorized in Table I. In the following section, these security properties (generic, register lock, and infrastructure) are evaluated with RTL-ConTest, and the violations for each of those properties are illustrated, which are represented as bugs in Table II. Moreover, Table II outlines the location and CWE category of each of the bugs and a comparative analysis on the efficiency of existing security vulnerability detection techniques against our proposed approach.

## B. Analysis of Security Properties

1) Generic Properties (GP): First, we describe the generic security properties that are valid for all functional units of the Pulpissimo and Ariane SoC. According to the design specifications, we have the following properties.

a) ${GP1}$ : The assignment to a register should not exceed its maximum address length. For example, logic [5:0] data define the maximum decimal value of register data to be 63 . An inappropriate assignment that results in overflow (in this

TABLE I

LIST OF SECURITY PROPERTIES FOR THE SOC

<table><tr><td>$\mathbf{{No}.}$</td><td>Security Property</td><td>Type</td><td/><td>$\mathbf{{No}.}$</td><td>Security Property</td><td>Type</td></tr><tr><td>GP1</td><td>Assignment to a register should not exceed its maximum address length.</td><td rowspan="5">Generic</td><td/><td>RLP4</td><td>Cryptographic should be protected by lock control.</td><td>Register Lock</td></tr><tr><td>GP2</td><td>All assignments should be operative in the RTL.</td><td/><td>IP1</td><td>JTAG interface should be protected by password and the password check function should check the whole password.</td><td rowspan="8">Infrastructure</td></tr><tr><td>GP3</td><td>The register address length should be equivalent to the assignment address length.</td><td/><td>IP2</td><td>Reset function should clear the function information registers.</td></tr><tr><td>GP4</td><td>Registers should not have negative address lengths.</td><td/><td>IP3</td><td>Timing calculation should operate in correct logic.</td></tr><tr><td>GP5</td><td>Variables in assignments should have a valid definition and value.</td><td/><td>IP4</td><td>Access to certain information should be in the correct privilege level.</td></tr><tr><td>RLP1</td><td>Lock control register can not be overwritten by the software.</td><td rowspan="4">Register Lock</td><td/><td>IP5</td><td>Status and privilege level of core units should be updated correctly.</td></tr><tr><td>RLP2</td><td>Reset function should not erase the lock control register.</td><td/><td>IP6</td><td>Encryption modules should not be integrated with regular modules.</td></tr><tr><td>RLP3</td><td>Only one peripheral should be activated by</td><td/><td>IP7</td><td>Exception of the unit should be checked and handled.</td></tr><tr><td/><td>the lock control in an unit.</td><td/><td>IP8</td><td>LSB of instruction address should be updated correctly.</td></tr></table>

TABLE II

Comparative Analysis of Vulnerability Detection Efficiency of Formal Verification Approaches (JasperGold Security Path Verification (SPV) [35] and Formal Property Verification (FPV) [36]), Coppelia [10], and Our Proposed RTL-ConTest. Preck, Check, Cross and Question Marks Represent Successful, Unsuccessful, and Unavailable Vulnerability Detection Results, Respectively. The Bugs in PULPissimo and Ariane SoC Are Shown in Black and Blue, Respectively. Out of 26 Security Vulnerabilities in Both SoCs. SPV, FPV, AND COPPELIA CAN DETECT 10, 19, AND 11 BUGS, RESPECTIVELY. ON THE OTHER HAND, RTL-CONTEST IS SUCCESSFUL IN DETECTING 24 SECURITY VULNERABILITIES ACROSS BOTH SOCS case, data register greater than 63) is considered as a violation of GP1. RTL-ConTest detects a violation of this property in the Advanced Debug Unit of the JTAG module in PULPissimo SoC during password check. As shown in Listing 1, register bitindex attains a value of 32 , after 31 correct inputs for the password. Since the index of pass has an address range from 0 to 31, register bitindex encounters an overflow, thereby violating GP1. This engenders a security-critical bug (b3 in Table II), because it will result in an incorrect password check function with a faulty value of bitindex.

<table><tr><td>$\mathbf{{No}.}$</td><td>Bug Description</td><td>Prop. Viol.</td><td>Location</td><td>HW CWE Category [3]</td><td>SPV</td><td>$\mathbf{{FPV}}$</td><td>Coppelia</td><td>RTL-ConTest</td></tr><tr><td>${bl}$</td><td>Address range overlap between peripherals SPI Master and SoC</td><td>RLP3</td><td>SoC Peripherals Bus definition</td><td>Peripherals and Fabric (CWE-1203)</td><td>✓</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>${b2}$</td><td>Address range overlap between GPIO, SPI, and SoC peripherals</td><td>RLP3</td><td>SoC Peripherals Bus definition</td><td>Peripherals and Fabric (CWE-1203)</td><td>✓</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>b3</td><td>Incorrect password checking logic in debug unit</td><td>GP1</td><td>Advanced Debug Unit</td><td>Debug and Test (CWE-1207)</td><td>✘</td><td>✓</td><td>?</td><td>✓</td></tr><tr><td>${b4}$</td><td>Advanced debug unit only checks 31 of the 32 bits of the password</td><td>IP1</td><td>Advanced Debug Unit</td><td>Debug and Test (CWE-1207)</td><td>✘</td><td>✓</td><td>?</td><td>✓</td></tr><tr><td>${b5}$</td><td>Reset for the advanced debug unit not operational</td><td>IP2</td><td>Advanced Debug Unit</td><td>Power, Clock, and Reset (CWE-1206)</td><td>✘</td><td>✘</td><td>?</td><td>✓</td></tr><tr><td>${b6}$</td><td>Password is hard-coded and set on reset.</td><td>IP1</td><td>Advanced Debug Unit</td><td>Debug and Test (CWE-1207)</td><td>✘</td><td>✘</td><td>?</td><td>✓</td></tr><tr><td>${b7}$</td><td>Faulty logic in the RTC causing inaccurate time calculation</td><td>IP3</td><td>RTC Clock Unit</td><td>Peripherals and Fabric (CWE-1203)</td><td>✘</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>${b8}$</td><td>GPIO lock control register can be written by software</td><td>RLP1</td><td>Advanced Peripheral Bus</td><td>Circuit and Logic (CWE-1199)</td><td>✓</td><td>✓</td><td>?</td><td>✓</td></tr><tr><td>${b9}$</td><td>Reset function clears the GPIO lock control register</td><td>RLP2</td><td>Advanced Peripheral Bus</td><td>Circuit and Logic (CWE-1199)</td><td>✓</td><td>✓</td><td>?</td><td>✓</td></tr><tr><td>bl0</td><td>Incorrect address range for APB allows memory aliasing</td><td>GP3</td><td>Advanced Peripheral Bus</td><td>Peripherals and Fabric (CWE-1203)</td><td>✘</td><td>✓</td><td>?</td><td>✓</td></tr><tr><td>b11</td><td>AXI address decoder ignores errors</td><td>IP5</td><td>Advanced Extensible Interface</td><td>Circuit and Logic (CWE-1199)</td><td>✓</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>bl2</td><td>JTAG interface is not password protected</td><td>IP1</td><td>JTAG Interface Tap</td><td>Debug and Test (CWE-1207)</td><td>✘</td><td>✘</td><td>?</td><td>✓</td></tr><tr><td>b13</td><td>Output of MAC is not erased on reset</td><td>IP2</td><td>Multiplexer Function Unit,</td><td>Power, Clock, and Reset (CWE-1206)</td><td>✓</td><td>✓</td><td>?</td><td>✓</td></tr><tr><td>bl4</td><td>Cryptographic information is not stored in secure memory</td><td>RLP4</td><td>Multiplexer Function Unit.</td><td>Memory and Storage Issues (CWE-1202)</td><td>✘</td><td>✘</td><td>?</td><td>✘</td></tr><tr><td>b15</td><td>Temperature module is muxed with encryption modules</td><td>IP6</td><td>Multiplexer Function Unit</td><td>Circuit and Logic (CWE-1199)</td><td>✘</td><td>✘</td><td>?</td><td>✘</td></tr><tr><td>bl6</td><td>Unit is able to access debug register when in halt mode</td><td>IP4</td><td>PULPino Core</td><td>Privilege and Access Control (CWE-1198)</td><td>✓</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>bl7</td><td>Processor assigns privilege level of execution incorrectly from CSR</td><td>IP5</td><td>PULPino Core</td><td>Privilege and Access Control (CWE-1198)</td><td>✘</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>bl8</td><td>Secure mode is not required to write to interrupt registers</td><td>GP2</td><td>PULPino Core</td><td>Privilege and Access Control (CWE-1198)</td><td>✓</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>b19</td><td>LSB of jump target address is not set as 0</td><td>IP8</td><td>PULPino Core</td><td>Circuit and Logic (CWE-1199)</td><td>✘</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>${b20}$</td><td>Flits register overflow when sending maximum number</td><td>GP1</td><td>Ariane Core</td><td>Circuit and Logic (CWE-1199)</td><td>✘</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>b21</td><td>Exception of CSR is not updated to controller unit</td><td>IP7</td><td>Ariane Core</td><td>Circuit and Logic (CWE-1199)</td><td>✘</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>${b22}$</td><td>Address ranges of PMP configuration register don't match</td><td>GP3</td><td>Ariane Core</td><td>Peripherals and Fabric (CWE-1203)</td><td>✘</td><td>✓</td><td>✓</td><td>✓</td></tr><tr><td>${b23}$</td><td>Incorrect assignment assigns corrupted value to register</td><td>GP5</td><td>JTAG Interface</td><td>Debug and Test (CWE-1207)</td><td>✘</td><td>✘</td><td>?</td><td>✓</td></tr><tr><td>${b24}$</td><td>Password check function not operative</td><td>GP2</td><td>JTAG Interface</td><td>Debug and Test (CWE-1207)</td><td>✓</td><td>✓</td><td>?</td><td>✓</td></tr><tr><td>${b25}$</td><td>Reset function doesn't reset the successful check</td><td>IP2</td><td>JTAG Interface</td><td>Power, Clock, and Reset (CWE-1206)</td><td>✓</td><td>✓</td><td>?</td><td>✓</td></tr><tr><td>b26</td><td>JTAG data instruction is not set at the LSB of ID code</td><td>IP8</td><td>JTAG Interface</td><td>Circuit and Logic (CWE-1199)</td><td>✘</td><td>✘</td><td>?</td><td>✓</td></tr></table>

---

begin

														if (tms_pad_i && (passchk))

																								next_TAP_state = 'state_select_dr_scan;

													else begin

																									next_TAP_state = 'state_run_test_idle;

																									if (correct >= 32' h0000_001F)

																																				passchk = 1;

																								else if (tdi_o == pass[bitindex]) begin

																																					correct++;

																																				bitindex++;

																									end

													end

		end

---

Listing 1. Code for password check in Pulpissimo SoC.

RTL-ConTest detects another violation of GP1 in the packet filter unit of the Ariane core. In this unit, register flits_sent_next contains the number of flitting instances that packet filter unit sends to the interconnect matrix unit. However, flits_sent_next counts the header of the flits (which makes the register counter starts with 1 ). Hence, it overflows the maximum range of flits_sent_next by 1 . This bug (b20 in Table II) is security critical since it will send faulty information to the core and cause unintentional behavior.

b) GP2: All assignments should be operative in the RTL. To validate this property, RTL-ConTest selects all the assignments in the ${CFG}$ as targets to generate unique test cases. If no test case can be generated for an assignment in ${CFG}$ , it means that GP2 is violated. RTL-ConTest detects a violation of GP2 in the PULPissimo SoC core. It indicates that a Secure Mode trigger ${PULP}\_ {SEC}$ is not switchable as intended. In the RTL implementation, the value of register PULP_SEC is already defined to interrupt registers since it is predetermined with localparam ${PULP}\_ {SEC} = 1$ . Due to this definition, there is no assignment that changes the value of PULP_SEC, and RTL-ConTest will never be able to reach the target. This is a security-critical bug (b18) since this faulty design can cause unintended behavior in the core.

RTL-ConTest detects another violation of GP2 in the JTAG unit of the Ariane SoC. As shown in Listing 2, when the target is set to reach the line pass_check $= 1$ ’ ${b1}$ , RTL-ConTest returns an error that the target cannot be reached, due to an unassigned variable. The variable pass_hash needs to have the same value as hash to reach the target line. However, the variable pass_hash does not have a valid assignment to change its value. Hence, it is impossible to satisfy the condition hash $=  =$ pass_hash. This bug (b24) is critical since no access can be gained to the JTAG unit with this faulty password function.

c) GP3: The register address length should be equivalent to the assignment address length. In the RTL design, the assignments of each variable are usually demonstrated in bits, which create the possibility of mismatched address range. For example, the register data mentioned in GP1 is a 6-bit register; if it is assigned with a value such as 7 b1000101, it can result in an unintended error or misbehavior.

---

PassChkValid: begin

													if (hashValid) begin

																							if (hash == pass_hash) begin

																																			pass_check = 1'b1;

																									end else begin

																																			pass_check = 1'b0;

																									end

																									state_d = Idle;

													end else begin

																									state_d = PassChkValid;

											end

		end

---

Listing 2. Password check in Ariane SoC.

---

parameter PER_ADDR_WIDTH = 15,

parameter APB_ADDR_WIDTH = 32

	output logic [PER_ADDR_WIDTH-1:0] per_master_add_o,

input logic [APB_ADDR_WIDTH-1:0] PADDR,

		...

	assign per_master_add_o = PADDR;

---

Listing 3. Incorrect address range assignment.

RTL-ConTest detects a violation of GP3 in the APB Unit of PULPissimo SoC. Listing 3 shows that the address length of the peripheral master register per_master_add_o is declared as 15. However, it is assigned with PADDR, which is a 32-bit input. This can cause a security-critical bug (b10) in APB, or unintentional behavior and exception in the process.

RTL-ConTest detects another violation of GP3 when checking the RISC-V Peripherals unit of Ariane SoC. The address range of input register pmpcfg_i in RISC-V Peripheral is defined as input [16-1:0] pmpcfg_i. However, the system definition shows that its address range is defined as wire [8*16-1:0] pmpcfg. Hence, their address ranges do not match each other, causing a security-critical bug (b22), that results in memory aliasing in the PMP configuration register.

d) GP4: Registers should not have negative address lengths. In RTL, it is common to have an address range defined by local parameters. However, when the parameter is defined incorrectly by other modules, it might contain a negative decimal value, causing an incorrect register declaration. No violation of GP4 is detected during the simulation of PULPissimo SoC and Ariane SoC, implying none of the implementations of RTL modules have negative address range declaration or faulty local address range parameter. Hence, both the SoCs satisfy this security property.

e) GP5: Variables in assignments should have a valid definition and value. If the registers are assigned with undefined variables, it will result in data corruption. RTL-ConTest indicates a violation of GP5 when testing the JTAG interface in the Ariane SoC. With detailed investigation, it infers that the violation is caused by the variable dmi_resp.data, as it is neither assigned to any value nor has been defined in the module. This bug (b23) is security critical, since corrupted data can be assigned to the variable.

RTL-ConTest detected another violation of GP5 in the core of Ariane SoC. The invalidation message shows that the variable template_0 is not a valid register in the assignment template $0 = r$ since the type of template0is not declared in its unit. However, after manual inspection, we conclude that it is not a security vulnerability. The file containing this assignment is a Python preprocessor file for Verilog; thus, it should not be considered as a RTL in SoC. Hence, this can be regarded as a false positive detection.

2) Register Lock Properties (RLP): To prevent malicious use of resources, registers of some peripherals are locked behind register locks. These locks prevent complete access to the peripherals irrespective of the processor privilege levels. The following properties validate the security of these locks.

a) RLP1: The lock control register cannot be overwritten by a software. This property ensures that the user cannot manipulate lock control and gain access to the sensitive information. Generally, this property is implemented with an user interface. RTL-ConTest checks the validation of this property in the APB unit, AXI, JTAG Interface and Register Restore Unit of both Ariane and PULPissimo SoC. Except the APB unit of the PULPissimo SoC, others satisfy this property. RTL-ConTest detects that the APB GPIO lock control register r_gpio_lock has the same value as user input PWDATA, which violates RLP1. Hence, it engenders a security-critical bug (b8) that allows the software interrupt to manipulate the lock control of APB GPIO interface.

b) RLP2: The reset function should not erase the lock control register. Erasing the lock control register will provide the attacker the privilege to access or manipulate sensitive information. This property has been validated by APB GPIO unit, AXI unit, JTAG Interface, and Register Restore Unit modules with RTL-ConTest in both the SoCs. However, RTL-ConTest detects a violation of RLP2 in the APB GPIO unit of the PULPissimo SoC. The reset function assigns r_gpio_lock with 0 , which invalidates RLP2. This is a critical security bug (b9), since resetting r_gpio_lock will give an unintentional privilege of the GPIO control to the attacker.

c) RLP3: Only one peripheral should be active in an unit. In order to control the I/O flow, different address ranges for specific peripherals are defined to activate the intended functionality. This property checks whether there is any overlap in these address range definitions.

A violation of RLP3 is detected in the APB unit of the PULPissimo SoC. RTL-ConTest validates the input paddr_i by trying to solve (start address $<  =$ paddr_i $<  =$ end address) for each peripheral; in order to figure out whether there are values of paddr_i that satisfy multiple conditions simultaneously. During the simulation, a property invalidation message is shown indicating that more than one peripheral is activated with one input paddr_i, thereby indicating that address overlaps are manifested in the SoC, as shown in Listing 4. As per the specifications, GPIO's address is defined from 32'h1A10_1000 to 32'h1A10_AFFF, and SPI's address is defined from 32'h1A10_2000 to 32'h1A10_4FFF. Hence, the address range of SPI falls in the range of GPIO. Similarly, SoC control peripheral's address range is defined from 32'h1A10_4000 to 32'h1A10_4FFF, which overlaps with the address range of SPI, thus violating RLP3. These bugs (b1 and b2) are security critical since they will cause faulty transactions and instruction in the SoC.

---

	<table><tr><td colspan="2">...</td></tr><tr><td>define GPIO_START_ADDR</td><td>32′h1A10_1000</td></tr><tr><td>define GPIO_END_ADDR</td><td>32′h1A10_AFFF</td></tr><tr><td>'define SOC_CTRL_START_ADDR</td><td>32′h1A10_4000</td></tr><tr><td>define SOC_CTRL_END_ADDR ...</td><td>32′h1A10_4FFF</td></tr></table>

---

Listing 4. Overlap address definition.

d) RLP4: Cryptographic information should be protected by lock control register. Cryptographic information requires high privilege to access, and lock control is used to establish the privilege level and prevent the leakage of cryptographic information. RTL-ConTest tests this property in the multiplexer function unit (MUX) unit of AES, MD5, and SHA encryption modules in PULPissimo SoC and the HMAC unit of SHA encryption module in Ariane SoC. RTL-ConTest cannot find any violation in both SoCs. However, we discover with manual inspection that there is a violation in the MUX unit in PULPissimo SoC. The cryptographic key of the AES module is stored in an unprotected register $b$ . This result shows that RTL-ConTest could not recognize the difference between a protected memory and unprotected memory. Hence, it needs to be improved to be able to detect the security bug (b14) that involves architecture flaws.

3) Infrastructure Properties: These properties ensure that the execution process of a module follows the design specifications. We declare infrastructure properties for functionally identical modules in both the SoCs.

a) IP1: The JTAG module should be protected by a password, and the password check function should verify every bit of the password. This property ensures that the password check function is implemented and executed correctly. RTL-ConTest detects three violations of IP1 in the JTAG module of the Pulpissimo SoC. Two violations are manifested in the Advanced Debug Unit, and one violation in the JTAG Interface of the module.

As shown in Listing 1, in the Advanced Debug Unit, pass[bitindex] stores each bit of the 32-bit password. When the value of register ${tdi}\_ o$ matches pass[bitindex], bitindex is incremented by 1 . In the next cycle, the process is repeated for the next bit of the password. Once the value of the register correct is equal to or higher than32, h0000_001F, it indicates that the inputs have matched all bits of the password pass; and the process allows the user to pass the password check. Since the value of ${tdi}\_ o$ is assigned with user input ${tdi}\_ {pad}\_ i$ , RTL-ConTest analyzes the test patterns of tdi_pad_i to help determining the validation of password check function in the Advanced Debug Unit.

The first violation of IP1 in the Pulpissimo Advanced Debug Unit is detected when the test cases show that this unit requires only 31 correct input bits of generated tdi_pad_i to pass the password check. Although the password is of 32 bits, the register correct is set to check only up to 31 bit positions. Hence, it implies that the function will not check the last bit of the password. This bug (b4) is security critical since it allows the attacker to access the debug unit with an incorrect password. The second violation of IP1 is detected in same unit, as RTL-ConTest generates identical input values of tdi_pad_i for all the test cases to pass the password check. This observation indicates that the password is hardcoded. This bug (b6) creates a critical security vulnerability. Once the adversary obtains this password, she can easily pass the password check to gain unauthorized access to the debug unit.

---

	...

										missing if (rst == 1) c = 128′h0

												if (d[0] == 1'b1) c = aes_out;

										else if (d[1] == 1'b1) c = sha_out;

										else if (d[2] == 1'b1) c = md5_out;

											else if(d[3] == 1'b1) c = temperature_out;

										else $c = {128}^{\prime } +$ h0000_0000_0000_0000_0000_0000_0000;

...

---

Listing 5. Output in MUX unit.

RTL-ConTest detects another violation of IP1 in the JTAG Interface of the Pulpissimo SoC. The 1-bit test data input $t{d}_{ - }i$ acts as the input to the password check function. However, RTL-ConTest cannot generate any valid test cases for $t{d}_{ - }i$ . Hence, it is inferred that the password check function is not implemented in RTL. This violates IP1, as the JTAG interface is accessible without any password, thereby generating a security critical bug (b12).

b) IP2: The reset function should clear the function information registers. This property is set to check whether these registers are reset to their initial values and disable an adversarial access to sensitive information. RTL-ConTest returns an invalidation message that IP2 is violated in the JTAG Unit of Ariane SoC. This indicates that the reset does not clear the value of pass_check, inferring an incorrect operation.

Violation of IP2 in the JTAG Debug Unit of Pulpissimo SoC is also detected by RTL-Contest. When the debug unit resets the password check function, the value of the correct password input counter correct does not reset to 0 . Hence, even after the input fails the password check, it will still assign passchk with 1 and continue providing the user access to the debug interface. These bugs (b25 and b5) cause very critical security exploits. Anytime the process passes the password check, it can access the JTAG debug unit and, hence, gain access to the SoCs internal signals without a correct password.

In the multiplexer function unit (MFU) of the PULPissimo SoC, the input reset signal rst is supposed to erase the values of MAC address, which controls the output from aes_out, sha_out, and md5_out. However, the test cases generated by RTL-ConTest show when input rst is 1 , the MFU does not erase the value of MAC address aes_out, and the output register $c$ is not equal to 0 . As shown in Listing 5, the reset function for the MAC address is not implemented in the RTL (b13); thereby violating IP2.

c) IP3: Timing calculation should operate in the correct logic. For example, time calculation functions of Pulpissimo SoC are designed to reset the seconds register $r$ _seconds when the value of $r$ _seconds is equal to or larger than 59, and simultaneously increases the value of minutes register r_minute by 1.

---

unique case (debug_addr_i[13:8])

...

																6'b10_0000: begin

																							debug_gnt_o = 1'b1;

																									\\\\Missing if (debug_halt == 1)

																											rdata_sel_n = RD_DBGS;

													end

			...

---

Listing 6. Unintentional register access.

---

csr_restore_mret_i: begin //MRET

												unique case (mstatus_q.mpp)

																								...

																										PRIV_LVL_M: begin

																																					mstatus_n.mie = mstatus_q.mpie;

																																				priv_lvl_n = PRIV_LVL_M;

																																				mstatus_n.mpie = 1'b1;

																																				mstatus_n.mpp = PRIV_LVL_U;

																										end

---

Listing 7. Code for privilege change in Pulpissimo SoC.

A violation of IP3 is detected in RTC module of Pulpissimo SoC by RTL-ConTest. The trigger value for $r$ _seconds is defined as 8 , h59 in hexadecimal, which is 89 in decimal, not the desired 59. Therefore, when the value of $r$ _seconds is higher than 60, the maximum value of second, this property is violated. This bug (b7) results in an inaccurate time calculation logic, which can cause security-critical errors.

d) IP4: Access to certain information should be in the correct privilege level. For example, the Debug Register can be only accessible to the Data Register when the process is in debug mode. Violation of this property indicates that privilege check of the unit is not correctly implemented.

An invalidation message, obtained from RTL-ConTest, is shown when validating the Control Store Register Unit in the Pulpissimo SoC. IP4 is not satisfied since the debug register ${RD}\_ {DBGS}$ is also accessible even when the core is not in debug halt mode, as shown in Listing 6. This bug (b16) is security critical, which will allow the attacker to access sensitive information even when the Control Store Register unit is not in the debug mode.

e) IP5: The status and privilege level of core units should be updated correctly. An incorrect update will cause error and unintentional behavior in the core. RTL-ConTest detects a violation of IP5 in the control and status register (CSR) of the Pulpissimo core. Listing 7 shows that when CSR restores Master privilege-level status, the privilege-level register priv_lvl_n is assigned as Master privilege level. However, privilege status register mstatus_n.mpp is assigned incorrectly as User privilege level in RTL. This bug (b17) is security critical since the core will represent an incorrect privilege status when it is in the Master privilege level.

Moreover, when simulating the AXI address decoder of the PULPissimo SoC for IP5, RTL-ConTest shows an invalidation message. It indicates that the current status register ${CS}$ and the next status register ${NS}$ do not have the same value after updating the status. This means that the AXI unit does not update the operation information correctly, thereby violating IP5. This is demonstrated in Listing 8. his bug (b11) will cause AXI address decoder to ignore the errors and create critical security exceptions in the core.

---

	case (CS)

...

														ERROR:

																								begin

																																						...

																																						if (outstanding_trans_i)

																																					begin

																																																		NS = OPERATIVE;

																																						end

---

Listing 8. AXI error update statement.

---

always_comb begin : exception_handling

														...

														if (commit_instr_i[0].valid) begin

																								if (csr_exception_i.valid && !amo_valid_commit_o)

																						begin

																																			...

																									end

...

---

Listing 9. Incorrect CSR exception update.

f) IP6: Encryption modules should not be integrated with regular modules. This property prevents regular modules from being implemented with the crytographic modules, avoiding over-protection on regular modules and unintentional leakage of sensitive information. RTL-ConTest checks this property in the MUX unit of PULPissimo SoC and the HMAC unit of Ariane SoC. It does not detect any violation during the simulation. However, our manual inspection discovers that the temperature module is integrated in the MUX unit of PULPissimo SoC along with the cryptographic modules, engendering bug b15. This test result shows that RTL-ConTest is still not capable to identify the differences between sensitive output and regular output. Hence, it needs improvement to verify that a security-critical infrastructure is not implemented with regular ones.

g) IP7: Exception of the CSR unit should be checked and handled. During the process of assigning control instructions, the information flows occurring in the commit stage unit or from other units might raise exceptions. These exceptions need to be checked and handled in the controller unit so that they will not cause unintentional behaviors. RTL-ConTest detects violations of IP7 in the commit stage unit of the Ariane SoC. The exception of CSR unit csr_exception_i is not correctly updated to the exception output execption_o. Listing 9 shows that the update assignment is guarded by an incorrect event trigger. Since atomic memory operation (AMO) does not participate in the CSR exception handling, the register of a valid AMO operation amo_valid_commit_o should not be in the if condition. This bug (b21) is security critical, as exception from the CSR unit might not be handled by the controller unit. and will cause unintentional behavior in the core.

h) IP8: Least significant bit (LSB) of the instruction register should be assigned with a specific value. Incorrect instruction assignment to the LSB will cause unintentional behavior in SoCs. A violation of IP8 is detected in the decoder unit of PULPissimo SoC after updating the LSB of jump target address register jump_target. According to the specification, the LSB of the jump_target is supposed to be set to 0 . However, no assignment is implemented to complete this operation, leaving the LSB of jump_target as 1 . Therefore, the implemented code does not validate the property, resulting in a faulty address update. This bug (b19) is security critical since the attacker could leverage it to redirect the program counter without being detected.

RTL-ConTest detects another violation of IP8 in the JTAG interface of Ariane SoC. The ID code instruction register next_idcode is supposed to store the JTAG data instruction jtag_datain in the LSB of the next_idcode. However, the jtag_datain is stored in the MSB of next_idcode. This bug (b26) will cause a faulty instruction output of the ID code, and further misbehavior.

## VI. CONCLUSION

This article proposed RTL-ConTest, a scalable hardware vulnerability detection technique for detecting security exploits directly in the RTL design of an SoC. Starting from an RTL-level description of an SoC, RTL-ConTest generates the ${CFG}$ of each SoC component (including the core and the peripherals) and performs concolic testing to detect the vulnerabilities against prespecified security properties. The security properties engender security restrictions, such as condition check and boundary check, which aid the RTL-ConTest framework in detecting runtime violations. When evaluated using two state-of-the-art RISC-V-based SoCs, RTL-ConTest successfully detected seven security bugs in core units, and 17 intermodular bugs manifested outside the cores. The tool can finish each bug detection in around ${10}\mathrm{\;s}$ on average. Overall, PULpissimo SoC took approximately $1\mathrm{\;h}$ and ${40}\mathrm{\;{min}}$ to complete, and Ariane SoC took approximately $1\mathrm{\;h}$ and ${20}\mathrm{\;{min}}$ to complete. Thereby, RTL-Contest demonstrated its scalability and efficiency over the traditional SoC security vulnerability detection strategies.

RTL-ConTest can be used by a design house to be incorporated into the verification flow at the design review stage of the SDL of the SoC, as shown in Fig. 1. This tool will identify security vulnerabilities in the RTL design before proceeding to the next steps, such as logic synthesis, placement, and routing. At present, RTL-ConTest can generate test cases for each RTL component in an SoC design. Manual intervention is required to review the results of these test cases against the security properties and locate the buggy lines directly in the Verilog/SystemVerilog files. In the future, we intend to include automated test case validation into the tool by creating test case references from security property generation. Moreover, RTL-ConTest falls short in identifying the differences between a cryptographic module and a regular module. In the future, we will develop an efficient technique to provide a flow map of the whole SoC design as input to RTL-ConTest, which connects all RTL modules together. This would significantly improve the systematic analysis of the test cases across the various modules of the SoC. Furthermore, in addition to the proposed technique, we still need more effort in developing detailed security features for each SoC. Thereby, we can identify undetected vulnerabilities by improving and refining the process of security property generation. REFERENCES

[1] N. Farzana, F. Rahman, M. M. Tehranipoor, and F. Farahmandi, "SoC security verification using property checking," in Proc. ITC. 2019, pp. 1-10.

[2] G. Dessouky et al., "HardFails: Insights into software-exploitable hardware bugs," in Proc. USENIX Security Symp., 2019, pp. 213-230.

[3] CWE-CWE-1194: Hardware Design (4.0). Accessed: May 15, 2020. [Online]. Available: https://cwe.mitre.org/data/definitions/1194.html

[4] Secure Development Lifecycle. Accessed: May 16, 2020. [Online]. Available: https://www.cisco.com/c/dam/en_us/about/doing_business/ trust-center/docs/cisco-secure-development-lifecycle.pdf

[5] Navigating a Product Security Journey. Accessed: May 16, 2020. [Online]. Available: https://www.lenovo.com/content/dam/lenovo/pcsd/ north-america/en/solutions/government/brochures/na-brochure-security-by-design.pdf

[6] S. L. He et al., "Model of the product development lifecycle," Sandia Nat. Lab., Albuquerque, NM, USA, Rep. SAND2015-9022, 2015.

[7] A. Cimatti et al., "NuSMV 2: An OpenSource tool for symbolic model checking," in Proc. CAV. 2002, pp. 359-364.

[8] P. Godefroid, N. Klarlund, and K. Sen, "DART: Directed automated random testing," in Proc. ACM SIGPLAN PLDI, 2005, pp. 213-223.

[9] W. Snyder, "Verilator and systemperl," in Proc. North Amer. SystemC Users' Group DAC, 2004.

[10] R. Zhang, C. Deutschbein, P. Huang, and C. Sturton, "End-to-end automated exploit generation for validating the security of processor designs," in Proc. MICRO. 2018, pp. 815-827.

[11] K. Sen, D. Marinov, and G. Agha, "Cute: A concolic unit testing engine for C," ACM SIGSOFT Softw. Eng. Notes, vol. 30, no. 5, pp. 263-272, 2005.

[12] L. Liu and S. Vasudevan, "STAR: Generating input vectors for design validation by static analysis of RTL," in Proc. HLDVT, 2009, pp. 32-37.

[13] A. Ahmed, F. Farahmandi, and P. Mishra, "Directed test generation using concolic testing on RTL models," in Proc. DATE, 2018, pp. 1538-1543.

[14] G. E. Suh, J. W. Lee, D. Zhang, and S. Devadas, "Secure program execution via dynamic information flow tracking," ACM SIGPLAN Notices, vol. 39, no. 11, pp. 85-96, 2004.

[15] M. Tiwari, H. M. G. Wassel, B. Mazloom, S. Mysore, F. T. Chong, and T. Sherwood, "Complete information flow tracking from the gates up," in Proc. ASPLOS, 2009, pp. 109-120.

[16] J. Oberg, W. Hu, A. Irturk, M. Tiwari, T. Sherwood, and R. Kastner, "Theoretical analysis of gate level information flow tracking," in Proc. ${DAC}$ . 2010, pp. 244-247.

[17] A. Ardeshiricham, W. Hu, J. Marxen, and R. Kastner, "Register transfer level information flow tracking for provably secure hardware design," in Proc. DATE, 2017, pp. 1691-1696.

[18] M. Qin, X. Wang, B. Mao, D. Mu, and W. Hu, "A formal model for proving hardware timing properties and identifying timing channels," Integration, vol. 72, pp. 123-133, May 2020.

[19] A. Ardeshiricham, Y. Takashima, S. Gao, and R. Kastner, "VeriSketch: Synthesizing secure hardware designs with timing-sensitive information flow properties," in Proc. ACM SIGSAC CCS, 2019, pp. 1623-1638.

[20] M. Bilzor, T. Huffmire, C. E. Irvine, and T. E. Levin, "Evaluating security requirements in a general-purpose processor by combining assertion checkers with code coverage," in Proc. HOST, 2012, pp. 49-54.

[21] M. Hicks, C. Sturton, S. T. King, and J. M. Smith, "SPECS: A lightweight runtime mechanism for protecting software from security-critical processor bugs," in Proc. ASPLOS, 2015, pp. 517-529.

[22] R. Zhang and C. Sturton, "Transys: Leveraging common security properties across hardware designs," in Proc. IEEE Symp. Security Privacy, 2020, pp. 1713-1727.

[23] W. Hu, A. Ardeshiricham, M. S. Gobulukoglu, X. Wang, and R. Kastner, "Property specific information flow analysis for hardware security verification," in Proc. ICCAD, 2018, p. 89.

[24] R. Zhang and C. Sturton, "A recursive strategy for symbolic execution to find exploits in hardware designs," in Proc. ACM SIGPLAN FSM, 2018, pp. 1-9.

[25] Y. Lyu and P. Mishra, "Automated test generation for activation of assertions in RTL models," in Proc. ASP-DAC, 2020, pp. 223-228.

[26] C. Cadar, D. Dunbar, and D. R. Engler, "KLEE: Unassisted and automatic generation of high-coverage tests for complex systems programs," in Proc. OSDI, vol. 8, 2008, pp. 209-224.

[27] L. Shen et al., "Symbolic execution based test-patterns generation algorithm for hardware Trojan detection," Comput. Security, vol. 78, pp. 267-280, Sep. 2018.

[28] VerifOx-A Tool for Path-Wise Symbolic Execution. Accessed: Aug. 27, 2020. [Online]. Available: http://www.cprover.org/verifox/

[29] GitHub-PFNet-Research/ATPG4SV: A Prototype of Concolic Testing Engine for SystemVerilog, Developed as Part of PFN Summer Internship 2018. Accessed: Aug. 27, 2020. [Online]. Available: https://github.com/pfnet-research/ATPG4SV

[30] X. Qin and P. Mishra, "Scalable test generation by interleaving concrete and symbolic execution," in Proc. VLSID ICESS, 2014, pp. 104-109.

[31] A. Kölbi, J. H. Kukula, and R. F. Damiano, "Symbolic RTL simulation," in Proc. DAC. 2001, pp. 47-52.

[32] Y. Zhang, W. Feng, and M. Huang, "Automatic generation of high-coverage tests for RTL designs using software techniques and tools," in Proc. ICIEA, 2016, pp. 856-861.

[33] GitHub: Seth-Lab-Tamu/HackDac-2018-SoC: Buggy SoC Used for the Second Phase of the Hack@DAC 2018 Hardware Security Competition. Accessed: May 15, 2020. [Online]. Available: https://github.com/seth-lab-tamu/hackdac-2018-soc

[34] GitHub-PrincetonUniversity/OpenPiton: The OpenPiton Platform. Accessed: Aug. 23, 2020. [Online]. Available: https://github.com/PrincetonUniversity/openpiton

[35] Cadence. Jaspergold Formal Verification Platform (Apps). Accessed: May 15, 2020. [Online]. Available: https:// www.cadence.com/en_US/home/tools/system-design-and-verification/ formal-and-static-verification/jasper-gold-verification-platform.html

[36] D. Wang et al., "Formal property verification by abstraction refinement with formal, simulation and hybrid engines," in Proc. DAC, 2001, pp. 35-40.

Xingyu Meng (Student Member, IEEE) is currently pursuing the Doctoral degree with the Department of Electrical and Computer Engineering, University of Texas at Dallas, Richardson, TX, USA.

Shamik Kundu (Student Member, IEEE) is currently pursuing the Doctoral degree with the Department of Electrical and Computer Engineering, University of Texas at Dallas, Richardson, TX, USA.

Arun K. Kanuparthi (Member, IEEE) received the Ph.D. degree in electrical engineering from New York University, New York, NY, USA, in 2013.

He is an Offensive Security Researcher with Intel Corporation, Hillsboro, OR, USA. He leads the offensive security research efforts on Intel's datacenter networking and communication products.

Kanad Basu (Senior Member, IEEE) received the Ph.D. degree from the Department of Computer and Information Science and Engineering, University of Florida, Gainesville, FL, USA, in 2012.

He is an Assistant Professor with the Electrical and Computer Engineering Department, University of Texas at Dallas, Richardson, TX, USA.