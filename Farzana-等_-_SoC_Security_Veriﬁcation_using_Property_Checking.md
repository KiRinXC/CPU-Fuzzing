# SoC Security Verification using Property Checking

Nusrat Farzana, Fahim Rahman, Mark Tehranipoor, Farimah Farahmandi

Department of Electrical and Computer Engineering, University of Florida, Gainesville, Florida

Email:\{ndipu, fahim034\}@ufl.edu, \{tehranipoor, farimah\}@ecce.ufl.edu

${Abstract}$ -Security of a system-on-chip (SoC) can be weakened by exploiting the inherent and potential vulnerabilities of the intellectual property (IP) cores used to implement the design as well as the interaction among the IPs. These vulnerabilities not only increase the security verification effort but also can increase design complexity and time-to-market. If the design and verification engineers are equipped with a comprehensive set of security properties at the early stage of a design process, SoC security validation effort can be greatly reduced. In this paper, we propose a property-driven approach to design a secure SoC. Our goal is to develop a comprehensive set of reusable and architecture-agnostic properties acting as security-aware design rules and guidelines. Moreover, we develop metrics from these properties to facilitate quantitative security assessment. Finally, we present design examples to demonstrate the efficacy of our approach under different threat models.

Index Terms-Vulnerabilities, Threat Models, Security Properties, Verification, Metrics, Architecture-agnostic

## I. INTRODUCTION

System-on-chips (SoCs) have become the core components of modern computing systems and applications. An SoC may contain tens of intellectual property (IP) cores, each potentially having its unique functionality, while at the same time bringing in unique security challenges. Further, these IP cores (digital, analog, memory, re-configurable fabric, etc.) can introduce vulnerabilities when interacting with one another under the real workload. Hence, the security of SoCs must be verified at the design stage before fabrication and deployment.

Designing a secure $\mathrm{{SoC}}$ is quite challenging since vulnerabilities may be introduced during different stages of its life cycle. In general, SoC vulnerabilities can be divided into several classes, e.g., information leakage, access control violation, side-channel/covert channel leakage, presence of malicious functions, exploitation of test and debug structure, and fault-injection attacks [1]-[3]. Some of these security vulnerabilities can be introduced unintentionally by the designer's mistakes or lack of understanding of security problems. Besides, computer-aided design (CAD) tools can unintentionally introduce additional vulnerabilities in the SoCs [1]. Moreover, malicious design modifications can be made by rouge designers during various stages of the design cycle. These vulnerabilities can create a backdoor to the design through which critical information can be leaked, functionalities can be altered, or an attacker can take control of the system.

Traditionally, design houses spend a major part of their effort and resources for $\mathrm{{SoC}}$ verification to meet different criteria, e.g., the functional correctness, timing, area, power, and performance [4]. As the SoC design complexity increases and time-to-market shrinks, verification has become even more challenging not only for ensuring correct functionality but also for security. Moreover, verification engineers are seldom mindful of potential security threats during design time.

Considering the above-mentioned challenges in verifying SoC security, a set of well-defined security properties is needed to put in place to help verification engineers quickly verify them at the early stage of the design process. We believe a set of standard properties can be developed and used as rules for common $\mathrm{{SoC}}$ architectures. In general, a security property is a statement that can check assumptions, conditions, and expected behaviors of a design in the context of security validation. For every security vulnerability, the respective properties will formally describe what should be the expected behaviors of the design. The coverage of security properties can be used as a metric and will provide confidence to the designers in terms of the security assessment of their SoC design.

Till now, a little amount of research has been conducted on developing standard properties for modern SoCs [5], [6]. The reason behind this could be (i) difficulty in formally representing security properties, (ii) diversity of security assets, and (iii) imperceptible vulnerabilities existing in modern SoCs. The security properties should have the adaptability feature for easier transformation from one design to another. In this paper, we develop a framework for identifying a comprehensive set of security properties for security evaluation of SoCs and their utilized IPs by covering a wide spectrum of vulnerabilities, design functionalities, and security requirements for different designs, implementations, and abstraction levels. We also suggest different techniques to systematically verify the security properties using available toolsets at various levels of abstraction. In summary, our goal is to develop (i) security properties which will act as guidelines for security assurance; and (ii) metrics for security assessment. We demonstrate our property-driven security framework using concrete design examples. Specifically, our contributions in this paper are as follows.

- Identifying assets (primary and secondary), probable vulnerabilities, and threat models of a design under consideration to identify and formulate security properties;

- Mapping the security properties formally to asses the vulnerability level of a design and define security metrics;

- Developing a comprehensive security property database;

- Demonstrating our approach on design examples like Debug and HALT unit, AES, trace buffer, true random number generator (TRNG), current program status register (CPSR), and bus protocols.

The rest of the paper is organized as follows. In Section II, we provide a brief overview of the sources of vulnerabilities in the SoC design flow, diversity of security assets, and the application of formal properties in hardware verification. In Sections III and IV, we describe our hypothesis - how security properties can be identified and utilized for different implementations, respectively. We present our results and case studies in Section V. Challenges and future research scopes are presented in Section VI. Finally, we conclude the paper with Section VII.

![0196f766-cf22-757e-922c-65f5a26286b1_1_135_137_773_254_0.jpg](images/0196f766-cf22-757e-922c-65f5a26286b1_1_135_137_773_254_0.jpg)

Fig. 1. Sources of potential vulnerabilities (marked in red) in different design stages.

## II. BACKGROUND

## A. Sources of SoC Vulnerabilities

Today, there is a lack of secure design guidelines, assessment metrics, security-aware tools, and skilled workforce when it comes to designing secure SoCs. As shown in Figure 1, an SoC design can encounter security vulnerabilities during different stages of its design and life cycle. Different sets of vulnerabilities can be introduced from the very early stage of the SoC design as IPs are developed by different designers and design teams, with limited knowledge of the IP interactions and the final product.

Many security vulnerabilities in SoCs can be unintentionally created by designer's mistakes or lack of understanding of security problems due to the high complexity of the designs, variety of assets, and diversity of security vulnerabilities. Moreover, todays CAD tools are not well-equipped with the understanding of the security vulnerabilities [7] in integrated circuits (ICs) and can, therefore, introduce additional vulnerabilities in the design. During logic synthesis, placement, clock-tree synthesis, and routing, CAD tools perform design flattening and multiple optimization processes. They could potentially merge trusted blocks with the untrusted ones. As CAD tools are designed to optimize power, performance, and area, they can potentially introduce additional unintentional states during finite state machine (FSM) optimization and may provide indirect or direct access to security-critical states of the design.

For modern SoCs, design-for-test (DFT) and design-for-debug (DFD) are designed to increase the observability and controllability for post-silicon validation and debug efforts. However, increasing the observability and preserving security are two conflicting factors. Test and debug infrastructures may create confidentiality and integrity violations, as demonstrated in the literature where trace buffers [4], [8], and scan chains [6] have been exploited to leak security-critical information. These example vulnerabilities are unintentionally introduced in the design flow, demonstrating that the current design flow is not security-aware as design flaws and lack of well-defined policies are largely affecting the security of a design. Apart from the unintentionally added vulnerabilities, SoC designs are also prone to many maliciously introduced threats, such

![0196f766-cf22-757e-922c-65f5a26286b1_1_978_139_616_291_0.jpg](images/0196f766-cf22-757e-922c-65f5a26286b1_1_978_139_616_291_0.jpg)

Fig. 2. Example of an asset propagating through different modules in an SoC. as hardware Trojans, introduced by untrusted entities involved in the design process (from RTL $\rightarrow$ Gate level $\rightarrow$ Physical Design $\rightarrow$ GDSII).

## B. Diversity of Security Assets

In an SoC, an asset can be defined as a resource with a security-critical value that is worth protecting from adversaries [9]. Assets in an SoC can be classified as: (i) Primary Assets - those that are the ultimate target for protection as deemed by the designer. Device keys, protected data (e.g., password), firmware, device configuration, communication credentials, the entropy of true random number generator (TRNG) and physically unclonable functions (PUFs) are good examples of primary assets. (ii) Secondary Assets - the infrastructures that require protection to ensure the security of the primary assets during rest or transition. A shared bus is a secondary asset as its protection ensures the security of a key moving from an on-chip ROM to an encryption engine within an SoC.

Referring to the example shown in Figure 2, the random numbers, generated by the TRNG module, are being used as a session key by the encryption module for running an encryption algorithm. Here, the session key is considered as a primary asset. The modules which are generating and utilizing the primary asset (marked in green) are considered as the confidential modules. However, other modules e.g., interconnects, CPU, and key management units (marked in blue) are responsible for propagating the encryption key from TRNG to the encryption module. Since the confidentiality of this operation could be broken by weaknesses in the supporting modules, critical states of these modules must be considered as secondary assets. Therefore, asset definition is not the same for different modules within an SoC hence leading to a large and diverse set of assets. Propagation of session keys from TRNG to encryption module is safe whereas leakage of the keys through the debug unit and RAM interface (marked in red) is considered as a vulnerability.

The number of assets associated with an SoC increases with the number of IPs embedded within it. An asset can be a concrete (e.g., a signal) or an abstract object (e.g., the controllability of a signal) which can be exploited by accessing the design's test/debug structures, bus snooping, side-channels, untrusted IPs, and unprotected communication channels.

## C. Security Verification using Property Checking

1) Related Work: To formally model the expected behavior of a design and its specification, functional property checking has been widely used in pre-silicon verification. These properties are used to automatically find the sources of functionality violation in simulation-based verification [10]. Moreover, these properties are synthesizable and placed on silicon as a monitor for certain events at the run-time or provide closures for post-silicon validation [11]. However, such functional properties are not sufficient for security validation, as there are fundamental differences between the two from the verification point of view. Similar to functional properties, we need to create security properties for SoC security verification and validation to either prove the trustworthiness of the design or find a counter-example in the case of security violations.

To date, only a few works have proposed a limited number of security policies for modern SoCs. Zhang et al. [12] presented a methodology to identify security-critical properties for dynamic verification. They used known processor errata to establish an initial set of security-critical invariants of the processor. Manufacturers release errata documents enumerating known bugs. However, unknown security vulnerabilities exist in an SoC design and are not enlisted by available errata documents. Moreover, the identified properties could only be helpful for dynamic verification. Static or boot-time verification are not covered by this approach.

The authors in [13]-[17] also proposed property-driven approaches for hardware security that allow automatic synthesis and verification of both qualitative and quantitative security properties. They have described threat models based on information flow tracking (IFT) and statistical models and have mapped them to security properties. Their approach only considers information leakage but does not cover other threats like denial of service, access control violation, etc. Further, the IFT technique increases the complexity of the design and statistical modeling, as all the input variables need to be tainted to utilize IFT. If an entity, such as the value of a program counter, is tainted, all the entities executing an operation and pointed by the program counter will be tainted as well. On the other hand, IFT will not be scalable, i.e., area, timing, and power consumption increase with the instrumentation needed by IFT. We believe a more effective approach would be to create a database of security properties and map them into assertions to verify them using appropriate tool-sets.

The authors in [1] developed a technique, called AVFSM, to identify and mitigate vulnerabilities to fault injection attack for finite state machine (FSM). Like the FSM in the SoC controller module, there are other modules in crypto-systems, exception handlers, test and debug infrastructures, random number generators, etc. which are prone to intentional and unintentional vulnerabilities and they must be inspected during security validation. Authors in [10] proposed an automatic test generation technique for the activation of malicious implants like hardware Trojan at the RTL. They targeted the activation of rare branches (which can be a good host to include hardware Trojans) by converting them to assertions and use Concolic testing [44] to check (activate) them. Other approaches for developing security properties and metrics consider only a small subset of vulnerabilities, e.g., the vulnerability in hardware crypto-systems [5], side-channel vulnerabilities [18], or are limited to a particular abstraction level, e.g., behavioral model [6]. In summary, there is a lack of security-aware SoC design practices.

![0196f766-cf22-757e-922c-65f5a26286b1_2_934_138_693_145_0.jpg](images/0196f766-cf22-757e-922c-65f5a26286b1_2_934_138_693_145_0.jpg)

Fig. 3. Framework for security-critical property identification.

2) Writing Properties: In the context of verification, a property can be defined as a statement that can check assumptions, conditions, and expected behaviors of a design [19]. A property can be described/mapped as an assertion or cover statement. An assertion is a check embedded in or bound to a design unit during the simulation. A single bit associated with assert can indicate the pass or fail status of the assertion. However, a cover statement can check if a certain scenario has ever occurred in the design during the simulation. At the end of the execution of the cover statement, an assertion is triggered if the assertion is not covered during the run-time. Designers mostly use Linear Temporal Logic (LTL) and Computational Tree Logic (CTL) formulas to describe properties [20]. Then properties are translated into assertions using PSL (Property Specification Language) [21] and SystemVerilog Assertions (SVA) [22] and bind with different Hardware Description Languages (HDL) for verification. In the following, we provide examples of general properties translated into assertions and written into SVA language:

Example 1: Signals ready and load cannot be true at the same time.

assertion_1: assert property (@(posedge clk) (!(ready && load)));

Example 2: No path should exist between points A and B.

assertion_2: assert property (No_path_between_A_B);

assertion_1 and assertion_2 can be checked using formal verification and information flow tracking tools such as JasperGold and JasperGold SPV [23], respectively.

## III. SECURITY-CRITICAL PROPERTY IDENTIFICATION

Due to the high complexity of modern SoCs, it will be very difficult for design engineers to manually analyze design implementation weaknesses (especially at the lower levels of abstraction) in order to detect security vulnerabilities. Therefore, design houses that are concerned about the security of their SoCs, need to maintain a large group of experts to find security issues. Such a manual approach is both time-consuming and expensive to sustain. Moreover, according to the rule-of-ten principle, the cost of detecting a security issue at a later stage of the chip design is 10 times more costly than the previous stage. All these issues highlight the necessity for generating a comprehensive set of security properties that can be checked automatically. Moreover, the designers can utilize these properties as rules/guidelines at the very beginning of the SoC design cycle. Hence, here, we introduce a framework, as illustrated in Figure 3, for security-critical property identification.

To identify a set of properties for an SoC, we have followed several steps as described in Algorithm 1 which expresses the framework in detail. The algorithm takes an SoC or IP design denoted by $\mathbb{D}$ as input and identifies a set of security-critical properties $\mathbb{P}$ as output by going through the following steps -1 ) asset identification, 2) vulnerability identification for identified assets and vulnerability database creation, 3) threat model development, and 4) property formulation as described in this section.

Algorithm 1: Security Critical Property Identification

---

Input: Design of an $\mathrm{{SoC}}/\mathrm{{IP}}, D$

Output: Set of security properties, $\mathbb{P}$

/*Step1 : Identify Primary and Secondary Assets*/

$\mathbb{A} =$ Identify_Assets(D);

for Asset $A \in  \mathbb{A}$ do

		if $A \in$ Primary_Assets then

			$\mathbb{A} = \mathbb{A} \cup$ Identify_Secondary_Assets(A);

		end

end

$\mathbb{P} = \{ \}$ ;

$\mathbb{{AL}} =$ Abstraction_levels(D)

for Asset $A \in  \mathbb{A}$ do

		/*Step 2: Identify Possible Vulnerabilities and Associated Abstraction

		Levels*/

		for Various Abstraction levels ${AL} \in  \mathbb{{AL}}$ do

			${\mathbb{V}}_{AL} =$ Identify_Vulnerabilities(D, AL, A)

			for Vulnerability $V \in  {\mathbb{V}}_{AL}$ do

					/*Step 3: Develop Threat Models*/

					$\mathrm{T} =$ Identify_Threat_Model(A, V, AL);

					/*Step 4: Property Formulation*/

					$\mathbb{P} = \mathbb{P} \cup$ Formulate_Property(D, A, AL, V, T);

			end

		end

end

return $\mathbb{P}$

---

1) Step 1 - Choice of Security Assets: To preserve the security of an SoC against various adversarial threats, a designer must understand the underlying assets as well as their corresponding security levels. Additionally, the choice and specification of an asset can be different from one abstraction level to another, in addition to the required security level of the design as well as varied adversarial capabilities and intents.

Carefully choosing assets is the first step for security-critical property identification. The chosen asset can be primary or secondary in terms of protection required, as discussed in Section II-B. Moreover, the asset under consideration can be static or dynamic, based on how it is being generated and utilized. For example, while considering AES as a single IP module, only the key and the plaintext are considered as primary assets that are needed to be protected. However, when the AES unit is executing, the intermediate results should be protected as they may leak some information about the secret key or the plaintext [15]. On the other hand, the assets of the AES unit are expanded when it is placed within an SoC and expected to interact with other modules. Signals that are involved in transferring primary assets such as the AES key, random numbers, as well as the plaintexts from various modules (e.g., key management unit) to/from the AES module, should be considered as dynamic/secondary assets. Therefore, a set of assets $\mathbb{A}$ will be available at the end of this step by analyzing the design $D$ , as shown in Algorithm 1 (lines 3-7). After choosing the assets, the designer must analyze their propagation paths to find potential vulnerabilities and weaknesses along with their access points.

2) Step 2 - Development of Vulnerability Database: Unintentional vulnerabilities can arise at front-end and back-end design stages along with intentional malicious modification and exploitation. With identified assets, the next step, therefore, should be finding the points of vulnerabilities through which an attacker can get access to the assets to perform exploitation.

Vulnerabilities in SoCs can be broadly divided into the following three categories - (i) Vulnerabilities introduced due to poor design practices as traditional design objectives do not consider security constraints. Additionally, security-aware design practices are not well established (e.g., the number of execution cycles should not be dependent on the asset value). Therefore, these factors lead to design mistakes that introduce vulnerabilities. (ii) Vulnerabilities introduced by CAD tools as described in Section II-A. These tools are not designed with the security in mind (e.g., they do not consider the security level of different modules while performing optimizations), and can introduce additional vulnerabilities in the design. (iii) Vulnerabilities introduced by DFT and DFD infrastructures as they increase design controllability and observability, which leads to numerous vulnerabilities by allowing attackers to control or observe internal states of the SoC.

By surveying literature and studying the implementation of the design in detail, an attack's points of entry or probable vulnerabilities can be enlisted. Eventually, a database of potential vulnerabilities ${\mathbb{V}}_{AL}$ can be established for the design, identified assets, and associated abstraction levels (as shown in line 13 of Algorithm 1). For example, it has been shown that the AES key can be extracted using different attack strategies such as power and timing side-channel attacks [24]-[27], trace buffer [28] and scan attacks [29], hardware Trojan insertion [30], physical attacks [31], etc. that have been developed over time. With these existing attack information and design resources, we can point out essential vulnerabilities/weaknesses of a weak AES implementation, e.g., the access of the debug unit to intermediate results of AES, the attacker's access to the key through the plaintext or control signals, etc. [15].

Security assurance in one abstraction level does not guarantee that the design under consideration is secured in another abstraction level. Moreover, it has been shown that the security provided for one abstraction level can be bypassed in the next abstraction level by exploiting the newly introduced vulnerabilities. For example, even if the designers verify the security of the design at high-level specification $\left( {\mathrm{C}/\mathrm{C} +  + }\right)$ , new vulnerabilities will be introduced during high-level synthesis (HLS) due to the automatic transformation of an untimed high-level program to a timed RTL implementation. Additional control signals can be introduced by HLS tools and exploited for timing channel attack [32]. Moreover, the introduction of 'don't-care' states that have direct access to the protected states of the design during RTL to gate-level synthesis can lead to access control and integrity violations. It has been shown that an attacker can inject faults to access the don't-care states and consequently, the protected states of the design to bypass the intermediate rounds of the AES algorithm to extract the key or the plaintext [1]. Moreover, as mentioned in Section II, during physical design, the CAD tools may merge trusted security modules with untrusted ones for performance and power optimization, and an attacker can exploit such vulnerabilities as the points of entry. DFT and DFD infrastructures are also inserted in the gate-level design, which can introduce a new set of security vulnerabilities due to the increased observability/controllability of the design. Therefore, abstraction levels, $\mathbb{{AL}}$ , should be considered while defining design vulnerabilities. As a result, identifying vulnerabilities associated with each abstraction level and the inclusion of them into the vulnerability database should be a part of this step, as shown in Algorithm 1 (lines 9-13). The vulnerability database can be used to define the threat models and develop the corresponding security properties in the next steps.

3) Step 3 - Development of Threat Models: With the prior knowledge of design implementation, assets to protect, identified vulnerabilities, and associated abstraction levels, we develop threat models $\mathbb{T}$ for each asset (as shown in line 16 of Algorithm 1). Examples include deviation from the expected functionality, illegal access path, and corrupting the critical data and FSM states. In this section, we have classified threat models as following:

- Information Leakage: The flow of assets to an untrusted IP or observable points is considered as Confidentiality Violation, whereas illegal modification or corruption of them is regarded as Integrity Violation. These two violations can be analyzed under the umbrella of the information leakage threat model since both involve sensitive data access. In this case, an example of a security property could be as such - design assets should not be propagated to and leaked through any observable point, such as primary outputs, DFT, or DFD infrastructures.

- Access Control/Isolation Violation: Unauthorized interaction between trusted and untrusted IPs, illegal accesses to protected memory addresses, and protected states of a controller module, and out of bound memory access fall under this threat model. An example of security property for such a case would be - security-critical entities should be isolated from entities with lower security levels to circumvent content observation and leakage through low-security entities. Another property would be - two entities with the same security requirement should not affect the integrity and confidentiality of each other.

- Denial of Service: Service or connectivity disruption of the within-SoC modules can be categorized as a denial of service (DoS) threat. Numeric exceptions (e.g., divide by zero) and deadlocks can make a resource unavailable for a particular time and are examples of DoS threats. Moreover, DoS attacks can introduce timing (latency) overhead for recovery which can be utilized for leaking information or violating access controls. In such cases, a sample property would be - access to security-critical entities and memory should be restricted during deadlock and recovery resulted from a DoS occurrence.

- Design Tampering: Any intended or unintended modification, which makes a design vulnerable to information leakage, access control violation, or denial of service fall under the category of design tempering. An example of such a threat is hardware Trojans that can be inserted in the unspecified functionality of the design to create covert-channel, side-channel leakage, and result in unauthorized access to design assets [33]. Therefore, a sample property for this example can be written as follows: there should not be any state in the design that shares the key of a crypto-module with any non-crypto module [34].

- Side Channel Leakage: Any potential vulnerability that leads to timing, power consumption, or electromagnetic emanation, can be inferred by the attacker for sensitive information extraction, and labeled as a side-channel leakage threat. Example property would be - the control flow of a program should not be dependent on the value of an asset to avoid extraction of the asset based on measuring the power and timing consumed by various paths of the design. If the number of execution cycles, as well as variation on power consumption, depends on the asset value, it allows an attacker to employ the divide-and-conquer approach to reveal the asset value. Moreover, fault injection by power/clock glitching, varying temperature or laser injection can be performed by an attacker to bypass security mechanism and violate the confidentiality or integrity of a design. Therefore, security property should be developed to ensure that a single bit-flip or stuck-at fault cannot grant access to a protected state or memory address.

Multiple threat models $\mathbb{T}$ can be developed for a single asset $A$ . For example, unauthorized access to the intermediate results of the encryption performed by an AES module through a debug port can result in information leakage as well as access control violation. Properties should be written in a way such that each asset under consideration should be checked against any of these vulnerabilities and threats.

After identifying assets, vulnerabilities, associated levels, and threat models, the next step is the selection of IPs as well as SoC transactions that either contain those vulnerabilities or involve in propagating the corresponding assets. For example, if we are concerned about information leakage and side-channel attacks, the focus of the security properties should be on crypto IPs, TRNG modules, and asset management units. On the other hand, we need to focus on halt/debug units and the exception handler unit in the SoC if we consider the denial of service or access control violation attacks.

4) Step 4 - Identification of Properties: The security level of a design is dependent on the asset and design attributes, associated vulnerabilities and potential threats. Therefore, these factors can be used to define metrics to identify the security properties of a design. For example, if a design is isolated physically, the side-channel leakage (e.g., power side-channel leakage) should not be a concern for a security verification engineer. As a result, security properties can be developed by focusing on other vulnerabilities.

From the earlier steps, we will be able to obtain ample information to define security metrics and identify security properties based on them (as shown in line 18 of Algorithm 1). Here, we have listed some security metric definitions as examples and their use for security evaluation:

- Signal Observability: In RTL, the hardness of signal observability or accessing a statement measures the probability of the presence/existence of hardware Trojan in the design. Security properties should cover all rare statements of the design to detect any malicious functionality.

- Confidentiality and Integrity Assessment: One must measure the accessibility to assets through observable points like DFT/DFD infrastructures at the gate-level.

We create a database that assists in formulating security properties $\mathbb{P}$ . For a certain entity, a single threat model can be mapped into multiple properties or vice-versa. A sample database is presented in Table I where we have enlisted security properties for some well-known IP modules by following the steps described previously. In short, our proposed approach for security validation is a fusion of analytical and inquisitive methodologies in which we analyze a design considering all assets, vulnerabilities, and threat models associated with it and extract necessary security properties from the analysis.

Example: Let us consider MSP430 micro-controller as a target design. In the asset identification step, we can enlist the value of critical CPU registers and intermediate buffers, the content of data or program memory as the primary assets of MSP430 by surveying the design specification. In debug mode, a debugger can initiate HALT operation to perform off-chip debugging. The debug feature can be exploited as an attacker's entry point. When the primary assets propagate through the design, incorrect design implementation, intermediate states introduced by the CAD tools, accessibility to intermediate results of an operation, and the activity of different functional modules during HALT operation can be listed as potential vulnerabilities to develop the vulnerability database. These vulnerabilities can provide the debuggers with direct access to data memory space or program counter or redirect the sequence of code execution during HALT operation which can result in different threat models, e.g., information leakage, control flow violation, etc. Moreover, write access to program memory can facilitate firmware modification and code injection attacks. On the other hand, intermediate state accessibility during HALT operation provides an opportunity for physical attack (e.g., probing). Based on the identified asset, vulnerability database, and classified threats, a set of security-critical properties can be developed for MSP430. The security properties should be the negation of the vulnerabilities enlisted into the database.

## IV. SECURITY-CRITICAL PROPERTY UTILIZATION

In this section, we discuss possible directions for utilizing the identified security properties for various implementations and attack scenarios.

## A. Using Security Properties for Security Validation

The identified security-critical properties can be utilized for quantitative and qualitative security analysis of a design implementation. To do this, security-critical properties should be translated into formal assertions and checked at a proper time by using available toolsets.

1) Time to Check Security Properties: To check a security property, we need to consider the time at which the property should be verified and formally model or instrument a design according to that requirement. Some properties should be checked statically at the design time. For example, the registers containing the critical status of a CPU should only be accessed in valid ways, and any undefined access to these registers is considered as a threat. On the other hand, during boot time/reset time, intermediate buffers of the design may be initialized to 'don't-care' values. There should be properties that will ensure that those 'don't-cares' in boot time do not lead to vulnerabilities such that giving the attackers the authority to start from a security-critical state of the design and steal some critical information or cause denial of service.

It is worth mentioning that some assets would be generated at run-time. Therefore, we require security-critical properties in the form of sensors or monitors [35] that can operate in run-time to prevent malicious functionality, denial of service, etc. Thus, the properties obtained from the framework should be classified into 1) static/design-time, 2) boot-time, and 3) run-time rules.

2) Tools to Check Security Properties: Based on the nature of security properties and associated threat models they target, different tools and techniques should be used to verify them. Note that based on the tool used for verification/validation of security properties, formal modeling or design instrumentation may be needed. For example, model checker tools can be used to formally verify the static or boot-time properties. Design $D$ can be translated into formal models [35], and properties can be mapped to LTL/CTL formulas [20] to feed the model checker tools. Moreover, simulation-based assertion checking is also useful for verifying such properties. On the other hand, design instrumentation for information flow tracking and taint analysis is helpful to detect information leakage and confidentiality violations in run-time since there are several ways that a secret value can be leaked to observable points of a design. Malicious modifications and safe design transformations can be checked using equivalence checking tools. The security validation using property checking generates a pass or fail result. Therefore, the result of property checking can be used for the security assessment of the design under consideration.

## B. Architecture-agnostic Security Properties

To reduce the effort of security validation in the SoC design flow, security properties should be designed in architecture-agnostic fashion. Architecture-agnostic security properties can be defined as a set of properties which are universally applicable to a diverse set of devices and implementations rather than being tied to a specific one. Though all the security properties identified for one target implementation cannot be mapped to another, they can be used as a reference for generating new properties that are more appropriate for the latter. If a good percentage of security properties for a certain entity in one implementation can be reusable for the same or similar entity in a different implementation, the effort for security validation will be reduced to a great extent.

Let us consider the example shown in Figure 4(a), where a particular IP has been utilized in Implementation1 & Implementation2. ${P}_{1},{P}_{2},{P}_{3},{P}_{4}$ are the identified security properties for the IP in Implementation 1 based on the type of asset it is dealing with, possible vulnerabilities, and threat models. Some of these identified properties can be universal and applied for any implementation of the IP. In the example, ${P}_{1}\& {P}_{2}$ are the universal properties and remain same for ${\mathrm{{IP}}}^{\prime }$ in Implementation2. However ${P}_{3}\& {P}_{4}$ are the architecture-specific security properties of Implementation 1 and can not be reused for Implementation2. Therefore, new security properties ${P}_{3}^{\prime }\& {P}_{4}^{\prime }$ will come into consideration for ${\mathrm{{IP}}}^{\prime }$ in Implementation2. The properties which do not match, can be generated by tuning different parameters of the implementations or using the previously identified properties as a reference to generate them automatically to ensure security validation.

![0196f766-cf22-757e-922c-65f5a26286b1_6_138_137_757_280_0.jpg](images/0196f766-cf22-757e-922c-65f5a26286b1_6_138_137_757_280_0.jpg)

Fig. 4. (a) Same IP in different implementations (b) Same implementation of an IP in different SoCs.

For example, the number of rounds can be different for different AES implementations and security properties can be obtained by just modifying the timing of the existing properties according to the specification of the implementations. FSM designs would be another good example here. Conditions for secure state transitions for an FSM can vary for different implementations and security properties for a newer implementation of FSM can be identified by tuning the existing security properties based on the new controller signals. Note that for the new features of the newer implementations of an IP, we need to write a new set of properties to cover them and update the security property database accordingly.

In the case of SoCs, a similar approach can be applied to identify security properties. The analogy has been illustrated in Figure 4(b). The same IP has been integrated in two different SoCs. Since the implementation remains the same for the IP, no new security properties are required. Moreover, some IP interactions $\left( {\mathrm{{IP}} \rightarrow  {\mathrm{{IP}}}_{1}}\right)$ , as well as the corresponding security properties $P{P}_{1}\& P{P}_{2}$ are indifferent in both SoCs. However, other IP interaction has changed (from IP $\rightarrow  {\mathrm{{IP}}}_{2}$ to $\mathrm{{IP}} \rightarrow  {\mathrm{{IP}}}_{2}^{\prime }$ ) with the new implementation of ${\mathrm{{IP}}}_{2}\left( {\mathrm{{IP}}}_{2}^{\prime }\right)$ in SoC2 and new security property $\left( {P{P}_{3} \rightarrow  P{P}_{3}^{\prime }}\right)$ needs to be mapped. Moreover, in SoC2, a new IP module $\left( {\mathrm{{IP}}}_{3}\right)$ is interacting with the IP. Therefore, a new property $P{P}_{4}$ is required for completing the security property set for SoC2. The new security property can be generated automatically by using the previously identified properties of SoC1 as templates.

## C. Property Mapping at Different Abstraction Levels

Even for the same implementation, security properties need to be mapped at different abstraction levels of the SoC design life cycle (high-level specification $\rightarrow$ RTL $\rightarrow$ Gate level $\rightarrow$ Layout). The security properties identified for high-level specifications can be necessary but not sufficient for security validation in lower levels of abstraction like RTL, gate level, or layout level. Vulnerability definition and asset generation are dynamic as they are tied to different abstraction levels. For example, when we are dealing with an AES, a common security property can be listed as: Key should NOT be accessed using the output of AES. After insertion of DFT, or trace buffers for testing and debugging purposes, a new set of properties comes into play such as: Key should NOT leak through the observable test points. Moreover, RTL to gate-level transformation introduces 'don't-care' states, which requires new security properties. As illustrated in Figure 5, we either map the security properties from higher levels of abstractions or introduce new properties to cover new vulnerabilities introduced in lower levels. Security properties will enrich the property database eventually and the database should be updated in the case of new attacks and vulnerabilities. Moreover, in the case of zero-day attacks, the vulnerability database will be helpful in creating new security properties to identify the underlying weaknesses that facilitate new attacks. In short, the process of security property identification will be a back and forth approach, where some properties will be identified manually, some will be mapped from the previous implementation, and the rest will be generated automatically (using the templates generated from the database) to cover the broad set of security vulnerabilities.

![0196f766-cf22-757e-922c-65f5a26286b1_6_974_138_627_292_0.jpg](images/0196f766-cf22-757e-922c-65f5a26286b1_6_974_138_627_292_0.jpg)

Fig. 5. Security-critical property identification at different abstraction levels.

## D. Metric: Property Coverage & Property Severity

The security evaluation of a design is quite difficult both qualitatively and quantitatively. In general, several properties may be required to cover one security vulnerability. In such a scenario, security property coverage can be used as metrics for security validation closures. Also, some properties may be more important than others, and the impact of their neglection would be more critical. Therefore, properties should be ranked according to the severity of vulnerabilities and their importance. The percentage of covered security properties that are checked using available tool-sets alongside with their importance (weight) define "property coverage" and "property severity" metrics, respectively. Let us consider, ${P}_{1}\& {P}_{2}$ are covered security properties of a design $D$ with equal weights $\left( {{50}\% }\right)$ . Among these properties, ${P}_{1}$ passes in the property checking step while ${P}_{2}$ fails. Therefore, the designer’s confidence level for the design is ${50}\%$ . Figure 6(a) shows the back and forth relationship between security vulnerabilities, properties, and metrics. Poor security property coverage is used as a metric to modify the design and remove vulnerabilities. Therefore, security properties act as a guideline for design engineers (as shown in Figure 6(b)).

![0196f766-cf22-757e-922c-65f5a26286b1_6_972_1847_620_207_0.jpg](images/0196f766-cf22-757e-922c-65f5a26286b1_6_972_1847_620_207_0.jpg)

Fig. 6. (a) The relation between security vulnerabilities, properties, and metrics, (b) Usages of security properties.

## V. RESULTS

In this section, first, we provide a sample database (Table I) of security properties that we have developed by surveying literature, available open-source architectures, design implementations, and different attack strategies. We also demonstrate our property-driven security assessment framework on a number of modules. We conducted the experiments on open source MSP430 microprocessor [36] and Trojan-inserted AES designs from Trust-Hub [34]. We used Cadence JasperGold model checker and JasperGold SPV (Security Path Verification tool) [23] along with our custom TCL scripts to perform security verification. The tools formally verify the properties (written in the form of assertions) and provide results whether they are violated or not.

1) Security Property Database: We have provided sample security-critical properties for different entities alongside with the abstraction levels as well as the time that these properties will need to be checked in Table I.

- Debug and HALT Unit: Features used for debug unit include the ability to read and modify register values of processors and peripherals. Usually, debugging involves halting the execution once a failure has been observed and collects state information retrospectively to investigate the problem [36]. HALT operation can be initiated by a host who can be an entity like an off-chip debugger connected to an on-chip Debug Access Port (DAP) via a JTAG interface or an on-chip processor. During HALT operation, a processor stops executing according to the instruction indicated by the program counter's (PC) value, and a debugger can examine and modify the processor state via DAP.

As the user has read/write access to memory and CPU registers in the debug mode, several threat models can be developed [37]. As shown in Table I, (first five rows), we identified assets of an SoC with debugging features and their vulnerabilities first. Then we enlisted probable threat models of the vulnerabilities. For example, when we consider the PC's value as an asset, we will need to check whether it is accessible from debug access port or not. If yes, an attacker can freeze the system and modify the PC's value in such a way that the sequence of code execution can get redirected after a HALT operation is executed.

Similarly, when the asset is in a particular memory address, reading from that address during HALT operation may result in leaking sensitive information like password, key, etc. Moreover, writing to a protected memory address from DAP can result in firmware modification or malicious code injection threats. The third property shown in Table I takes the bus or interconnects into consideration. At the time of debugging, a secret key (e.g., encryption key) should not be loaded into the bus as an attacker who has illegally initiated the HALT operation has direct access to the bus and can exploit the asset. Moreover, HALT operation should not be initiated during the functional mode. In the case of inter-processor debugging, the privileged status should be maintained strictly. Otherwise, a low-privileged host can access a high-privileged target. Therefore, we have identified a set of properties for a safe HALT operation behavior.

- Advanced Encryption Standard (AES): AES is a well-known block cipher standard used for encryption/decryption module. The assets of an AES module are: keys (including round keys), intermediate results, plaintexts, etc. Exploitable hardware vulnerabilities exist in the hardware implementation of AES [38] as prior works exhibit that disclosure of internal signals, via implementation flaws or rogue debug interfaces, can significantly reduce the effort in recovering the assets of AES [24], [28]. By surveying existing attack scenarios, we have enlisted possible threats and developed a set of properties for AES similar to the debug and HALT unit.

- Trace Buffer: Trace buffer stores all the signal traces required for debugging. Attack strategies in [28] show that the stored signals of trace buffers can be dumped to extract sensitive information of a design. Therefore, trace buffer should exhibit security properties to protect the trace signals.

- Exception Handler: The exception handler is an entity that raises an exception when an interrupt or exception happens. This entity can be exploited to obtain privileged access from an unprivileged mode. We develop security properties to ensure that the exception handler does not give unprivileged user access to privileged memory.

- TRNG: For a TRNG module, entropy is the most security-critical asset, and contamination of it reduces the unpredictability of TRNG. The security violation for the TRNG module can cause confidentiality violations. So we have added a set of properties for TRNG to ensure that entropy is not compromised.

- CPSR: CPSR stores the current status of CPU. It can be a good source of the denial of service attack. Therefore, there should be security properties to ensure that the value of CPSR cannot be modified from I/O pins.

- Bus Protocols: Lastly, bus protocols can be exploited for information leakage and the denial of service attack. Therefore, we have enlisted security properties to ensure that there can be no interference between sub-modules of a system connected through a bus to prevent information leakage and the frequency of clock signal cannot vary out of range to prevent Trojan insertion by clock glitching.

2) Case Study1: Debug and HALT Unit: We checked the identified security properties for debug and HALT unit on the open-source MSP430 microprocessor [36] [39]. We verified the properties using JasperGold SPV. Table II enlists our obtained results. In order to check the first two properties from Table I, we mapped these properties into multiple assertions to check if there is any direct path between i) debugger receiver/transmitter port and program counter, and ii) debugger receiver/transmitter port and memory unit of MSP430. All assertions were proven 'true' which indicates that such vulnerable paths exist in this architecture and security properties are violated.

3) Case Study2: Advanced Encryption Standard (AES): We used Trust-Hub [34] benchmarks AES-T100 and AES-T400 for checking the identified security properties for AES. In Table II, we have reported the result of the first two properties of AES from Table I. According to the first property from Table I, the secret key should not flow to the output of AES. Hence, when the key is loaded to the AES, the ready signal should be low at that moment. Again, according to the second property of AES, mentioned in Table I, the key of AES should not be shared with any other component of SoC which may lead to information leakage. We have mapped these two properties to sample assertions and verified them using JasperGold. The verification results (Table II) showed that there are scenarios where these assertions failed and exhibited vulnerabilities in implementations.

TABLE I

DATABASE OF SECURITY CRITICAL PROPERTIES

<table><tr><td>Different Entities</td><td>Asset</td><td>Property Description</td><td>Associated Levels</td><td>Threat models</td><td>Time-to- Check</td><td>Abstraction Lev- els</td></tr><tr><td>Debug & HALT Unit</td><td>Program counter's value</td><td>Program Counter's value should not be mod- ified during HALT operation.</td><td>IP, SoC, Inter- connects</td><td>Confidentiality & integrity vi- olation, existence of hardware Trojan, side channel leakage</td><td>Static</td><td>RTL, gate-level</td></tr><tr><td>Debug & HALT Unit</td><td>Privilege memory</td><td>Privileged memory access should be denied in HALT operation through DAP.</td><td>IP, SoC, Inter- connects</td><td>Confidentiality & integrity vi- olation, existence of HW Tro- jan</td><td>Static</td><td>High level specifi- cation, RTL, gate- level</td></tr><tr><td>Debug & HALT Unit</td><td>Crypto key, TRNG value</td><td>Assets of an SoC should not be loaded into bus/inter-connects in HALT operation.</td><td>IP, SoC, Inter- connects</td><td>Confidentiality violation, side channel leakage</td><td>Static</td><td>High level specifi- cation, RTL, gate- level</td></tr><tr><td>Debug & HALT Unit</td><td>Functionality of an SoC</td><td>Halt operation should be initiated only in test mode not in functional mode</td><td>IP, SoC, Inter- connects</td><td>Confidentiality & integrity vi- olation, access control viola- tion, denial of service, side channel leakage</td><td>Static</td><td>High level specifi- cation, RTL, gate- level, Layout</td></tr><tr><td>Debug & HALT Unit</td><td>Assets of the target processor</td><td>For inter-processor debugging, both target and host should be in same privilege level</td><td>SoC</td><td>Confidentiality & integrity vi- olation, access control viola- tion, denial of service</td><td>Static</td><td>Instruction level, High level Speci- fication</td></tr><tr><td>AES</td><td>Key</td><td>AES key should not flow to the output</td><td>IP</td><td>Confidentiality violation, side channel leakage</td><td>Static</td><td>RTL</td></tr><tr><td>AES</td><td>Key</td><td>AES key should not be shared with other modules</td><td>IP, SoC, Inter- connects</td><td>Existence of HW Trojan, side channel leakage</td><td>Static</td><td>RTL, gate-level, Layout</td></tr><tr><td>AES</td><td>Round Key</td><td>Round key should be derived correctly from cipher key and round constant</td><td>IP</td><td>Confidentiality violation, side channel leakage</td><td>Static</td><td>RTL, gate-level, Layout</td></tr><tr><td>AES</td><td>Intermediate state</td><td>Each intermediate state should be derived correctly by taking the bitwise XOR of round key and corresponding state</td><td>IP</td><td>Confidentiality violation, side channel leakage</td><td>Static</td><td>RTL, gate-level, Layout</td></tr><tr><td>AES</td><td>Intermediate result of encryption</td><td>The intermediate encryption results can only flow to the output in debug mode but prohib- ited during normal operation</td><td>IP, SoC, Inter- connects</td><td>Access control violation</td><td>Static</td><td>RTL, gate-level, Layout</td></tr><tr><td>AES</td><td>Intermediate result of encryption</td><td>The cipher-text ready (R) signal should be only asserted from final round state</td><td>IP</td><td>Confidentiality and access control violation, side channel leakage</td><td>Static</td><td>RTL</td></tr><tr><td>Trace Buffer</td><td>Trace buffer con- tents</td><td>Trace Signals should be encrypted before being stored/recorded in the trace buffer [4]</td><td>SoC</td><td>Confidentiality & integrity vi- olation</td><td>Static</td><td>RTL, gate-level, Layout</td></tr><tr><td>Trace Buffer</td><td>Privileged memory</td><td>Authorized access to trace buffer</td><td>IP, SoC</td><td>Confidentiality & integrity vi- olation</td><td>Static</td><td>RTL, gate-level, Layout</td></tr><tr><td>Exception Handler</td><td>Exception vector ta- ble</td><td>Exception vector table (EVT) should not be stored into low memory address like $0 \times  0$</td><td>SW/HW. u-Architecture</td><td>Isolation Violation, denial of service</td><td>Static, Boot time</td><td>Instruction level</td></tr><tr><td>Exception Handler</td><td>Exception register value</td><td>Exception register value should be updated correctly</td><td>SW/HW. u-Architecture</td><td>Isolation Violation, denial of service</td><td>Static</td><td>Instruction level</td></tr><tr><td>Exception Handler</td><td>Mode handeling</td><td>A reset/ software interrupt exception should only be handled in supervisor mode</td><td>SW/HW. u-Architecture</td><td>Isolation Violation</td><td>Run-time, boot time</td><td>Instruction level. RTL</td></tr><tr><td>TRNG</td><td>Entropy</td><td>An entropy source of TRNG should not ex- hibit any biases</td><td>IP</td><td>Confidentiality violation</td><td>Static. Run-time</td><td>RTL, gate-level, Layout</td></tr><tr><td>TRNG</td><td>Entropy</td><td>The unpredictability of the TRNG should not be based on the complexity of the harvesting mechanism</td><td>IP</td><td>Confidentiality violation</td><td>Static</td><td>RTL. gate-level, Layout</td></tr><tr><td>CPSR</td><td>CPSR content</td><td>Only in privilege mode, CPSR content can be copied to/from any general-purpose register</td><td>SoC. SW/HW. u-Architecture</td><td>Confidentiality, integrity & access control violation & de- nial of service</td><td>Run- Time</td><td>High Level Specification. RTL, gate-level</td></tr><tr><td>Bus Protocol (USB/I2C)</td><td>Message from host to device or master to slave</td><td>There should be no interference between de- vices (USB) / slaves (I2C)</td><td>Inter-connects. SoC</td><td>Confidentiality, integrity and isolation violation</td><td>Static, Run-time</td><td>RTL. gate-level, Layout</td></tr><tr><td>I2C Pro- tocol</td><td>Clock signal</td><td>The frequency of the clock in the protocol should vary within an allowed range</td><td>Inter-connects. SoC</td><td>Denial of service</td><td>Run- Time</td><td>gate-level, Layout</td></tr></table>

## VI. DISCUSSION

In this section, we mainly discuss the future research directions of our work. Security property identification is the first step for automatic property generation techniques. Existing mining tools like Texada [40] and UNDINE [41] require user-defined LTL property type template as input for automatic property generation. Developing a comprehensive database of security vulnerabilities will be helpful for developing the LTL template for mining tools. Using temporal logic, a precise description of the expected behavior of a design can be captured, and any deviation from it can be verified by simulation or formal methods. The automatic generation of properties will make security verification easier and less time-consuming.

Note that security properties are not static. Diversity of assets, new implementations of a common entity, new attack scenarios which exploit new vulnerabilities require new security properties. Therefore, the more attacks are developed, the more vulnerabilities should be identified and the designers will need to bring a quick change in security properties to assess and mitigate them. In short, the security property database will need to be updated frequently to cover new attack strategies.

Farahmandi et al. [11] has presented a technique to map constraints to assertion (Constraints $\rightarrow$ Temporal Logic $\rightarrow$ Assertion) and use assertions for post-silicon coverage analysis. Boule et al. [42] showed how to transform properties into automata. They proved that PSL/SVA properties are synthesizable and transformable to circuits. Hence, a checker circuit can be designed for post-silicon monitoring of the asserted properties. The same technique can be used to generate security coverage monitors from the developed security properties. Moreover, security properties can be used for automatic test generation to activate security violations in an SoC [10]. Hence, the security policy engine [43] can be another potential implementation of security properties. Since most of the properties should be checked with formal tools, making formal tools scalable is a major challenge. A modern SoC design is quite complex with multiple clocks, therefore, security property verification using the available toolset is quite difficult, and scalable formal verification tools and techniques need to be developed.

TABLE II

VERIFICATION RESULTS OBTAINED FROM JASPERGOLD FOR MSP430 AND AES ARCHITECTURES

<table><tr><td>Entity</td><td>Property</td><td>Mapping</td><td>Sample Assertion</td><td>Status</td><td>Duration</td></tr><tr><td>Debug & HALT Unit</td><td>Program Counter's value should not be modified during HALT op- eration.</td><td>Any path? (Debug/HALT Unit Receiver Pin $\rightarrow$ Program Counter Input)</td><td>check_spv -create -from \{dbg_uart_rxd\} -to \{frontend_0.pc\}</td><td>Proven (leakage)</td><td>0.3s</td></tr><tr><td>Debug & HALT Unit</td><td>Program counter's value should not be read through DAP.</td><td>Any path? (Program Counter Output $\rightarrow$ Debug/HALT Unit Transmitter Pin)</td><td>check_spv -create -from \{frontend_0.pc\} -to \{dbg_uart_txd\}</td><td>Proven (leakage)</td><td>0.1s</td></tr><tr><td>Debug & HALT Unit</td><td>Privileged memory access should be denied through DAP in HALT operation.</td><td>Any path? (Debug/HALT Unit Receiver Pin $\rightarrow$ Memory Unit Input)</td><td>check_spv -create -from \{dbg_uart_rxd\} -to \{dmem_din\}</td><td>Proven (leakage)</td><td>7.8s</td></tr><tr><td>Debug & HALT Unit</td><td>Sensitive memory content should not leak through DAP.</td><td>Any path? (Memory Unit Output $\rightarrow$ De- bug/HALT Unit Transmitter Pin)</td><td>check_spv -create -from \{dmem_dout\} -to \{dbg_uart_txd\}</td><td>Proven (leakage)</td><td>0.1s</td></tr><tr><td>AES-T100</td><td>The secret key should not flow to the output of AES.</td><td>Never load key when output is ready (Key_load $\mid   -  >$ ! Output_ready)</td><td>assert property(@(posedge_clk) (ld) $| -  >$ (!ready))</td><td>Failed</td><td>0.031s</td></tr><tr><td>AES-T400</td><td>AES Key should not be shared with other modules in SoC.</td><td>AES_key_input $\neq$ key_input_other_modules</td><td>assert property(@(posedge_clk) (!top.AES.key==top.TSC.key))</td><td>Failed</td><td>0.009s</td></tr></table>

## VII. CONCLUSION

In this paper, we have proposed a property driven approach for secure hardware design. Our approach is able to compare the security level of different implementation of the same design and add/remove necessary security property for the new implementation. We have demonstrated the proposed framework on a number of hardware modules.

## REFERENCES

[1] A. Nahiyan et al., "AVFSM: a framework for identifying and mitigating vulnerabilities in FSMs", DAC, 2016.

[2] M. Tehranipoor & F. Kaushanfar, "A survey of hardware trojan taxonomy and detection", IEEE Design & Test of Computers, 2010.

[3] G. K. Contreras et al., "Security vulnerability analysis of design-for-test exploits for asset protection in SoCs", ASP-DAC, 2017.

[4] P. Mishra et al., "Hardware IP security and trust", Springer, 2017.

[5] B. Yuce et al., "TVVF: Estimating the vulnerability of hardware cryp-tosystems against timing violation attacks", HOST, 2015.

[6] H. Salmani & M. Tehranipoor. "Analyzing circuit vulnerability to hardware Trojan insertion at the behavioral level", DFTS, 2013.

[7] A. Nahiyan et al., "Security-aware FSM Design Flow for Identifying and Mitigating Vulnerabilities to Fault Attacks", TCAD, vol. 38, 2018.

[8] J. Rajendran et al., "Building trustworthy systems using untrusted components: A high-level synthesis approach", IEEE Transactions on VLSI Sytems, vol. 24, 2016.

[9] A ARM, "Security technology building a secure system using trustzone technology (white paper)", ARM Limited, 2010.

[10] A. Ahmed et al., "Scalable hardware Trojan activation by interleaving concrete simulation and symbolic execution", ITC, 2018.

[11] F. Farahmandi et al., "Cost-effective analysis of post-silicon functional coverage events", DATE, 2017.

[12] R. Zhang et al., "Identifying security critical properties for the dynamic verification of a processor", ACM, 2017.

[13] W. Hu et al., "Towards Property Driven Hardware Security", MTV, 2016.

[14] W. Hu et al., "On the complexity of generating gate level information flow tracking logic", IEEE Transactions on Information Forensics and Security, 2012.

[15] W. Hu et al., "Property specific information flow analysis for hardware security verification", ICCAD, 2018.

[16] J. Oberg et al., "Leveraging gate-level properties to identify hardware timing channels", TCAD, 2014.

[17] J. Oberg et al., "Information flow isolation in I2C and USB", DAC, 2011.

[18] J. Demme et al., "Side-channel vulnerability factor: A metric for measuring information leakage", ISCA, 2012.

[19] F. Chang, "Property Specification: The key to an Assertion-Based Verification Platform".

[20] M. Boulé & Z. Zilic, "Automata-based assertion-checker synthesis of PSL properties", TODAES, ACM, 2008.

[21] M. Gruninger & C. Menzel, The process specification language (PSL) theory and applications, AI magazine, 2013.

[22] A. Pnueli, "The temporal logic of programs", SFCS, 1977.

[23] https://www.cadence.com/

[24] W. Diehl, "Attack on AES Implementation Exploiting Publicly-visible Partial Result", IACR, 2017.

[25] K. Schramm et al., "A Collision-Attack on AES: Combining Side Channel-and Differential-Attack", Joye and Quisquater [8], pp. 163- 175.

[26] C. Ashokkumar, " "S-Box" Implementation of AES is NOT side-channel resistant", IACR, 2018.

[27] H. Gamaarachchi & H. Ganegoda, "Power Analysis Based Side Channel Attack", arXiv preprint arXiv:1801.00932, 2018.

[28] Y. Huang and P. Mishra, "Trace buffer attack on the aes cipher", Journal of Hardware and Systems Security, 2017.

[29] B. Ege et al.,"Differential scan attack on aes with $x$ -tolerant and $x$ - masked test response compactor", Euromicro DSD, 2012.

[30] T. Reece and W. H. Robinson, "Analysis of data-leak hardware trojans in aes cryptographic circuits", International Conference on Technologies for Homeland Security, 2013.

[31] A. A. Pammu et al., "Interceptive side channel attack on aes-128 wireless communications for iot applications", APCCAS, 2016.

[32] Z. Jiang et al., "High-level synthesis with timing-sensitive information flow enforcement", ICCAD, 2018.

[33] N. Fern et al., "Hardware trojans in incompletely specified on-chip bus systems", DATE, 2016.

[34] https://trust-hub.org/home

[35] H. D. Foster et al., "Assertion-based design", Springer , 2004.

[36] https://github.com/olgirard/openmsp430

[37] Z. Ning and F. Zhang, "Understanding the security of arm debugging features", SMP, 2019.

[38] B. Schneier, "Cryptographic design vulnerabilities", Computer, 1998.

[39] O. Girard, "openmsp430", 2009.

[40] C. Lemieux et al., "General LTL specification mining (t)", AES, 2015.

[41] C. Deutschbein and C. Sturton, "Mining security critical linear temporal logic specifications for processors", MTV, 2018.

[42] M. Boulé & Z. Zilic, "Efficient automata-based assertion-checker synthesis of psl properties", International High Level Design Validation and Test Workshop, 2006.

[43] S. Ray & Y. Jin, "Security policy enforcement in modern soc designs", ICCAD, 2015.

[44] S. Koushik, "Concolic Testing", ASE'07, pages 571-572, ACM, 2007.