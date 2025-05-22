# SHarPen: SoC Security Verification by Hardware Penetration Test

Hasan Al-Shaikh, Arash Vafaei, Mridha Md Mashahedur Rahman,

Kimia Zamiri Azar, Fahim Rahman, Farimah Farahmandi, Mark Tehranipoor

The Department of Electrical & Computer Engineering, University of Florida, Gainesville, FL, USA.

\{hasanalshaikh, arash.vafaei, mrahman1, k.zamiriazar\}@ufl.edu, \{fahimrahman, farimah, tehranipoor\}@ece.ufl.edu

## ABSTRACT

As modern SoC architectures incorporate many complex/heterogeneous intellectual properties (IPs), the protection of security assets has become imperative, and the number of vulnerabilities revealed is rising due to the increased number of attacks. Over the last few years, penetration testing (PT) has become an increasingly effective means of detecting software (SW) vulnerabilities. As of yet, no such technique has been applied to the detection of hardware vulnerabilities. This paper proposes a PT framework, SHarPen, for detecting hardware vulnerabilities, which facilitates the development of a SoC-level security verification framework. SHarPen proposes a formalism for performing gray-box hardware (HW) penetration testing instead of relying on coverage-based testing and provides an automation for mapping hardware vulnerabilities to logical/- mathematical cost functions. SHarPen supports both simulation and FPGA-based prototyping, allowing us to automate security testing at different stages of the design process with high capabilities for identifying vulnerabilities in the targeted SoC.

## KEYWORDS

SoC Security Verification, Penetration Testing, BPSO, Cost Function.

## ACM Reference Format:

Hasan Al-Shaikh, Arash Vafaei, Mridha Md Mashahedur Rahman,, Kimia Zamiri Azar, Fahim Rahman, Farimah Farahmandi, Mark Tehranipoor. 2023. SHarPen: SoC Security Verification by Hardware Penetration Test. In 28th Asia and South Pacific Design Automation Conference (ASPDAC '23), January 16-19, 2023, Tokyo, Japan. ACM, New York, NY, USA, 6 pages. https://doi.org/10.1145/3566097.3567918

## 1 INTRODUCTION

With the continuous growth of the SoC market coupled with increasingly rigid time-to-market (TTM), an increasing trend of reusing third parties' intellectual property (3PIPs), and the globalization of the integrated circuits (IC) supply chain, multiple untrustworthy entities are introduced into the IC supply chain. So, there is a growing possibility of introducing more vulnerabilities in the SoC architecture with irreparable damage [18]. It is crucial to detect these vulnerabilities at the earliest possible IC design stage, and ideally, before prototyping, to minimize the need for re-spins. Since the SoCs architecture is becoming increasingly complex and

difficult to test/verify, security vulnerabilities are arising at a significantly higher rate, which originates from a greater variety of sources, including: (1) designer errors during different stages of the IC supply chain, (2) the re-use of IPs from untrusted 3PIP vendors, (3) rogue employees within design houses (insider threats), and (4) a lack of security awareness in EDA tools. Since these vulnerabilities originate from unexpected SoC-level behavior, none of the existing security countermeasures, including logic obfuscation, watermarking, metering, etc. [6, 16], are able to cope with them.

Numerous studies have investigated the efficacy of various methods for IP/SoC-level security verification, such as formal-based verification $\left\lbrack  {4,{17},{28}}\right\rbrack$ and information flow tracking (IFT) for security validation [20, 27]. However, the efficacy of these methods is low as (1) the definition of security policies relies on the verification engineers' expertise, (2) the policy generation is mostly done manually, (3) a detailed understanding of the design is necessary to have accurate policies, and (4) they often suffer from excessive and unwanted computation, resulting in low scalability. So, more recent studies aim to overcome the accuracy/scalability using evolutionary mechanisms. Fuzz testing is one of the most recent evolutionary techniques $\left\lbrack  {3,{24},{26}}\right\rbrack$ that enables SoC-level verification. It, however, mainly focuses on increasing the coverage, which isn't directly related to a significant portion of IP-based and SoC-based vulnerabilities. Hence, the majority of them are still unable to automatically identify vulnerabilities, and the identifications are mostly the result of manual inspections of fuzzer behavior.

Similar to fuzzing, penetration test (PT) is another evolutionary mechanism with the capability of self-refinement for verification purposes, mostly involving simulating attacks on the software or networks [5]. The high efficacy of PT in detecting unforeseen bugs and vulnerabilities in networks and software has led to robust standardization and guidelines by various commercial (CREST), non-profit (MITRE, OWASP), and governmental organizations (e.g., OSTMM). However, due to intrinsic differences between the vulnerabilities in HW vs. those in software (e.g., a segmentation fault in SW vs. illegal access to HW), migrating the testing techniques from SW to HW requires careful consideration, formulation, and adaptation. The lack of a well-defined PT infrastructure for hardware has led to studies collectively referring to PT as an umbrella term, without any clear delineation of the steps to follow [13]. Moreover, to the best of our knowledge, no study has been conducted to compare the efficacy of gray-box PT vs. white-box PT in HW.

In this paper, we propose SHarPen to put forth the first precise methodology for gray-box pre-silicon HW PT framework compatible with both RTL simulation and FPGA-based prototyping. We examine the efficacy of SHarPen on multiple forms of vulnerabilities ranging from malicious leakage of information to denial of service. The main contributions of this work are as follows: (1) We formalize a gray-box HW PT framework and justify its efficacy in detecting pre-silicon vulnerabilities.

---

Permission to make digital or hard copies of all or part of this work for personal or classroom use is granted without fee provided that copies are not made or distributed for profit or commercial advantage and that copies bear this notice and the full citation on the first page. Copyrights for components of this work owned by others than ACM must be honored. Abstracting with credit is permitted. To copy otherwise, or republish, to post on servers or to redistribute to lists, requires prior specific permission and/or a fee. Request permissions from permissions@acm.org.

ASPDAC '23, January 16-19, 2023, Tokyo, Japan

© 2023 Association for Computing Machinery.

ACM ISBN 978-1-4503-9783-4/23/01...\$15.00

https://doi.org/10.1145/3566097.3567918

---

(2) SHarPen automates the generic specification to generic cost function transformation, paving the way for Binary Particle Swarm Optimization (BPSO) to be used for security verification purposes. (3) The proposed PT framework provides compatibility with both behavioral (RTL) simulation and FPGA-based prototyping.

(4) We also provide a detailed experimental analysis of the proposed cost functions to demonstrate how PT can be effectively invoked to automate the detection of security vulnerabilities.

## 2 BACKGROUND

Security assets are information in the design, e.g., sensitive user data, TRNG output, and encryption key, whose leakage may lead to economic/credibility losses for the manufacturer. To avoid such incidents, as a set of rules, security policies or properties will be defined that must be met for security requirements $\left\lbrack  {{12},{25}}\right\rbrack$ , and the adherence of a device to these policies will be confirmed by security verification. HW-based security verification is inherently more challenging (than functional verification) as security policies can be violated through compromise of HW, FW, or SW. In addition, there may be numerous sources of security vulnerabilities, including the malicious modification of functionality by an insider, weak implementations of specifications, security-unaware optimizations of CAD tools, and end users' malicious actions.

In formal-based verification techniques [4, 28], verification engineers are responsible for specifying how security policies and properties are to be implemented (e.g., via SystemVerilog assertions) for identifying the vulnerabilities. The expertise and meticulousness of the verification engineers play a crucial role in the transformation of the specification and security policies into effective outcomes [17]. Formal-based techniques, therefore, require comprehensive detailed knowledge, also known as white-box models [8], and even in the white-box model, they cannot discover unknown cases [1, 22]. An example is Augury, a recent attack reported on Apple M series and A14 SoCs that exploits data memory prefetcher (DMP) rather than speculative execution to leak sensitive information [10]. Similar to formal-based approaches, in symbolic execution and concolic testing $\left\lbrack  {2,{21}}\right\rbrack$ , assertions must be crafted into the design, and as such they require the white-box model. Since they aim local points in processors, they may rely on a uniform branch selection strategy, which can suffer from path explosion problems.

Table 1: Comparison of SoC Security Verification Techniques

<table><tr><td>Method</td><td>Model</td><td>Limit</td><td>Scale</td><td>Automation</td></tr><tr><td>Formal [9, 17]</td><td>white-box (Manual)</td><td>False positive, state explo- sion, low accuracy</td><td>low</td><td>low</td></tr><tr><td>Concolic [2]</td><td>white-box (Manual)</td><td>Trojan only, path explo- sion, low accuracy</td><td>low</td><td>moderate</td></tr><tr><td>Symbolic [21]</td><td>white-box (Manual)</td><td>Only processor-level</td><td>low _____</td><td>moderate</td></tr><tr><td>Genetic [14]</td><td>white-box</td><td>Trojan only, reliance on structural features</td><td>low</td><td>moderate</td></tr><tr><td>ML [29]</td><td>white-box</td><td>Trojan only, reliance on structural features</td><td>high _____</td><td>moderate</td></tr><tr><td>Fuzzing $\left\lbrack  {3,{23}}\right\rbrack$</td><td>gray-box (Branch Signals Instrument)</td><td>metrics</td><td>high</td><td>high</td></tr></table>

![0196f75f-ffb0-72a6-8daf-bb4df53f43dc_1_928_239_714_193_0.jpg](images/0196f75f-ffb0-72a6-8daf-bb4df53f43dc_1_928_239_714_193_0.jpg)

Figure 1: The Proposed SHarPen Framework Overall Architecture.

In IFT $\left\lbrack  {{20},{27}}\right\rbrack$ , the verification engine is built on a label-based propagation, in which, based on the policy/property, a set of input signals will be labeled and based on the propagation of the labels detection of vulnerabilities will be accomplished. Similar to formal-based techniques, IFT suffers from excessive and unwanted propagation, resulting in poor accuracy and limited scalability.

Fuzz testing is an evolutionary mechanism with the capability of self-refinement for hardware vulnerability detection [3, 11, 24, 26]. All fuzzing techniques on hardware utilize coverage-based testing. However, as the nature of errors, bugs, and vulnerabilities in hardware differs from that in software (e.g., a segmentation fault in SW and illegal access to HW), coverage-based techniques cannot accurately model all vulnerabilities in an SoC platform. Additionally, they suffer from other weaknesses, e.g., the inability to support HDLs commonly used, the reliance on designers' expertise, and the inability to identify vulnerabilities at multiple abstraction levels.

Table 1 compares the existing methodologies. Compared to these techniques, our proposed BPSO-based PT technique relies on a precise set of prerequisites which gives enough leeway on strict knowledge requirements, does not rely on structural features or coverage metrics, and offers even greater scalability/automation.

## 3 PROPOSED FRAMEWORK: SHarPen

Unlike almost all existing verification methods, which rely on coverage metrics ${}^{1}$ , in SHarPen, we propose generic cost functions, whose goal is to find how or when the system may not follow the security rules ${}^{2}$ . The cost functions in SHarPen follow the gray-box model and require minimal high-level knowledge from the verification engineer with no dependency on the circuit structures, which makes the flow scalable and adaptable to a wide variety of circuit designs. Figure 1 shows the top view of the SHarPen framework. The SHarPen framework automates security rule checks to logical/mathematical transformation, and by doing so, it invokes BPSO for accomplishing automated PT on direct hardware. The following demonstrates how formalism is built in SHarPen, how cost functions are automating the process of testing, and how the framework is implemented.

### 3.1 Formalism of PT in Hardware

In general, PT, also colloquially referred to as ethical hacking, identifies corner cases and unforeseen loopholes that an attacker might exploit to compromise the security of the device. In order to achieve this, our methodology uses a metaheuristic optimization algorithm. SHarPen, by relying on the following proposition:

Proposition ${P}_{1} : {HW}$ vulnerabilities, either IP-level or SoC-level, can be described as mathematical representations such that finding their optima will lead to their detection.

---

${}^{1}$ The goal is to find a positive, i.e. achieve a satisfactory ratio of the upper bound of a prescribed metric.

${}^{2}$ The goal is to find a negative which does not have a quantifiable upper bound.

---

The Particle Swarm Optimization (PSO) and its variants, as a class of evolutionary optimization algorithms, meta-heuristically find the global optima of a mathematical function. In the standard PSO [7], for a D-dimensional feature space, PSO describes the ${i}^{th}$ particle as ${x}_{i} = \left\lbrack  {{x}_{i1},{x}_{i2},{x}_{i3},\ldots ,{x}_{iN}}\right\rbrack$ , in which, ${x}_{{i1}..N}$ are real valued quantities, and the swarm is the collection of $\mathrm{N}$ such particles as $X = \left\lbrack  {{x}_{1},{x}_{2},\ldots ,{x}_{N}}\right\rbrack$ , in which ${x}_{i}$ is called the position of the article. In this case, the ultimate goal of PSO is to find the best position (min/max of the function). To accomplish that, according to the equations 1 and 2, the position of each particle (for specific numbers of iterations) will be updated:

$$
{v}_{i, j}\left( {t + 1}\right)  = w{v}_{i, j}\left( t\right)  + {c}_{1}{R}_{1}\left( {{p}_{\text{best }i, j} - {x}_{i, j}\left( t\right) }\right)  + {c}_{2}{R}_{2}\left( {{g}_{\text{best }i, j} - {x}_{i, j}\left( t\right) }\right)  \tag{1}
$$

$$
{x}_{i, j}\left( {t + 1}\right)  = {x}_{i, j}\left( t\right)  + {v}_{i, j}\left( {t + 1}\right)  \tag{2}
$$

${R}_{1}$ and ${R}_{2}$ are $\left\lbrack  {0,1}\right\rbrack$ uniformly distributed real random numbers. ${c}_{1}$ and ${c}_{2}$ denote acceleration coefficients and $w$ is termed as positive inertia constant. ${v}_{i, j}\left( t\right)$ is the velocity at ${\mathrm{j}}^{\text{th }}$ index in the ${\mathrm{i}}^{\text{th }}$ particle of the swarm at iteration $t$ . The ${p}_{\text{best }}$ and ${g}_{\text{best }}$ stand for so-called personal best and global best. So, the updates are according to each particle's own experience as well as the global best experience.

A simple modification on this model makes it binarized, which is suitable for solving problems in binary space [7], known as ${\mathrm{{BPSO}}}^{3}$ . To map all real valued quantities into binary space, the following constraint must be applied to eq. (2).

$$
{x}_{i, j}\left( {t + 1}\right)  = 0\text{[if}\operatorname{rand}\left( \right)  \geq  S\left( {{v}_{i, j}\left( {t + 1}\right) }\right) \rbrack \;\text{or 1 [otherwise]} \tag{3}
$$

where $S\left( \text{.}\right) {isthesigmoidfunctionusedforvelocitytransforming}.$ As all interaction signaling in SW/FW/HW can be described using binary bit vectors, we use this binarized PSO (BPSO) in $S{\text{HarPen}}^{4}$ .

### 3.2 Cost Function Formalism in SHarPen

Based on proposition ${P}_{1}$ , the cost function is the mathematical model (function) that needs to be optimized for detecting the vulnerabilities. In SHarPen, we propose the generic gray-box cost function $\mathbb{F}$ as $\mathbb{F} = \mathop{\sum }\limits_{{i = 1}}^{N}{\alpha }_{i}g\left( p\right)  + \sum \mathcal{G}$ , in which $g : \mathcal{P} \rightarrow  f$ and $f : {\mathcal{Z}}_{2} \rightarrow  \mathbb{R}$ . Here, $\mathcal{P} \subset  \mathbb{P}$ are the sub-set of policies for the targeted design, $f$ is a function with the range of real numbers $\mathbb{R}$ and ${\mathcal{Z}}_{2}$ is the set of binary strings that enumerates all possible values of the relevant observable point in the design ${}^{5} \cdot  \mathbb{P}$ is the set of all security policies of the device. The set $\mathbf{\alpha } = \left\{  {{\alpha }_{1},{\alpha }_{2},{\alpha }_{3},\ldots ,{\alpha }_{N}}\right\}$ is a set of constants which consists of tester configurable positive real numbers, and $\mathcal{G}$ are a set of optional optimizing functions that allows for quicker convergence. To form the subset $\mathcal{P}$ and ${\mathcal{Z}}_{2}$ , we suppose a set of prerequisites are met which are as follows:

Prerequisite $P{R}_{1}$ : The tester possesses a high-level knowledge of potential vulnerabilities (violations of $\mathcal{P}$ ) and their high-level impact on neighboring signals/modules. Here is an example of what being high-level means in SHarPen. For instance, the AES key in the SoC must not be leaked to untrusted regions. In this case, the neighboring signals/modules are the associated read and write channel of the shared bus among the trusted and untrusted IPs ${}^{6}$ .

Table 2: Targeted Vulnerabilities inserted in Ariane SoC.

<table><tr><td>Index</td><td>Description</td><td>SRV</td><td>Reference</td></tr><tr><td>SV1</td><td>A Trojan leaks the AES secret key through the common bus in the SoC</td><td>C</td><td>AES-T1300, BASICRSA-T100, CVE-2018-8933</td></tr><tr><td>SV2</td><td>A Trojan injects delay in the AES IP (certain # of clock cycles.</td><td>A</td><td>AES-T500, BASICRSA-T200, BASICRSA-T400</td></tr><tr><td>SV3</td><td>SATP register read/write allowed in su- pervisor mode even when the TVM bit is set in mstatus register.</td><td>I AC</td><td>CVE-2018-7522</td></tr><tr><td>SV4</td><td>Execution of WFI instruction possible (or does not complete within bounded time) in supervisor mode even when TW bit is set in mstatus register.</td><td>AC, A</td><td>CVE-2018-18068</td></tr><tr><td>SV5</td><td>Faulty implementation of password check logic in debug module</td><td>A, I</td><td>CVE-2018-18347</td></tr><tr><td colspan="2">SV: Security Vulnerability C: ConfidentialityA: Availability</td><td>I: Integrity</td><td>SRV: Security Requirement Violated AC: Access Control</td></tr></table>

Prerequisite $P{R}_{2}$ : The tester can observe a sub-set of signals of the design that might be impacted by the vulnerabilities (gray-box). Prerequisite $P{R}_{3}$ : The tester can control a sub-set of relevant signals of the design to the vulnerabilities (gray-box).

## 4 IMPLEMENTATION DETAILS

The vulnerabilities to be evaluated on SHarPen are based on well-studied cases reported in Mitre CWE and hackathons, all tested on the Ariane SoC. Table 2 lists these vulnerabilities for the evaluation of SHarPen. SV1/2 is related to malicious modification of the crypto core of the SoC. SV1 may lead to illegal access to the security-critical asset (AES key). SV2 may lead to denial-of-service across the whole SoC. These HW Trojans may be introduced to the SoC through a 3P crypto IP procured from an untrusted vendor. SV3/4, on the other hand, is related to privilege level specifications described in RISC-V ISA, and these privilege (Machine (M), Supervisor (S), and User (U)) levels are used to provide proper execution environment isolation of the software stack. Specifically, SV3 may be exploited to cause invalid or major page faults in the processor, and SV4 can be exploited to enable an unauthorized processor halt which can result in DoS attacks. SV5 is another vulnerability that can be leveraged to gain access to debug facilities otherwise protected by password check mechanism. The password protection can be skipped and the password itself can be guessed because of SV5's existence which violates the integrity and availability of design assets. SV3/4/5 may be introduced through designer mistakes in the HDL code.

### 4.1 Cost Function per SV

To have a more comprehensive set of generic cost functions that will be tested by BPSO, the generic definition of SVs is a crucial pre-processing step in SHarPen. With more accumulated SVs, which eventually results in having a wider variety of generic cost functions, the automation of SoC security verification promised by SHarPen becomes more extensive as it can evaluate more scenarios to determine which vulnerabilities may be exploitable. Assuming that these SVs are defined meticulously, by following the Mitre CWE database and hackathons, and by using the generic gray-box cost function $\mathbb{F}$ , automation can be completed for penetration testing. Note that based on the Prerequisites $P{R}_{1,2,3},{SHarPen}$ can easily be upgraded in an automatic manner for covering more and more SVs.

---

${}^{3}$ BPSO has been utilized in RTL verification purposes previously[19], to the best of our knowledge, it has never been used for hardware security verification purposes.

${}^{4}$ All inter-IP signals as well as firmware and user space programs dissolved into assembly are binary vectors, and can be modeled using BPSO.

${}^{5}g\left( p\right)$ is the function that defines what set of inputs $\left( {\mathcal{Z}}_{2}\right)$ can lead to invalidating (global optima) of the policies(P).

${}^{6}{\mathrm{{PR}}}_{1}$ can be reasonably fulfilled for a large number of pre-silicon vulnerabilities as the reference dataset of documented common vulnerabilities are getting more and more populated (e.g., TrustHub (trust-hub.org), CWE (cwe.mitre.org)).

---

Here we define how generic gray-box cost function $\mathbb{F}$ will be built for SVs listed in Table 2 by following Prerequisites $P{R}_{1,2,3}$ . $\mathbb{F}\left( {{SV1}\& {SV2}}\right)  : \left( 1\right) \left( {P{R}_{2}/P{R}_{3}}\right)$ : The tester has access to the AES IP key/plaintext input, handshaking signal (valid_in/valid_out), and ciphertext output; (2) $\left( {P{R}_{1}}\right)$ No AES key leakage through the shared bus (preventing AES key to be on AXI read channel of AES and ${AXI}$ write channel of other IPs); (3) $\left( {P{R}_{1}}\right)$ No AES key leakage through unprotected memory locations; (4) $\left( {P{R}_{1}}\right)$ Maximum no. of clock cycles allowed for the assertion of valid_out; (5) $\left( {P{R}_{1}}\right)$ Tester assumes that a specific k -bit character in the plaintext can potentially trigger a Trojan. Based on (5), mutation of k -bit and its position will be done by BPSO-based PT. Accordingly, the generic cost function $\mathbb{F}$ can be written as eq. 4.

$$
\mathbb{F}\left( {{SV1}\& {SV2}}\right)  = {\alpha }_{1}{HD}\left( {{r}_{\text{data }},{SC}}\right)  + {\alpha }_{2}{HD}\left( {\text{ vo }\_ {clk},0}\right)
$$

$$
+ {\alpha }_{3}\mathop{\sum }\limits_{{j = 1}}^{M}{HD}\left( {{w}_{\text{data }},{SC}}\right)  + {\alpha }_{4}\mathop{\sum }\limits_{{j = 1}}^{K}{HD}\left( {{me}{m}_{\text{data }},{SC}}\right)  - \beta  \tag{4}
$$

In eq. 4, ${HD}$ : hamming distance, ${r}_{\text{data }}$ : read data signal (AES read channel), ${SC}$ : secret key used for encryption, vo_clk: valid_out signal after a certain no. of clock cycles, ${w}_{\text{data }}$ : write data signal (other IPs’ write channel), $M$ : Number of IPs in the design, $K$ : Number of memory locations accessible by IPs, $\beta$ : Tester configurable constant parameter, and ${\alpha }_{i}$ : tester configurable parameters.

Now, by converting the logical propositions (policies) to mathematical formulas, which is the ultimate goal of SHarPen, it may be possible to utilize BPSO-based PT to seek out test cases that result in the vulnerability being exploited. Using this form, if there is a violation of stated security policy (due to $\mathbf{{SV}}1/2$ ), the value of $\mathbb{F}\left( {{SV1}\& {SV2}}\right)$ would be at the global minima of $- \beta$ . Note that when any one (or more) of the policy is violated, at least one of the terms evaluates to 0 ; subsequently all others are set to zero as well.

It is highly worth mentioning that SHarPen builds these equations, like eq. 4, for the BPSO at the host system, and then the only requirement is to provide these signals' activities to the SHarPen framework. Additionally, SHarPen includes many factors per each vulnerability that might be impacted, e.g., by the existence of trojan, instead of assuming which will be impacted. Also, in SHarPen, no structural details about the AES or SoC $\mu$ architecture is assumed. Together, the preceding two facts relieve the tester from the responsibility of possessing intricate knowledge of the system. This is why we call this type of cost function a gray-box cost function(s), which generically can be considered as part of SHarPen framework. $\mathbb{F}\left( {{SV3}\& {SV4}}\right)  : \left( 1\right) \left( {P{R}_{1}}\right)$ The design is a 64-bit RISC-V ISA compliant SoC (e.g., v1.10) with three privilege levels (M/S/U); (2) (PR1) While in supervisor mode, attempts to read or write the SATP control and status register (CSR) or to execute the SFENCE. VMA is illegal if TVM in mstatus is set; $\left( 3\right) \left( {P{R}_{1}}\right)$ While in supervisor mode, attempts to execute the SRET instruction should raise an illegal instruction exception if the Trap SRET (TSR) bit in mstatus register is set. (4) $\left( {P{R}_{2}/P{R}_{3}}\right)$ The tester can use an user space assembly code to trigger this vulnerability. (5) $\left( {P{R}_{2}/P{R}_{3}}\right)$ The tester can set the configuration bits in e.g. mstatus, and use candidate processor instructions, e.g., SFENCE. VMA, WFI, SRET, and CSR read/write. So, the generic cost function $\mathbb{F}$ can be written as eq. ${5}^{7}$ , where ${I}_{ex}$ : Illegal instruction exception raised, $\mathcal{H}$ : the Heaviside step function ${}^{8}, T$ : load immediate instruction, ${I}_{x}$ : Binary encoding of assembly instruction $x, S$ : configuration bits (TVM, TW and TSR) of mstatus, $U :$ CSR read/write instruction, $W$ : candidate processor instructions, ${I}_{x}{rd}/{rs} : {rd}/{rs}$ of binary encoding of instruction $x$ .

$$
\mathbb{F}\left( {{SV3}\& {SV4}}\right)  = {\alpha }_{1}{I}_{ex} + {\alpha }_{2}\mathcal{H}\left( {-\mathop{\sum }\limits_{{j \in  T}}{HD}\left( {{I}_{1}, j}\right) }\right)  + {\alpha }_{3}\mathcal{H}\left( {-\mathop{\sum }\limits_{{i \in  S}}{HD}\left( {i,1}\right) }\right)
$$

$$
+ {\alpha }_{4}\mathcal{H}\left( {-\mathop{\sum }\limits_{{k \in  U}}{HD}\left( {{I}_{2}, k}\right) }\right)  + {\alpha }_{5}{HD}\left( {{I}_{2rd},{0x300}}\right)  \tag{5}
$$

$$
+ {\alpha }_{6}\mathcal{H}\left( {-\mathop{\sum }\limits_{{m \in  W}}{HD}\left( {{I}_{3}, m}\right) }\right)  + {\alpha }_{7}{HD}\left( {{I}_{{3rd}/{rs}},{0x180}}\right)
$$

![0196f75f-ffb0-72a6-8daf-bb4df53f43dc_3_929_238_713_271_0.jpg](images/0196f75f-ffb0-72a6-8daf-bb4df53f43dc_3_929_238_713_271_0.jpg)

Figure 2: SHarPen through Software Model Simulation.

Similar to $\mathbb{F}\left( {{SV1}\& {SV2}}\right)$ , the cost function $\mathbb{F}\left( {{SV3}\& {SV4}}\right)$ will evaluate to 0 (global minima) when either SV3 or SV4 exists. Please note that the evaluation is at the user space instruction level, showing how it can be modeled and tested on any RISC-V SoC without relying on the intricate knowledge of $\mu$ -architectural details. This is how SHarPen converts test scenarios to mathematical relations and tests them with the BPSO-based PT framework.

$\mathbb{F}\left( {SV5}\right)$ : With JTAG debug, we can read and write to the Data Register (DR) through JTAG signals by writing the proper instructions to the Instruction Register (IR) of the JTAG TAP controller. The external debugger can access RISC-V debug modules and memory space through specific registers. Shifting in values of address, data and operation will activate bus transactions, and response messages from these transactions will give SHarPen the required feedback information. The debug module password check logic may $\left( 1\right) \left( {P{R}_{1}}\right)$ work for only one of the read and write operations; (2) $\left( {P{R}_{1}}\right)$ work for only a portion of the full password; (3) $\left( {P{R}_{1}}\right)$ not been properly initialized after the reset; (4) $\left( {P{R}_{2}/P{R}_{3}}\right)$ provide access to debug resources for an unauthorized user or even escalated levels of access for authorized ones holding the correct password. The general cost function can accordingly be written as:

$$
\mathbb{F}\left( {SV5}\right)  = {\alpha }_{1}{HD}\left( {\text{ pass_ch }{k}_{rd},0}\right)  + {\alpha }_{2}{HD}\left( {\text{ pass_ch }{k}_{wr},0}\right)
$$

$$
+ {\alpha }_{3}{HD}\left( {\text{pass_ch}{k}_{rst},0}\right)  + {\alpha }_{4}{HD}\left( {\text{pass_ch}{k}_{sub},0}\right)  \tag{6}
$$

In eq. 6, pass_chk variable's index shows: whether an error was raised during read(rd), write(wr), reset(rst)operation with a wrong password or when a portion ( e.g., 31 of 32 bits) of the correct password has been provided (sub).

### 4.2 Framework: SW-based and FPGA-based

We implemented SHarPen to be applicable in two frameworks, i.e., simulation and FPGA-based prototyping. In the simulation-based framework (Fig. 2), we rely on building software equivalent model (e.g., through Verilator) of the SoC, on which user space programs are executable in . elf format. SHarPen encodes the appropriate portion of the code as binary bit patterns, followed by decoding bit patterns for each new generation of the swarm in BPSO, leading to new . elf after compilation. During decoding, illegal bit patterns generated that do not have valid contextual mapping (e.g. all 0s for a RISC-V instruction) are discarded. The decoding step also allows us to reduce the search space dramatically. By parsing the simulation waveform for tracing the signals of interest, the cost function will be evaluated and based on this feedback, the code is mutated for the next generations of the swarm. In the FPGA-based prototyping (Fig. 3), according to $P{R}_{2}/P{R}_{3}$ , we instrument the RTL code for additional observable points (① of Fig. 3). In this model, the host PC runs the BPSO program and is responsible for bitstream generation and load (2)- ④), and monitoring is done via JTAG debugging SW such as OpenOCD/UrJTAG (⑤) and based on tracing the signals, BPSO mutation will be done in the host (⑥).

---

${}^{7}$ 0x300 and 0x180 are the hex encoding of mstatus and satp registers.

${}^{8}$ Zero for negative arguments and one for non-negative arguments.

---

![0196f75f-ffb0-72a6-8daf-bb4df53f43dc_4_155_236_711_268_0.jpg](images/0196f75f-ffb0-72a6-8daf-bb4df53f43dc_4_155_236_711_268_0.jpg)

Figure 3: SHarPen through FPGA Prototyping.

## 5 EXPERIMENTAL SETUP & RESULTS

To construct the cost functions and engage BPSO in SHarPen, we customized the open-source pyswarms library [15]. We also engaged Ariane SoC to test SHarPen on the selected vulnerabilities. For the simulation model, a customized Verilated Ariane SoC is used. Also, a VCD parser was developed to evaluate the signals of interest corresponded to $\mathbb{F}$ s $\left( {\mathrm{{SV}}}_{1.5}\right)$ . For FPGA-based prototyping, all instrumentation has been done on RTL representation of Ariane SoC. This instrumentation involves making the signals that need to be mutated (like data used by cost functions) available to the memory space accessible by the JTAG debugger. We engaged Digilent Genesys 2 board as the prototype and used the JTAG port as the interface between the host PC and the target board. The JTAG signaling is performed by software interfaces such as UrJ-TAG/OpenOCD. In the FPGA prototyping, traces of feedback signals used in cost function are monitored via both the logic analyzer (ILA) core and UrJTAG/OpenOCD. All experiments have been done on AMD Ryzen ${4600}\mathrm{H}$ processor with $8\mathrm{{GBs}}$ of memory as the host PC.

Fig. 4 demonstrates the result of performing BPSO-based PT on Ariane SoC for detecting occurrence potential of SV1/2. Fig. $4\left( {b, c}\right)$ show the global experience of the population of the particles. Here for each iteration, the best value achieved by the swarm is plotted against that iteration number. This gradual acquirement of minima indicates that, by relying on mapping the properties to mathematical cost functions, we evolutionary move towards the test case that triggers the vulnerability. Fig. 4(a) reflects the continuous behavior of 5 out of 20 particles used for triggering SV2. In general, as demonstrated, each particle in the population gets gradually closer to the minima (less fluctuation in more iterations).

Figure 5 reflects the global experience of the population of the particles for SV3/4/5. For SV3/4, based on eq. 5, a user space program was prepared allowing us to mutate the 32-bit RISC-V instructions. Each time BPSO minimized one term of the cost function, a new swarm, which represents a new instruction, was created.

![0196f75f-ffb0-72a6-8daf-bb4df53f43dc_4_923_236_723_1043_0.jpg](images/0196f75f-ffb0-72a6-8daf-bb4df53f43dc_4_923_236_723_1043_0.jpg)

Fig. 5(a, b) demonstrates the aggregate swarm cost history when detecting SV3/4. Listing 1 shows the snippet code ${}^{9}$ generated by SHarPen, mutated by BPSO, used for triggering SV3. For triggering SV4, a similar code-snippet is generated by the BPSO algorithm except the final instruction is replaced by the WFI instruction.

For SV5, an adversary could write to the protected regions of the memory space, e.g., FW or bootup ROMs, via debug module without providing the correct password. BPSO mutated on the 41- bit value of the RISC-V data register which was shifted in through the software interface of UrJTAG and provided the pass_chk value required for cost function evaluation. The gradual achievement of the minima is also evident for this vulnerability in Fig. 5(c).

The generic cost function in SHarPen includes few configuration parameter $\left( {\alpha \mathrm{s}}\right)$ and additional optimization function $\left( {\sum \mathcal{G}}\right)$ . For instance, for $\mathbf{{SV}}1/2$ , a new generic term as ${\mathcal{G}}_{1}.\left| {L - {L}_{\text{best }}}\right|  +$ ${\mathcal{G}}_{2}.\left| {{N}_{\text{match }} - k}\right|$ can be defined as an additional term to $\mathbb{F}$ . In this additional term, $\mathcal{G}$ : a real valued function $\left( \left( {-\infty , + \infty }\right) \right) , L$ : the position of character, ${L}_{\text{best }}$ : The best position of the character, and ${N}_{\text{match }}$ : the no. of specific match bits. So, in BPSO-based PT, determining the configuration parameters, i.e., $\alpha \mathrm{s}/\mathcal{G}\mathrm{s}$ , can affect the convergence rate. Fig. 6 shows the results for one trial of different standard choices for ${\mathcal{G}}_{1/2}$ for SV1. In our trials, for different choices of ${\mathcal{G}}_{1/2}$ , there exists no significant difference in convergence rate.

---

${}^{9}$ For the sake of clarity, it has been written with assembler pseudo-instructions.

---

![0196f75f-ffb0-72a6-8daf-bb4df53f43dc_5_152_237_716_343_0.jpg](images/0196f75f-ffb0-72a6-8daf-bb4df53f43dc_5_152_237_716_343_0.jpg)

Figure 6: Cost Function Convergence for $\mathbb{F}\left( {{SV1}\& {SV2}}\right)$ with Different Tester Configurable Optimizing Function $\left( {\mathcal{G}}_{1/2}\right)$ .

Table 3: Impact of Tester Configured Parameters on Performance.

<table><tr><td>Parameter Changed $\rightarrow$</td><td>${\alpha }_{1}$</td><td>${\alpha }_{2}$</td><td>${\alpha }_{3}$</td><td>${\alpha }_{4}$</td></tr><tr><td>AVG min # of iterations</td><td>76</td><td>72</td><td>81</td><td>81</td></tr><tr><td>$\sigma$ of min # of iterations</td><td>0.52</td><td>0.46</td><td>0.66</td><td>0.64</td></tr></table>

Table 4: Performance Comparison in FPGA Prototyping vs. simulation.

<table><tr><td>Targeted vulnerability for cost function $\rightarrow$</td><td>SV1</td><td>SV2</td></tr><tr><td>Simulation-based</td><td>3050-3150</td><td>3050-3150</td></tr><tr><td>FPGA-based Prototyping</td><td>180-190</td><td>190-200</td></tr></table>

In general, choosing both ${\mathcal{G}}_{1/2}$ to be the exponential function produced the best average result (lowest iterations required) ${}^{10}$ . We also experimented with different values of each $\alpha$ s for different $\mathbb{F}$ s. The result for trials with $\mathbb{F}\left( {{SV1}\& {SV2}}\right)$ summarized in Table 3. All parameters were swept, in turn, in the range 1-100 with a step of 0.25 while others were kept constant. Table 3 implies that the variation in performance as measured by the minimum no. of iterations required to reach the minima, is relatively same for different $\alpha \mathrm{s}$ . Consequently, pending further rigorous investigation, we suggest that all $\alpha$ s can be set to 1 as BPSO follows the same performance rate for different values (indicated by the standard deviation).

---

Assembly:

	call setup_csr

	la t1, sm_trap_handler

	csrw mepc, t1

	mret

Setup_CSR:

	/*setup_CSR should set appropriate values in

	mstatus to enter supervisor mode*/

	li a1, 0x1009A2

	csrrw x0, mstatus, a1

	ret

sm_trap_handler:

	/*trap handler code to be executed in S-mode*/

	csrrw x5, satp, a1

	ret

			Listing 1: Code Snippet to Trigger Vulnerability SV3.

---

The complexity (execution time) of the framework for SV1/SV2 for both simulation/FPGA-based framework is reflected in Table 4. As shown, $\sim  {15}\mathrm{x}$ speed-up in performance is achieved when we use the FPGA-based prototyping. Also, depending on the vulnerabilities, their cost functions, instrumentations of the SoC, instructions under the test, and monitoring, the execution time differs. For instance, for SV3/SV4, with almost the same speed-up, it only takes around ${135}\mathrm{\;s}$ per iteration in simulation mode, affirming the acceptable performance acquired by SHarPen.

## 6 CONCLUSION

In this paper, we proposed SHarPen, a BPSO-based penetration testing framework for automating security verification at the SoC level. SHarPen formalizes the transformation of security policies, properties, and rule-checks to generic cost functions. Using this formalism, SHarPen could build the mathematical representation of rule-checks to be tested using BPSO-based PT at the hardware level. SHarPen framework is built for both simulation and FPGA-based prototyping environments. We tested SHarPen on Ariane SoC with a selective set of vulnerabilities, and our experiments show the effectiveness and performance of the proposed framework in both simulation-based and FPGA-based models.

## REFERENCES

[1] A. Adamov et al. 2009. The problem of Hardware Trojans detection in System-on-Chip. In CADSM. 178-179.

[2] A. Ahmed et al. 2018. Scalable hardware trojan activation by interleaving concrete simulation and symbolic execution. In ITC. 1-10.

[3] A. Tyagi et al. 2022. TheHuzz: Instruction Fuzzing of Processors Using Golden-Reference Models for Finding Software-Exploitable Vulnerabilities. arXiv 2201.09941 (2022).

[4] G. Dessouky et al. 2019. \{HardFails\}: Insights into \{Software-Exploitable\} Hardware Bugs. In USENIX Security. 213-230.

[5] H. Al Shebli et al. 2018. A study on penetration testing process and tools. In LISAT. 1-7.

[6] H. M. Kamali et al. 2022. Advances in Logic Locking: Past, Present, and Prospects. Cryptology ePrint Archive (2022).

[7] J. Kennedy et al. 1997. A discrete binary version of the particle swarm algorithm. In IEEE SMC, Vol. 5. 4104-4108.

[8] J. Portillo et al. 2016. Building trust in 3PIP using asset-based security property verification. In VTS. 1-6.

[9] J. Rajendran et al. 2016. Formal security verification of third party intellectual property cores for information leakage. In VLSID. 547-552.

[10] J. Vicarte et al. 2022. Augury: Using data memory-dependent prefetchers to leak data at rest. In IEEE S&P. 1518-1518.

[11] K. Laeufer et al. 2018. RFUZZ: Coverage-directed fuzz testing of RTL on FPGAs. In ${ICCAD}.1 - 8$ .

[12] K. Z. Azar et al. 2022. Fuzz, Penetration, and AI Testing for SoC Security Verification: Challenges and Solutions. Cryptology ePrint Archive (2022).

[13] M. Fischer et al. 2020. Hardware penetration testing knocks your SoCs off. IEEE Design & Test 38, 1 (2020), 14-21.

[14] M. Nourian et al. 2018. Hardware Trojan detection using an advised genetic algorithm based logic testing. JETC 34, 4 (2018), 461-470.

[15] Lester James V. Miranda. 2018. PySwarms, a research-toolkit for Particle Swarm Optimization in Python. JOSS 3 (2018). Issue 21.

[16] N. Anandakumar et al. 2022. Rethinking Watermark: Providing Proof of IP Ownership in Modern SoCs. Cryptology ePrint Archive (2022).

[17] N. Farzana et al. 2019. Soc security verification using property checking. In ITC.

[18] P. Kocher et al. 2019. Spectre attacks: Exploiting speculative execution. In IEEE $S\& P.1 - {19}$ .

[19] P. Puri et al. 2015. Fast stimuli generation for design validation of rtl circuits using binary particle swarm optimization. In ISVLSI. 573-578.

[20] P. Subramanyan et al. 2014. Formal verification of taint-propagation security properties in a commercial SoC design. In DATE. IEEE, 1-2.

[21] R. Zhang et al. 2018. End-to-end automated exploit generation for validating the security of processor designs. In MICRO. 815-827.

[22] S. Bhunia et al. 2018. The Hardware Trojan War. Springer (2018).

[23] S. Canakci et al. 2021. Directfuzz: Automated test generation for rtl designs using directed graybox fuzzing. In DAC. 529-534.

[24] S. Muduli et al. 2020. Hyperfuzzing for soc security validation. In ICCAD. 1-9.

[25] S. Ray et al. 2019. Security Policy in System-on-Chip Designs: Specification, Implementation and Verification. Springer.

[26] T. Trippel et al. 2021. Fuzzing hardware like software. arXiv 2102.02308 (2021).

[27] W. Hu et al. 2021. Hardware information flow tracking. ACM CSUR 54, 4 (2021), $1 - {39}$ .

[28] X. Guo et al. 2016. Scalable SoC trust verification using integrated theorem proving and model checking. In HOST. 124-129.

[29] Z. Pan et al. 2021. Automated test generation for hardware trojan detection using reinforcement learning. In ASP-DAC. 408-413.

---

${}^{10}$ For all experiments, ${\mathcal{G}}_{1/2}$ were both chosen to be the exponential. For instance, for SV1, for adding additional term, it would be ${\mathcal{G}}_{1}$ . $\exp \left( \left| {L - {L}_{\text{best }}}\right| \right)  + {\mathcal{G}}_{2}$ . $\exp \left( \left| {{N}_{\text{match }} - }\right. \right.$ $k \mid  )$ (base of the natural log) function i.e., exp(abs(L_Lbest)) and exp(abs(N_Nmatch).

---