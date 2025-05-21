# TaintFuzzer: SoC Security Verification using Taint Inference-enabled Fuzzing

Muhammad Monir Hossain, Nusrat Farzana Dipu, Kimia Zamiri Azar, Fahim Rahman, Farimah Farahmandi, and Mark Tehranipoor

Electrical and Computer Engineering, University of Florida, Gainesville, Florida 32611, USA \{hossainm, ndipu, k.zamiriazar\}@ufl.edu, \{fahimrahman, farimah, tehranipoor\}@ece.ufl.edu

Abstract-Modern System-on-Chip (SoC) designs containing sensitive information have become targets of malicious attacks. Unfortunately, current verification practices still undermine the importance of SoCs security verification due to extreme time-to-market constraints, lack of autonomous methodologies, and low coverage. This results in SoC designs moving forward to production with security holes, making them insecure and exploitable by adversaries. Traditional taint analysis and formal approaches are losing applicability to industrial applications due to labor-intensive, slow, and scalability issues. Some approaches apply fuzz testing for hardware vulnerability detection using state-of-the-art software fuzzers, also utilizing information flow tracking for better coverage. However, these approaches prove to be inefficient and cannot be applied to SoCs integrated with third-party IPs (3PIP) for several reasons: laborious white-box-based taint analysis, inconsiderate cross-layer co-verification, and lacking hardware-centric input mutations. This paper proposes TaintFuzzer, a fuzzing-driven automated SoC security verification framework leveraging taint inference (feasible in gray-box verification) for detecting SoC security vulnerabilities. Unlike previous studies relying on traditional (code) coverage-related metrics, in TaintFuzzer, we develop (i) schemes for generating smart seeds, (ii) a security-oriented cost function, and (iii) run-time feedback for the mutation engine to choose the appropriate strategies to mutate stimuli targeting SoC modules. TaintFuzzer is powered by FPGA emulation of SoC, making it extremely fast and scalable, especially for cross-layer co-verification. TaintFuzzer's cost function and feedback enable dynamic tuning of mutation strategies to generate hardware-centric inputs. Our experiments with RISC-V-based SoC demonstrate the TaintFuzzer's effectiveness in detecting both known and unknown vulnerabilities in significantly less time.

Index Terms-SoC Security Verification, Testing, Fuzzing, Cost Function, Taint Inference

## I. INTRODUCTION

In modern SoCs with numerous intellectual properties (IPs) obtained from different vendors globally, the lack of reciprocal trustworthiness between entities may lead to emerging malicious functionality in the design [1]. Therefore, SoC designs may be compromised, posing security threats to critical applications. These ever-increasing concerns over SoC security demonstrate the importance of comprehensive verification during the pre-silicon stage before manufacturing, as post-silicon verification is particularly challenging and expensive [2].

The primary objective of verification is to ensure the confidentiality of SoC's critical assets, integrity, and availability (CIA triad) of all functionalities, according to the design and specification. Any threats to these properties must be identified during verification so that they can be resolved in the pre-silicon stage of the SoC before proceeding to the manufacturing stage [3]. There exists a variety of verification approaches such as formal techniques [3]-[5] and information flow tracking [6]- [9] to detect hardware (HW) vulnerabilities. However, these techniques suffer from some drawbacks. As SoCs are getting larger and more complex, formal verification approaches have become less effective due to scalability, state-space explosion issues, false positives/negatives, and incompatibility with multilayer HW/SW components. Information flow tracking requires manual taint labels, which may be erroneous and require technical expertise regarding the SoC design under verification. Taint tracking logic also increases code size and design overhead and results in false positives by tainting unnecessary signals (over-tainting) or ignoring necessary ones (under-tainting).

In recent studies, fuzz testing [10]-[16], a dynamic approach has become popular because of its applicability in both post-silicon and pre-silicon verification. Fuzzing was originally used by software (SW) engineers to identify SW's weaknesses and vulnerabilities by exposing it to unexpected inputs [17]-[20]. The majority of HW-oriented fuzz testing frameworks [10]-[16] have utilized these SW fuzzing engines (e.g., AFL [18]) for detecting the HW vulnerabilities in the HW design. However, such re-use of fuzzing in HW suffers from a few major flaws: (1) No consideration of accurate HW-oriented input mutation (i.e., no security-driven mutation and feedback); (2) Sole reliance on traditional SW-based code coverage approach with no direct (meaningful) mapping to the HW design; (3) Only applicable to white-box (full access to the design) model; and (4) No support of dynamic feedback generation/evaluation for runtime improvement. These shortcomings affect the automation of vulnerability detection in the HW domain, and the need for expert contribution is still inevitable. Additionally, some SW fuzzing approaches leverage advanced techniques like taint inference [21], [22] for more efficiency by prioritizing which branch to explore, which bytes to mutate, and how to mutate. Taint inference suggests inferring taints of variables by tracking their changes due to mutating input data. In SW, these approaches are still based on the white-box model and none of the HW-based fuzzing solutions utilizes such techniques for HW-oriented input generation at the SoC level.

To address the shortcomings mentioned above and benefit from such advanced technology from SW fuzzing, in this paper, we introduce TaintFuzzer, a gray-box-based verification approach that extends the state-of-the-art SW fuzzing engine, AFL [18], for SoC security verification. Using a vulnerability database and based on targeted SoC specification, a set of security-oriented cost functions is developed in TaintFuzzer, whose value is an indication of proximity to triggering a vulnerability, and the vulnerability is indeed triggered when the cost function reaches its minimum value (i.e., global minima). This generic gray-box cost function guides the mutation more efficiently toward unexplored cornerstone HW-oriented input generation. In addition, TaintFuzzer employs taint inference to generate smart seeds, which makes fuzzing highly efficient and guides mutation strategies through cost-function-based dynamic feedback. Also, cost function-based feedback evaluation is done using an emulation-based integrated logic analyzer. TaintFuzzer is examined via real HW vulnerabilities from different Hackathons on a large enough RISC-V SoC. Our main contributions to this work are as follows:

---

The authors would like to acknowledge funding support through Semiconductor Research Corporation (Grant/Award No. 2022-HW-3124 / AWD13345).

---

(1) Implementing a gray-box HW-oriented fuzzing framework leveraging taint inference for detecting $\mathrm{{SoC}}$ vulnerabilities.

(2) Developing a technique to derive smart seeds with higher efficiency from the initial seed using taint inference.

(3) Formulating an HW-oriented generic cost function and dynamic feedback leveraging taint inference, vulnerability characteristics, and SoC design specifications to guide the fuzzing mutation engine.

(4) Validate the effectiveness (performance) of the proposed fuzzing framework in detecting vulnerabilities on a real SoC.

## II. BACKGROUND AND PRIOR ART

Security-oriented verification is the process of protecting the security asset of a design (either primary or secondary [23]) and detects unintentional bugs or malicious functions that may compromise the security of these assets. The expected behavior of the design in terms of security can be verified using a set of security policies [3]. A variety of mechanisms have been used to implement/evaluate these policies, e.g., formal methods (also symbolic/concolic) [3]-[5], [24], [25] and information flow tracking (IFT) [6]-[9], [26]. Formal verification approaches suggest formally representing the policies, such as translating them into SystemVerilog [27] assertions, which are validated using formal tools (e.g., Cadence JasperGold [28]). For these approaches, cycle-accurate assertions must be written, which require expert knowledge and source codes (white-box) [3], [24]. Also, in the case of symbolic/concolic testing, test cases are generated to effectively achieve branch coverage in the design. It is, however, susceptible to scalability issues since it is based on the propagation of symbolic patterns [4], [25]. On the other hand, taint analysis requires the definition of the taint propagation rules from one operation to the next in the design, depending on the used language [29]. The taints of the variables are stored in additional variables, which incurs a considerable increase in memory and processing power. Moreover, IFT-based verification, while assuming white-box model, suffers from excessive and unwanted propagation, resulting in poor accuracy, limited scalability, false alarms due to excessive tainting, and false negatives for insufficient tainting [7], [8].

A recent trend in HW vulnerability detection has been fuzzing or fuzz testing because of (i) its applicability in both black-box and gray-box testing, (ii) scalability, and (iii) more cornerstone-directed mutation using feedback. Fuzz testing is the process of automatically testing programs [30], [31]. Generally, the fuzzing process consists of four steps starting with the initial seed(s): (i) Mutation, generates new randomized/directed inputs; (ii) Executing test case, runs the program under verification with inputs mutated from (i); (iii) Output observation, monitors the run-time behavior of output signals/ports to detect vulnerabilities (malicious activity or information leakage); and (iv) Feedback, guides the mutation engine for generating efficient input patterns to increase coverage. Few studies have investigated SW fuzzing applications for HW (RTL) verification [10]-[15]. It is possible to perform SW fuzzing on the HW in two variants: (1) Fuzzing the HW as SW and (2) Directly fuzzing the HW. For (1), by first translating the HW into its SW representation (e.g., using Verilator [32]), any SW fuzzer may be invoked [11], [14]. In case of (2), the fuzzer will be applied directly to the HW to generate mutated stimuli [10], [12], [13], [15], [16]. Fuzzing the HW design, through either (1) or (2) sounds promising. However, (i) Reliance on SW-based metrics, e.g., branch and code coverage, makes no clear connection with HW-oriented vulnerabilities, specifically those related to CIA violation; (ii) They adopt the white-box model for feedback generation, which is a tight assumption, specifically moving towards zero knowledge environment; (iii) They have no HW-oriented mutated input generation; (iv) They cannot derive assertions for fuzzer from the specifications of the HW design and vulnerability database; and (v) They support no run-time monitoring and correcting of mutation strategies for more efficiency. The authors in [33] attempted to overcome existing limitations using cost function-based feedback to some extent. However, none of the HW fuzzing frameworks benefits from the advanced techniques used in SW fuzzing. For instance, taint inference is a lightweight and efficient approach to gray-box fuzzing, widely used in SW fuzzing to overcome the challenges with traditional IFT [21], [22]. We can infer taint from the fact that a variable's value changes as a result of a mutation of an input byte that the variable depends on that input in any way, explicitly or implicitly, i.e. the variable can be controlled by changing the input, which might be used in a branch or data operation in the program. Integration of taint inference into HW-oriented fuzzing to generate the feedback sounds promising hybrid approach for taking the advantages of traditional IFT but in a gray-box model, almost no area overhead, and fewer false positives/negatives [21].

## III. Proposed Framework: TaintFuzzer

TaintFuzzer is a gray-box-based fuzzing framework using emulation powered by a dynamic cost function leveraging taint inference to automatically detect security vulnerabilities in SoCs that are associated with the CIA triad. The TaintFuzzer framework is designed to support the security verification engineers who consider gray-box model-based verification that meets shortened time-to-market constraints, removes the burden of understanding complex designs, and enables verification of third-party modules in the SoC. As the name suggests, to perform gray-box model-based verification, the verification engineers should have high-level knowledge regarding the security implications of all functionalities and operations of the SoC, based on its specifications. Moreover, to derive the security policies (required for cost function development) for TaintFuzzer, they need to identify potential attack scenarios, related to the CIA triad, by utilizing open-source vulnerability databases [34], [35] associated with such attacks.

![0196f2c1-8302-7121-bffb-e24f98581afb_2_92_182_727_253_0.jpg](images/0196f2c1-8302-7121-bffb-e24f98581afb_2_92_182_727_253_0.jpg)

Fig. 1: A high-level diagram of proposed TaintFuzzer framework.

Fig. 1 illustrates a high-level diagram of TaintFuzzer. Taint-Fuzzer involves a host machine to prepare the SoC's bitstream (under verification) for FPGA emulation platform. An initial seed and a driving program targeting SoC modules are taken as input to TaintFuzzer. As seed's quality may significantly impact fuzzing performance [36], TaintFuzzer proposes to generate smart seeds from the initial one irrespective of its quality provided by the verification engineer, ensuring efficient fuzzing. The fuzzing engine mutates new inputs and drives the SoC module(s) with them. TaintFuzzer observes the output and updates the cost function accordingly. For detecting vulnerabilities, when there is a rapid decrease in the cost function, TaintFuzzer compares the output behaviors against the vulnerability database without using any golden model. If the vulnerability is really triggered (matched malicious output behavior or information leakage), the cost function indeed reaches the global minimum. As TaintFuzzer is based on a cost function, detecting unknown vulnerabilities becomes possible by observing a significant decrease in the cost function. Taint-Fuzzer employs FPGA-supported emulation for monitoring (limited and required) HW signals and behaviors in real time. This is achieved by instrumenting the design based on a set of cost functions derived from HW-specific security policies to add the pivotal observation points to the SoC via a logic analyzer. When developing the cost functions, we assume the verification engineer possesses very minimal design (e.g., specifications) knowledge. TaintFuzzer provides feedback based on the cost function (w.r.t security policies), and the most-fitting mutation technique is selected based on that feedback, enforcing faster convergence of triggering a vulnerability. Since a gray-box model verification approach leverages a limited amount of HW knowledge, run-time evaluation of fuzzing efficiency (quality of inputs) and generated seeds is super challenging. To overcome this challenge, TaintFuzzer utilizes taint inference to assess the impact of inputs on the HW system by observing the output ports/behaviors without direct instrumentation of internal design parts (following gray-box model) and thereby generating smart seeds, cost function, and feedback in an effective manner. In this section, we discuss the taint inference rule, cost function, and feedback, proposed for the TaintFuzzer framework.

## A. TaintFuzzer Taint Inference Rule

Taint inference in HW is the process of mapping output changes to the mutation of the input port, inferring that this output port depends on the input port either explicitly or implicitly. Therefore, output ports are defined as tainted for the specified inputs. Consider the targeted SoC has ${l}_{i}$ number of input ports(S)that form the seed and mutated inputs and $z$ output ports. TaintFuzzer maps the applied inputs with the corresponding tainted output ports(t)by applying the random mutation (See Eq. 1). Since the mutation of a particular input port may have multiple tainted output ports, we define the number of tainted output ports as the tainting weight $\left( {n}_{t}\right)$ for the corresponding HW input port (Eq. 2). TaintFuzzer decides the priority on input port-wise HW-oriented mutation technique based on this tainting weight and mapping as discussed later (Section III-E).

$$
\operatorname{Map}\left\lbrack  {S\left\lbrack  k\right\rbrack  }\right\rbrack   = \left\{  {{t}_{1},{t}_{2},\ldots ,{t}_{z}}\right\}  ;k \in  \left\{  {1,2,\ldots ,{l}_{i}}\right\}   \tag{1}
$$

$$
{W}_{S\left\lbrack  k\right\rbrack  } = {n}_{{t}_{k}};0 \leq  {n}_{{t}_{k}} \leq  z, k \in  \left\{  {1,2,\ldots ,{l}_{i}}\right\}   \tag{2}
$$

## B. Smart Seeds Evolution

A seed may play a vital role in gray-box fuzzing in terms of expedited convergence to global minima [36]. A seed is defined as the initial input that allows the fuzzer to trigger the vulnerability by using mutation strategies on this seed. In TaintFuzzer framework, we propose to generate some efficient seeds that we define as smart seeds for efficient HW fuzzing and thereby triggering the vulnerabilities faster. However, smart seeds can neither be selected in a trivial process nor generated arbitrarily. A verification engineer has difficulty selecting a smart seed due to the need for a thorough understanding of the internal HW design, which is not feasible for gray-box verification. In TaintFuzzer, we develop a metric based on which it mutates some smart seeds from the initial seed. The metric measures the impact on HW behavior and the proximity of vulnerability-triggering behavior. Higher the impact on the HW output ports due to the inputs (seed), the higher chance for faster convergence. We call this metric ${HW}$ seed impact factor $\left( {I}_{s}\right)$ shown in Eq. 3.

$$
{I}_{s} = \frac{1}{{l}_{i} \times  z}\mathop{\sum }\limits_{{k = 1}}^{{l}_{i}}{W}_{S\left\lbrack  k\right\rbrack  } + \left\{  {1 - \frac{1}{m}\mathop{\sum }\limits_{{k = 1}}^{m}h\left( {{o}_{r, k},{o}_{t, k}}\right) }\right\}   \tag{3}
$$

Where, ${l}_{i}$ : total number of input ports, $z$ : total number of output ports, ${W}_{S\left\lbrack  k\right\rbrack  }$ : tainting weight of ${k}^{th}$ input port, $h\left( *\right)$ : hamming distance (HD) in percentage, $m$ : total number of targeted output ports/secret data bits or behavior to monitor a certain vulnerability, ${o}_{t, k}$ : expected asset or behavior of ${k}^{th}$ target output port, and ${o}_{r, k}$ : runtime observed value or behavior of ${k}^{th}$ target output port.

The first term in Eq. 3 indicates the impacted HW ports due to the taint inference for the applied seed input based on Eq. 2. If an input's taint penetrates the HW ports as much as possible, the chance of activating more control paths is significantly high. Therefore, taint inference is a crucial parameter to measure the impact of seed on internal HW behavior and determines which seed(s) might have a higher chance of triggering the vulnerability. The second term in Eq. 3 indicates the proximity of the output behavior to a potential malicious behavior or asset leakage w.r.t. the candidate seed. A seed is a good choice, i.e., a smart seed, if it develops the closest behavior to malicious behavior or asset leakage that might require a few mutations to reach the global minima. Efficient fuzzing depends on an optimum number of smart seeds. Fuzzing with fewer smart seeds may result in inefficient randomness in mutation, while a larger number of smart seeds may cost a significant time to mutate these seeds before fuzzing.

## C. TaintFuzzer Cost Function

We develop a generic gray-box-based cost function to guide the fuzzer based on the real-time monitoring of HW behavior and taint inference to discover the vulnerability in significantly less time. The cost function is derived based on evaluating run-time fuzzing performance with input seed, determining the proximity of the target behavior and asset leakage of the HW design (vulnerability triggering constraints), and assessing how mutated inputs (i.e., mutation strategy) affect internal signals/ports due to taint inference. Eq. 4 represents the normalized cost function, where the average value of all considered parameters is subtracted from 1 (zeroth term). With the decreasing ${F}_{c}$ , it approaches closer to triggering the vulnerability.

$$
{F}_{c} = 1 - \frac{1}{4} \times  \left\lbrack  {\frac{{n}_{t}}{z} + \left\{  {1 - \frac{2}{r\left( {r - 1}\right) }\mathop{\sum }\limits_{{j = 1}}^{{r - 1}}\mathop{\sum }\limits_{{k = j + 1}}^{r}h\left( {{S}_{j},{S}_{k}}\right) }\right\}  }\right.
$$

$$
+ \left\{  {1 - \frac{1}{m}\mathop{\sum }\limits_{{k = 1}}^{m}h\left( {{o}_{r, k},{o}_{t, k}}\right) }\right\}   + \frac{{n}_{u}}{{2}^{N}}\rbrack
$$

(4)

Where, $z$ : total number of output ports ${n}_{t}$ : total tainted output ports, $r$ : total number of mutated inputs, ${S}_{j}$ : mutated input for ${j}^{th}$ execution, ${S}_{k}$ : mutated input for ${k}^{th}$ execution, $h\left( {{S}_{j},{S}_{k}}\right)$ : HD (percentage) between ${S}_{j}$ and ${S}_{k}, m$ : total number of target output ports/secrets (in bits)/behavior, ${o}_{t, k}$ : expected asset value or behavior of ${k}^{th}$ target output port, ${o}_{r, k}$ : runtime observed value or behavior of ${k}^{th}$ target output port, ${n}_{u}$ : total fuzzed unique inputs, and $N$ : total number of effective input bits of SoC to be fuzzed.

Taint inference $\left( {1}^{\text{st }}\right.$ term in the bracket of Eq. 4) plays a vital role in the cost function. This measures the impacted HW ports for a specific input, i.e., the percentage of tainted output ports. As long as the mutated inputs can taint HW output ports more indicating higher tainting weight $\left( {F}_{c}\right.$ decreases), this increases the chances of fuzzing strategy to expose more control paths (i.e., cornerstone inputs) and, eventually, this will probably trigger more unseen bug(s). The ${2}^{\text{nd }}$ term presents the randomness among certain mutated inputs. Fuzzing with no feedback produces complete random test inputs, resulting in lower coverage than feedback-enabled fuzzing. So, the fuzzing must generate inputs away from the ideal hamming distance (HD) of ${50}\%$ (HD of complete randomness) while creating unique inputs per execution. To have a decreasing cost function in case of less randomness (i.e., generated intelligently and not arbitrarily), we subtract normalized HD from 1 . The ${3}^{\text{rd }}$ term in Eq 4 gives the percentile of how much output ports achieve the corresponding value or behavior that collectively may lead to the triggering of a vulnerability. For example, consider the confidentiality violation of a cryptographic accelerator interfaced with an SoC. With an increasing number of leaking cryptographic key, this term decreases and reaches a minimum of zero when all key bits are exposed. As an output behavior, this term of Eq 4 also considers the repeated outputs even for the changes of input values, which may refer to stuck faults to specific outputs. The ${4}^{\text{th }}$ term measures the traversed input space of SoC-under-test (number of executions). The increasing value of this term indicates the fuzzing tool covers more input space and hence gets closer to the trigger of the vulnerability (exploring corner cases with a high chance of bugs).

Run-time cost function evaluation allows real-time correction of mutation strategies through a feedback parameter (Section III-D). The proposed framework is able to tune the best mutation strategies dynamically for HW design and it achieves faster convergence of the cost function to hit the vulnerability. A decreasing cost function indicates that the fuzzing tool is gradually approaching vulnerability-triggering conditions. After ${F}_{c}$ reaches the global minima, the vulnerability is triggered, indicated in the final test patterns.

## D. TaintFuzzer Feedback

We develop dynamic feedback from the cost function to guide the fuzzer for higher efficiency in mutating cornerstone inputs. This feedback is derived from our defined parameter, Cumulative Differential Cost Function (CDCF) in Eq. 5, where ${F}_{{c}_{k}}$ is the cost function value for the ${k}^{th}$ execution/iteration. ${CDCF}$ is updated after a certain number of iterations (at least two iterations), which is defined as the frequency of feedback evaluation $\left( {f}_{f}\right)$ . Per each ${f}_{f}$ executions, the fuzzer may change the mutation strategy based on the value of ${CDCF}$ . If positive, it indicates that the fuzzer is performing better by developing more smart inputs, i.e., approaching global minima of ${F}_{c}$ . A negative value of ${CDCF}$ indicates the fuzzer is failing to achieve the objective, so TaintFuzzer alters the mutation technique. The optimal value for ${f}_{f}$ is necessary to avoid situations when mutation techniques change too rapidly or very slowly, causing an incomplete assessment of the HW internals or requiring significant verification time, respectively.

$$
{CDCF} =  - \mathop{\sum }\limits_{{k = 2}}^{{f}_{f}}\left( {{F}_{{c}_{k}} - {F}_{{c}_{k - 1}}}\right)  \tag{5}
$$

### E.HW Oriented Mutated Fuzzing Inputs

In TaintFuzzer, we leverage an existing SW fuzzer for mutation. Considering that HW-oriented input patterns are different from SW, which focuses on underlying registers, memory addresses, and so on, we examine some effective strategies for mutating HW-oriented inputs:

(A) Fixed Input Length: Varying the input's length is mostly necessary for SW verification. For example, SW applications including data processing may hit different branches based on the length of the input. However, HW requires a specific length of input (based on the communication protocols) as designed in the Hardware Description Language. While considering a sequence of inputs to $\mathrm{{SoC}}$ , the sequence’s length still could be variant for a particular number of clock cycles.

(B) Control Signal Invariants: Some control signals may not need to mutate or there might be a limited range of all possibilities. For instance, the signal to initiate encryption must be fixed, otherwise, the ciphertext can't be generated. (C) Importance of Data Coverage: Data coverage is also important along with path coverage in HW verification. In SW fuzzing, higher path coverage may significantly cover the entire SW verification, but in HW, data coverage is also very important. We can mention AES key leakage for a specific plaintext as an example, where it has only a few control paths but data coverage is ${2}^{128}$ (128-bit AES Module). Therefore, covering all paths does not guarantee the absence of this information leakage vulnerability.

![0196f2c1-8302-7121-bffb-e24f98581afb_4_92_185_727_299_0.jpg](images/0196f2c1-8302-7121-bffb-e24f98581afb_4_92_185_727_299_0.jpg)

Fig. 2: Diagram for TaintFuzzer framework implementation.

(D) Input Port-wise Mutation: Byte-wise mutation irrespective of considering variables length used in SW verification is not much effective for HW fuzzing. Rather bit or byte-level mutation focusing on a particular port is effective. This approach may ensure the effect of mutating one input port while others are intact, where input port selection is prioritized based on Eq. 2. For example, from a 16-bit input port, one or some bits of the input may be used as a selector of a MUX for controlling data flow (input $\rightarrow$ output) of the MUX. TaintFuzzer emphasizes cross-input ports mutation (selected based on Eq. 1) for exposing more diversified HW behavior.

To implement these strategies, TaintFuzzer acquires some high-level specifications of targeted SoC along with primary I/O ports from the verifier and utilizes this information in mutation.

## IV. IMPLEMENTATION OF TAINTFUZZER

TaintFuzzer has been implemented for evaluating the performance in detecting vulnerabilities in the SoC. We considered an open-source RISCV-based Ariane SoC [37] as the SoC in the framework implementation for experiment purposes. Fig. 2 illustrates how different components of TaintFuzzer are built and connected. Internal HW debugging is enabled by integrating the Xilinx Integrated Logic Analyzer (ILA) [39], and it is emulated using the Genesys 2 Kintex-7 FPGA Development Board [40]. We enabled real-time monitoring via a JTAG debug port, and a Linux kernel with a built-in fuzzing engine (based on AFL [18]) was directly mounted on the SoC. The AFL source code has been extended in multiple ways: (i) determining the tainting weight and mapping of input and tainted output ports (Section III-A); (ii) integration of input generation strategies for HW-oriented mutation (Section III-E); (iii) smart seed generation and evaluation (Algorithm 1); (iv) receiving HW signal's status from the host machine through UART; (v) calculating cost function and feedback (CDCF); and (vi) integration of feedback-based mutation techniques (Algorithm. 2). The high-level C code executables have been written to drive specific components of the SoC to fuzz and detect vulnerabilities. Compiling these codes using the RISC-V compiler and providing the executable binary (.elf) to the fuzzer on the SoC with the required initial seeds allows the fuzzer to detect vulnerabilities.

Algorithm 1: Evolution of smart seeds for TaintFuzzer

---

Input: Initial seed $\left( {S}_{0}\right)$ , Number of smart seeds(n)

Output: Sorted $n$ smart seeds, ${S}_{s}\left\lbrack  n\right\rbrack$

initialize $i \leftarrow  1$

Calculate Impact ${I}_{0}$ for ${S}_{0}$

while $i \leq  n$ do

	Get a randomly mutated candidate seed ${S}_{i}$ from ${S}_{0}$

	Calculate Impact ${I}_{i}$ for ${S}_{i}\;/ \star$ See Eq. 2 and

		3 */

	if ${I}_{i} > {I}_{o}$ then

		$S\left\lbrack  i\right\rbrack   \leftarrow  {S}_{i}, i \leftarrow  i + 1$

out ${S}_{s}\left\lbrack  n\right\rbrack  \;/ \star$ By sorting $S\left\lbrack  n\right\rbrack$ using $I\left\lbrack  n\right\rbrack   \star$ /

---

Algorithm 2: Cost function-based feedback and fuzzing

---

Input: Sorted $n$ smart seeds, ${S}_{s}\left\lbrack  n\right\rbrack$ , Frequency of

		Feedback Evaluation, ${f}_{f}$

Output: Bug detection, global minima of ${F}_{c}$

initialize $i, r \leftarrow  1,{F}_{{c}_{\min }} \leftarrow  1,{CDCF} \leftarrow  0$

while $i < n$ do

	while True do

		Fuzzing w/ deterministic mutation & ${S}_{s}\left\lbrack  i\right\rbrack$ seed

			Calculate cost function for ${r}^{th}$ iteration, ${F}_{{c}_{r}}$

		if $r > 2$ then

				${CDCF} \leftarrow   - \left( {{F}_{{c}_{r}} - {F}_{{c}_{r - 1}}}\right)  + {CDCF}$

		if ${F}_{{c}_{r}} < {F}_{{c}_{\min }}$ then

				${S}_{\min \left( {F}_{c}\right) } \leftarrow  {S}_{s}\left\lbrack  i\right\rbrack  \;/ \star$ Seed for local

					minima */

				${F}_{{c}_{\min }} \leftarrow  {F}_{{c}_{r}}$

			/* Check if CDCF > 0 (Eq. 5) when

				feedback is evaluated */

			if $r > 2\& r\% {f}_{f} = 0\& {CDCF} < 0$ then

				Change mutation strategy, reset ${CDCF} \leftarrow  0$

		if Global Minima of ${F}_{{c}_{r}}$ or TimeOut then

				Exit Fuzzing /* Bug triggered */

		$r \leftarrow  r + 1$

	$i \leftarrow  i + 1$

while !(Global Minima of ${F}_{{c}_{r}}$ ) or !TimeOut do

	Fuzzing w/ non-deterministic mutation and ${S}_{\min \left( {F}_{c}\right) }$

---

In Algorithm 1, we show how TaintFuzzer generates smart seeds (highly correlated to vulnerability-triggering input). Smart seeds are derived based on the impact factor discussed in Eq. 3. TaintFuzzer at first creates a candidate seed by randomly mutating the initial seed. For a candidate seed with improved impact factor (vs. intial seed's impact factor), it may be considered a smart seed. Finally, $n$ smart seeds are chosen with the best impact factors provided for fuzzing.

TaintFuzzer calculates the run-time cost function and feedback for guiding the fuzzer as described in Algorithm 2 by gathering the output activity through ILA. When it comes to mutation, TaintFuzzer initially takes deterministic approaches, such as bit flipping, arithmetic operations, etc. for all individual smart seeds. Deterministic mutations are generally much more effective than non-deterministic ones in fuzzing. Therefore, TaintFuzzer emphasizes this approach at the beginning. The proposed framework changes the mutation strategy based on the feedback ${CDCF}$ as discussed in Eq. 5 when a particular deterministic approach fails to improve the cost function. Upon completion of all mutations for all seeds using deterministic approaches available in AFL, the framework starts fuzzing with non-deterministic mutations (havoc), with the best seed being recognized as having the lowest cost function achieved in deterministic mutations. As randomized mutations are prone to hit the cornerstone inputs by sheer luck, this approach is tried at the last phase of fuzzing.

TABLE I: List of inserted vulnerabilities in Ariane SoC [37] based on CWE MITRE database [38].

<table><tr><td/><td>dex Vulnerability</td><td>Location</td><td>Triggering Condition</td></tr><tr><td/><td>SV1 Allow executing machine-level (M) instructions from user mode (U)</td><td>CPU (dec)</td><td>Execute "mret" instruction</td></tr><tr><td/><td>SV2 Incorrect implementation of logic to detect the FENCE.I instruction</td><td>CPU (dec)</td><td>${imm} \neq  0 \vee  {rs1} \neq  0$</td></tr><tr><td>SV3</td><td>Improper privilege management by the memory management unit (MMU)</td><td>CPU (LSU)</td><td>Unauthorized memory access</td></tr><tr><td>SV4</td><td>Access to CSRs from lower privilege level</td><td>Register file</td><td>mstatus_reg rd/wr from user space</td></tr><tr><td>SV5</td><td>Improper access control to CSRs</td><td>Register File</td><td>Unauthorized write access to CSR from user mode</td></tr><tr><td/><td>SV6 Leaking the AES key through SoC common bus</td><td>AES IP</td><td>Specific plaintext</td></tr><tr><td/><td>SV7 A Trojan delays cipher transformation in AES module in SoC</td><td>AES IP</td><td>Specific plaintext</td></tr></table>

<table><tr><td>Output Behaviour</td><td>Reference</td></tr><tr><td>No exception exhibited</td><td>CWE-1242</td></tr><tr><td>Illegal instr exception raised</td><td>CWE-440</td></tr><tr><td>No access error is raised</td><td>CWE-269</td></tr><tr><td>No exception exhibited</td><td>CWE-1262</td></tr><tr><td>CSRs get updated even if priv- ilege violation is raised</td><td>CWE-284</td></tr><tr><td>AES Key leaks to PO</td><td>CVE-2018-8922</td></tr><tr><td/><td>AES-T500</td></tr></table>

SV: Security Vulnerability.

## V. EXPERIMENTAL SETUP AND RESULTS

### A.SoC Vulnerabilities for Experiments

This paper investigates the efficiency of TaintFuzzer on a set (7) of bugs reported in MITRE CWE database [38] in the Ariane SoC, listed in Table I. RISC-V ISA implements machine (M), supervisor (S), and user (U) modes for controlling legitimate permissions. SV1 states that even if an illegal instruction (e.g., mret ) is executed in user mode, no exception is exhibited. As indicated in SV2, the instruction decoder of a RISC-V processor ignores the immediate(imm)and first source register (rs1) fields of FENCE.I instruction, violating the RISC- $\mathrm{V}$ specification. The illegal rejection of execution of FENCE.I may result in cache coherence problems. According to SV3, the memory management unit (MMU) can be tweaked in a way such that a user from one privilege level can access a page of another privilege level which is not permitted, without generating a data/instruction access error. Both SV4 and SV5 allow unauthorized read and/or write to control and status registers (CSRs) from a lower privilege mode resulting in privilege escalation. The SV6 and SV7 are based on malicious Trojans introduced into the AES core integrated with Ariane SoC. 3PIPs acquired from an untrustworthy vendor may cause these vulnerabilities. In SV6, the Trojan is triggered when plaintext matches a preset value. It leaks the encryption key, the primary security-critical asset of AES IP, through the ciphertext. SV7 states that a timing violation in the AES crypto module can result in a denial of service for time-critical applications.

## B. Cost Function Development Per Vulnerability

We developed the taint inference-based cost function (Eq. 4) to guide the mutation strategy for faster detection of vulnerabilities. Table II lists the parameters associated with each vulnerability's cost function according to Eq. 4, which are derived from the specifications and vulnerability database. Here we show an example of how the cost function development process is done (for SV1). A C-program (.elf) is developed to run a 32-bit instruction (effective input length, $N = {32}$ ) in the user mode, where the mutated data goes to the Instruction Register(IR). The fuzzed ${IR}$ may have tainted operand or other fields $\left( {n}_{t}\right.$ ranges up to 4 depending on IR type representing various fields, such as opcode, destination register, 1 or 2 source registers, or immediate field) which lead to either executing the instruction or incur a segmentation fault. Therefore, one input (IR) manipulates the output behaviors $\left( {o}_{r, k}\right)$ , such as raising no exception, skipping IR flushing, and privilege escalation [37], resulting in $m = 3$ . While fuzzing, illegitimate instruction is executed from the user mode, thus target behavior $\left( {o}_{t, k}\right)$ becomes No Exception (number of behaviors are represented in bits). As this is supposed to be executed only in machine mode, the ISA specification is violated, hence, triggering the vulnerability. The randomness is calculated when the feedback is evaluated, i.e., after ${f}_{f}$ executions $\left( {r = {f}_{f}}\right)$ . Thus, the cost function is deduced to be Eq. 6 with the parameters for SV1.

$$
{\left( {F}_{c}\right) }_{SV1} = 1 - \frac{1}{4}\left\lbrack  {\frac{{n}_{t}}{z} + \left\{  {1 - \frac{2}{{f}_{f}\left( {{f}_{f} - 1}\right) }\mathop{\sum }\limits_{{j = 1}}^{{{f}_{f} - 1}}\mathop{\sum }\limits_{{k = j + 1}}^{{f}_{f}}h\left( {I{R}_{j}, I{R}_{k}}\right) }\right\}  }\right.  \tag{6}
$$

$$
\left. {+\left\{  {1 - \frac{1}{m}\mathop{\sum }\limits_{{k = 1}}^{m}h\left( {{o}_{r, k},\text{ NoException }}\right) }\right\}   + \frac{{n}_{u}}{{2}^{N}}}\right\rbrack
$$

TABLE II: Cost function parameters per vulnerability.

<table><tr><td rowspan="2">Index</td><td rowspan="2">Max ${n}_{t}$</td><td colspan="2">Input</td><td colspan="2">Output</td><td rowspan="2">$N$ (bits)</td></tr><tr><td>Data</td><td>Length</td><td>Port/Behavior(s)</td><td>$m$</td></tr><tr><td>SV1</td><td>4</td><td>Instruction</td><td>32 bits</td><td>Exception, Flush, PE</td><td>3</td><td>32</td></tr><tr><td>SV2</td><td>4</td><td>Instruction</td><td>32 bits</td><td>Exception for ${rs1}/{imm}$</td><td>2</td><td>32</td></tr><tr><td>SV3</td><td>1</td><td>Address</td><td>64 bits</td><td>NAE, Out of range, ${\mathrm{W}}^{ * }$</td><td>3</td><td>64</td></tr><tr><td>SV4</td><td>1</td><td>CSR Ins.</td><td>32 bits</td><td>Exception, ${\mathrm{W}}^{ * }$ , PE</td><td>3</td><td>12</td></tr><tr><td>SV5</td><td>1</td><td>CSR Ins</td><td>32 bits</td><td>Exception , ${\mathrm{W}}^{ * }$ , PE</td><td>3</td><td>12</td></tr><tr><td>SV6</td><td>2</td><td>Plaintext, CR</td><td>128 bits</td><td>Ciphertext, CR</td><td>2</td><td>128</td></tr><tr><td>SV7</td><td>2</td><td>Plaintext, CR</td><td>32 bits</td><td>Ciphertext, CR</td><td>2</td><td>128</td></tr></table>

*:Memory/Register Write NAE: No Access Error Ins: Instruction PE: Privilege Level Escalation CR: AES Module Control Register

On the contrary, SV2 incurs an exception for a valid instruction, otherwise, the cost function is almost the same as SV1. In case of SV3, the memory address is fuzzed, which is 64-bits(N) for Ariane SoC. As the CSR specifier is ${12}\left( N\right)$ bits in the 32-bits of CSR RISC-V instruction [41], the effective length is 12 for SV4 and SV5. In the case of SV6 and SV7, the inputs are plaintext (128-bits) and control register (32-bits) $\left( {{n}_{t} = 2}\right.$ , representing a total number of inputs), where the target output is the ciphertext and CR for SV6 $\left( {m = 2}\right)$ and the behavior of delayed CR and cipher update for SV7 $\left( {m = 2}\right)$ . Thus the parameters for gray-box fuzzing are derived based on the high-level characteristics of the target module and vulnerability database rather than detailed source-code information.

![0196f2c1-8302-7121-bffb-e24f98581afb_6_96_188_712_217_0.jpg](images/0196f2c1-8302-7121-bffb-e24f98581afb_6_96_188_712_217_0.jpg)

Fig. 3: Impact of smart seeds in TaintFuzzer performance.

## C. Results and Analysis

This section presents the experimental results and analysis of how taint inference-based smart seeds, cost function $\left( {F}_{c}\right)$ , and feedback (CDCF) impact the performance of TaintFuzzer for vulnerability detection.

Role of Smart Seeds: TaintFuzzer mutates some smart seeds using taint inference (Eq. 3) for efficient fuzzing. The time (number of executions) required to generate these smart seeds is negligible as compared to the entire fuzzing session (shown in Table IV). From the experiments, we observed that, for two primary reasons the number of executions required for converging to global minima decreases with more utilization of smart seeds: (i) using efficient seeds (higher tainting capability) and (ii) focusing more on deterministic mutations with the smart seeds. This allows TaintFuzzer to detect the vulnerabilities in significantly less time with smart seeds (Fig. 3). However, the number of smart seeds cannot be very high as may become a bottleneck in verification. In our experiments, the best number of smart seeds ranges from 5 to 30 (obtained through a few trials for which vulnerabilities were triggered within a minimum number of executions comparatively) for vulnerabilities with low to high input space, respectively (shown in Table IV).

Impact of Cost Function and Feedback: In TaintFuzzer, the feedback is developed based on the taint inference-enabled cost function. The feedback guides the mutation engine calculated based on the dynamic cost function values. We observe that TaintFuzzer with this feedback outperforms the conventional fuzzing framework. For all vulnerabilities, we plot the runtime cost function in Fig. 4. An increasing ${F}_{c}$ implies a negative ${CDCF}$ , forcing the TaintFuzzer to change the mutation technique. On the contrary, if ${F}_{c}$ is decreasing, no mutation technique change is necessary. We observe a notable fluctuation in ${F}_{c}$ . However, it becomes more stable as it finds the most effective mutation technique, resulting in faster convergence to global minima. With any ${f}_{f}$ value, TaintFuzzer with feedback were able to converge to the global minima significantly faster than the system without feedback, as reflected in Table IV.

Role of ${f}_{f}$ : The frequency of feedback evaluation $\left( {f}_{f}\right)$ affects the convergence rate of the cost function, which is reflected in our experiments. We swept ${f}_{f}$ for SV2 and SV4 and plotted the number of iterations/executions required for triggering the vulnerabilities as shown in Fig. 5. We observed that the number of required executions is comparatively high for small and large ${f}_{f}$ values. A small ${f}_{f}$ indicates a rapid update of the mutation technique (e.g., 2). Consequently, an inadequate number of samples are evaluated, which results in the global minima taking a long time to reach. In contrast, when ${f}_{f}$ is large (e.g., $5,6)$ , the fuzzer spends more time on the same inefficient mutation technique until new feedback is evaluated, causing slower convergence. For SV2 and SV4, we observed that the optimum choice of ${f}_{f}$ can improve the convergence by ${15}\mathrm{x}$ and $4\mathrm{x}$ , respectively as compared to fuzzing without the proposed feedback. Therefore, ${f}_{f}$ must be selected meticulously to provide the highest efficiency for TaintFuzzer.

![0196f2c1-8302-7121-bffb-e24f98581afb_6_838_185_713_247_0.jpg](images/0196f2c1-8302-7121-bffb-e24f98581afb_6_838_185_713_247_0.jpg)

Fig. 4: TaintFuzzer cost function analysis in detecting all vulnerabilities.

![0196f2c1-8302-7121-bffb-e24f98581afb_6_841_494_710_223_0.jpg](images/0196f2c1-8302-7121-bffb-e24f98581afb_6_841_494_710_223_0.jpg)

Fig. 5: Performance evaluation of TaintFuzzer with feedback by varying ${f}_{f}$ .

Unknown Vulnerability Detection: Apart from the listed vulnerability mentioned in Table I, TaintFuzzer was able to find three unknown vulnerabilities in some experiments while targeting the known vulnerabilities in the AES-integrated Ariane SoC. After finding the vulnerability, we review the code and confirm the presence of the vulnerabilities in the RTL. The first unknown vulnerability (USV1) detected by TaintFuzzer suggests revealing of last encryption result (cipher) while taking new plaintext and cipher key as inputs, which may have a critical security impact when the adversarial process is also running in the same machine. The ciphertext from the last encryption operation should be flushed or erased when the machine takes new input, otherwise, it may cause data leakage or unauthorized access to confidential cipher. TaintFuzzer was able to detect this vulnerability since the third term in the cost function becomes significantly higher as there is no change on the output ciphertext although the plaintext (i.e., as the input) changes (hamming distance of previous and new ciphers becomes zero). Therefore, it significantly reduces the cost function, and a local minima of the cost function is found. The cost function variations for all unknown vulnerabilities with the corresponding targeted known vulnerabilities are shown in Fig. 6. TaintFuzzer detected another unknown vulnerability (USV2) in the AES, which suggests leaking intermediate encryption results as the ciphertext. While checking the integrity of ciphertext as a part of AES evaluation and cost function development for other known vulnerabilities, TaintFuzzer was able to detect it. This also causes a remarkable increase in the third term of the cost function as mentioned in Section III-C. During targeting known vulnerability SV1, TaintFuzzer generates random 32-bit instructions and executes them in SoC. TaintFuzzer was able to detect another unknown vulnerability (USV3) when it mutated ${MULH}$ instruction where the destination register(rd)becomes the same as either first source register (rs1) or second source register (rs2). According to RISC-V ISA [42], the decoder should throw an exception for the aforementioned illegal formation of ${MULH}$ instruction while executing in order to increase the performance of processors. As TaintFuzzer did not receive any exception, this clearly violates the ISA specification and therefore decreases the cost function (vulnerability detection) shown in Fig. 6.

![0196f2c1-8302-7121-bffb-e24f98581afb_6_843_1782_708_242_0.jpg](images/0196f2c1-8302-7121-bffb-e24f98581afb_6_843_1782_708_242_0.jpg)

Fig. 6: Cost function of detected unknown vulnerabilities by TaintFuzzer.

TABLE III: List of detected unknown vulnerabilities by TaintFuzzer.

<table><tr><td>Index</td><td>Vulnerability</td><td>Location</td><td>Triggering Condition</td><td>Output Behaviour</td><td>Reference</td></tr><tr><td>USV1</td><td>A Trojan allows retrieving last ciphertext of encryption operation for AES AE module in SoC</td><td>AES IP</td><td>Specific plaintext</td><td>Read last ciphertext</td><td>CWE-401</td></tr><tr><td>USV2</td><td>A Trojan releases intermediate result of encryption as final ciphertext</td><td>AES IP</td><td>Specific plaintext</td><td>Intermediate ciphertext</td><td>[3]</td></tr><tr><td>USV3</td><td>Decoder does not raise any exception while executing an illegal instruction</td><td>CPU (Dec)</td><td>( ${rd} = {rs1} \vee  {rd} = {rs2}$ ) in MULH</td><td>No exception raised</td><td>CWE-440</td></tr></table>

USV: Unknown Security Vulnerability.

Summary Results: We presented the summary of our experimental results with the optimum number of smart seeds and ${f}_{f}$ in detecting all vulnerabilities in Table IV. To avoid the case of sheer luck in fuzzing, we demonstrated the average results after running each experiment thrice. Table IV suggests that TaintFuzzer with taint inference-enabled cost function surpasses the performance as compared to fuzzing without the proposed feedback in terms of executions in detecting each of the known vulnerabilities. We observe TaintFuzzer required only a few iterations at the beginning to obtain the necessary smart seeds using Algorithm 1 from the provided initial seed, which is usually $\sim  6\mathrm{x}$ to $\sim  {12}\mathrm{x}$ of the number of smart seeds considered while targeting a vulnerability. TaintFuzzer with the feedback, saved at least ${32}\%$ of verification time compared with the framework without the proposed feedback (disabling CDCF feedback in TaintFuzzer, rather using the built-in feedback system of AFL). TaintFuzzer with the proposed cost function detected three unknown vulnerabilities, proving its capability in exploring more unknown bugs in the SoC.

Comparison with Prior Fuzzing Techniques: TaintFuzzer's strengths lie in its independence from golden model and white-box model, usages of taint inference leveraged smart seeds, dynamic cost function, and feedback, which can be integrated to any SW fuzzing engine. There are no HW-based fuzzing engines that support these features collectively as shown in Table

TABLE IV: Summary of verification results performed by TaintFuzzer.

<table><tr><td rowspan="2">Index</td><td rowspan="2">#SS</td><td rowspan="2">${f}_{f}$</td><td rowspan="2">Iterations for Smart Seeds</td><td colspan="2">No. of Iterations</td><td rowspan="2">Reduction of Verification Time</td></tr><tr><td>$T{F}_{NF}$</td><td>$T{F}_{CDCF}$</td></tr><tr><td>SV1</td><td>10</td><td>5</td><td>79</td><td>1610</td><td>629</td><td>56.02%</td></tr><tr><td>SV2</td><td>10</td><td>4</td><td>64</td><td>808</td><td>197</td><td>67.70%</td></tr><tr><td>SV3</td><td>10</td><td>6</td><td>88</td><td>4789</td><td>1945</td><td>57.55%</td></tr><tr><td>SV4</td><td>5</td><td>5</td><td>34</td><td>454</td><td>272</td><td>32.56%</td></tr><tr><td>SV5</td><td>5</td><td>5</td><td>31</td><td>413</td><td>181</td><td>48.69</td></tr><tr><td>SV6</td><td>20</td><td>7</td><td>243</td><td>13311</td><td>5261</td><td>58.65%</td></tr><tr><td>SV7</td><td>30</td><td>5</td><td>315</td><td>6063</td><td>2597</td><td>51.97%</td></tr><tr><td>USV1</td><td>20</td><td>7</td><td>225</td><td>N/A</td><td>10783</td><td>N/A</td></tr><tr><td>USV2</td><td>30</td><td>5</td><td>327</td><td>N/A</td><td>7311</td><td>N/A</td></tr><tr><td>USV3</td><td>10</td><td>5</td><td>83</td><td>N/A</td><td>1147</td><td>N/A</td></tr></table>

#SS: No. of Smart Seeds ${f}_{f}$ : Frequency of Feedback Evaluation.

$T{F}_{NF}$ : TaintFuzzer w/o proposed Feedback $T{F}_{CDCF}$ : TaintFuzzer w/ CDCF.

TABLE V: Comparison of TaintFuzzer with existing Fuzzing Frameworks. Emu: Emulation Sim: Simulation

<table><tr><td>Framework</td><td>Target Design</td><td>Approach</td><td>Feedback/Coverage</td><td>Auto- mation</td><td>Gray- box</td><td>Golden Model</td></tr><tr><td>HyPFuzz [16]</td><td>CPU</td><td>HDL Sim</td><td>Branch</td><td>No</td><td>No</td><td>Yes</td></tr><tr><td>TheHuzz [15]</td><td>CPU</td><td>HDL Sim</td><td>FSM, Statement</td><td>No</td><td>No</td><td>Yes</td></tr><tr><td>Hyper fuzzing [11]</td><td>SoC</td><td>SW Sim</td><td>NoC, Bitflip</td><td>No</td><td>Yes</td><td>Yes</td></tr><tr><td>DifuzzRTL [43]</td><td>CPU</td><td>FPGA Emu</td><td>Control-register</td><td>No</td><td>No</td><td>Yes</td></tr><tr><td>RFUZZ [10]</td><td>IP</td><td>FPGA Emu</td><td>MUX</td><td>No</td><td>No</td><td>Yes</td></tr><tr><td>Fuzzing HW as SW [14]</td><td>SoC</td><td>SW Sim</td><td>Branch/Code</td><td>Yes</td><td>No</td><td>Yes</td></tr><tr><td>TaintFuzzer</td><td>SoC</td><td>FPGA Emu</td><td>Cost Function</td><td>Yes</td><td>Yes</td><td>No</td></tr><tr><td>Emu: Emulation</td><td/><td>Sim: Simulation</td><td/><td/><td/><td/></tr></table>

V, which opens the possibility of utilizing it on large/complex SoCs with less scalability issues using our emulation-based framework with taint inference-enabled cost function. Again, TaintFuzzer verification platform is extremely faster due to leveraging FPGA emulation as compared to SW simulation [44] and does not incur any potential SW bugs [45] due to the translation of HW code to equivalent SW model.

## VI. CONCLUSION

In this paper, we proposed TaintFuzzer, a gray-box-based HW-oriented fuzzing-driven framework for SoC security verification. TaintFuzzer leverages the taint inference technique on inputs/outputs while being powered by advanced and efficient concepts for developing smart seeds, generating dynamic cost function, and feedback-based run-time correction of mutation strategies. All these features improve the performance/scalability of fuzzing in gray-box fashion. TaintFuzzer has been demonstrated to be capable and efficient in detecting SoC-level vulnerabilities including unknowns by mutating cornerstone inputs significantly faster.

[1] K. Z. Azar, M. M. Hossain, A. Vafaei, H. Al Shaikh, N. N. Mondol, F. Rahman, M. Tehranipoor, and F. Farahmandi, "Fuzz, penetration, and ai testing for soc security verification: Challenges and solutions," Cryptology ePrint Archive, 2022.

[2] P. Mishra, R. Morad, A. Ziv, and S. Ray, "Post-silicon validation in the soc era: A tutorial introduction," IEEE Design & Test, vol. 34, no. 3, pp. 68-92, 2017.

[3] N. Farzana, F. Rahman, M. Tehranipoor, and F. Farahmandi, "Soc security verification using property checking," in 2019 IEEE International Test Conference (ITC). IEEE, 2019, pp. 1-10.

[4] X. Meng, S. Kundu, A. K. Kanuparthi, and K. Basu, "Rtl-contest: Concolic testing on rtl for detecting security vulnerabilities," IEEE Transactions on Computer-Aided Design of Integrated Circuits and Systems, vol. 41, no. 3, pp. 466-477, 2021.

[5] C. Kern and M. R. Greenstreet, "Formal verification in hardware design: a survey," ACM Transactions on Design Automation of Electronic Systems (TODAES), vol. 4, no. 2, pp. 123-193, 1999.

[6] W. Hu, D. Mu, J. Oberg, B. Mao, M. Tiwari, T. Sherwood, and R. Kastner, "Gate-level information flow tracking for security lattices," ACM Transactions on Design Automation of Electronic Systems (TODAES), vol. 20, no. 1, pp. 1-25, 2014.

[7] A. Ardeshiricham, W. Hu, J. Marxen, and R. Kastner, "Register transfer level information flow tracking for provably secure hardware design," in Design, Automation & Test in Europe Conference & Exhibition (DATE), 2017. IEEE, 2017, pp. 1691-1696.

[8] W. Hu, A. Ardeshiricham, and R. Kastner, "Hardware information flow tracking," ACM Computing Surveys (CSUR), vol. 54, no. 4, pp. 1-39, 2021.

[9] W. Hu, X. Wang, and D. Mu, "Security path verification through joint information flow analysis," in 2018 IEEE Asia Pacific Conference on Circuits and Systems (APCCAS). IEEE, 2018, pp. 415-418.

[10] K. Laeufer, J. Koenig, D. Kim, J. Bachrach, and K. Sen, "Rfuzz: Coverage-directed fuzz testing of rtl on fpgas," in 2018 IEEE/ACM International Conference on Computer-Aided Design (ICCAD). ACM, 2018, pp. 1-8.

[11] S. K. Muduli, G. Takhar, and P. Subramanyan, "Hyperfuzzing for soc security validation," in Proceedings of the 39th International Conference on Computer-Aided Design, 2020, pp. 1-9.

[12] S. Canakci, L. Delshadtehrani, F. Eris, M. B. Taylor, M. Egele, and A. Joshi, "Directfuzz: Automated test generation for rtl designs using directed graybox fuzzing," in 2021 58th ACM/IEEE Design Automation Conference (DAC). IEEE, 2021, pp. 529-534.

[13] J. Hur, S. Song, D. Kwon, E. Baek, J. Kim, and B. Lee, "Difuzzrtl: Differential fuzz testing to find cpu bugs," in 2021 IEEE Symposium on Security and Privacy (SP). IEEE, 2021, pp. 1286-1303.

[14] T. Trippel, K. G. Shin, A. Chernyakhovsky, G. Kelly, D. Rizzo, and M. Hicks, "Fuzzing hardware like software," in 31st USENIX Security Symposium (USENIX Security 22), 2022, pp. 3237-3254.

[15] R. Kande, A. Crump, G. Persyn, P. Jauernig, A.-R. Sadeghi, A. Tyagi, and J. Rajendran, "\{TheHuzz\}: Instruction fuzzing of processors using \{Golden-Reference\} models for finding \{Software-Exploitable\} vulnerabilities," in 31st USENIX Security Symposium (USENIX Security 22), 2022, pp. 3219-3236.

[16] C. Chen, R. Kande, N. Nyugen, F. Andersen, A. Tyagi, A.-R. Sadeghi, and J. Rajendran, "Hypfuzz: Formal-assisted processor fuzzing," 2023.

[17] X. Yang, Y. Chen, E. Eide, and J. Regehr, "Finding and understanding bugs in c compilers," in Proceedings of the 32nd ACM SIGPLAN conference on Programming language design and implementation, 2011, pp. 283-294.

[18] M. Zalewski, "American Fuzzy Lop," https://lcamtuf.coredump.cx/afl/.

[19] K. Serebryany, "Continuous fuzzing with libfuzzer and addresssanitizer," in 2016 IEEE Cybersecurity Development (SecDev). IEEE, 2016, pp. 157-157.

[20] K. Serebryany, M. Lifantsev, K. Shtoyk, D. Kwan, and P. Hochschild, "Silifuzz: Fuzzing cpus by proxy," arXiv preprint arXiv:2110.11519, 2021.

[21] S. Gan, C. Zhang, P. Chen, B. Zhao, X. Qin, D. Wu, and Z. Chen, "Greyone: Data flow sensitive fuzzing." in USENIX Security Symposium, 2020, pp. 2577-2594.

[22] S. Rawat, V. Jain, A. Kumar, L. Cojocar, C. Giuffrida, and H. Bos, "Vuzzer: Application-aware evolutionary fuzzing." in NDSS, vol. 17, 2017, pp. 1-14.

[23] N. Farzana, A. Ayalasomayajula, F. Rahman, F. Farahmandi, and M. Tehranipoor, "Saif: Automated asset identification for security verification at the register transfer level," in 2021 IEEE 39th VLSI Test Symposium (VTS). IEEE, 2021, pp. 1-7.

[24] X. Guo, R. G. Dutta, P. Mishra, and Y. Jin, "Scalable soc trust verification using integrated theorem proving and model checking," in 2016 IEEE International Symposium on Hardware Oriented Security and Trust (HOST). IEEE, 2016, pp. 124-129.

[25] R. Zhang, C. Deutschbein, P. Huang, and C. Sturton, "End-to-end automated exploit generation for validating the security of processor designs," in 2018 51st Annual IEEE/ACM International Symposium on Microarchitecture (MICRO). IEEE, 2018, pp. 815-827.

[26] W. Hu, A. Ardeshiricham, and R. Kastner, "Identifying and measuring security critical path for uncovering circuit vulnerabilities," in 2017 18th International Workshop on Microprocessor and SOC Test and Verification (MTV). IEEE, 2017, pp. 62-67.

[27] C. Spear, SystemVerilog for verification: a guide to learning the testbench language features. Springer Science & Business Media, 2008.

[28] Cadence, "Jaspergold formal verification." [Online]. Available: https://www.cadence.com/en_US/home/tools/system-design-and-verification/formal-and-static-verification/jasper-gold-verification-platform.html

[29] M. M. Hossain, F. Farahmandi, M. Tehranipoor, and F. Rahman, "Boft: Exploitable buffer overflow detection by information flow tracking," in 2021 Design, Automation & Test in Europe Conference & Exhibition (DATE). IEEE, 2021, pp. 1126-1129.

[30] B. P. Miller, L. Fredriksen, and B. So, "An empirical study of the reliability of unix utilities," Communications of the ACM, vol. 33, no. 12, pp. 32-44, 1990.

[31] P. Godefroid, M. Y. Levin, D. A. Molnar et al., "Automated whitebox fuzz testing." in NDSS, vol. 8, 2008, pp. 151-166.

[32] "Verilator." [Online]. Available: https://www.veripool.org/verilator/

[33] M. M. Hossain, A. Vafaei, K. Z. Azar, F. Rahman, F. Farahmandi, and M. Tehranipoor, "Socfuzzer: Soc vulnerability detection using cost function enabled fuzz testing," in 2023 Design, Automation & Test in Europe Conference & Exhibition (DATE). IEEE, 2023, pp. 1-6.

[34] "Trust-hub vulnerability database for soc." [Online]. Available: https://trust-hub.org/#/vulnerability-db/soc-vulnerabilities

[35] N. Farzana, F. Farahmandi, and M. Tehranipoor, "Soc security properties and rules," Cryptology ePrint Archive, 2021.

[36] A. Herrera, H. Gunadi, S. Magrath, M. Norrish, M. Payer, and A. L. Hosking, "Seed selection for successful fuzzing," in Proceedings of the 30th ACM SIGSOFT International Symposium on Software Testing and Analysis, 2021, pp. 230-243.

[37] "Risc-v arian soc." [Online]. Available: https://github.com/openhwgroup/cva6

[38] MITRE, "HW CWEs," https://cwe.mitre.org/data/definitions/1194.html.

[39] [Online]. Available: https://www.xilinx.com/products/intellectual-property/ila.html

[40] "Genesys 2 fgpga development board." [Online]. Available: https://digilent.com/reference/programmable-logic/genesys-2/start

[41] A. Waterman, Y. Lee, D. A. Patterson, and K. Asanovi, "The risc-v instruction set manual. volume 1: User-level isa, version 2.0," California Univ Berkeley Dept of Electrical Engineering and Computer Sciences, Tech. Rep., 2014.

[42] "Risc-v." [Online]. Available: https://riscv.org/

[43] S. Nilizadeh, Y. Noller, and C. S. Pasareanu, "Diffuzz: differential fuzzing for side-channel analysis," in 2019 IEEE/ACM 41st International Conference on Software Engineering (ICSE). IEEE, 2019, pp. 176-187.

[44] "When to use simulation, when to use emulation." [Online]. Available: https://www.electronicproducts.com/when-to-use-simulation-when-to-use-emulation/

[45] "Software bug." [Online]. Available: https://github.com/verilator/verilator/issues/1533