# FormalFuzzer: Formal Verification Assisted Fuzz Testing for SoC Vulnerability Detection

Nusrat Farzana Dipu, Muhammad Monir Hossain, Kimia Zamiri Azar, Farimah Farahmandi, and Mark Tehranipoor Electrical and Computer Engineering, University of Florida, Gainesville, Florida 32611, USA

\{ndipu, hossainm, k.zamiriazar\}@ufl.edu, \{farimah, tehranipoor\}@ece.ufl.edu

Abstract-Modern Systems-on-Chips (SoCs) integrate numerous insecure intellectual properties to meet design-cost and time-to-market constraints. Incorporating these SoCs into security-critical systems severely threatens users' privacy. Traditional formal/simulation-based verification techniques detect vulnerabilities to some extent. However, these approaches face challenges in detecting unknown vulnerabilities and suffer from significant manual efforts, false alarms, low coverage, and scalability. Several fuzzing techniques have been developed to mitigate pre-silicon hardware verification limitations. Nevertheless, these techniques suffer from major challenges such as slow simulation platforms, extensive design knowledge requirements, and lacking consideration of untrusted inter-module communications. To overcome these shortcomings, we developed FormalFuzzer, an emulation-based hybrid framework by combining formal verification and fuzz testing, leveraging their own benefits. FormalFuzzer incorporates formal-verification-based pre-processing using template-based assertion generation to narrow down the search space for fuzz testing and appropriate mutation strategy selection by dynamic feedback derived from a security-oriented cost function. The cost function is developed using vulnerability databases and specifications, indicating the likelihood of triggering a vulnerability. A vulnerability is detected when the cost function reaches global or local minima. Our experiments on RISC-V-based Ariane SoC demonstrate the efficiency of proposed formal-verification-based pre-processing strategies and cost function-driven feedback on fuzzing in detecting both known and unknown vulnerabilities expeditiously.

Index Terms-SoC, Fuzzing, Cost Function, Formal Method

## I. INTRODUCTION

System-on-Chips (SoCs) are being introduced every year with strict time-to-market constraints, resulting in a shorter verification time-frame. Multi-source intellectual properties (IPs) are integrated into a single SoC to satisfy user requirements. These third-party IPs (3PIPs) may lack trustworthiness and potentially harbor malicious functionality that could adversely affect SoC's security. These IPs may also expose new vulnerabilities and compromise trusted IP security [1]. However, assurance that all IPs can effectively collaborate and support SoC's secure functioning, requires rigorous testing to detect any malicious activities associated with 3PIPs.

SoCs may face confidentiality, integrity, or availability (CIA triad) threats due to untrusted 3PIPs, insecure design transformations, and malicious design modifications. Security countermeasures should be implemented at each abstraction in the design life-cycle upon detecting vulnerabilities at earlier silicon stages (pre-silicon) to prevent higher costs of bug-fixing at a later stage [2]. To detect hardware (HW) vulnerabilities, numerous verification approaches have been developed, such as formal-verification [2]-[4] and information flow tracking (IFT) [5], [6]. The formal method verifies the design by manually writing security properties. More properties result in more coverage, but they are time-consuming and error-prone. Consequently, these methods fail to scale and deal with state-space explosion on large and complex SoCs. Moreover, IFT involves manual taint labeling, which may be inaccurate and requires technical expertise regarding the SoC's design. Taint tracking/propagation logic increases design overhead. False positives are generated as unnecessary signals are tainted (over-tainting), or false negatives due to necessary signals being ignored (under-tainting). These traditional techniques use white-box verification models, which require extensive design knowledge and are mostly not applicable to 3PIPs.

An emerging technique called HW fuzzing [7]-[15] has proven efficiency in identifying HW vulnerabilities. An HW fuzzer generates inputs through mutations of previously interesting inputs and applies them to HW modules or SoCs to uncover unknown behavior. SoCs can be verified with fuzz testing during both pre-silicon and post-silicon stages, making it an interactive verification technique. Several frameworks [7]- [12] that focus on HW-oriented fuzz testing have attempted to use software(SW)-centric fuzzing engines like AFL [16] to uncover HW-specific vulnerabilities. There are, however, some notable limitations to these approaches: (1) utilization of SW-based coverage metrics overlooks a significant portion of IPs and SoCs, (2) adding more randomness in mutations and increased input space (irrelevant to targeted HW), and (3) lacking security-driven mutation and dynamic feedback [7]- [12]. To overcome some of these limitations, a hybrid HW fuzzer [14] has been developed inspired by the success of hybrid SW fuzzers, such as [17], [18]. This approach utilizes formal verification to target hard-to-reach design spaces. However, this simulation-based approach prolongs verification time and demands significant design knowledge (white-box model). These limitations impede vulnerability detection in the HW, necessitating experts' participation in this research domain.

This paper proposes a hybrid technique, FormalFuzzer, to address the aforementioned challenges, which extends the widely used AFL fuzzing engine for SoC security verification. FormalFuzzer utilizes formal verification (as pre-processing targeting a module under verification) to narrow down the fuzzing search space. An FPGA-based emulation platform powers FormalFuzzer, reducing the overall verification time. Inspired by SoCFuzzer [13], we establish the same type of security cost function in the foundation of FormalFuzzer to guide the fuzzing process toward unexplored parts of the design in relation to security properties. The performance of FormalFuzzer has been tested with real HW vulnerabilities obtained from various Hack@ DAC [19], [20], using a large RISC-V SoC platform. Our contributions are:

(1) Developing a formal verification-assisted pre-processing technique using security assertion templates to reduce input space before starting fuzz testing.

(2) Developing a mutation engine for efficient fuzzing for detecting SoC security vulnerabilities utilizing postulations from pre-processing step and cost-function enabled feedback. (3) Analyzing the performance of the proposed framework in detecting vulnerabilities using a real SoC.

## II. BACKGROUND AND PRIOR ARTS

SoCs may contain critical information worth protecting from adversaries, which are called security assets [2]. These assets may be leaked due to unintentional design bugs or intentionally inserted malicious logic [1]. For security assurance, identifying leak sources is mandatory, which can be accomplished by defining and evaluating specific design behaviors, also known as security properties. The CIA-triac (confidentiality, integrity, and availability) of SoC components can be assured by a comprehensive assessment of security properties [2]. These properties can be verified using a variety of security validation approaches, including formal methods [2]-[4], [21], [22]. However, different formal verification techniques have their own limitations. For example, symbolic execution-based formal approach is susceptible to scalability issues as it deals with a large set of constraints [3], [22] for verifying the properties. Moreover, as a prerequisite to property-driven or model-checking-based formal verification, security properties need to be explicitly represented as assertions to feed into tools that require knowledge of the subject matter and source code (white-box) [2], [21].

Fuzzing or fuzz testing can mitigate some limitations of formal techniques. For example, fuzz testing supports both black and gray-box model verification, unlike formal methods. Fuzzing initially inputs seed(s) from the user and mutates randomized inputs using run-time HW behavior-based feedback. The mutated inputs are fed to the design under test and discrepancies in the output are monitored to detect bugs and vulnerabilities. With recent successes in SW verification, few studies have explored applying SW fuzzing to HW (RTL) [7]-[12]. These frameworks have significant limitations: 1) the feedback system relies on SW-based metrics lacking a correlation between fuzzer assertions and HW-oriented errors and 2) lacking a self-evolving cost-function formulation to offer a better sense of proximity to the vulnerability. To mitigate the existing shortcomings to some extent, the authors in [13], [15] developed fuzzing-based frameworks by introducing a novel cost function and feedback system for detecting SoC security vulnerabilities. However, fuzzing limits solving path constraints due to its random nature. To compensate for individual limitations, formal methods have been combined with fuzzing to bring tremendous success in SW verification [17], [18], [23], [24], which has largely not been investigated for HW verification. The authors in [25] introduced symbolic simulation enhanced coverage directed dynamic verification, which enables the ability to trace and get feedback. However, this approach focuses on test generation for detecting HW faults rather than security vulnerabilities. The authors in [14] proposed HyPFuzz framework that dynamically links the formal approach with fuzzing for higher and faster coverage. Unfortunately, these HW fuzzing frameworks require rigorous design knowledge and lengthen verification time due to simulation.

![0196f2c0-a9e6-7e3d-99cb-9f132459e245_1_916_143_743_284_0.jpg](images/0196f2c0-a9e6-7e3d-99cb-9f132459e245_1_916_143_743_284_0.jpg)

Fig. 1: An Overview of FormalFuzzer Framework.

## III. Proposed Framework: FormalFuzzer

## A. Overview

We present FormalFuzzer, a hybrid framework that utilizes property-driven formal verification to reduce the input space significantly before starting fuzzing for triggering vulnerabilities faster. FormalFuzzer uses an existing cost function-enabled fuzzing strategy and feedback [13] to dynamically tune the mutation strategies for efficient fuzzing. Fig. 1 illustrates the framework's high-level overview. Before fuzzing an SoC module, FormalFuzzer uses template-based assertion generation technique to convert pre-defined security properties into assertions [26]. The cover scenario or violation obtained from assertion checking is used as a high-level inquiry (postulations derived) for covering all corner cases. Hence, it searches for malicious behaviors in the surrounding regions related to the scenario or violation(s) in the SoC module. Moreover, FormalFuzzer relies on FPGA-based emulation with real-time monitoring of internal HW signals on the host machine to estimate the cost function (based on a set of security properties). FormalFuzzer executes program(s) with fuzzed inputs on the SoC's CPU, targeting a specific module for vulnerability detection at a time. Initially, FormalFuzzer accepts a seed from which mutation initiates. During runtime, FormalFuzzer tracks interesting HW behavior and tunes the mutation strategy based on the cost function, which indicates a vulnerability trigger upon reaching a minimum value. To summarize, in this hybrid approach, the formal method discovers the generic scope of potential vulnerabilities in preprocessing, while the fuzzer targets specific cases dynamically. Following a gray-box model, FormalFuzzer assumes limited design knowledge from the verification engineer, without a golden model.

TABLE I: System Verilog Assertion Template.

<table><tr><td>Template</td><td>Event</td><td>Template Description</td></tr><tr><td>${ATI}$</td><td>G(E)</td><td>Always: Event $\mathrm{E}$ will always be true</td></tr><tr><td>AT2</td><td>$\mathrm{A} \mid   \rightarrow  \mathrm{C}$</td><td>Immediate: Event A will immediately cause event C</td></tr><tr><td>${AT3}$</td><td>$\mathrm{A} \mid   \rightarrow  \mathrm{F}\left( \mathrm{C}\right)$</td><td>Eventual: If $A$ happens, then $C$ will happen eventually</td></tr><tr><td>${AT4}$</td><td>$\mathrm{A} \mid   \rightarrow  \mathrm{U}\left( \mathrm{C}\right)$</td><td>Until: Event A holds until Event C happens</td></tr><tr><td>${AT5}$</td><td>$\mathrm{A} \mid   \rightarrow  \mathrm{N}\left( \mathrm{C}\right)$</td><td>Next: If A happens, then $\mathrm{C}$ will happen in next specific clock cycles</td></tr></table>

## B. Pre-processing Using Formal Verification

① Defining Security Properties: Before verifying a particular module, we develop security properties that FormalFuzzer verifies by keeping gray-box verification in mind and leveraging the module's external behaviors. The security properties can be defined from the prior knowledge of design specification, security assets to protect, known security vulnerabilities, attacker's accessibility to the security asset and corresponding attack examples as described in [2].

② Assertion Template Library: To verify compliance of the security properties with the design under verification (DUV), FormalFuzzer translates the properties into formal assertions by relying on the concept of developing assertion templates (AT)[26]. In this paper, we develop an event-based template library for generating assertions using the System Verilog Assertion (SVA) language in which a set of events(E) are associated with a security property or design behavior (Examples are provided in Table I). An ${AT}$ is comprised of a set of events (i.e., a statement which can only be true or false) which are connected through logical connectives (e.g., (~ not), (& & and), (\\ or), etc.) and/or temporal operators (e.g., always (G), eventually (F), next(X), etc) [26]. In addition, if one event leads to another, we denote the causal event as $A$ ntecedent(A)and the resulting event as Consequent(C)and connect them with the implication $\left(  \rightarrow  \right)$ operator. These ${AT}$ s serve as a generic representation of the property under consideration and can be instantiated multiple times with different signals or conditions to check various aspects of the design's behavior.

③ Identification of Interface Signals: As the security properties are based on the gray-box knowledge of the DUV, to instantiate the associated ${AT}\left( \mathrm{\;s}\right)$ , we need to identify corresponding interface signals. However, an IP module can have hundreds of interface signals [27] depending on various factors, such as the IP's functionality and complexity, the number of features it supports, and the level of configurability and flexibility it provides. Depending on their functionality, interface signals are divided into (i) Data signal: Input and/or output signals that are used for transferring data between the IP module and other parts of the design; (ii) Control signal: Control signals govern the IP module's behavior and operation by enabling or disabling some functions, selecting particular operations, specific parameters, or controlling internal states; (iii) Clock and Reset Signals: IP modules typically require clock and reset signals to synchronize and initialize their internal operations; and (iv) Status and Interrupt Signals: IP modules often provide status signals to indicate their current state and interrupt signals to notify the rest of the design of specific events or conditions.

![0196f2c0-a9e6-7e3d-99cb-9f132459e245_2_147_1863_742_208_0.jpg](images/0196f2c0-a9e6-7e3d-99cb-9f132459e245_2_147_1863_742_208_0.jpg)

Fig. 2: Example of Interface Signals.

Algorithm 1: Interface Signals Identification.

---

Input: Output signal, ${S}_{o}$ ; Design $D$

Output: List of control ${\mathbb{C}}_{\mathrm{i}}$ and data ${\mathbb{D}}_{\mathrm{i}}$ signals

initialize ${\mathbb{S}}_{\mathrm{i}}$

${\mathbb{S}}_{\mathrm{i}} \leftarrow  \operatorname{Fan}\operatorname{In}\left( {{S}_{o}, D}\right)$

while ${S}_{i} \in  {\mathbb{S}}_{i}$ do

	/* Information flow analysis

		between ${S}_{i}\& {S}_{o}\; * /$

	$I \leftarrow  {IFA}\left( {{S}_{i},{S}_{o}}\right) / \star$ Store IFA results

		*/

	if $I \Rightarrow$ ExplicitFlow then

		${\mathbb{D}}_{\mathrm{i}} \leftarrow  {\mathbb{D}}_{\mathrm{i}} + {S}_{i}$

	else

		${\mathbb{C}}_{\mathrm{i}} \leftarrow  {\mathbb{C}}_{\mathrm{i}} + {S}_{i}$

out ${\mathbb{C}}_{\mathrm{i}},{\mathbb{D}}_{\mathrm{i}}$

---

Although some of the input signal's functionality can be easily identified using the design specification or naming conventions (e.g., clock, reset etc.), sometimes the distinction is subjective or context-dependent. By analyzing the connections and inter-actions of an input signal with other components of the modules, we can specify whether the signal is a control or data signals. As shown in Fig. 2 control or control-like signals are involved with specific control paths and dependencies within the design, while data or data-like signals are involved in data paths. Algorithm 1 shows how to evaluate these features. The Algorithm takes the output signal ${S}_{o}$ , correspondent to the property and RTL design $D$ as inputs and returns lists of input control ${\mathbb{C}}_{\mathrm{i}}$ and ${\mathbb{D}}_{\mathrm{i}}$ signals. At first, Algorithm 1 performs backward structural analysis [28] to find out and enlist all primary inputs as ${\mathbb{S}}_{\mathrm{i}}$ in the fan-in cone of ${S}_{o}$ (Lines 1-2). Then for each input signal ${S}_{i}$ in ${\mathbb{S}}_{\mathrm{i}}$ , information flow analysis (IFA) [29] is performed to check the information propagation between ${S}_{i}$ and ${S}_{o}$ (Line 5). If the information flow is explicit, then ${S}_{i}$ is enlisted as a data signal otherwise as a control signal (Lines 6-9).

④ Assertion Generation and Verification: By instantiating the corresponding ${AT}$ with identified interface signals, the security assertions are generated. To check the design against the generated assertions, the FormalFuzzer applies formal verification tools, such as model checkers or property checkers [2]. These tools will explore the design's state space and verify whether the properties hold and provide necessary cover traces or if there are counterexamples that violate the properties. ⑤ Postulations Extraction: From the cover traces of proven assertions or counter-example of violated assertions, we derive some postulations for guiding the mutation engine. These postulations are interpreted using SoC instruction set architecture (ISA) and design specifications. Based on the postulations, the verification engineer specifies the input fields to the mutation engine that should be mutated (or not) in fuzzing.

## C. Cost Function and Feedback

In terms of fuzzing, FormalFuzzer mutates test inputs that are more intelligent than random ones, within the input space guided by the postulations via applying mutation strategies such as bit-flipping, shift operation, permutation, etc. Formal-Fuzzer also aims to improve coverage and convergence using an efficient cost function-based feedback mechanism proposed in [13]. This feedback assists in generating intelligent inputs that align with specific security properties and run-time HW statuses. The normalized cost function is formulated as Eq. 1.

$$
{F}_{c} = 1 - \frac{1}{n}\left\lbrack  {\left\{  {1 - \frac{2}{r\left( {r - 1}\right) }\mathop{\sum }\limits_{{j = 1}}^{{r - 1}}\mathop{\sum }\limits_{{k = j + 1}}^{r}\frac{h\left( {{i}_{j},{i}_{k}}\right) }{{l}_{i}}}\right\}   + \frac{{u}_{i}}{{2}^{N - d}}}\right.
$$

$$
\left. {+\frac{\sigma }{\mathop{\sum }\limits_{{j = 1}}^{z}{l}_{z}}\mathop{\sum }\limits_{{k = 1}}^{z}\left( {h\left( {{o}_{r - 1, k},{o}_{r, k}}\right) }\right)  + \frac{1}{m}\mathop{\sum }\limits_{{k = 1}}^{m}\left( {{o}_{r, k} =  = {o}_{t, k}}\right) }\right\rbrack
$$

(1)

Where, $r$ : #fuzzed inputs, ${i}_{j}$ : mutated input value for ${j}^{\text{th }}$ run, ${i}_{k}$ : mutated input value for ${k}^{th}$ run, ${l}_{i}$ : input length in bits, $h\left( {{i}_{j},{i}_{k}}\right)$ : hamming distance (HD) of ${j}^{th}$ and ${k}^{th}$ fuzzed inputs, ${u}_{i}$ : #unique fuzzed inputs, $d$ : #fixed bits (exclusion), $N$ : #input bits of SoC module under verification, $\sigma  \in  \{ 0,1\}$ , $z$ : #output signals, ${l}_{z}$ : length of ${z}^{\text{th }}$ output signal (bits), $h\left( {{o}_{r - 1, k},{o}_{r, k}}\right)$ : HD of output ${k}^{th}$ values between ${r}^{th}$ and ${\left( r - 1\right) }^{th}$ iteration, $m$ : #target output signals/behavior, ${o}_{t, k}$ : expected value/behavior of ${k}^{th}$ target output signal, and ${o}_{r, k}$ : runtime observed value or behavior of ${k}^{th}$ target output signal.

The second term in Eq. 1 measures the quality of mutated inputs by determining randomness (less indicates better-crafted inputs), while the third term measures unique input coverage. The fourth term measures how mutated inputs affect output changes (to avoid stuck or idle states). The last term quantifies malicious behavior or asset leakage. To develop the normalized cost function, these terms are normalized and subtracted from 1. A decreasing value in ${F}_{c}$ indicates approaching the vulnerability-triggering condition gradually. Finally, when ${F}_{c}$ reaches global or local minima, a vulnerability is triggered.

In order to evaluate the run-time performance of Formal-Fuzzer, feedback is determined based on the parameter, cost function improvement rate (CFIR).

$$
{CFIR} =  - \frac{\delta {F}_{c}}{\delta r} =  - \frac{{F}_{c2} - {F}_{c1}}{{r}_{2} - {r}_{1}} \tag{2}
$$

Where ${F}_{ci}$ presents the cost function values for ${r}_{i}^{th}$ execution.

A positive value of ${CFIR}\left( { \downarrow  {F}_{c}}\right)$ indicates that the fuzzer is improving by mutating smart inputs. Conversely, a negative CFIR indicates that the fuzzer is not progressing toward its objective. In such cases, FormalFuzzer switches the mutation strategy. CFIR can be calculated after each iteration or after a specific number of iterations, defined as the frequency of feedback generation $\left( {f}_{f}\right)$ . After every ${f}_{f}$ execution, the fuzzer may adjust the mutation strategy based on the positive or negative value of CFIR for improving performance.

![0196f2c0-a9e6-7e3d-99cb-9f132459e245_3_912_140_744_318_0.jpg](images/0196f2c0-a9e6-7e3d-99cb-9f132459e245_3_912_140_744_318_0.jpg)

Fig. 3: Implementation of FormalFuzzer Framework.

## IV. FORMALFUZZER IMPLEMENTATION

To assess the effectiveness and performance of Formal-Fuzzer, we utilized the open-source RISC-V-based Ariane SoC [30] as shown in Fig. 3. For implementing FormalFuzzer, we used different industry standard tools accompanied by our custom scripts. In the pre-processing step, we utilized a Python-based Hardware design parser called PyVerilog [31] to identify interface signals and Cadence JasperGold [32] to verify the generated assertions. The verification results are expressed in terms of postulations providing guidance to the mutation engine. We incorporated Integrated Logic Analyzer (ILA) core (from Xilinx) to support internal HW debugging on the Genesys 2 Kintex-7 FPGA Development Board, using the SoC bitstream synthesized and implemented with Xilinx Vivado. JTAG debug port was used for real-time monitoring. The fuzzing engine, based on AFL [16], was deployed on a Linux kernel directly mounted on the SoC via an SD card. We made several modifications to AFL (version 2.57b) for various purposes: 1) utilizing postulations to mutate from targeted input space,2) computation of ${F}_{c}$ and CFIR, and 3) utilizing feedback to change AFL's mutation techniques and mutate fuzzing inputs tailored for HW. To perform SoC fuzzing and vulnerability detection, high-level executable driving programs (by RISC-V compiler) are developed to interact with specific SoC components. FormalFuzzer executes these codes through the fuzzer, gathering output activity via ILA, and calculating cost function and feedback to guide the fuzzer. FormalFuzzer begins non-deterministic fuzzing after trying deterministic mutation with the initial seed. During fuzzing, when the cost function reaches a minimum value (global or local), it indicates the triggering of a vulnerability.

## V. EXPERIMENTS AND RESULTS

This section presents the experimental results to evaluate the performance of FormalFuzzer in detecting vulnerabilities.

## A. Formal Verification Results

Table II shows the step-by-step assertion generation results using interface signal list for some example security properties of several modules (in Columns 1-5) in Ariane SoC. Assertion-checking results are extracted as postulates(P)and reduced input space (RIS) (i.e., the ratio of reduced input space and total input space) for the mutation engine are presented in Columns 6 and 7, respectively.

TABLE II: List of Security Properties, Corresponding Templates, and Assertions per SoC Module.

<table><tr><td>Index/ Module</td><td>Security Property (SP)</td><td>#IS Temp- latesC D S</td><td>Generated Security Assertion (SA)</td><td>Postulation (P)RIS</td></tr><tr><td>SM1/ CSR file</td><td>SP11: CSRs should raise an exception if the access priv- ${AI}$ ilege mode of CSR does not match with current privilege mode of the core; SP12: CSRs should raise an exception if a wrong access request is made to CSRs</td><td>$1;0;2$ ; $\begin{array}{lll} 2 & 0 & 1 \end{array}$</td><td>SA11: $({csr}\_ {addr}\_ i\left\lbrack  {9 : 8}\right\rbrack  ! =$ priv $\_ {lvl}\_ o| \rightarrow$ ${\# \# }\lbrack 1:\$ \rbrack ({csr}\_ {exception}\_ o.{valid}=={1}^{\prime }{b1});$ SA12: $\left( {{csr}\_ {addr}\_ i\left\lbrack  {{11} : {10}}\right\rbrack  ! = {csr}\_ {op}\_ i}\right)  \mid   \rightarrow$ ${\# \# }\lbrack 1:\$ \rbrack ({csr}\_ {exception}\_ o.{valid} =  = {1}^{\prime }{b1}$</td><td>P11: User's privilege escala- 65% tion to Machine-mode CSRs; 739 P12: Write request to read- only CSRs</td></tr><tr><td>SM2/ CPU De- coder</td><td>SP21: If privilege level changes, the corresponding in- AT3 struction should also change to avoid an instruction spec- ified for one privilege level, to be executed in another privilege level</td><td>$\begin{array}{lll} 1 & 1 & 0 \end{array}$</td><td>SA21: $\sim$ \$stable(priv_lvl_i) |→ ##$\left\lbrack  {1 : \$ }\right\rbrack   \sim  \$$ stable(instr_o)</td><td>P21: Machine-level instruc- ${21}\%$ tion from user mode; P22: Mutate specific bits (31:7) of instruction fields</td></tr><tr><td>SM3/ AES</td><td>SP31: Encryption key should not be leaked through cipher ATI; text output port; SP32: After receiving plain text and AT5 encryption key as inputs, the encryption result should be available at the output after a specified number of clock cycles</td><td>$0;1;1;$ $\begin{array}{lll} 0 & 2 & 1 \end{array}$</td><td>SA31: encryption_key! =encryption_result; SA32: $\sim$ \$stable(encryption_key)|| ~ \$stable(plain_text) |→ $\# \# \left\lbrack  {1 : a}\right\rbrack  \left( {\mathit{{encryption}\_ {result}}! = 0}\right)$ $\& \&  \sim  \$ {stable}\left( {{encryption}\_ {result}}\right)$</td><td>P31: Check (mutate) first 93% byte of plain text</td></tr><tr><td>SM4/ MMU</td><td>SP41: MMU should raise an exception when physical ${AT2}$ memory access is operated in U/S mode</td><td>$\begin{array}{lll} 1 & 0 & 2 \end{array}$</td><td>SA41: $\left( {\text{priv_lvl_i}! = \text{PRIV_LVL_M}}\right) \& \&$ $\left( {{icache}\_ {areq}\_ i.{fetch}\_ {req} =  = {1}^{\prime }{b1}}\right)$ $|\rightarrow ({icache\_ areq\_ o}.{fetch\_ exception}=={1}^{\prime }{b1})$</td><td>P41: Access physical mem- ${50}\%$ ory in user mode</td></tr></table>

IS: Interface Signals, C: Control, D: Data, S: Status, RIS: Reduced Input Space in percentage for fuzzing mutation engine using Ps.

SM1: SP11: For CSR module, if any CSR can be accessed from another mode than the privilege mode specified, an exception should be raised. According to RISC-V specifications, the ${8}^{th}$ and ${9}^{th}$ bits in the CSR address field indicate the lowest privilege level. To generate the assertion, we select template ${AT3}$ as it depicts ${SP11}$ closely. From the interface signals identification step, we identify that both csr_addr_i[9:8] and priv_lvl_o are in the fan-in cone of csr_exception_o and instantiate AT3. A cover trace (Fig. 4) from assertion checking demonstrates a design behavior where the core in User (U) mode requests access to a CSR in Machine (M) mode at 12'h3B0, resulting in an exception. Taking into account that the Ariane SoC has 4096 CSRs in total, the identified CSR at 12'h3B0 isn't the only one that should comply with SP11. The cover trace, however, provides the postulate ${P11}$ for fuzzing in the next phase, which searches for CSRs in $M$ mode while the core is in $U$ mode. As the Ariane SoC has 1104 CSRs with $M$ mode privilege, the RIS for the fuzzer will be $\sim  {73}\%$ (i.e., from 4096 to 1104). SP12: A second security assertion derived for CSR detects scenarios where the core requests access operations (e.g., READ, WRITE, SET, CLEAR) to a CSR that do not match the access permissions (i.e., determined by the ${10}^{th}$ and ${11}^{th}$ bits of the address field) specified for the CSR, resulting in an exception. In this case, control signals csr_addr_i[11:10] and csr_op_i are in fan-in cone of the status signal csr_exception_o. The assertion checking provides a postulate ${P12}$ for the fuzzer to focus on write operations, attempted for read-only CSRs (i.e., in total 1408), thus resulting RIS for the fuzzer as $\sim  {65}\%$ (i.e., from 4096 to 1408).

SM2: SP21: We select AT3 to check SP21 for decoder module, which determines whether the instruction under execution changes with privilege changes. instruction_o is the relevant output data signal for ${SP21}$ , to which priv_lvl_i is an input signal in its fan-in cone. A cover trace shows that instruction_o changes with privilege level change from $M$ to $U$ . However, the lowest 7 bits of instruction_o[6:0] remain constant, indicating that the fuzzer should mutate the rest 25 bits to find out the privilege escalation vulnerability, resulting in RIS of $\sim  {21}\%$ .

SM3: SP31: For the AES module integrated into the Ariane SoC, SP31 states that the encryption key should never leak through the cipher text output port. As the property should always remain valid under any condition, ${AT1}$ is chosen for assertion generation. The generated assertion checking provides a counter-example, where encryption_key leaks through encryption_result when the lowest byte of plain text retains a specific value. As the plain text is a 128-bit signal, the counterexample guides the fuzzer to mutate only the lowest 8 bits; hence reducing fuzzing effort by $\sim  {93}\%$ . SP32: If either the plain text or encryption key changes, the encryption result also changes after a specific number of clock cycles. AT5 is selected in this case, as the property checks for a specific number of clock cycles for assertion generation. According to the counter-example obtained from the generated assertion, the encryption result is delayed by several clock cycles for a particular value of the lowest byte of plain text. Similar to SP31, in this case, the fuzzer’s effort gets reduced by $\sim  {93}\%$ .

SM4: SP41: The memory management unit (MMU) raises an exception through the output signal icache_areq_o.fetch_exception when there is a request to access the physical memory icache_areq_i.fetch_req from instruction cache module in a mode different than $M$ mode. The counter-example obtained from this assertion shows that physical memory can be accessed in $U$ mode without errors. This violation scenario provides a postulate to check access to physical memory from $U$ mode and reduce the input space for fuzzer by ${50}\%$ (i.e., from 2 to 1).

![0196f2c0-a9e6-7e3d-99cb-9f132459e245_4_911_1844_744_224_0.jpg](images/0196f2c0-a9e6-7e3d-99cb-9f132459e245_4_911_1844_744_224_0.jpg)

Fig. 4: Cover Trace Obtained from Assertion SA11 for CSR Register File.

## B. Pre-processing Guided Fuzzing Results

In this section, we analyze the impact of pre-processing and cost function-based feedback on the performance of Formal-Fuzzer in detecting vulnerabilities. For experiments, we have considered a set of known vulnerabilities from MITRE CWE [33] as described in Table III for SoC modules SM1-SM4. SV1 allows privilege escalation, i.e., unauthorized CSR read. SV2 permits successful execution of machine-level instruction from $U$ mode, whereas SV3 throws an exception for valid instruction execution. SV4 & SV5 incur confidentiality violation and denial of service in the AES module, respectively. SV6 allows access to higher-privileged memory from lower-privilege mode via MMU.

FormalFuzzer equipped with the guidance from preprocessing along with the feedback(CFIR)detects all vulnerabilities significantly faster. Fig. 5 illustrates the runtime cost functions while executing test programs for stimulating various SoC modules, targeting all individual vulnerabilities. Initially, the cost function $\left( {F}_{c}\right)$ has a high value, but as iterations progress, it fluctuates. FormalFuzzer updates the mutation technique every ${f}_{f}$ (frequency offeedback generation) number of iterations based on the change of ${F}_{c}$ . If ${F}_{c}$ increases (negative CFIR), prompting FormalFuzzer to change the mutation technique. As long as ${F}_{c}$ decreases, no change in the mutation technique is required. While we observed more fluctuation in ${F}_{c}$ during the early iterations of fuzzing, it stabilizes as FormalFuzzer finds the most suitable mutation technique, resulting in faster convergence to the global minimum.

While targeting for known vulnerabilities, FormalFuzzer was also able to detect three unknown vulnerabilities as described in Table IV. The first vulnerability (UV1) creates a critical security flaw as it may leak the previous ciphertext. This was detected because of no change in the output which decreases the cost function significantly (local minima). While fuzzing the CSR module, FormalFzzer was able to read a value from an undefined hardware performance counter (HPC). FormalFuzzer detected UV3 when mutating MULH instruction and was able to execute an illegal form of this.

TABLE III: List of Targeted Known Vulnerabilities in Ariane SoC.

<table><tr><td colspan="2">Index Vulnerability</td><td>TC</td><td>Reference</td></tr><tr><td>SV1</td><td>1 Access to mstatus CSR from lower privilege level</td><td>CSR Read</td><td>CWE-1262</td></tr><tr><td>SV2</td><td>Allow executing mret machine-level ins. from user-mode</td><td>Execute</td><td>CWE-1242</td></tr><tr><td>SV3</td><td>Incorrect logic to decode FENCE.I ins.</td><td>imm ≠ 0V ${rs1} \neq  0$</td><td>CWE-440</td></tr><tr><td>SV4</td><td>Leaking AES key through SoC common bus (ciphertext)</td><td>Specific P</td><td>Ref.*</td></tr><tr><td>SV5</td><td>A Trojan delays cipher conversion in the AES module</td><td>Specific P</td><td>AES-T500</td></tr><tr><td>SV6</td><td>5 Unauthorized memory access via MMU</td><td>Mem access</td><td>CWE-269</td></tr></table>

SV: Security Vulnerability, TC: Triggering Condition, P: Plaintext, *: CVE-2018-8922.

![0196f2c0-a9e6-7e3d-99cb-9f132459e245_5_138_1854_744_215_0.jpg](images/0196f2c0-a9e6-7e3d-99cb-9f132459e245_5_138_1854_744_215_0.jpg)

Fig. 5: Analysis of Cost Function and Feedback in Detecting Vulnerabilities.

TABLE IV: List of Detected Unknown Vulnerabilities by FormalFuzzer

<table><tr><td colspan="2">Index Vulnerability</td><td>TC</td><td>Reference</td></tr><tr><td/><td>UV1 Retrieve last ciphertext in AES module (Trojan)</td><td>Specific plaintext</td><td>CWE-401</td></tr><tr><td/><td>UV2 CSR read access to undefined HPC</td><td>No exception</td><td>CWE1281</td></tr><tr><td/><td>UV3 No exception raised for illegal format of MULH</td><td>${rd} \in  \{ {rs1},{rs2}\}$</td><td>CWE1262</td></tr></table>

UV: Unknown Security Vulnerability, TC: Triggering Condition.

TABLE V: Summary Results of Vulnerability Detection by FormalFuzzer.

<table><tr><td rowspan="2">Index</td><td rowspan="2">${f}_{f}$</td><td colspan="3">Number of Executions (Average)</td><td colspan="2">Speed Up</td></tr><tr><td>${F}_{NPNF}$</td><td>SoCFuzzer [13]</td><td>FormalFuzzer</td><td>S1</td><td>S2</td></tr><tr><td>SV1</td><td>6</td><td>868</td><td>541</td><td>201</td><td>76.84%</td><td>62.85%</td></tr><tr><td>SV2</td><td>5</td><td>421</td><td>165</td><td>133</td><td>68.41%</td><td>19.39%</td></tr><tr><td>SV3</td><td>5</td><td>361</td><td>208</td><td>161</td><td>55.90%</td><td>22.60%</td></tr><tr><td>SV4</td><td>5</td><td>15254</td><td>6687</td><td>187</td><td>98.77%</td><td>97.2%</td></tr><tr><td>SV5</td><td>5</td><td>12896</td><td>5210</td><td>204</td><td>98.42%</td><td>96.08%</td></tr><tr><td>SV6</td><td>6</td><td>5331</td><td>2592</td><td>1937</td><td>63.67%</td><td>25.27%</td></tr><tr><td>UV1</td><td>5</td><td>N/A</td><td>N/A</td><td>7389</td><td>N/A</td><td>N/A</td></tr><tr><td>UV2</td><td>6</td><td>N/A</td><td>N/A</td><td>758</td><td>N/A</td><td>N/A</td></tr><tr><td>UV3</td><td>5</td><td>N/A</td><td>N/A</td><td>737</td><td>N/A</td><td>N/A</td></tr></table>

${F}_{NPNF}$ : Fuzzing w/o proposed pre-processing and cost-function-based feedback. S1: FormalFuzzer speedup w.r.t. ${F}_{NPNF}$ and S2: Speedup w.r.t. SoCFuzzer.

TABLE VI: Comparison of FormalFuzzer with Existing Fuzzing Frameworks.

<table><tr><td>Framework</td><td>Target Design</td><td>Approach</td><td>Feedback/ Coverage</td><td>Auto- mation</td><td>Gray- box</td><td>Golden Model</td></tr><tr><td>HyPFuzz [14]</td><td>CPU</td><td>HDL Sim</td><td>Branch</td><td>No</td><td>No</td><td>Yes</td></tr><tr><td>TheHuzz [7]</td><td>CPU</td><td>HDL Sim</td><td>FSM, Statement</td><td>No</td><td>No</td><td>Yes</td></tr><tr><td>HyperFuzzing [9]</td><td>SoC</td><td>SW Sim</td><td>NoC, Bitflip</td><td>No</td><td>Yes</td><td>Yes</td></tr><tr><td>DifuzzRTL [34]</td><td>CPU</td><td>FPGA Emu</td><td>Control-register</td><td>No</td><td>No</td><td>Yes</td></tr><tr><td>RFUZZ [11]</td><td>IP</td><td>FPGA Emu</td><td>MUX</td><td>No</td><td>No</td><td>Yes</td></tr><tr><td>Fuzzing HW as SW [8]</td><td>SoC</td><td>SW Sim</td><td>Branch/Code</td><td>Yes</td><td>No</td><td>Yes</td></tr><tr><td>FormalFuzzer</td><td>SoC</td><td>FPGA Emu</td><td>Cost Function</td><td>Yes</td><td>Yes</td><td>No</td></tr><tr><td>Emu: Emulation</td><td/><td>Simulation</td><td/><td/><td/><td/></tr></table>

## C. Summary Results

A summary of the verification results, along with optimum ${f}_{f}$ for each vulnerability, is provided in Table V. Since fuzzing involves randomness, two identical experiments may require a different number of iterations for vulnerability triggers. To address this, we conducted each experiment three times and averaged the results. Our proposed framework successfully detected all known vulnerabilities within a reasonable time along with the detection of three unknowns. FormalFuzzer, equipped with formal verification-based postulations to reduce the input space (shown in Table II) and cost function-enabled feedback, achieved a minimum of ${55}\%$ reduction in verification time compared to the system without our proposed pre-processing and feedback (disabling feedback) and 19% compared to SoCFuzzer [13] (lacking proposed pre-processing).

## D. Comparison with Prior Works

FormalFuzzer's strength resides in its unique combination of features, including its independence from golden models, formal method's assistance in reducing input space, usage of an analog cost function, and feedback. No current HW fuzzing engines offer this comprehensive set of features. In Table VI, we highlight the key differences among FormalFuzzer and other state-of-the-art HW fuzzing techniques. By leveraging our proposed FPGA emulation-based framework and integrating the formal method, we significantly enhance fuzzing efficiency and performance. This allows us to apply our testing methodology to larger SoCs with minimal scalability issues.

## VI. CONCLUSION

In this manuscript, we presented FormalFuzzer, a fuzzing-based formal assisted hybrid framework designed specifically for verifying SoC security. By incorporating strategies derived from formal verification, a cost function-enabled feedback system, and utilizing a real-time debugging platform, For-malFuzzer offers scalable and efficient vulnerability detection capabilities. Our experiments demonstrated the effectiveness and efficiency of FormalFuzzer in terms of vulnerability detection rates and verification time.

## REFERENCES

[1] K. Z. Azar, M. M. Hossain, A. Vafaei, H. Al Shaikh, N. N. Mondol, F. Rahman, M. Tehranipoor, and F. Farahmandi, "Fuzz, penetration, and ai testing for soc security verification: Challenges and solutions," Cryptology ePrint Archive, 2022.

[2] N. Farzana, F. Rahman, M. Tehranipoor, and F. Farahmandi, "Soc security verification using property checking," in 2019 IEEE International Test Conference (ITC). IEEE, 2019, pp. 1-10.

[3] X. Meng, S. Kundu, A. K. Kanuparthi, and K. Basu, "Rtl-contest: Concolic testing on rtl for detecting security vulnerabilities," IEEE Transactions on Computer-Aided Design of Integrated Circuits and Systems, vol. 41, no. 3, pp. 466-477, 2021.

[4] C. Kern and M. R. Greenstreet, "Formal verification in hardware design: a survey," ACM Transactions on Design Automation of Electronic Systems (TODAES), vol. 4, no. 2, pp. 123-193, 1999.

[5] A. Ardeshiricham, W. Hu, J. Marxen, and R. Kastner, "Register transfer level information flow tracking for provably secure hardware design," in Design, Automation & Test in Europe Conference & Exhibition (DATE), 2017. IEEE, 2017, pp. 1691-1696.

[6] W. Hu, D. Mu, J. Oberg, B. Mao, M. Tiwari, T. Sherwood, and R. Kastner,"Gate-level information flow tracking for security lattices," ACM Transactions on Design Automation of Electronic Systems (TODAES), vol. 20, no. 1, pp. 1-25, 2014.

[7] R. Kande, A. Crump, G. Persyn, P. Jauernig, A.-R. Sadeghi, A. Tyagi, and J. Rajendran, "\{TheHuzz\}: Instruction fuzzing of processors using \{Golden-Reference\} models for finding \{Software-Exploitable\} vulnerabilities," in 31st USENIX Security Symposium (USENIX Security 22), 2022, pp. 3219-3236.

[8] T. Trippel, K. G. Shin, A. Chernyakhovsky, G. Kelly, D. Rizzo, and M. Hicks, "Fuzzing hardware like software," in 31st USENIX Security Symposium (USENIX Security 22), 2022, pp. 3237-3254.

[9] S. K. Muduli, G. Takhar, and P. Subramanyan, "Hyperfuzzing for soc security validation," in Proceedings of the 39th International Conference on Computer-Aided Design, 2020, pp. 1-9.

[10] J. Hur, S. Song, D. Kwon, E. Baek, J. Kim, and B. Lee, "Difuzzrtl: Differential fuzz testing to find cpu bugs," in 2021 IEEE Symposium on Security and Privacy (SP). IEEE, 2021, pp. 1286-1303.

[11] K. Laeufer, J. Koenig, D. Kim, J. Bachrach, and K. Sen, "Rfuzz: Coverage-directed fuzz testing of rtl on fpgas," in 2018 IEEE/ACM International Conference on Computer-Aided Design (ICCAD). ACM, 2018, pp. 1-8.

[12] S. Canákci, L. Delshadtehrani, F. Eris, M. B. Taylor, M. Egele, and A. Joshi, "Directfuzz: Automated test generation for rtl designs using directed graybox fuzzing," in 2021 58th ACM/IEEE Design Automation Conference (DAC). IEEE, 2021, pp. 529-534.

[13] M. M. Hossain, A. Vafaei, K. Z. Azar, F. Rahman, F. Farahmandi, and M. Tehranipoor, "Socfuzzer: Soc vulnerability detection using cost function enabled fuzz testing," in 2023 Design, Automation & Test in Europe Conference & Exhibition (DATE). IEEE, 2023, pp. 1-6.

[14] C. Chen, R. Kande, N. Nyugen, F. Andersen, A. Tyagi, A.-R. Sadeghi, and J. Rajendran, "Hypfuzz: Formal-assisted processor fuzzing," arXiv preprint arXiv:2304.02485, 2023.

[15] M. M. Hossain, N. F. Dipu, K. Z. Azar, F. Rahman, F. Farah-mandi, and M. Tehranipoor, "Taintfuzzer: Soc security verification using taint inference-enabled fuzzing," in 2023 International Conference on Computer-Aided Design (ICCAD). IEEE/ACM, 2023, pp. 1-6.

[16] M. Zalewski, "American Fuzzy Lop," https://lcamtuf.coredump.cx/afl/.

[17] N. Stephens, J. Grosen, C. Salls, A. Dutcher, R. Wang, J. Corbetta, Y. Shoshitaishvili, C. Kruegel, and G. Vigna, "Driller: Augmenting fuzzing through selective symbolic execution." in NDSS, vol. 16, no. 2016, 2016, pp. 1-16.

[18] L. Zhao, Y. Duan, H. Yin, and J. Xuan, "Send hardest problems my way: Probabilistic path prioritization for hybrid fuzzing." in NDSS, 2019.

[19] G. Dessouky, D. Gens, P. Haney, G. Persyn, A. Kanuparthi, H. Khattri, J. M. Fung, A.-R. Sadeghi, and J. Rajendran, "When a patch is not enough-hardfails: Software-exploitable hardware bugs," arXiv preprint arXiv:1812.00197, 2018.

[20] G. Dessouky, D. Gens, P. Haney, G. Persyn, A. K. Kanuparthi, H. Khat-tri, J. M. Fung, A.-R. Sadeghi, and J. Rajendran, "Hardfails: Insights into software-exploitable hardware bugs." in USENIX Security Symposium, 2019, pp. 213-230.

[21] X. Guo, R. G. Dutta, P. Mishra, and Y. Jin, "Scalable soc trust verification using integrated theorem proving and model checking," in 2016 IEEE International Symposium on Hardware Oriented Security and Trust (HOST). IEEE, 2016, pp. 124-129.

[22] R. Zhang, C. Deutschbein, P. Huang, and C. Sturton, "End-to-end automated exploit generation for validating the security of processor designs," in 2018 51st Annual IEEE/ACM International Symposium on Microarchitecture (MICRO). IEEE, 2018, pp. 815-827.

[23] P. Godefroid, N. Klarlund, and K. Sen, "Dart: Directed automated random testing," in Proceedings of the 2005 ACM SIGPLAN conference on Programming language design and implementation, 2005, pp. 213- 223.

[24] B. S. Pak, "Hybrid fuzz testing: Discovering software bugs via fuzzing and symbolic execution," School of Computer Science Carnegie Mellon University, 2012.

[25] T. Li, H. Zou, D. Luo, and W. Qu, "Symbolic simulation enhanced coverage-directed fuzz testing of rtl design," in 2021 IEEE International Symposium on Circuits and Systems (ISCAS). IEEE, 2021, pp. 1-5.

[26] A. Danese, N. D. Riva, and G. Pravadelli, "A-team: Automatic template-based assertion miner," in 2017 54th ACM/EDAC/IEEE Design Automation Conference (DAC), 2017, pp. 1-6.

[27] A. Dobberfuhl and M. W. Lange, "Interfaces per module: Is there an ideal number?" in International Design Engineering Technical Conferences and Computers and Information in Engineering Conference, vol. 48999, 2009, pp. 1373-1385.

[28] N. Farzana, A. Ayalasomayajula, F. Rahman, F. Farahmandi, and M. Tehranipoor, "Saif: Automated asset identification for security verification at the register transfer level," in 2021 IEEE 39th VLSI Test Symposium (VTS). IEEE, 2021, pp. 1-7.

[29] W. Hu, X. Wang, and D. Mu, "Security path verification through joint information flow analysis," in 2018 IEEE Asia Pacific Conference on Circuits and Systems (APCCAS). IEEE, 2018, pp. 415-418.

[30] F. Zaruba and L. Benini, "The cost of application-class processing: Energy and performance analysis of a linux-ready 1.7-ghz 64-bit risc-v core in 22-nm fdsoi technology," IEEE Transactions on Very Large Scale Integration (VLSI) Systems, vol. 27, no. 11, pp. 2629-2640, Nov 2019.

[31] S. Takamaeda-Yamazaki, "Pyverilog: A python-based hardware design processing toolkit for verilog hdl," in 11th Int. Symposium, ARC 2015, Germany. Proceedings 11. Springer, pp. 451-460.

[32] Cadence, "Jaspergold formal verification." [Online]. Available: https://www.cadence.com/en_US/home/tools/system-design-and-verification/formal-and-static-verification/jasper-gold-verification-platform.html

[33] MITRE, "HW CWEs," https://cwe.mitre.org/data/definitions/1194.html.

[34] S. Nilizadeh, Y. Noller, and C. S. Pasareanu, "Diffuzz: differential fuzzing for side-channel analysis," in 2019 IEEE/ACM 41 st International Conference on Software Engineering (ICSE). IEEE, 2019, pp. 176-187.