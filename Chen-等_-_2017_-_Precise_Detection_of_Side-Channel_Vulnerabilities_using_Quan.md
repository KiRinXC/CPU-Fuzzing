# Precise Detection of Side-Channel Vulnerabilities using Quantitative Cartesian Hoare Logic

Jia Chen

University of Texas at Austin

Austin, Texas

jchen@cs.utexas.edu

Yu Feng

University of Texas at Austin

Austin, Texas

yufeng@cs.utexas.edu

Isil Dillig

University of Texas at Austin

Austin, Texas

isil@cs.utexas.edu

## Abstract

This paper presents THEMIS, an end-to-end static analysis tool for finding resource-usage side-channel vulnerabilities in Java applications. We introduce the notion of $\epsilon$ -bounded non-interference, a variant and relaxation of Goguen and Meseguer's well-known non-interference principle. We then present Quantitative Cartesian Hoare Logic (QCHL), a program logic for verifying $\epsilon$ -bounded noninterference. Our tool, THEMIS, combines automated reasoning in CHL with lightweight static taint analysis to improve scalability. We evaluate THEMIS on well known Java applications and demonstrate that THEMIS can find unknown side-channel vulnerabilities in widely-used programs. We also show that THEMIS can verify the absence of vulnerabilities in repaired versions of vulnerable programs and that TheMIS compares favorably against BLAZER, a state-of-the-art static analysis tool for finding timing side channels in Java applications.

## CCS CONCEPTS

- Security and privacy $\rightarrow$ Logic and verification; Software security engineering; - Theory of computation $\rightarrow$ Automated reasoning;

## KEYWORDS

vulnerability detection; side channels; static analysis; verification

## 1 INTRODUCTION

Side channel attacks allow an adversary to infer security-sensitive information of a system by observing its external behavior. For instance, in the case of timing side channels, an attacker can learn properties of a secret (e.g., user's password) by observing the time it takes to perform some operation (e.g., password validation). Similarly, compression side channel attacks allow adversaries to glean confidential information merely by observing the size of the compressed data (e.g., HTTP response). Numerous research papers and several real-world exploits have shown that such side channel attacks are both practical and harmful. For instance, side channels have been used to infer confidential data involving user accounts [26, 38], cryptographic keys $\left\lbrack  {5,{19},{45}}\right\rbrack$ , geographic locations $\left\lbrack  {62}\right\rbrack$ , and medical data [22]. Recent work has shown that side channels can also lead to information leakage in cyber-physical systems [23].

Side channel attacks are made possible due to the presence of an underlying vulnerability in the system. For example, timing attacks are feasible because the application exhibits different timing characteristics based on some properties of the secret. In general, the most robust defense against side-channel attacks is to eradicate the underlying vulnerabilities by ensuring that the resource usage of the program (time, space, power etc.) does not vary with respect to the secret. Unfortunately, it can be challenging to write programs in a way that follows this discipline, and side-channel vulnerabilities continue to be uncovered on a regular basis in real-world security-critical systems $\left\lbrack  {5,{19},{36},{68}}\right\rbrack$ .

Our goal in this paper is to help programmers develop side-channel-free applications by automatically analyzing correlations between variations in resource usage and differences in security-sensitive data. In particular, given a program $P$ and a "tolerable" resource deviation $\epsilon$ , we would like to verify that the resource usage of $P$ does not vary by more than $\epsilon$ no matter what the value of the secret. Following the terminology of Goguen and Meseguer [34], we refer to this property as $\epsilon$ -bounded non-interference. Intuitively, a program that violates $\epsilon$ -bounded non-interference for even large values of $\epsilon$ exhibits significant secret-induced differences in resource usage.

The problem of verifying $\epsilon$ -bounded non-interference is challenging for at least two reasons: First, the property that we would like to verify is an instance of a so-called 2-safety property [66] that requires reasoning about all possible interactions between pairs of program executions. Said differently, a witness to the violation of $\epsilon$ -bounded interference consists of a pair of program runs on two different secrets. Unlike standard safety properties that have been well-studied in verification literature and for which many automated tools exist, checking 2-safety is known to be a much harder problem. Furthermore, while checking 2-safety can in principle be reduced to standard safety via so-called product construction [12, 14] such a transformation either causes a blow-up in program size [12], thereby resulting in scalability problems, or yields a program that is practically very difficult to verify [14].

In this work, we solve these challenges by combining relatively lightweight static taint analysis with more precise relational verification techniques for reasoning about $k$ -safety (i.e., properties that concern interactions between $k$ program runs). Specifically, our approach first uses taint information to identify so-called hot spots, which are program fragments that have the potential to exhibit a secret-induced imbalance in resource usage. We then use much more precise relational reasoning techniques to automatically verify that such hot spots do not violate $\epsilon$ -bounded non-interference.

---

republish, to post on servers or to redistribute to lists, requires prior specific permission CCS '17, October 30-November 3, 2017, Dallas, TX, USA © 2017 Copyright held by the owner/author(s). Publication rights licensed to Associa-

---

At the core of our technique is a new program logic called Quantitative Cartesian Hoare Logic (QCHL) for verifying the $\epsilon$ -bounded non-interference property. QCHL leverages recent advances in relational verification by building on top of Cartesian Hoare Logic (CHL) [65] for verifying $k$ -safety properties. Specifically, QCHL allows us to prove triples of the form $\langle \phi \rangle S\langle \psi \rangle$ , where $S$ is a program fragment and $\phi ,\psi$ are first-order formulas that relate the program’s resource usage (e.g., execution time) between an arbitrary pair of program runs. Starting with the precondition that two runs have the same public input but different values of the secret, QCHL proof rules allow us to prove that the difference in resource usage is bounded from above by some (user-provided) constant $\epsilon$ . Similar to CHL, our QCHL logic allows effective relational verification by symbolically executing two copies of the program in lockstep. However, QCHL differs from CHL in that it reasons about the program's resource usage behavior and exploits domain-specific assumptions to improve both analysis precision and scalability. Furthermore, since the QCHL proof rules are deterministic (modulo an oracle for finding loop invariants and proving standard Hoare triples), QCHL immediately lends itself to a fully automated verification algorithm.

We have implemented our proposed solution as a tool called THEMIS ${}^{1}$ , a static analyzer for detecting resource-related side-channels in Java applications. To demonstrate its effectiveness, we evaluate THEMIS by performing a series of experiments. First, we compare THEMIS against BLAZER [8], a state-of-the-art static analysis tool for finding timing side channels in Java applications, and we show that TheMIs compares favorably with BLAZER, both in terms of accuracy and running time. Second, we use THEMIs to analyze known side-channel vulnerabilities in security-sensitive Java applications, such as Tomcat and Spring-Security. We show that THEMIS can identify the defects in the original vulnerable versions of these programs, and that THEMIs can verify the correctness of their repaired versions. Finally, we run THEMIS on several real-world Java applications and demonstrate that THEMIS uncovers previously unknown side-channel vulnerabilities in widely-used programs, such as the Eclipse Jetty HTTP web server.

Contributions. In summary, this paper makes the following key contributions:

- We propose the notion of $\epsilon$ -bounded non-interference, which can be used to reason about secret-induced variations in the application's resource usage behavior.

- We present Quantitative Cartesian Hoare Logic (QCHL), a variant of CHL that can be used to verify $\epsilon$ -bounded noninterference.

- We show how to build a scalable, end-to-end side channel detection tool by combining static taint analysis and QCHL.

- We implement our approach in a tool called THEMIS and evaluate it on multiple security-critical Java applications. We also compare TheMIs against BLAZER, a state-of-the-art timing side channel detector for Java. Our results demonstrate that THEMIS is precise, useful, and scalable.

---

BigInteger modPow(BigInteger base,

	BigInteger exponent, BigInteger modulus) \{

	BigInteger s = BigInteger.valueOf(1);

	// BigInteger r ;

	int width = exponent.bitLength(   );

	for ( int i = 0; i < width; i++ ) \{

		s = s.multiply(s).mod(modulus);

		if(exponent.testBit(width - i - 1))

			s = s.multiply(base).mod(modulus);

		//else r = s.multiply(base).mod(modulus);

	\}

	return s;

\}

---

## Figure 1: Gabfeed code snippet that contains a timing side channel (without the commented out lines). A possible fix can be obtained by commenting in lines 4 and 10 .

---

\{

	"epsilon":"0", "costModel":"time",

	"secrets": [ "<com.cyberpointllc.stac.auth.

								KeyExchangeServer:java.math.BigInteger

								secretKey>" ]

---

## Figure 2: THEMIS configuration file for Gabfeed.

- We use THEMIS to find previously unknown security vulnerabilities in widely-used Java applications. Five of the vulnerabilities uncovered by THEMIS were confirmed and fixed by the developers in less than 24 hours.

Organization. The rest of this paper is organized as follows. We start by giving an overview of THEMIS and explain our threat model (Section 2). After formalizing the notion of $\epsilon$ -bounded noninterference in Section 3, we then present our program logic, QCHL, for verifying this 2-safety property (Section 4). We then describe the design and implementation of THEMIS in Section 5 and present the results of our evaluation in Section 6. The limitations of the system as well as comparison against related work are discussed in Sections 7 and 8.

## 2 OVERVIEW

In this section, we give an overview of our technique with the aid of a motivating example and explain the threat model that we assume throughout the paper.

### 2.1 Motivating Example

Suppose that Bob, a security analyst at a government agency, receives a Java web application called Gabfeed, which implements a web forum that allows community members to post and search messages ${}^{2}$ . In this context, both the user names and passwords are considered confidential and are therefore encrypted before being stored in the database. Bob's task is to vet this application and verify that it does not contain timing side-channel vulnerabilities that may compromise user name or password information. However, Gabfeed contains around 30,000 lines of application code (not including any libraries); hence, manually searching for a vulnerability in the application is akin to finding a needle in the haystack.

---

${}^{2}$ Gabfeed is one of the challenge problems from the DARPA STAC project. Please see http://www.darpa.mil/program/space-time-analysis-for-cybersecurity for more details about the STAC project.

${}^{1}$ THEMIS is a Greek goddess for justice and balance, hence the name.

---

A security analyst like Bob can greatly benefit from THEMIS by using it to automatically verify the absence of side-channel vulnerabilities in the target application. To use THEMIs, Bob first identifies application-specific confidential data (in this case, secretKey) and annotates them as such in a THEMIS-specific configuration file, as shown in Figure 2. In the same configuration file, Bob also tells THEMIS the type of side channel to look for (in this case, timing) by specifying the costModel field and provides a reasonable value of $\epsilon$ , using the epsilon field. Here, Bob wants to be conservative and initially sets the value of $\epsilon$ to zero.

Using the information provided by Bob in the configuration file, THEMIS first performs static taint analysis to identify methods that are dependent on confidential data. In this case, one of the methods that access confidential data is modPow, shown in Figure 1. Specifically, THEMIS determines that the second argument (exponent) of modPow is tainted and marks it as a "hot spot" that should be analyzed more precisely using relational verification techniques.

In the next phase, THEMIS uses its Quantitative Cartesian Hoare Logic (QCHL) verifier to analyze modPow in more detail. Specifically, the QCHL verifier considers two executions of modPow that have the same values of base and modulus but that differ in the value of exponent. In this case, the QCHL verifier fails to prove that the resource usage of any such two runs is identical and therefore issues a warning about a possible timing side channel in the modPow procedure.

Next, Bob wonders whether the imbalance in resource usage is large enough to be actually exploitable in practice. For this reason, he plays around with different values of the bound $\epsilon$ , gradually increasing it to larger and larger constants. In the case of timing side channels, $\epsilon$ represents the difference in the executed number of Java bytecode instructions. However, no matter what value of $\epsilon$ Bob picks, THEMIS complains about a possible timing side channel. This observation indeed makes sense because the difference in resource usage is proportional to the secret and can therefore not be bounded by a constant.

Bob now inspects the source code of modPow and realizes that a possible vulnerability arises due to the resource imbalance in the secret-dependent branch from line 8. To fix the vulnerability, Bob adds the code from lines 4 and 10 , with the goal of ensuring that the timing behavior of the program is not dependent on exponent. To confirm that his fix is valid, Bob now runs THEMIS one more time and verifies that his repair eliminates the original vulnerability.

### 2.2 Threat Model

In this paper, we assume that an adversary can observe a program's total resource usage, such as timing, memory, and response size. When measuring resource usage, we further assume that any variations are caused at the application software level. Hence, side channels caused by the microarchitecture such as cache contention [70] and branch prediction [2] are out of the scope of this work. Physical side channels (including power and electromagnetic radiation [31]) can, in principle, be handled by our our system as long as a precise model of the corresponding resource usage is given. We assume that the attacker is not able to observe anything else about the program other than its resource usage.

One possible real-world setting in which the aforementioned assumptions hold could be that the attacker and the victim are connected through a network, and the victim runs a server or P2P software that interacts with other machines through encrypted communications. In this scenario, the attacker and the victim are physically separated; hence, the attacker cannot exploit physical side channels, such as power usage. Furthermore, the attacker does not have a co-resident process or VM running on the victim's machine, thus it is hard to passively observe or actively manipulate OS and hardware-level side channels. What the attacker can do is to either interact with the server and measure the time it takes for the server to respond, or observe the network traffic and measure request and response sizes. In our setting, we assume that data encryption has been properly implemented and the attacker cannot directly read the contents of any packet.

## 3 SIDE-CHANNELS AND BOUNDED NON-INTERFERENCE

In this section, we introduce the property of $\epsilon$ -bounded non-interference, which is the security policy that will be subsequently verified using the THEMIS system.

Let $P$ be a program that takes a list of input values $\overrightarrow{a}$ , and let ${R}_{P}\left( \overrightarrow{a}\right)$ denote the resource usage of $P$ on input $\overrightarrow{a}$ . Following prior work in the literature $\left\lbrack  {{30},{35},{60}}\right\rbrack$ , we assume that each input is marked as either high or low, where high inputs denote security-sensitive data and low inputs denote public data. Let ${\overrightarrow{a}}^{h}$ (resp. ${\overrightarrow{a}}^{l}$ ) be the sublist of the inputs that are marked as high (resp. low). Prior work in the literature $\left\lbrack  {6,{25},{66}}\right\rbrack$ considers a program to be side-channel-free if the following condition is satisfied:

DEFINITION 1. A program $P$ is free of resource-related side-channel vulnerabilities if

$$
\forall \overrightarrow{{a}_{1}},\overrightarrow{{a}_{2}}.\left( {{\overrightarrow{{a}_{1}}}^{l} = {\overrightarrow{{a}_{2}}}^{l} \land  {\overrightarrow{{a}_{1}}}^{h} \neq  {\overrightarrow{{a}_{2}}}^{h}}\right)  \Rightarrow  {R}_{P}\left( \overrightarrow{{a}_{1}}\right)  = {R}_{P}\left( \overrightarrow{{a}_{2}}\right)
$$

The above definition, which is a direct adaptation of the classical notion of non-interference [34], states that a program is free of side channels if the resource usage of the program is deterministic with respect to the public inputs. In other words, the program's resource usage does not correlate with any of its secret inputs.

We believe that Definition 1 is too strong in practice: There are many realistic programs that are considered side-channel-free but that would be deemed vulnerable according to Definition 1. For example, consider a setting in which the attacker is co-located with the victim on a slow network and the resource usage of the program varies by only a few CPU cycles depending on the value of the high input. Since the resource usage of the program is not identical for different high inputs, this program is vulnerable according to Definition 1, but it is practically impossible for an attacker to exploit this vulnerability given the noise in the program's execution environment.

In this paper, we therefore use a relaxed version of the above definition, with the goal of giving security analysts greater flexibility and helping them understand the severity of the resource usage imbalance. Specifically, we propose the following variant of non-interference that we call $\epsilon$ -bounded non-interference:

---

$$
\langle \text{expr}\rangle  :  \mathrel{\text{:=}} \langle \text{const}\rangle \left| {\langle \text{var}\rangle }\right| \langle \text{expr}\rangle  \circ  \langle \text{expr}\rangle
$$

$$
\left( {\circ  \in  \{ +,-,\times ,\vee ,\land ,\ldots \} }\right)
$$

$$
\langle \text{ stmt }\rangle  :  \mathrel{\text{:=}} \text{ skip }\left| {\text{ consume }\left( {\langle \text{ expr }\rangle }\right) }\right| \langle \text{ var }\rangle  \mathrel{\text{:=}} \langle \text{ expr }\rangle
$$

$$
\langle \text{ stmts }\rangle  :  \mathrel{\text{:=}} \langle \text{ stmt }\rangle  \mid  \langle \text{ stmt }\rangle ;\langle \text{ stmts }\rangle
$$

$$
\text{if}\langle \text{expr}\rangle \text{then}\langle \text{stmts}\rangle \text{else}\langle \text{stmts}\rangle
$$

$$
\text{while}\langle \text{expr}\rangle \text{do}\langle \text{stmts}\rangle
$$

$$
\langle \text{params}\rangle  :  \mathrel{\text{:=}} \langle \text{param}\rangle  \mid  \langle \text{param}\rangle ,\langle \text{params}\rangle
$$

$\langle$ param $\rangle  :  \mathrel{\text{:=}} \langle$ annot $\rangle \langle$ var $\rangle$

$\langle$ annot $\rangle  :  \mathrel{\text{:=}}$ low | high

$\langle$ prog $\rangle  :  \mathrel{\text{:=}} \lambda \langle$ params $\rangle .\langle$ stmts $\rangle$

			## Figure 3: Language used in our formalization

---

DEFINITION 2. A program $P$ obeys the $\epsilon$ -bounded non-interference property if

$$
\left. {\forall {\overrightarrow{a}}_{1},{\overrightarrow{a}}_{2}.\;\left( {{\overrightarrow{a}}_{1}{}^{l} = {\overrightarrow{a}}_{2}{}^{l} \land  {\overrightarrow{a}}_{1}{}^{h} \neq  {\overrightarrow{a}}_{2}{}^{h}}\right)  \Rightarrow  \left| {{R}_{P}\left( {\overrightarrow{a}}_{1}\right)  - {R}_{P}\left( {\overrightarrow{a}}_{2}\right) }\right|  \leq  \epsilon }\right)
$$

In this definition, any variation in resource usage below $\epsilon$ is deemed to be a minor imbalance that is unlikely to be exploitable under real-world scenarios. Hence, compared to Definition 1, the notion of $\epsilon$ -bounded interference considers a program to be secure as long as the difference in resource usage is "minor" according to the constant $\epsilon$ . In practice, the value of $\epsilon$ should be chosen by security analysts in light of the security requirements of the application and the underlying threat model. If Definition 2 is violated even for large values of $\epsilon$ , this means the application potentially exhibits large secret-induced variations in resource usage, and hence, the underlying vulnerability is potentially more serious.

## 4 VERIFYING BOUNDED NON-INTERFERENCE USING QCHL

One of the key technical contributions of this paper is a new method for verifying $\epsilon$ -bounded non-interference using QCHL, a variant of Cartesian Hoare Logic introduced in recent work for verifying $k$ -safety [65]. As mentioned in Section 1, QCHL proves triples of the form $\langle \phi \rangle S\langle \psi \rangle$ , where $S$ is a program fragment and $\phi ,\psi$ are first-order formulas that relate the program's resource usage between an arbitrary pair of program runs. Starting with the precondition that the program's low inputs are the same for a pair of program runs, QCHL tries to derive a post-condition that logically implies $\epsilon$ -bounded non-interference.

### 4.1 Language

We will describe our program logic, QCHL, using the simplified imperative language shown in Figure 3. In this language, program inputs are annotated as high or low, indicating private and public data respectively. Atomic statements include skip (i.e., a no-op), assignments of the form $x \mathrel{\text{:=}} e$ , and consume statements, where "consume(e)"indicates the consumption of $e$ units of resource. Our language also supports standard control-flow constructs, including sequential composition, if statements, and loops.

$$
\begin{matrix} P = \lambda \overrightarrow{p}.S\;\forall {p}_{i} \in  \overrightarrow{p}.\Gamma \left( {p}_{i}\right)  = {a}_{i}\;\Gamma  \vdash  S : {\Gamma }^{\prime }, r \\  {R}_{P}\left( \overrightarrow{a}\right)  = r \end{matrix}
$$

$$
\frac{S = \text{ skip }}{\Gamma  \vdash  S : \Gamma ,0}
$$

$$
\frac{S = \left( {x \mathrel{\text{:=}} e}\right) \;\Gamma  \vdash  e : v\;{\Gamma }^{\prime } = \Gamma \left\lbrack  {x \leftarrow  v}\right\rbrack  }{{\Gamma }^{\prime } \vdash  S : {\Gamma }^{\prime },0}
$$

$$
\frac{S = \mathbf{{consume}}\left( e\right) \;\Gamma  \vdash  e : v}{\Gamma  \vdash  S : \Gamma , v}
$$

$$
\begin{matrix} \Gamma  \vdash  {S}_{1} : {\Gamma }_{1},{r}_{1} \\  S = {S}_{1};{S}_{2}\;{\Gamma }_{1} \vdash  {S}_{2} : {\Gamma }_{2},{r}_{2} \\  \Gamma  \vdash  S : {\Gamma }_{2},{r}_{1} + {r}_{2} \end{matrix}
$$

$$
S = \text{ if }e\text{ then }{S}_{1}\text{ else }{S}_{2}\;\begin{array}{l} \Gamma  \vdash  e : \text{ true } \\  \Gamma  \vdash  {S}_{1} : {\Gamma }^{\prime },{r}^{\prime } \\  \Gamma  \vdash  S : {\Gamma }^{\prime },{r}^{\prime } \end{array}
$$

$$
\begin{array}{l} S = \text{ if }e\text{ then }{S}_{1}\text{ else }{S}_{2}\;\Gamma  \vdash  {S}_{2} : {\Gamma }^{\prime },{r}^{\prime } \\  S = \text{ if }e\text{ then }{S}_{1}\text{ else } \\  \end{array}
$$

$$
\frac{S = \text{ while }e\text{ do }{S}^{\prime }\;\Gamma  \vdash  e : \text{ false }}{\Gamma  \vdash  S : \Gamma , r}
$$

$$
\begin{aligned} S = & \text{ while }e\text{ do }{S}^{\prime } & \Gamma  \vdash  {S}^{\prime } : {\Gamma }_{1},{r}_{1} \\   & \frac{\Gamma  \vdash  e : \text{ true }}{\Gamma  \vdash  S : {\Gamma }_{2},{r}_{1} + {r}_{2}} & \\   & \Gamma  \vdash  S : {\Gamma }_{2},{r}_{1} + {r}_{2} &  \end{aligned}
$$

Figure 4: Rules for computing resource usage

Figure 4 defines the cost-instrumented operational semantics of this language using judgments of the form $\Gamma  \vdash  S : {\Gamma }^{\prime }, r$ . The meaning of this judgment is that, assuming we execute $S$ under environment $\Gamma$ (mapping variables to values), then $S$ consumes $r$ units of resource and the new environment is ${\Gamma }^{\prime }$ . As shown in Figure 4, we use the notation ${R}_{P}\left( \overrightarrow{a}\right)$ to denote the resource usage of program $P$ on input vector $\overrightarrow{a}$ . In cases where the resource usage is irrelevant, we simply omit the cost and write $\Gamma  \vdash  S : {\Gamma }^{\prime }$ .

### 4.2 QCHL Proof Rules

We now turn our attention to the proof rules of Quantitative Cartesian Hoare Logic (QCHL), which forms the basis of our verification methodology. Similar to CHL [65], QCHL is a relational program logic that allows proving relationships between multiple runs of the program. However, unlike CHL, QCHL is concerned with proving properties about the difference in resource usage across multiple runs. Towards this goal, QCHL performs cost instrumentation and explicitly tracks the program's resource usage. Furthermore, since our goal is to prove the specific property of $\epsilon$ -bounded noninterference, QCHL exploits domain-specific assumptions by incorporating taint information into the proof rules. Finally, since the QCHL proof rules we describe here are deterministic, our program logic can be immediately translated into a verification algorithm (modulo an oracle for providing loop invariants and proving standard Hoare triples).

Figure 5 presents the proof rules of QCHL. Here, all proof rules, with the exception of Rule (0), derive judgments of the form $\sum  \vdash$ $\langle \Phi \rangle {S}_{1} \circledast  {S}_{2}\langle \Psi \rangle$ , where ${S}_{1}$ and ${S}_{2}$ contain a disjoint set of variables and $\sum$ is a taint environment mapping variables to a taint value drawn from the set $\{$ low, high $\}$ . The notation ${S}_{1} \circledast  {S}_{2}$ describes a program that is semantically equivalent to ${S}_{1};{S}_{2}$ but that is somehow easier to verify (because it tries to execute loops from different executions in lock step). Hence, we have $\sum  \vdash  \langle \Phi \rangle {S}_{1} \circledast  {S}_{2}\langle \Psi \rangle$ if $\{ \Phi \} {S}_{1};{S}_{2}\{ \Psi \}$ is a valid Hoare triple. As we will see shortly, the taint environment $\sum$ is used as a way of increasing the precision and scalability of the analysis. In the remainder of this section, we assume that $\sum$ is sound, i.e., if $\sum \left( x\right)$ is low, then the value of $x$ does not depend (either explicitly or implicitly) on any high inputs. We now explain each of the rules from Figure 5 in more detail.

The first rule labeled (0) corresponds to the top-level verification procedure. If we can derive $\sum  \vdash$ SideChannelFree $\left( {P,\epsilon }\right)$ , then $P$ obeys the $\epsilon$ -bounded non-interference property. In this rule, we use the notation ${S}^{\tau }$ to denote the cost-instrumented version of $S$ , defined as follows:

DEFINITION 3. Given a program $P = \lambda \overrightarrow{p}.S$ , its cost-instrumented version is another program ${P}^{\tau }$ obtained by instrumenting $P$ with a counter variable $\tau$ that tracks its resource usage. More formally, ${P}^{\tau } = \gamma \left( P\right)$ where the instrumentation procedure $\gamma$ is defined as:

- $\gamma \left( {\lambda \overrightarrow{p} \cdot  S}\right)  = \lambda \overrightarrow{p} \cdot  \left( {\tau  \mathrel{\text{:=}} 0;\gamma \left( S\right) }\right)$

- $\gamma \left( \text{skip}\right)  =$ skip

- $\gamma \left( {x \mathrel{\text{:=}} e}\right)  = \left( {x \mathrel{\text{:=}} e}\right)$

- $\gamma$ (consume $\left( e\right) ) = \left( {\tau  \mathrel{\text{:=}} \tau  + e}\right)$

- $\gamma \left( {{S}_{1};{S}_{2}}\right)  = \gamma \left( {S}_{1}\right) ;\gamma \left( {S}_{2}\right)$

- $\gamma$ (if $e$ then ${S}_{1}$ else ${S}_{2}$ ) = if $e$ then $\gamma \left( {S}_{1}\right)$ else $\gamma \left( {S}_{2}\right)$

- $\gamma$ (while $e$ do $S$ ) = while $e$ do $\gamma \left( S\right)$

Essentially, the program ${P}^{\tau }$ is the same as $P$ except that it contains an additional variable $\tau$ that tracks the program’s resource usage. As stated by the following lemma, our instrumentation is correct with respect to the operational semantics from Figure 4.

LEMMA 4.1. Let program $P = \lambda \overrightarrow{p}.S$ and let ${P}^{\tau } = \lambda \overrightarrow{p}.{S}^{\tau }$ . We have

- ${S}^{\tau }$ does not contain any consume statement.

$$
\text{- If}\Gamma \left( \overrightarrow{p}\right)  = \overrightarrow{a}\text{and}\Gamma  \vdash  {S}^{\tau } : {\Gamma }^{\prime }\text{, then}{R}_{P}\left( \overrightarrow{a}\right)  = {\Gamma }^{\prime }\left( \tau \right) \text{.}
$$

Hence, rule (0) from Figure 5 instruments the original program $\lambda \overrightarrow{p}.S$ to obtain a new program $\lambda \overrightarrow{p}.{S}^{\tau }$ that uses a fresh variable $\tau$ to track the program's resource usage. Since bounded non-interference is a 2-safety property, it then creates two $\alpha$ -renamed copies ${S}_{1}^{\tau }$ and ${S}_{2}^{\tau }$ of ${S}^{\tau }$ that have no shared variables and uses the remaining QCHL proof rules to derive a triple

$$
\left\langle  {{\overrightarrow{{p}_{1}}}^{l} = {\overrightarrow{{p}_{2}}}^{l} \land  {\overrightarrow{{p}_{1}}}^{h} \neq  {\overrightarrow{{p}_{2}}}^{h}}\right\rangle  {S}_{1}^{\tau } \circledast  {S}_{2}^{\tau }\langle \Psi \rangle
$$

If the post-condition $\Psi$ logically implies $\left| {{\tau }_{1} - {\tau }_{2}}\right|  \leq  \epsilon$ , we have a proof that the program obeys bounded non-interference. Intuitively, this proof rule considers an arbitrary pair of executions of $S$ where the low inputs are the same and tries to prove that the resource usage of the two runs differs by at most $\epsilon$ .

The remaining rules from Figure 5 derive QCHL triples of the form $\langle \Phi \rangle {S}_{1} \circledast  {S}_{2}\langle \Psi \rangle$ . Our verification algorithm applies these rules in the reverse order shown in Figure 5. That is, we only use rule labeled $i$ if no rule with label $j > i$ is applicable. Hence, unlike standard CHL, our verification method does not perform backtracking search over the proof rules.

$$
\Phi  = \left( {{\overrightarrow{{p}_{1}}}^{l} = {\overrightarrow{{p}_{2}}}^{l} \land  {\overrightarrow{{p}_{1}}}^{h} \neq  {\overrightarrow{{p}_{2}}}^{h}}\right)
$$

$$
\lambda {\overrightarrow{p}}_{1} \cdot  {S}_{1}^{\tau } = \alpha \left( {\lambda \overrightarrow{p} \cdot  {S}^{\tau }}\right) \;\sum  \vdash  \langle \Phi \rangle {S}_{1}^{\tau } \circledast  {S}_{2}^{\tau }\langle \Psi \rangle
$$

$$
\frac{\lambda {\overrightarrow{p}}_{2} \cdot  {S}_{2}^{\tau } = \alpha \left( {\lambda \overrightarrow{p} \cdot  {S}^{\tau }}\right)  \vDash  \Psi  \rightarrow  \left| {{\tau }_{1} - {\tau }_{2}}\right|  \leq  \epsilon }{\sum  \vdash  \text{ SideChannelFree }\left( {\lambda \overrightarrow{p} \cdot  S,\epsilon }\right) }\left( 0\right)
$$

$$
\frac{\sum  \vdash  \langle \Phi \rangle {S}_{2} \circledast  {S}_{1}\langle \Psi \rangle }{\sum  \vdash  \langle \Phi \rangle {S}_{1} \circledast  {S}_{2}\langle \Psi \rangle }\left( 1\right)
$$

$$
S \neq  \left( {{S}_{1};{S}_{2}}\right)
$$

$$
\frac{\sum  \vdash  \langle \Phi \rangle S;\mathbf{{skip}} \circledast  {S}^{\prime }\langle \Psi \rangle }{\sum  \vdash  \langle \Phi \rangle S \circledast  {S}^{\prime }\langle \Psi \rangle }\left( 2\right)
$$

$$
\vdash  \{ \Phi \} {S}_{1}\left\{  {\Phi }^{\prime }\right\}
$$

$$
\sum  \vdash  \left\langle  {\Phi }^{\prime }\right\rangle  {S}_{2} \circledast  {S}_{3}\langle \Psi \rangle
$$

$$
\frac{{S}_{1} = \operatorname{skip} \vee  {S}_{1} = \left( {v \mathrel{\text{:=}} e}\right) }{\sum  \vdash  \langle \Phi \rangle {S}_{1};{S}_{2} \circledast  {S}_{3}\langle \Psi \rangle }\left( 3\right)
$$

$$
\frac{ \vdash  \{ \Phi \} S\{ \Psi \} }{\sum  \vdash  \langle \Phi \rangle S \circledast  \mathbf{{skip}}\langle \Psi \rangle }\left( 4\right)
$$

$$
\sum  \vdash  \langle \Phi  \land  e\rangle {S}_{1};S \circledast  {S}_{3}\left\langle  {\Psi }_{1}\right\rangle
$$

$$
\frac{\sum  \vdash  \langle \Phi  \land  \neg e\rangle {S}_{2};S \circledast  {S}_{3}\left\langle  {\Psi }_{2}\right\rangle  }{\sum  \vdash  \langle \Phi \rangle \text{ if }e\text{ then }{S}_{1}\text{ else }{S}_{2};S \circledast  {S}_{3}\left\langle  {{\Psi }_{1} \vee  {\Psi }_{2}}\right\rangle  }\left( 5\right)  \tag{5}
$$

$$
\vdash  \{ \Phi \} \text{while}{e}_{1}\text{do}{S}_{1}\left\{  {\Phi }^{\prime }\right\}
$$

$$
\vdash  \left\{  {\Phi }^{\prime }\right\}  \text{while}{e}_{2}\text{do}{S}_{2}\left\{  {\Psi }^{\prime }\right\}
$$

$$
\frac{\sum  \vdash  \left\langle  {\Psi }^{\prime }\right\rangle  S \circledast  {S}^{\prime }\langle \Psi \rangle }{\sum  \vdash  \langle \Phi \rangle \text{ while }{e}_{1}\text{ do }{S}_{1};S \circledast  \text{ while }{e}_{2}\text{ do }{S}_{2};{S}^{\prime }\langle \Psi \rangle }\left( 6\right)  \tag{6}
$$

$$
\sum  \vdash  \text{CanSynchronize}\left( {{e}_{1},{e}_{2},{S}_{1},{S}_{2}, I}\right)
$$

$$
\sum  \vdash  \left\langle  {I \land  {e}_{1} \land  {e}_{2}}\right\rangle  {S}_{1} \circledast  {S}_{2}\left\langle  {I}^{\prime }\right\rangle
$$

$$
\sum  \vdash  \left\langle  {I\land \neg {e}_{1}\land \neg {e}_{2}}\right\rangle  S \circledast  {S}^{\prime }\langle \Psi \rangle
$$

$$
\frac{ \vDash  \Phi  \rightarrow  I\; \vDash  {I}^{\prime } \rightarrow  I}{\sum  \vdash  \langle \Phi \rangle \text{ while }{e}_{1}\text{ do }{S}_{1};S \circledast  \text{ while }{e}_{2}\text{ do }{S}_{2};{S}^{\prime }\langle \Psi \rangle }(: \tag{7}
$$

Figure 5: QCHL proof rules. The notation $\alpha \left( S\right)$ denotes an $\alpha$ -renamed version of statement $S$ .

Let us now consider the remaining rules in more detail: Rule (1) is the same as commutativity rule in CHL and states that the - operator is symmetric. Intuitively, since ${S}_{1}$ and ${S}_{2}$ do not share variables, any interleaving of ${S}_{1}$ and ${S}_{2}$ will yield the same result, and we can therefore commute the two operands when deriving QCHL triples. As will become clear shortly, the commutativity rule ensures that our verification algorithm makes progress when none of the other rules are applicable.

The next rule states that we are free to append a skip statement to any non-sequential statement without affecting its meaning. While this rule may not seem very useful on its own, it allows us to avoid redundancies in the proof system by bringing each ${S}_{1} \circledast  {S}_{2}$ to a canonical form where ${S}_{1}$ is always of the form $S;{S}^{\prime }$ or ${S}_{2}$ is skip.

Rule (3) specifies the verification logic for ${S}_{1} \circledast  {S}_{2}$ when ${S}_{1}$ is of the form $A;S$ where $A$ is an atomic statement. In this case, we simply "consume" $A$ by deriving the Hoare triple $\{ \Phi \} A\left\{  {\Phi }^{\prime }\right\}$ and then use ${\Phi }^{\prime }$ as a precondition for $S \circledast  {S}_{2}$ .

Rule (4) serves as the base case for our logic. When we want to prove $\langle \Phi \rangle S \circledast$ skip $\langle \Psi \rangle$ , we immediately reduce this judgement to the standard Hoare triple $\{ \Phi \} S\{ \Psi \}$ because skip is just a no-op.

Example. Suppose we want to prove (0-bounded) non-interference for the following program:

---

$\lambda \left( {\operatorname{low}x}\right)$ . consume(x); skip;

---

First we apply transformation $\gamma$ and get the resource instrumented program:

---

$\lambda \left( {\operatorname{low}x}\right) .\tau  = 0;\tau  = \tau  + x;$ skip;

---

Ignore the taint environment for now, as we will not use it in this example. According to rule (0), we only need to prove

---

$$
\left\langle  {{x}_{1} = {x}_{2}}\right\rangle  {\tau }_{1} = 0;{\tau }_{1} = {\tau }_{1} + {x}_{1};\text{ skip; } \circledast
$$

---

$$
{\tau }_{2} = 0;{\tau }_{2} = {\tau }_{2} + {x}_{2};\operatorname{skip};\left\langle  {{\tau }_{1} = {\tau }_{2}}\right\rangle
$$

Applying rule (3) twice, we can reduce the above judgement to the following one:

$$
\left\langle  {{x}_{1} = {x}_{2} \land  {\tau }_{1} = {x}_{1}}\right\rangle  \text{ skip; } \circledast  {\tau }_{2} = 0;{\tau }_{2} = {\tau }_{2} + {x}_{2}\text{; skip; }\left\langle  {{\tau }_{1} = {\tau }_{2}}\right\rangle
$$

Swapping the two operands of $\circledast$ with rule (1), we get

$$
\left\langle  {{x}_{1} = {x}_{2} \land  {\tau }_{1} = {x}_{1}}\right\rangle  {\tau }_{2} = 0;{\tau }_{2} = {\tau }_{2} + {x}_{2};\text{ skip; } \circledast  \text{ skip; }\left\langle  {{\tau }_{1} = {\tau }_{2}}\right\rangle
$$

After applying rule (4), we get

$$
\left\{  {{x}_{1} = {x}_{2} \land  {\tau }_{1} = {x}_{1}}\right\}  {\tau }_{2} = 0;{\tau }_{2} = {\tau }_{2} + {x}_{2};\text{ skip; }\left\{  {{\tau }_{1} = {\tau }_{2}}\right\}
$$

Applying Hoare-style strongest postcondition computation, the above Hoare triple can be reduced to

$$
\left\{  {{x}_{1} = {x}_{2} \land  {\tau }_{1} = {x}_{1} \land  {\tau }_{2} = {x}_{2}}\right\}  \text{ skip; }\left\{  {{\tau }_{1} = {\tau }_{2}}\right\}
$$

Since this Hoare triple is clearly valid, we have proven non-interference using the QCHL proof rules.

Rule (5) specifies the general verification logic for branch statements. This rule is an analog of the conditional rule in standard Hoare logic: we can verify an if statement by embedding the branch condition $e$ into the true branch and its negation $\neg e$ into the false branch and carry out the proof for both branches accordingly.

Rule (6) specifies the general verification logic for loops. Without loss of generality, this rule requires both sides of the $\circledast$ operator to be loops: If one side is a not a loop, we can always apply one of the other rules, using rule (1) to swap the loop to the other side if necessary. The idea here is to apply self-composition [14]: we run the loop on the left-hand side first, followed by the loop on the right-hand side, and try to derive the proof as if the two loops are sequentially composed.

While rule (6) is sound, it is typically difficult to prove 2-safety using rule (6) alone. In particular, rule (6) does not allow us to synchronize executions between the two loops, so the resulting Hoare triples are often hard to verify. The following example illustrates this issue:

Example. Consider the following code snippet:

$$
\frac{{e}_{1} = \alpha \left( e\right) \;{e}_{2} = \alpha \left( e\right) }{{e}_{1}{ \equiv  }_{\alpha }{e}_{2}}
$$

$$
\frac{{S}_{1} = \alpha \left( S\right) \;{S}_{2} = \alpha \left( S\right) }{{S}_{1}{ \equiv  }_{\alpha }{S}_{2}}
$$

$$
\begin{matrix} {e}_{1}{ \equiv  }_{\alpha }{e}_{2} & \sum  \vdash  {e}_{1} : \text{ low } \\  {S}_{1}{ \equiv  }_{\alpha }{S}_{2} & \sum  \vdash  {e}_{2} : \text{ low } \\  \sum  \vdash  \operatorname{CanSynchronize}\left( {{e}_{1},{e}_{2},{S}_{1},{S}_{2}, I}\right) &  \end{matrix}
$$

$$
\frac{\mid  = I \rightarrow  \left( {{e}_{1} \leftrightarrow  {e}_{2}}\right) }{\sum  \vdash  \operatorname{CanSynchronize}\left( {{e}_{1},{e}_{2},{S}_{1},{S}_{2}, I}\right) }
$$

## Figure 6: Helper rules for figure 5

---

$\lambda \left( {\operatorname{low}n,\operatorname{low}k}\right)$ .

	i = 0 ;

	while (i < n) \{

		consume(i); i = i + k;

	\}

---

To prove that this program obeys $\epsilon$ -bounded non-interference, we need to show that the difference in resource consumption after executing the two copies of the loop is at most $\epsilon$ . However, to prove this property using rule (6), we would need to infer a precise post-condition about resource consumption. Unfortunately, this requires inferring a complex non-linear loop invariant involving $i, n, k$ . Since such loop invariants are difficult to infer, we cannot prove non-interference using rule (6).

Rule (7) is one of the most important rules underlying QCHL, as it allows us to execute loops from different executions in lockstep. This loop can be applied only when the CanSynchronize predicate is true, meaning that the two loops are guaranteed to execute the same number of times. The definition of the CanSynchronize predicate is shown in Figure 6: Given two loops ${L}_{1} \equiv$ while $\left( {e}_{1}\right)$ do ${S}_{1}$ and ${L}_{2} \equiv$ while $\left( {e}_{2}\right)$ do ${S}_{2}$ , and a loop invariant $I$ for the "fused" loop while $\left( {{e}_{1} \land  {e}_{2}}\right)$ do ${S}_{1};{S}_{2}$ , CanSynchronize determines if ${L}_{1}$ and ${L}_{2}$ must execute the same number of times. In the easy case, this information can be determined using only taint information: Specifically, suppose that ${L}_{1},{L}_{2}$ are identical modulo variable renaming and ${e}_{1},{e}_{2}$ contains only untainted (low) variables. Since we prove bounded non-interference under the assumption that low variables from the two runs have the same value, this assumption implies ${L}_{1}$ and ${L}_{2}$ must execute the same number of times. If we cannot prove the CanSynchronize predicate using taint information alone, we may still be able to prove it using the invariant $I$ for the fused loop. Specifically, if $I$ logically implies ${e}_{1} \leftrightarrow  {e}_{2}$ , we know that after each iteration ${e}_{1},{e}_{2}$ have the same truth value; hence, the loops must again execute the same number of times.

Now, suppose we can prove that CanSynchronize evaluates to true. In this case, rule (7) conceptually executes the two loops in lockstep. Specifically, the premise $\sum  \vdash  \left\langle  {I \land  {e}_{1} \land  {e}_{2}}\right\rangle  {S}_{1} \circledast  {S}_{2}\left\langle  {I}^{\prime }\right\rangle$ , together with $\vDash  {I}^{\prime } \rightarrow  I$ , ensures that $I$ is an inductive invariant of the fused loop while $\left( {{e}_{1} \land  {e}_{2}}\right)$ do ${S}_{1};{S}_{2}$ . Thus, $I$ must hold when the both loops terminate. Thus, we can safely use the predicate $I \land  \neg {e}_{1} \land  \neg {e}_{2}$ as a precondition when reasoning about the "continuations" $S$ and ${S}^{\prime }$ .

Algorithm 1 Relational Invariant Generation

---

Input: $\sum$ , the taint environment.

Input: $\Phi$ , the pre-condition of the loop.

Input: $e, S$ , loop condition and loop body.

Input: $V$ , the set of all variables appeared in the loop.

Output: An inductive relational loop invariant

	function RelationalInvGen $\left( {\sum ,\Phi , e, S, V}\right)$

			$\left( {{e}_{1},{S}_{1}}\right)  \leftarrow  \alpha \left( {e, S}\right)$

			$\left( {{e}_{2},{S}_{2}}\right)  \leftarrow  \alpha \left( {e, S}\right)$

			Guesses $\leftarrow  \left\{  {{v}_{1} = {v}_{2} \mid  v \in  V}\right\}$

			for $g \in$ Guesses do

				if $\vDash  \Phi  \rightarrow  g$ then

					Guesses $\leftarrow$ Guesses $\smallsetminus  \{ g\}$

			inductive $\leftarrow$ false

		while $\neg$ inductive do

				$I \leftarrow  \mathop{\bigwedge }\limits_{{g \in  {Guesses}}}g$

				assume $\sum  \vdash  \left\langle  {I \land  {e}_{1} \land  {e}_{2}}\right\rangle  {S}_{1} \circledast  {S}_{2}\left\langle  {I}^{\prime }\right\rangle$

				inductive $\leftarrow$ true

				for $g \in$ Guesses do

					if $\vDash  {I}^{\prime } \rightarrow  g$ then

						Guesses $\leftarrow$ Guesses $\smallsetminus  \{ g\}$

						inductive $\leftarrow$ false

		return $\mathop{\bigwedge }\limits_{{g \in  {Guesses}}}g$

---

Example. In the previous example, we illustrated that it is difficult to prove non-interference using rule (6) even for a relatively simple example. Let us now see why rule (7) makes verifying 2-safety easier. Since $i$ and $n$ are both low according to the taint environment $\sum$ , we can show that the the CanSynchronize predicate evaluates to true. To prove that the program obeys non-interference, we use the relational loop invariant $I = \left( {{i}_{1} = {i}_{2} \land  {\tau }_{1} = {\tau }_{2} \land  {k}_{1} = {k}_{2}}\right)$ . It is easy to see that $I$ is a suitable inductive relational loop invariant, because:

- ${i}_{1},{i}_{2},{\tau }_{1},{\tau }_{2}$ are all set to 0 before the loop starts.

- We know ${k}_{1} = {k}_{2}$ from the precondition (since they are low inputs)

- ${i}_{1}$ and ${i}_{2}$ are increased by the same amount in each iteration of the loop since ${k}_{1} = {k}_{2}$ .

- ${\tau }_{1}$ and ${\tau }_{2}$ are also increased by the same amount in each iteration of the loop since ${i}_{1} = {i}_{2}$ .

- $I$ implies the post condition $\left| {{\tau }_{1} - {\tau }_{2}}\right|  \leq  0$ .

Observe that the use of rule (7) allows us to prove the desired property without reasoning about the total resource consumption of the loop. Hence, we do not need complicated non-linear loop invariants, and the verification task becomes much easier to automate.

THEOREM 4.2 (SOUNDNESS). Assuming soundness of taint environment $\sum$ , if $\sum  \vdash$ SideChannelFree $\left( {\lambda \overrightarrow{p}.S,\epsilon }\right)$ , then the program $\lambda \overrightarrow{p}.S$ does not have an $\epsilon$ -bounded resource side-channel.

Proof of this theorem can be found in appendix A.

### 4.3 Loop Invariant Generation

In the previous subsection, we assumed the existence of an oracle for finding suitable relational loop invariants (recall rule 7). Here, by "relational loop invariant", we mean a simulation relation over variables in programs ${S}_{1},{S}_{2}$ . Specifically, we use such relational loop invariants in two ways: First, we use them to check whether two loops execute the same number of times. Second, we use the relational loop invariant to compute the precondition for the continuations of the two programs. Hence, to apply rule 7, we need an algorithm for computing such relational loop invariants.

![0196f761-044b-7a80-b532-ca40657b18a2_6_922_230_756_436_0.jpg](images/0196f761-044b-7a80-b532-ca40657b18a2_6_922_230_756_436_0.jpg)

Figure 7: Workflow of the THEMIS tool

Algorithm 1 shows our inference engine for computing relational loop invariants. This algorithm can be viewed as an instance of monomial predicate abstraction (i.e., guess-and-check) [27, 47, 61]. Specifically, we consider the universe Guesses of predicates ${v}_{1} = {v}_{2}$ relating variables from the two loops. Because synchronizable loops execute the same number of times, they typically contain one or more "anchor" variables that are pairwise equal. Thus, we can often find useful relational invariants over this universe of predicates.

Considering Algorithm 1 in more detail, we first filter our those predicates that are not implied by the precondition $\Phi$ (lines 5-7). In lines 9-16, we then further filter out those predicates that are not preserved in the loop body. In particular, on line 10 , we construct the candidate invariant by conjoining all remaining predicates in our guess set, and, on line 11, we compute the post condition ${I}^{\prime }$ of the loop using the proof rules shown in figure 5 . Since we have $\sum  \vdash  \left\langle  {I \land  {e}_{1} \land  {e}_{2}}\right\rangle  {S}_{1} \circledast  {S}_{2}\left\langle  {I}^{\prime }\right\rangle$ and $\vDash  {I}^{\prime } \rightarrow  g$ , this means predicate $g$ is not preserved by the loop body and is therefore removed from our set of predicates. When the loop in lines 9-16 terminates, we have $\sum  \vdash  \left\langle  {I \land  {e}_{1} \land  {e}_{2}}\right\rangle  {S}_{1} \circledast  {S}_{2}\left\langle  {I}^{\prime }\right\rangle$ and $\vDash  {I}^{\prime } \rightarrow  I$ ; thus, $I$ is an inductive relational loop invariant.

## 5 SYSTEM DESIGN AND IMPLEMENTATION

In this section, we discuss the design and implementation of THEMIS, our end-to-end static analysis tool for verifying bounded noninterference. While the QCHL verifier discussed in the previous section is one of the key components of TheMIs, it is not necessary (and also not scalable) to employ such precise relational reasoning throughout the entire program. Hence, as mentioned earlier, our approach employs taint analysis to identify program parts that require more precise analysis.

### 5.1 Design Overview

Figure 7 gives a high-level schematic overview of THEMIS's architecture. In addition to the QCHL verifier discussed in detail in Section 4, THEMIS also incorporates pointer and taint analyses and instruments the program to explicitly track resource usage. We now give a brief overview of each of these components.

Pointer analysis. Given the bytecode of a Java application, THEMIS performs (field- and object-sensitive) pointer analysis to build a precise call graph and identify all variables that may alias each other. The resulting call graph and alias information are used by the subsequent taint analysis as well as the QCHL verifier.

Taint analysis. The use of taint analysis in THEMIS serves two goals: First, the QCHL verifier uses the results of the taint analysis to determine whether two loops can be synchronized. Second, we use taint analysis to identify hotspots that need to be analyzed more precisely using the QCHL verifier.

The taint analyzer uses the annotations in THEMIS's configuration file to determine taint sources (i.e., high inputs) and propagates taint using a field- and object-sensitive analysis. Our taint analyzer tracks both explicit and implicit flows. That is, a variable $v$ is considered tainted if (a) there is an assignment $v \mathrel{\text{:=}} e$ such that $e$ is tainted (explicit flow), or (b) a write to $v$ occurs inside a branch whose predicate is tainted (implicit flow).

We use the results of the taint analysis to identify methods that should be analyzed by the QCHL verifier. A method $m$ is referred to as hot spot if it reads from a tainted variable. We say that a hot spot $m$ dominates another hot spot ${m}^{\prime }$ if ${m}^{\prime }$ is a transitive callee of $m$ but not the other way around. Any hot spot that does not have dominators is given as an entry point to the QCHL verifier. In principle, this strategy of running the QCHL verifier on only hot spots can cause our analysis to report false positives. For instance, consider the following example:

---

main(...) \{ foo(   ); bar(   ); \}

foo(   ) \{

	int x = readSecret(   );

	if(x > 0) consume(1); else consume(100);

\}

bar(   ) \{

	int y = readSecret(   );

	if(y <= 0) consume(1); else consume(100);

\}

---

While this program does not have any secret-dependent imbalance in resource usage, foo and bar individually do not obey noninterference, causing our analysis to report false positives. However, in practice, we have not observed any such false positives, and this strategy greatly increases the scalability of the tool.

Resource usage instrumentation. The language we considered for our formalization in Section 4 is equipped with a consume(x) statement that models consumption of $x$ units of resource. Unfortunately, since Java programs do not come with such statements, our implementation uses a cost model to instrument the program with such consume statements. In principle, our framework can detect different classes of side channels, provided that the tool is given a suitable cost model for the corresponding resource type.

Our current implementation provides cost models for two kinds of resource usage, namely, timing and response size. For timing, we use a coarse cost model where every byte code instruction is assumed to have unit cost. For response size, each string $s$ that is appended to the response consumes s.length(   ) units of resource.

Counterexample generation. If THEMIs fails to verify the bounded non-interference property for a given $\epsilon$ , it can also generate counterexamples by using the models provided by the underlying SMT solver. In particular, when the verification condition (VC) generated by THEMIs is invalid, the tool asks the SMT solver for a falsifying assignment and pretty-prints the model returned from Z3 by replacing Z3 symbols with their corresponding variable names. Since the VCs depend on automatically inferred loop invariants, the counterexamples generated by THEMIS may be spurious if the inference engine does not infer sufficiently strong loop invariants.

### 5.2 System Implementation

The THEMIS system is implemented in a combination of Java and OCaml and leverages multiple existing tools, such as Soot [67], Z3 [24], and Apron [44]. Specifically, our pointer analysis builds on top of Soot [67], and we extend the taint analysis provided by FlowDroid [9], which is a state-of-the-art context-, field-, flow-, and object-sensitive taint analyzer, to also track implicit flows. Our QCHL verifier is implemented in OCaml and uses the Z3 SMT solver [24] to discharge the generated verification conditions. To prove the Hoare triples that arise as premises in the QCHL proof rules, we perform standard weakest precondition computation, leveraging the Apron numerical abstract domain library [44] to infer standard loop invariants. Recall that we infer relational loop invariants using the monomial predicate abstraction technique described in Section 4.3.

Our formal description of QCHL in section 4 uses a simplified programming language that does not have many of the complexities of Java. TheMIS handles these complexities by first leveraging the Soot framework to parse the Java bytecode to Soot IR, and then using an in-house "front-end" that further lowers Soot IR into a form closer to what is presented in section 4. In particular, the transformation from Soot to our IR recovers program structures (loops, conditionals etc.) and encodes heap accesses in terms of arrays. The verifier performs strongest postcondition calculation over our internal IR and encodes verification conditions with SMT formulae. In the remainder of this section, we explain how we handle various challenges that we encountered while building the THEMIS frontend.

Object encoding. Since objects are pervasive in Java applications, their encoding has a significant impact on the precision and scala-bility of the approach. In THEMIS, we adopt a heap encoding that is similar to ESC-Java [28]. Specifically, instance fields of objects are represented as maps from object references (modeled as integer variables) to the value of the corresponding field. Reads and writes to the map are modeled using select and update functions defined by the theory of arrays in SMT solvers. If two object references are known not to be the same (according to the results of the pointer analysis), we then add a disequality constraint between the corresponding variables.

Method invocation. Since the simplified language from Section 4 did not allow function calls, we only described an intraprocedural version of the QCHL verifier. We currently perform interprocedural analysis by function inlining, which is performed as a preprocessing step at the internal IR level before the analysis takes place. Since the QCHL verifier only needs to analyze hot spots (which typically constitute a small fraction of the program), we do not find inlining to be a major scalability bottleneck. However, since recursive procedures cannot be handled using function inlining, our current implementation requires models for recursive procedures that correspond to hot spots.

Virtual calls and instanceof encoding. The result of certain operations in the Java language, such as virtual calls and the instanceof operator, depends on the runtime values of their operands. To faithfully model those operations, we encode the type of each allocation site as one of its field, and we transform virtual calls and instanceof to a series of if statements that branch on this field. For example, if variable a may point to either allocation ${A}_{1}$ of type T1 or allocation ${A}_{2}$ of type T2, then the polymorphic call site a. foo(   ) will be modeled as:

---

if (a.type == T1)

	((T1)a).foo(   );

else if (a.type == T2)

	((T2)a).foo(   );

---

We handle the instanceof operator in a similar way.

## 6 EVALUATION

In this section, we describe our evaluation of THEMIS on a set of security-critical Java applications. Our evaluation is designed to answer the following research questions:

Q1. How does THEMIs compare with state-of-the-art tools for side channel detection in terms of accuracy and scalability?

Q2. Is THEMIS able to detect known vulnerabilities in real-world Java applications, and can THEMIS verify their repaired versions?

Q3. Is THEMIS useful for detecting zero-day vulnerabilities from the real world?

In what follows, we describe a series of experiments that are designed to answer the above questions. All experiments are conducted on an Intel Xeon(R) computer with an E5-1620 v3 CPU and 64G of memory running on Ubuntu 16.04.

### 6.1 Comparison Against BLAZER

To evaluate how competitive THEMIS is with existing tools, we compare THEMIS against BLAZER [8], a state-of-the-art tool for detecting timing side channels in Java bytecode. BLAZER is a static analyzer that uses a novel decomposition technique for proving non-interference properties. Since the BLAZER tool is not publicly available, we compare TheMIs against BLAZER on the same 22 benchmarks that are used to evaluate BLAZER in their PLDI'17 paper [8]. These benchmarks include a combination of challenge problems from the DARPA STAC program, classic examples from previous literature $\left\lbrack  {{33},{46},{55}}\right\rbrack$ , and some microbenchmarks constructed by the developers of BLAZER. Since BLAZER verifies standard noninterference (rather than our proposed $\epsilon$ -bounded variant), we set the value of $\epsilon$ to be 0 when running THEMIS.

<table><tr><td rowspan="2">Benchmark</td><td rowspan="2">Version</td><td rowspan="2">Size</td><td colspan="2">Time (s)</td></tr><tr><td>Blazer</td><td>Themis</td></tr><tr><td colspan="5">MicroBench</td></tr><tr><td>array</td><td>Safe</td><td>16</td><td>1.60</td><td>0.28</td></tr><tr><td>array</td><td>Unsafe</td><td>14</td><td>0.16</td><td>0.23</td></tr><tr><td>loopAndbranch</td><td>Safe</td><td>15</td><td>0.23</td><td>0.33</td></tr><tr><td>loopAndbranch</td><td>Unsafe</td><td>15</td><td>0.65</td><td>0.16</td></tr><tr><td>nosecret</td><td>Safe</td><td>7</td><td>0.35</td><td>0.20</td></tr><tr><td>notaint</td><td>Unsafe</td><td>9</td><td>0.28</td><td>0.12</td></tr><tr><td>sanity</td><td>Safe</td><td>10</td><td>0.63</td><td>0.41</td></tr><tr><td>sanity</td><td>Unsafe</td><td>9</td><td>0.30</td><td>0.17</td></tr><tr><td>straightline</td><td>Safe</td><td>7</td><td>0.21</td><td>0.49</td></tr><tr><td>straightline</td><td>Unsafe</td><td>7</td><td>22.20</td><td>5.30</td></tr><tr><td colspan="5">STAC</td></tr><tr><td>modPow1</td><td>Safe</td><td>18</td><td>1.47</td><td>0.61</td></tr><tr><td>modPow1</td><td>Unsafe</td><td>58</td><td>218.54</td><td>14.16</td></tr><tr><td>modPow2</td><td>Safe</td><td>20</td><td>1.62</td><td>0.75</td></tr><tr><td>modPow2</td><td>Unsafe</td><td>106</td><td>7813.68</td><td>141.36</td></tr><tr><td>passwordEq</td><td>Safe</td><td>16</td><td>2.70</td><td>1.10</td></tr><tr><td>passwordEq</td><td>Unsafe</td><td>15</td><td>1.30</td><td>0.39</td></tr><tr><td colspan="5">Literature</td></tr><tr><td>k96</td><td>Safe</td><td>17</td><td>0.70</td><td>0.61</td></tr><tr><td>k96</td><td>Unsafe</td><td>15</td><td>1.29</td><td>0.54</td></tr><tr><td>gpt14</td><td>Safe</td><td>15</td><td>1.43</td><td>0.46</td></tr><tr><td>gpt14</td><td>Unsafe</td><td>26</td><td>219.30</td><td>1.25</td></tr><tr><td>login</td><td>Safe</td><td>16</td><td>1.77</td><td>0.54</td></tr><tr><td>login</td><td>Unsafe</td><td>11</td><td>1.79</td><td>0.70</td></tr></table>

Figure 8: Comparison between THEMIS and BLAZER.

We summarize the results of our comparison against BLAZER in Table 8. ${}^{3}$ One of the key points here is that THEMIS is able to automatically verify all 25 programs from the BLAZER data set. Moreover, we see that THEMIS is consistently faster than BLAZER except for a few benchmarks that take a very short time to analyze. On average, THEMIS takes a median of 7.73 seconds to verify a benchmark, whereas the median running time of BLAZER is 376.92 seconds.

### 6.2 Detection of Known Vulnerabilities

To demonstrate that TheMIS can be used to detect non-trivial vulnerabilities in real-world Java programs, we further evaluate THEMIS on security-sensitive Java frameworks. The benchmarks we collect come from the following sources:

(1) Response-size side-channel benchmarks from existing publication [73] ${}^{4}$ .

---

${}^{3}$ The BLAZER paper reports two sets of numbers for running time, namely time for safety verification alone, and time including attack specification search. Since THEMIS does not perform the latter, we only compare time for safety verification. For the "Size" column in the table, we use the original metric from BLAZER, which indicates the number of basic blocks.

${}^{4}$ We are only able to obtain the source codes for 2 of 3 benchmarks mentioned in the paper.

---

<table><tr><td>Benchmark</td><td>Version</td><td>LOC</td><td>LOC’</td><td>$\epsilon  = {64}$</td><td>$\epsilon  = 0$</td><td>Time (s)</td></tr><tr><td>Spring-Security</td><td>Safe</td><td>1630</td><td>41</td><td>✓</td><td>✓</td><td>1.70</td></tr><tr><td>Spring-Security</td><td>Unsafe</td><td>1602</td><td>32</td><td>✓</td><td>✓</td><td>1.09</td></tr><tr><td>JDK7-MsgDigest</td><td>Safe</td><td>633</td><td>30</td><td>✓</td><td>✓</td><td>1.27</td></tr><tr><td>JDK6-MsgDigest</td><td>Unsafe</td><td>619</td><td>27</td><td>✓</td><td>✓</td><td>1.33</td></tr><tr><td>Picketbox</td><td>Safe</td><td>208</td><td>73</td><td>✓</td><td>✘</td><td>1.79</td></tr><tr><td>Picketbox</td><td>Unsafe</td><td>180</td><td>65</td><td>✓</td><td>✓</td><td>1.55</td></tr><tr><td>Tomcat</td><td>Safe</td><td>12221</td><td>100</td><td>✓</td><td>✘</td><td>9.93</td></tr><tr><td>Tomcat</td><td>Unsafe</td><td>12173</td><td>96</td><td>✓</td><td>✓</td><td>8.64</td></tr><tr><td>Jetty</td><td>Safe</td><td>2667</td><td>77</td><td>✓</td><td>✓</td><td>2.50</td></tr><tr><td>Jetty</td><td>Unsafe</td><td>2619</td><td>76</td><td>✓</td><td>✓</td><td>2.07</td></tr><tr><td>orientdb</td><td>Safe</td><td>19564</td><td>134</td><td>✓</td><td>✘</td><td>37.99</td></tr><tr><td>orientdb</td><td>Unsafe</td><td>19413</td><td>131</td><td>✓</td><td>✓</td><td>38.09</td></tr><tr><td>pac4j</td><td>Safe</td><td>1978</td><td>104</td><td>✓</td><td>✘</td><td>3.97</td></tr><tr><td>pac4j</td><td>Unsafe</td><td>1900</td><td>105</td><td>✓</td><td>✓</td><td>1.85</td></tr><tr><td>boot-auth</td><td>Safe</td><td>7106</td><td>74</td><td>✓</td><td>✘</td><td>9.12</td></tr><tr><td>boot-auth</td><td>Unsafe</td><td>6977</td><td>69</td><td>✓</td><td>✓</td><td>8.31</td></tr><tr><td>tourPlanner</td><td>Safe</td><td>7735</td><td>46</td><td>✓</td><td>✓</td><td>22.22</td></tr><tr><td>tourPlanner</td><td>Unsafe</td><td>7660</td><td>34</td><td>✓</td><td>✓</td><td>22.01</td></tr><tr><td>Dyna_table</td><td>Unsafe</td><td>175</td><td>40</td><td>✓</td><td>✓</td><td>1.165</td></tr><tr><td>Advanced_table</td><td>Unsafe</td><td>232</td><td>55</td><td>✓</td><td>✓</td><td>2.01</td></tr></table>

Figure 9: Evaluation on existing vulnerabilities. A checkmark (✓) indicates that THEMIS gives the correct result, while $x$ indicates a false positive.

(2) One benchmark that contains a response-size side channel from the DARPA STAC project.

(3) A well-known timing side channel in the MessageDigest class from JDK6.

(4) Seven other benchmarks with known vulnerabilities collected from Github.

Benchmarks that fall in the first two categories contain response-size side-channel vulnerabilities, and all other benchmarks contain timing side-channels. All benchmarks except for those in category (1) also come with a repaired version that does not exhibit the original vulnerability.

Before running THEMIS, we need to specify the entry points of each application. Since most applications come with test cases, we use these test harnesses as entry points. For those applications for which we do not have suitable drivers, we manually construct a harness and specify it as the entry point.

Main results. The table in Figure 9 shows the accuracy and running time of THEMIS on these benchmarks. Using a value of $\epsilon  = {64}$ , THEMIS successfully finds vulnerabilities in the original vulnerable versions of these frameworks and is able to verify that the original vulnerability is no longer present in the repaired versions. The running time of THEMIS is also quite reasonable, taking an average 8.81 seconds to analyze each benchmark.

Benefit of taint analysis. Recall from Sections 1 and 5 that THEMIS performs taint analysis to identify hot spots, which overapproxi-mate program fragments that may contain a side-channel vulnerability. The QCHL verifier only analyzes such hot spots rather than the entire program. To demonstrate the usefulness of taint analysis, we compare the lines of code (in Soot IR) in the original application (reported in the LOC column) with the lines of code (also in Soot IR) with those analyzed by the QCHL verifier (reported in the LOC' column). As we can see from Figure 9, taint analysis significantly prunes security-irrelevant parts of the application in terms of lines of codes. This pruning effect can also be observed using other statistics. For example, the number of reachable methods ranges from 15 to 1487, with an average of 479 , before taint analysius, whereas the number of reachable methods after taint analysis ranges from 6 to 35, with an average of 15 , after taint analysis. Thus, pruning using taint information makes the job of the QCHL verifier significantly easier.

<table><tr><td>Benchmark</td><td>LOC</td><td>Category</td><td>#Reports</td><td>Time (s)</td></tr><tr><td>Jetty</td><td>2619</td><td>Server</td><td>4</td><td>10.17</td></tr><tr><td>Tomcat</td><td>12173</td><td>Server</td><td>1</td><td>5.86</td></tr><tr><td>OpenMRS</td><td>10721</td><td>Healthcare</td><td>1</td><td>9.71</td></tr><tr><td>OACC</td><td>78</td><td>Authentication</td><td>1</td><td>1.83</td></tr><tr><td>Apache Shiro</td><td>4043</td><td>Authentication</td><td>0</td><td>6.54</td></tr><tr><td>Apache Crypto</td><td>4505</td><td>Crypto</td><td>0</td><td>4.33</td></tr><tr><td>bc-java</td><td>5759</td><td>Crypto</td><td>0</td><td>6.89</td></tr></table>

Figure 10: Evaluation THEMIS on identifying zero-day vulnerabilities from popular Java applications

Benefit of $\epsilon$ . To justify the need for our relaxed notion of noninterference, Figure 9 also shows the results of the same experiment using an $\epsilon$ value of 0 . Hence, the $\epsilon  = 0$ column from Figure 9 corresponds to the standard notion of non-interference. As we can see from the table, THEMIS reports several false positives using an $\epsilon$ value of 0 . In particular, the repaired versions of some programs still exhibit a minor resource usage imbalance but this difference is practically infeasible to exploit, so the developers consider these versions to be side-channel-free. However, these programs are deemed unsafe using standard non-interference. We believe this comparison shows that our relaxed policy of $\epsilon$ -bounded non-interference is useful in practice and allows security analysts to understand the severity of the side channel.

Benefit of relational analysis. To investigate the benefit of relational invariants, we analyze the safe versions of the 20 benchmarks from Figures 8 and 9 with relational invariant generation disabled. In this case, THEMIS can only verify the safety of 10 of the benchmarks.

Although this number can potentially be increased by using a more sophisticated non-relational loop invariant generation algorithm, THEMIS circumvents this need, instead using simple relational in- variants that are conjunctions of simple equality constraints. This experiment corroborates the hypothesis that QCHL makes verification easier by requiring simpler loop invariants compared to other techniques like self-composition.

### 6.3 Discovery of Zero-Day Vulnerabilities

To evaluate whether THEMIS can discover unknown vulnerabilities in real world Java applications, we conduct an experiment on seven popular Java frameworks. Our data set covers a wide range of Java applications from different domains such as HTTP servers,

---

public boolean check (Object credentials)

\{

	if (credentials instanceof char[])

		credentials = new String((char[]) credentials);

	if (!(credentials instanceof String) && !(credentials

				instanceof Password))

		LOG.warn("Can't check " + credentials.getClass(   ) + "

					against CRYPT");

	String passwd = credentials.toString(   );

	// FIX: return stringEquals(_cooked, UnixCrypt.crypt(   )

				passwd,_cooked));

	return _cooked.equals(UnixCrypt.crypt(passwd,_cooked));

\}

/**

	* <p>Utility method that replaces String.equals(   ) to

			avoid timing attacks.</p>

	*/

static boolean stringEquals(String s1, String s2)

\{

	boolean result = true;

	int l1 = s1.length(   );

	int l2 = s2.length(   );

	if(ll != 12) result = false;

	int n = (11 < 12) ? 11 : 12;

	for ( int i = 0; i < n; i++ )

		result &= s1.charAt(i) == s2.charAt(i);

	return result;

\}

---

Figure 11: Eclipse Jetty code snippet that contains a timing side channel. Line 10 is the original buggy code. This vulnerability can be fixed by implementing stringEquals (lines 14 - 26) and calling it instead of the built-in String.equals method.

health care platforms, authentication frameworks, etc. For example, Eclipse Jetty is a well-known web server that is embedded in products such as Apache Spark, Google App Engine, and Twitter's Streaming API. OpenMRS is the world's leading open source enterprise electronic medical record system platform; OACC is a well-known Java application security framework, and bc-java is an implementation of the Bounty Castle crypto API in Java.

As in our previous experiment, we first manually annotate each application to indicate the sources of confidential information. We then use THEMIS to find timing side channels in these applications using an $\epsilon$ value of 10 . The results of this experiment are summarized in Figure 10. As we can see from this figure, THEMIS reports a total of seven vulnerabilities in four of the analyzed applications. We manually inspected each report and confirmed that the detected vulnerabilities are indeed true positives. We also reported the vulnerabilities detected by THEMIs to the developers, and the majority of these vulnerabilities were confirmed and fixed by the developers in less than 24 hours. However, the vulnerability that we reported for OpenMRS was rejected by the developers. The reason for this false positive is that the leaked password is actually hashed and salted in the database, but, because the logic for hashing and salting is not part of the Java implementation, THEMIS was not able to reason about this aspect.

To give the reader some intuition about the kinds of side channels detected by THEMIS, Figure 11 shows a security vulnerability from the Eclipse Jetty web server. The check procedure from Figure 11 checks whether the password provided by the user matches the expected password (_cooked). The original code performs this check by calling the built-in equality method provided by the java . lang. String library. Since the built-in equality method returns false as soon as it finds a mismatch between two characters, line 10 in the check method introduces a timing side-channel vulnerability.

The developers have fixed the vulnerability [1] in this code snippet by replacing line 10 with the (commented out) code shown in line 9. In particular, the fix involves calling the safe version of equals, called stringEquals, which checks for equality between all characters in the strings. This repaired version of the check method no longer contains a vulnerability for any $\epsilon  > 1$ , and THEMIS can verify that the check procedure is now safe.

## 7 LIMITATIONS

Like any other program analysis tool, THEMIS has a number of limitations. First, due to the fundamental undecidability of the underlying static analysis problem, THEMIS is incomplete and may report false positives (e.g., due to imprecision in pointer analysis or loop invariant generation). For example, our method for inferring relational invariants is based on monomial predicate abstraction using a fixed set of pre-defined templates, and we restrict our templates to equalities between variables. In addition, our non-relational invariant generator is based on traditional abstract interpretation, which does not distinguish array elements precisely.

Second, dynamic features of the Java language, such as reflective calls, dynamic class loading, and exceptional handling, pose challenges for THEMIS. Our current implementation can handle some cases of reflection (e.g., reflective calls with string constants), but reflection can, in general, cause THEMIs to have false negatives. We plan to mitigate this issue by integrating the Tamiflex tool [16] for reasoning about reflection into the THEMIS tool chain.

Finally, THEMIS unconditionally trusts all human inputs into the system, which may result in false negatives if the user inputs are not accurate. Said user inputs include application entry points, taint sources, cost instrumentations, and models of library methods.

## 8 RELATED WORK

In this section, we survey related work from the security and program analysis communities and explain how THEMIS differs from prior techniques.

Side channel attacks. Side-channel attacks related to resource usage have been known for decades. Specifically, side channels have been used to leak private cryptographic keys [3, 19, 46], infer user accounts [17], steal cellphone and credit card numbers [32], obtain web browsing history [26], and recover the plaintext of encrypted TLS traffic [5]. Chen et al. presents a comprehensive study of side-channel leaks in web applications [22].

Verification for non-interference. As mentioned in Section 3, we can prove that a program is free of side channel leaks by proving that it obeys a certain kind of non-interference property. There has been a significant body of work on proving non-interference. The simplest and most well-known technique for proving non-interference (and, in general, any 2-safety property) is self-composition [14]. The general idea underlying self-composition is as follows: Given a program $P$ and 2-safety property $\phi$ , we create a new program ${P}^{\prime }$ which sequentially composes two $\alpha$ -renamed copies of $P$ and then asserts that $\phi$ holds. Effectively, self-composition reduces verification of 2-safety to standard safety. While this self-composition technique is sound and relatively complete, successfully verifying the new program often requires the safety checker to come up with intricate invariants that are difficult to infer automatically [66]. Dufay et al. try to solve this problem by providing those invariants through JML annotations [25]; however, the resulting approach requires significant manual effort on the part of the developer or security analyst.

Another popular approach for proving $k$ -safety is to construct so-called product programs $\left\lbrack  {{12},{13},{71}}\right\rbrack$ . Similar to self-composition, the product program method also reduces $k$ -safety to standard safety by constructing a new program containing an assertion. While there are several different methods for constructing the product program, the central idea -shared in this work- is to execute the different copies of the program in lock step whenever possible. One disadvantage of this approach is that it can cause a blow-up in program size. As shown in the work of Sousa and Dillig [65], the product program approach can therefore suffer from scalability problems.

The approach advocated in this paper is most closely related to relational program logic, such as Cartesian Hoare Logic [65] and Relational Hoare Logic [15]. Specifically, the QCHL program logic introduced in Section 4 builds on top of CHL by instantiating it in the $\epsilon$ -bounded non-interference setting and augmenting it with additional rules for tracking resource usage and utilizing taint information. One advantage of this approach over explicit product construction is that we decompose the proof into smaller lemmas by constructing small product programs on-the-fly rather than constructing a monolithic program that is subsequently checked by an off-the-shelf verifier.

The approach described in this paper also shares similarities with the work of Terauchi and Aiken, in which they extend self-composition with type-directed translation [52, 66]. In particular, this technique uses a type system for secure information flow to guide product construction. Specifically, similar to our use of taint information to determine when two loops can be synchronized, Ter-auchi and Aiken use type information to construct a better product program than standard self-composition. Our verification technique differs from this approach in two major ways: First, our algorithm is not guided purely by taint information and uses other forms of semantic information (e.g., relational loop invariants) to determine when two loops can be executed in lock step. Second, similar to other approaches for product construction, the type-directed translation method generates a new program that is subsequently verified by an off-the-shelf verifier. In contrast, our method decomposes the proof into smaller lemmas by constructing mini-products on-the-fly, as needed.

Almeida et al. implement a tool named ct-verif based on aforementioned techniques (involving both product programs and self-composition) [6]. In particular, ct-verif is designed for verifying the constant-time policy, which roughly corresponds to our notion of 0 -bounded non-interference instantiated with a timing cost model. In addition to using different techniques based on QCHL and taint analysis, THEMIS provides support for verifying a more general property, namely $\epsilon$ -bounded non-interference for any value of $\epsilon$ .

An alternative approach for verifying $k$ -safety is the decomposition method used in BLAZER [8]: This method decomposes execution traces into different partitions using taint information and then verifies $k$ -safety of the whole program by proving a standard safety property of each partition. One possible disadvantage of this approach is that, unlike our method and product construction techniques, BLAZER does not directly reason about the relationship between a pair of program executions. As illustrated through some of the examples in Section 4, such relational reasoning can greatly simplify the verification task in many cases. Furthermore, as we demonstrate in Section 6, THEMIS can verify benchmarks that cannot be proven by BLAZER within a 10-minute time limit.

In their recent work, Ngo et al. propose a language-based system for verifying and synthesizing synthesizes programs with constant-resource usage, meaning that every execution path of the program consumes the same amount of resource [53]. This technique uses a novel type system to reason both locally and globally about the resource usage bounds of a given program. Similar to work for verifying constant-time policy, this technique also does not allow proving $\epsilon$ -bounded non-interference for arbitrary values of $\epsilon$ . Furthermore, as a type-based solution for a functional language, this technique puts heavier annotation burden on the developer and is not immediately applicable to standard imperative languages like Java or C.

Secure information flow. There has been a significant body of work on language-based solutions for enforcing information flow properties [51, 56, 69, 72]. For instance, Zhang et al. [72] propose a language-based approach that tracks side-channel leakage, and Pottier et al. [56] design a type-based information flow analysis inside an ML-style language. THEMIS differs from these language-based solutions in that it requires minimal annotation effort and works on existing Java programs.

One of the most popular tools for tracking information flow in existing Java applications is FlowDroid [9], and TheMIS builds on top of FlowDroid to identify secret-tainted variables. FlowTracker [59] is another information flow analysis for $\mathrm{C}/\mathrm{C} +  +$ featuring efficient representation of implicit flow. We believe these techniques are complimentary to our approach, and a tool like THEMIs can directly benefit from advances in such static taint tracking tools.

There have also been attempts at verifying the constant-time policy directly using information-flow checking [11]. However, this approach is flow-insensitive (and therefore imprecise) and imposes a number of restrictions on the input program.

Automatic resource bound computation. There has been a flurry of research on statically computing upper bounds for the resource usage of imperative programs. Existing techniques for this purpose leverage abstract interpretation [37], size-change abstraction [74], lossy vector addition systems [63], linear programming [20], difference constraints [64], recurrence relations [4, 7, 29], and term rewriting [18]. Another line of research, called AARA [39-43], performs bound analysis on functional languages.

Our approach differs from these approaches in that we perform relational reasoning about resource usage. That is, rather than computing an upper bound on the resource usage of the program, we use QCHL to prove an upper bound on the difference between the resource usage of two program runs. Similar to our QCHL, recent work by Çiçek et al. performs relational cost analysis to reason about the difference in resource usage of a pair of programs [21]. Their work shares with us the insight that relational analysis may be simplified by exploiting the structural similarity between the inputs as well as the program codes. However, their non-relational reasoning relies on range analysis while THEMIS relies on Hoare-style weakest precondition computation; as a result THEMIs is more precise. Also, THEMIS analyzes real-world Java programs, while [21] is built on top of a hypothetical higher-order functional language.

Other defenses against side channels. In this paper, we consider a purely static approach for detecting resource side channels. However, there are other possible ways of detecting vulnerabilities and preventing against side channel attacks. For instance, Bang et al. use symbolic execution and model counting to quantify leakage for a particular type of side channel [10]. Pasareanu et al. have recently implemented a symbolic execution based algorithm for generating inputs that maximize side channel measurements (namely timing and memory usage) [54]. Sidebuster [73] uses a hybrid static/dynamic analysis to detect side-channels based on irregularities in the One key advantage of our approach compared to these other techniques is that it can be used to verify the absence of side-channel vulnerabilities in programs.

There has also been a line of research that focuses for defending against side channels using runtime systems [49], compilers [50, 57, 58], or secure hardware [48]. Unlike these techniques, our approach does not result in runtime overhead.

## 9 CONCLUSIONS

We have proposed a new security policy called $\epsilon$ -bounded noninterference that can be used to verify the absence of resource side channels in programs. We have also proposed an automated verification algorithm, implemented in a tool called THEMIS, for proving this property. Our approach verifies the absence of side channels by combining lightweight static taint analysis with precise relational verification using a new program logic called QCHL.

We have evaluated our tool, THEMIS, in a number of ways and have shown that (a) it can find previously unknown vulnerabilities in widely used Java programs, (b) it can verify that the repaired versions of vulnerable programs do not exhibit the original vulnerability, and (c) it compares favorably against BLAZER, a state-of-the-art tool for finding timing side channels in Java programs.

There are a number of directions for future work. First, our current implementation only provides cost models for timing and response size; however, we would like to broaden the applicability of THEMIS by providing cost models for other kinds of resources. Second, our current implementation can verify bounded non-interference for a given $\epsilon$ , but we would also like to automatically infer the smallest $\epsilon$ for which the program is safe. While this extension can be easily done by solving an optimization rather than a satisfiability problem, our current implementation does not provide this capability due to limitations in Z3's OCaml API.

## ACKNOWLEDGEMENT

We would like to thank the anonymous reviewers for their helpful feedback. We also thank Calvin Lin for his insightful comments, Marcelo Sousa for setting up the DESCARTES [65] tool, and the Jetty developer team for their responsiveness as well as the assistance they provide.

This material is based on research sponsored by DARPA award FA8750-15-2-0096 as well as NSF Award CCF-1712067. The U.S. Government is authorized to reproduce and distribute reprints for Governmental purposes notwithstanding any copyright notation thereon. The views and conclusions contained herein are those of the authors and should not be interpreted as necessarily representing the official policies or endorsements, either expressed or implied, of the U.S. Government.

## REFERENCES

[1] 2017. A timing channel in Jetty. https://github.com/eclipse/jetty.project/commit/ 2baa1abe4b1c380a30deacca1ed367466a1a62ea. (2017).

[2] Onur Aciiçmez, Cetin Kaya Koç, and Jean-Pierre Seifert. 2007. On the Power of Simple Branch Prediction Analysis. In Proceedings of the 2Nd ACM Symposium on Information, Computer and Communications Security (ASIACCS '07). ACM, 312-320.

[3] Onur Aciiçmez and Werner Schindler. 2008. A Vulnerability in RSA Implementations Due to Instruction Cache Analysis and Its Demonstration on OpenSSL. In Proceedings of the 2008 The Cryptopgraphers' Track at the RSA Conference on Topics in Cryptology (CT-RSA'08). Springer-Verlag, 256-273.

[4] Elvira Albert, Jesús Correas Fernández, and Guillermo Román-Díez. 2015. Noncumulative Resource Analysis. In Proceedings of the 21st International Conference on Tools and Algorithms for the Construction and Analysis of Systems - Volume 9035. Springer-Verlag New York, Inc., 85-100.

[5] Nadhem J. AlFardan and Kenneth G. Paterson. 2013. Lucky Thirteen: Breaking the TLS and DTLS Record Protocols. In 2013 IEEE Symposium on Security and Privacy, SP 2013, Berkeley, CA, USA, May 19-22, 2013. 526-540.

[6] José Bacelar Almeida, Manuel Barbosa, Gilles Barthe, François Dupressoir, and Michael Emmi. 2016. Verifying Constant-Time Implementations. In 25th USENIX Security Symposium, USENIX Security 16, Austin, TX, USA, August 10-12, 2016. 53-70.

[7] Diego Esteban Alonso-Blas and Samir Genaim. 2012. On the Limits of the Classical Approach to Cost Analysis. In Proceedings of the 19th International Conference on Static Analysis (SAS'12). Springer-Verlag, 405-421.

[8] Timos Antonopoulos, Paul Gazzillo, Michael Hicks, Eric Koskinen, Tachio Ter-auchi, and Shiyi Wei. 2017. Decomposition Instead of Self-Composition for k-Safety. In Proceedings of the ACM Conference on Programming Language Design and Implementation (PLDI).

[9] Steven Arzt, Siegfried Rasthofer, Christian Fritz, Eric Bodden, Alexandre Bartel, Jacques Klein, Yves Le Traon, Damien Octeau, and Patrick D. McDaniel. 2014. FlowDroid: precise context, flow, field, object-sensitive and lifecycle-aware taint analysis for Android apps. In ACM SIGPLAN Conference on Programming Language Design and Implementation, PLDI '14, Edinburgh, United Kingdom - June 09 - 11, 2014. 259-269.

[10] Lucas Bang, Abdulbaki Aydin, Quoc-Sang Phan, Corina S. Păsăreanu, and Tevfik Bultan. 2016. String Analysis for Side Channels with Segmented Oracles. In Proceedings of the 2016 24th ACM SIGSOFT International Symposium on Foundations of Software Engineering (FSE 2016). ACM, 193-204.

[11] Gilles Barthe, Gustavo Betarte, Juan Campo, Carlos Luna, and David Pichardie. 2014. System-level Non-interference for Constant-time Cryptography. In Proceedings of the 2014 ACM SIGSAC Conference on Computer and Communications Security (CCS '14). ACM, 1267-1279.

[12] Gilles Barthe, Juan Manuel Crespo, and César Kunz. 2011. Relational Verification Using Product Programs. In FM 2011: Formal Methods - 17th International Symposium on Formal Methods, Limerick, Ireland, June 20-24, 2011. Proceedings. 200-214.

[13] Gilles Barthe, Juan Manuel Crespo, and César Kunz. 2013. Beyond 2-Safety: Asymmetric Product Programs for Relational Program Verification. In Logical Foundations of Computer Science, International Symposium, LFCS 2013, San Diego, CA, USA, January 6-8, 2013. Proceedings, Sergei N. Artëmov and Anil Nerode (Eds.), Vol. 7734. Springer, 29-43.

[14] Gilles Barthe, Pedro R D'Argenio, and Tamara Rezk. 2004. Secure information flow by self-composition. In Computer Security Foundations Workshop, 2004. Proceedings. 17th IEEE. IEEE, 100-114.

[15] Nick Benton. 2004. Simple relational correctness proofs for static analyses and program transformations. In ACM SIGPLAN Notices, Vol. 39. ACM, 14-25.

[16] Eric Bodden, Andreas Sewe, Jan Sinschek, Hela Oueslati, and Mira Mezini. 2011. Taming reflection: Aiding static analysis in the presence of reflection and custom class loaders. In Proceedings of the 33rd International Conference on Software Engineering, ICSE 2011, Waikiki, Honolulu, HI, USA, May 21-28, 2011. 241-250.

[17] Andrew Bortz and Dan Boneh. 2007. Exposing Private Information by Timing Web Applications. In World Wide Web. ACM, 621-628.

[18] Marc Brockschmidt, Fabian Emmes, Stephan Falke, Carsten Fuhs, and Jürgen Giesl. 2016. Analyzing Runtime and Size Complexity of Integer Programs. ACM Trans. Program. Lang. Syst. 38, 4, Article 13 (Aug. 2016), 50 pages.

[19] David Brumley and Dan Boneh. 2003. Remote Timing Attacks Are Practical. In Proceedings of the 12th USENIX Security Symposium, Washington, D.C., USA, August 4-8, 2003.

[20] Quentin Carbonneaux, Jan Hoffmann, and Zhong Shao. 2015. Compositional Certified Resource Bounds. In Proceedings of the 36th ACM SIGPLAN Conference on Programming Language Design and Implementation (PLDI '15). ACM, 467-478.

[21] Ezgi Çiçek, Gilles Barthe, Marco Gaboardi, Deepak Garg, and Jan Hoffmann. 2017. Relational Cost Analysis. In Proceedings of the 44th ACM SIGPLAN Symposium on Principles of Programming Languages (POPL 2017). ACM, 316-329.

[22] Shuo Chen, Rui Wang, XiaoFeng Wang, and Kehuan Zhang. 2010. Side-Channel Leaks in Web Applications: A Reality Today, a Challenge Tomorrow. In 31st IEEE Symposium on Security and Privacy, S&P 2010, 16-19 May 2010, Berleley/Oakland, California, USA. 191-206.

[23] Sujit Rokka Chhetri and Mohammad Abdullah Al Faruque. 2017. Side-Channels of Cyber-Physical Systems: Case Study in Additive Manufacturing. IEEE Design & Test (2017).

[24] Leonardo De Moura and Nikolaj Bjørner. 2008. Z3: An efficient SMT solver. In Tools and Algorithms for the Construction and Analysis of Systems. Springer, 337-340.

[25] Guillaume Dufay, Amy Felty, and Stan Matwin. 2005. Privacy-sensitive Information Flow with JML. In Proceedings of the 20th International Conference on Automated Deduction (CADE' 20). Springer-Verlag, 116-130.

[26] Edward W. Felten and Michael A. Schneider. 2000. Timing attacks on Web privacy. In CCS 2000, Proceedings of the 7th ACM Conference on Computer and Communications Security, Athens, Greece, November 1-4, 2000. 25-32.

[27] Cormac Flanagan and K. Rustan M. Leino. 2001. Houdini, an Annotation Assistant for ESC/Java. In Proceedings of the International Symposium of Formal Methods Europe on Formal Methods for Increasing Software Productivity (FME '01). Springer-Verlag, 500-517.

[28] Cormac Flanagan, K. Rustan M. Leino, Mark Lillibridge, Greg Nelson, James B. Saxe, and Raymie Stata. 2002. Extended Static Checking for Java. In Proceedings of the ACM SIGPLAN 2002 Conference on Programming Language Design and Implementation (PLDI '02). ACM, New York, NY, USA, 234-245.

[29] Antonio Flores-Montoya and Reiner Hähnle. 2014. Resource Analysis of Complex Programs with Cost Equations. Springer International Publishing, Cham, 275-295.

[30] Riccardo Focardi and Roberto Gorrieri. 1995. A Classification of Security Properties for Process Algebras1. Journal of Computer security 3, 1 (1995), 5-33.

[31] Karine Gandolfi, Christophe Mourtel, and Francis Olivier. 2001. Electromagnetic Analysis: Concrete Results. In Proceedings of the Third International Workshop on Cryptographic Hardware and Embedded Systems (CHES '01). Springer-Verlag, 251-261.

[32] Nethanel Gelernter and Amir Herzberg. 2015. Cross-Site Search Attacks. In Proceedings of the 22nd ACM SIGSAC Conference on Computer and Communications Security. ACM, 1394-1405.

[33] Daniel Genkin, Itamar Pipman, and Eran Tromer. 2014. Get Your Hands Off My Laptop: Physical Side-Channel Key-Extraction Attacks on PCs. In Cryptographic Hardware and Embedded Systems - CHES 2014 - 16th International Workshop, Busan, South Korea, September 23-26, 2014. Proceedings. 242-260.

[34] Jan Goguen and Meseguer Jose. 1982. Security policies and security models. In Symposium on Security and Privacy. IEEE Computer Society Press, 11-20.

[35] James W Gray III. 1992. Toward a mathematical foundation for information flow security. Journal of Computer Security 1, 3-4 (1992), 255-294.

[36] David Gullasch, Endre Bangerter, and Stephan Krenn. 2011. Cache Games - Bringing Access-Based Cache Attacks on AES to Practice. In 32nd IEEE Symposium on Security and Privacy, S&P 2011, 22-25 May 2011, Berkeley, California, USA. 490- 505.

[37] Sumit Gulwani, Krishna K. Mehra, and Trishul Chilimbi. 2009. SPEED: Precise and Efficient Static Estimation of Program Computational Complexity. In Proceedings of the 36th Annual ACM SIGPLAN-SIGACT Symposium on Principles of Programming Languages (POPL '09). ACM, 127-139.

[38] Andreas Haeberlen, Benjamin C. Pierce, and Arjun Narayan. 2011. Differential Privacy Under Fire. In 20th USENIX Security Symposium, San Francisco, CA, USA, August 8-12, 2011, Proceedings.

[39] Jan Hoffmann, Klaus Aehlig, and Martin Hofmann. 2012. Multivariate Amortized Resource Analysis. ACM Trans. Program. Lang. Syst. 34, 3, Article 14 (Nov. 2012), 62 pages.

[40] Jan Hoffmann, Ankush Das, and Shu-Chun Weng. 2017. Towards Automatic Resource Bound Analysis for OCaml. In Proceedings of the 44th ACM SIGPLAN Symposium on Principles of Programming Languages (POPL 2017). ACM, 359-373.

[41] Jan Hoffmann and Martin Hofmann. 2010. Amortized Resource Analysis with Polynomial Potential: A Static Inference of Polynomial Bounds for Functional Programs. In Proceedings of the 19th European Conference on Programming Languages and Systems (ESOP'10). Springer-Verlag, 287-306.

[42] Jan Hoffmann and Zhong Shao. 2014. Type-Based Amortized Resource Analysis with Integers and Arrays. Springer International Publishing, 152-168.

[43] Martin Hofmann and Steffen Jost. 2003. Static Prediction of Heap Space Usage for First-order Functional Programs. In Proceedings of the 30th ACM SIGPLAN-SIGACT Symposium on Principles of Programming Languages (POPL '03). ACM, 185-197.

[44] Bertrand Jeannet and Antoine Miné. 2009. Apron: A Library of Numerical Abstract Domains for Static Analysis. In Proceedings of the 21st International Conference on Computer Aided Verification (CAV '09). Springer-Verlag, 661-667.

[45] Paul Kocher. 1996. Timing attacks on implementations of Diffie-Hellman, RSA, DSS, and other systems. In Advances in Cryptologyâ $\widetilde{A}\widetilde{T}{CRYPTO}\widehat{a}\widetilde{A}\widetilde{Z}{96}$ . Springer, ${104} - {113}$ .

[46] Paul C. Kocher. 1996. Timing Attacks on Implementations of Diffie-Hellman, RSA, DSS, and Other Systems. In CRYPTO '96, 16th Annual International Cryptology Conference, Santa Barbara, California, USA, August 18-22, 1996, Proceedings. 104- 113.

[47] Shuvendu K. Lahiri and Shaz Qadeer. 2009. Complexity and Algorithms for Monomial and Clausal Predicate Abstraction. In Proceedings of the 22Nd International Conference on Automated Deduction (CADE-22). Springer-Verlag, 214-229.

[48] Chang Liu, Austin Harris, Martin Maas, Michael Hicks, Mohit Tiwari, and Elaine Shi. 2015. GhostRider: A Hardware-Software System for Memory Trace Oblivious Computation. In Proceedings of the Twentieth International Conference on Architectural Support for Programming Languages and Operating Systems (ASPLOS '15). ACM, 87-101.

[49] Robert Martin, John Demme, and Simha Sethumadhavan. 2012. TimeWarp: Rethinking Timekeeping and Performance Monitoring Mechanisms to Mitigate Side-channel Attacks. In Proceedings of the 39th Annual International Symposium on Computer Architecture (ISCA '12). IEEE Computer Society, 118-129.

[50] David Molnar, Matt Piotrowski, David Schultz, and David Wagner. 2006. The Program Counter Security Model: Automatic Detection and Removal of Control-flow Side Channel Attacks. In Proceedings of the 8th International Conference on Information Security and Cryptology (ICISC'05). Springer-Verlag, 156-168.

[51] Andrew C Myers, Lantian Zheng, Steve Zdancewic, Stephen Chong, and Nathaniel Nystrom. 2001. Jif: Java information flow. Software release. Located at http://www.cs.cornell.edu/jif 2005 (2001).

[52] David A. Naumann. 2006. From Coupling Relations to Mated Invariants for Checking Information Flow. Springer Berlin Heidelberg, 279-296.

[53] Van Chan Ngo, Mario Dehesa-Azuara, Matthew Fredrikson, and Jan Hoffmann. 2017. Verifying and Synthesizing Constant-Resource Implementations with Types. In 38st IEEE Symposium on Security and Privacy, S&P.

[54] Corina Pasareanu, Quoc-Sang Phan, and Pasquale Malacaria. 2016. Multi-Run Side-Channel Analysis Using Symbolic Execution and Max-SMT. In Computer Security Foundations Symposium. IEEE.

[55] Corina S. Pasareanu, Quoc-Sang Phan, and Pasquale Malacaria. 2016. Multi-run Side-Channel Analysis Using Symbolic Execution and Max-SMT. In IEEE 29th Computer Security Foundations Symposium, CSF 2016, Lisbon, Portugal, June 27 - July 1, 2016. 387-400.

[56] François Pottier and Vincent Simonet. 2003. Information Flow Inference for ML. ACM Trans. Program. Lang. Syst. 25, 1 (Jan. 2003), 117-158.

[57] Ashay Rane, Calvin Lin, and Mohit Tiwari. 2015. Raccoon: Closing Digital Side-Channels through Obfuscated Execution. In 24th USENIX Security Symposium (USENIX Security 15). USENIX Association, Washington, D.C., 431-446.

[58] Ashay Rane, Calvin Lin, and Mohit Tiwari. 2016. Secure, Precise, and Fast Floating-Point Operations on x86 Processors. In 25th USENIX Security Symposium (USENIX Security 16). USENIX Association, Austin, TX, 71-86.

[59] Bruno Rodrigues, Fernando Magno Quintão Pereira, and Diego F. Aranha. 2016. Sparse Representation of Implicit Flows with Applications to Side-channel Detection. In Proceedings of the 25th International Conference on Compiler Construction (CC 2016). ACM, 110-120.

[60] Andrei Sabelfeld and Andrew C Myers. 2003. Language-based information-flow security. IEEE Journal on selected areas in communications 21,1 (2003), 5-19.

[61] Rahul Sharma, Saurabh Gupta, Bharath Hariharan, Alex Aiken, Percy Liang, and Aditya V. Nori. 2013. A Data Driven Approach for Algebraic Loop Invariants. In Proceedings of the 22Nd European Conference on Programming Languages and Systems (ESOP'13). Springer-Verlag, 574-592.

[62] Reza Shokri and Vitaly Shmatikov. 2015. Privacy-Preserving Deep Learning. In Proceedings of the 22nd ACM SIGSAC Conference on Computer and Communications Security, Denver, CO, USA, October 12-6, 2015. 1310-1321.

[63] Moritz Sinn, Florian Zuleger, and Helmut Veith. 2014. A Simple and Scalable Static Analysis for Bound Analysis and Amortized Complexity Analysis. Springer International Publishing, 745-761.

[64] Moritz Sinn, Florian Zuleger, and Helmut Veith. 2017. Complexity and Resource Bound Analysis of Imperative Programs Using Difference Constraints. Journal of Automated Reasoning (2017), 1-43.

[65] Marcelo Sousa and Isil Dillig. 2016. Cartesian Hoare Logic for Verifying K-safety Properties. In Proceedings of the 37th ACM SIGPLAN Conference on Programming Language Design and Implementation (PLDI '16). ACM, New York, NY, USA, 57-69.

[66] Tachio Terauchi and Alexander Aiken. 2005. Secure Information Flow as a Safety Problem. In Static Analysis, 12th International Symposium, SAS 2005, London, UK, September 7-9, 2005, Proceedings. 352-367.

[67] Raja Vallée-Rai, Phong Co, Etienne Gagnon, Laurie Hendren, Patrick Lam, and Vijay Sundaresan. 1999. Soot - a Java Bytecode Optimization Framework. In Proceedings of the 1999 Conference of the Centre for Advanced Studies on Collaborative Research (CASCON '99). IBM Press, 13-.

[68] Serge Vaudenay. 2002. Security Flaws Induced by CBC Padding - Applications to SSL, IPSEC, WTLS ... In Advances in Cryptology - EUROCRYPT 2002, International Conference on the Theory and Applications of Cryptographic Techniques, Amsterdam, The Netherlands, April 28 - May 2, 2002, Proceedings. 534-546.

[69] Jean Yang, Kuat Yessenov, and Armando Solar-Lezama. 2012. A language for automatically enforcing privacy policies. In ACM SIGPLAN Notices, Vol. 47. ACM, 85-96.

[70] Yuval Yarom, Daniel Genkin, and Nadia Heninger. 2016. CacheBleed: A Timing Attack on OpenSSL Constant Time RSA. Springer Berlin Heidelberg, 346-367.

[71] Anna Zaks and Amir Pnueli. 2008. CoVaC: Compiler Validation by Program Analysis of the Cross-Product. In Proceedings of the 15th International Symposium on Formal Methods (FM '08). Springer-Verlag, Berlin, Heidelberg, 35-51.

[72] Danfeng Zhang, Aslan Askarov, and Andrew C. Myers. 2012. Language-based Control and Mitigation of Timing Channels. In Proceedings of the 33rd ACM SIGPLAN Conference on Programming Language Design and Implementation (PLDI '12). ACM, 99-110.

[73] Kehuan Zhang, Zhou Li, Rui Wang, XiaoFeng Wang, and Shuo Chen. 2010. Side-buster: Automated Detection and Quantification of Side-channel Leaks in Web Application Development. In Computer and Communications Security. ACM, 595- 606.

[74] Florian Zuleger, Sumit Gulwani, Moritz Sinn, and Helmut Veith. 2011. Bound Analysis of Imperative Programs with the Size-change Abstraction. In Proceedings of the 18th International Conference on Static Analysis (SAS'11). Springer-Verlag, 280-297.

## APPENDIX A: PROOF OF SOUNDNESS

LEMMA .1. Let program $P = \lambda \overrightarrow{p}$ . S. If the following premises hold:

- $\overrightarrow{{p}_{1}} = \alpha \left( \overrightarrow{p}\right) ,\overrightarrow{{p}_{2}} = \alpha \left( \overrightarrow{p}\right)$

$\bullet   \vDash  I \rightarrow  {\overrightarrow{{p}_{1}}}^{l} = {\overrightarrow{{p}_{2}}}^{l} \land  {\overrightarrow{{p}_{1}}}^{h} \neq  {\overrightarrow{{p}_{2}}}^{h}$

- Σ is sound

- $\sum  \vdash$ CanSynchronize $\left( {{e}_{1},{e}_{2},{S}_{1},{S}_{2}, I}\right)$

Then $\vDash  I \rightarrow  \left( {{e}_{1} \leftrightarrow  {e}_{2}}\right)$ .

Proof. According to figure 6, if $\sum  \vdash$ CanSynchronize $\left( {{e}_{1},{e}_{2},{S}_{1},{S}_{2}, I}\right)$ , then at least one of the two conditions must be true:

$$
\text{-} \mid   = I \rightarrow  \left( {{e}_{1} \leftrightarrow  {e}_{2}}\right)
$$

- ${e}_{1}{ \equiv  }_{\alpha }{e}_{2} \land  {S}_{1}{ \equiv  }_{\alpha }{S}_{2} \land  \sum  \vdash  {e}_{1} : \mathbf{{low}} \land  \sum  \vdash  {e}_{2} : \mathbf{{low}}$

If the first condition is true, then the conclusion trivially holds Otherwise, since $\sum$ is sound, we know that ${e}_{1}$ and ${e}_{2}$ depend solely on ${\overrightarrow{{p}_{1}}}^{l}$ and ${\overrightarrow{{p}_{2}}}^{l}$ , respectively. According to the first two premises, $I \rightarrow  {\overrightarrow{{p}_{1}}}^{l} = {\overrightarrow{{p}_{2}}}^{l}$ . It follows that $I \rightarrow  {e}_{1} = {e}_{2}$ and therefore $\vDash  I \rightarrow$ $\left( {{e}_{1} \leftrightarrow  {e}_{2}}\right)$ .

LEMMA .2. Let $\operatorname{vars}\left( S\right)$ be the set of all free variables in $S$ . If $\operatorname{vars}\left( {S}_{1}\right)  \cap  \operatorname{vars}\left( {S}_{2}\right)  = \varnothing$ , then ${S}_{1};{S}_{2}$ is semantically equivalent to ${S}_{2};{S}_{1}$ .

Proof. Suppose $\Gamma  \vdash  {S}_{1};{S}_{2} : {\Gamma }^{\prime }, r$ . Since $\operatorname{vars}\left( {S}_{1}\right)$ and $\operatorname{vars}\left( {S}_{2}\right)$ are mutually disjoint, we could break $\Gamma$ into three partitions $\Gamma  =$ ${\Gamma }_{1} \sqcup  {\Gamma }_{2} \sqcup  {\Gamma }_{3}$ , where $\operatorname{dom}\left( {\Gamma }_{1}\right)  = \operatorname{vars}\left( {S}_{1}\right) ,\operatorname{dom}\left( {\Gamma }_{2}\right)  = \operatorname{vars}\left( {S}_{2}\right)$ and $\operatorname{dom}\left( {\Gamma }_{3}\right)  = \operatorname{dom}\left( \Gamma \right)  \smallsetminus  \operatorname{vars}\left( {S}_{1}\right)  \smallsetminus  \operatorname{vars}\left( {S}_{2}\right)$ . Since ${S}_{i}$ does not touch ${\Gamma }_{j}$ where $i \neq  j$ , we have

$$
{\Gamma }_{1} \vdash  {S}_{1} : {\Gamma }_{1}^{\prime },{r}_{1}\;{\Gamma }_{2} \vdash  {S}_{2} : {\Gamma }_{2}^{\prime },{r}_{2}
$$

It follows that

$$
\Gamma  \vdash  {S}_{1} : {\Gamma }_{1}^{\prime } \sqcup  {\Gamma }_{2} \sqcup  {\Gamma }_{3},{r}_{1}\;\Gamma  \vdash  {S}_{2} : {\Gamma }_{1} \sqcup  {\Gamma }_{2}^{\prime } \sqcup  {\Gamma }_{3},{r}_{2}
$$

$$
{\Gamma }_{1}^{\prime } \sqcup  {\Gamma }_{2} \sqcup  {\Gamma }_{3} \vdash  {S}_{2} : {\Gamma }_{1}^{\prime } \sqcup  {\Gamma }_{2}^{\prime } \sqcup  {\Gamma }_{3},{r}_{1} + {r}_{2}
$$

$$
{\Gamma }_{1} \sqcup  {\Gamma }_{2}^{\prime } \sqcup  {\Gamma }_{3} \vdash  {S}_{1} : {\Gamma }_{1}^{\prime } \sqcup  {\Gamma }_{2}^{\prime } \sqcup  {\Gamma }_{3},{r}_{2} + {r}_{1}
$$

Using the operational semantics rule for sequential composition shown in figure 4 , this means

$$
\Gamma  \vdash  {S}_{1};{S}_{2} : {\Gamma }_{1}^{\prime } \sqcup  {\Gamma }_{2}^{\prime } \sqcup  {\Gamma }_{3},{r}_{1} + {r}_{2}
$$

$$
\Gamma  \vdash  {S}_{2};{S}_{1} : {\Gamma }_{1}^{\prime } \sqcup  {\Gamma }_{2}^{\prime } \sqcup  {\Gamma }_{3},{r}_{2} + {r}_{1}
$$

As ${S}_{1};{S}_{2}$ and ${S}_{2};{S}_{1}$ both have the same effect on $\Gamma$ and consume the same amount of resource, they are semantically equivalent.

LEMMA .3. Let program $P = \lambda \overrightarrow{p}$ . S. Under the assumption that the following premises hold:

- $\overrightarrow{{p}_{1}} = \alpha \left( \overrightarrow{p}\right) ,\overrightarrow{{p}_{2}} = \alpha \left( \overrightarrow{p}\right)$

$\bullet   \vDash  \Phi  \rightarrow  {\overrightarrow{{p}_{1}}}^{l} = {\overrightarrow{{p}_{2}}}^{l} \land  {\overrightarrow{{p}_{1}}}^{h} \neq  {\overrightarrow{{p}_{2}}}^{h}$

- Σ is sound

If $\sum  \vdash  \langle \Phi \rangle {S}_{1} \circledast  {S}_{2}\langle \Psi \rangle$ , then $\vdash  \{ \Phi \} {S}_{1};{S}_{2}\{ \Psi \}$ .

Proof. By structural induction on proof rules shown in figure 5.

- Rule (1).

By inductive hypothesis, $\vdash  \{ \Phi \} {S}_{2};{S}_{1}\{ \Psi \}$ . Since ${S}_{1}$ and ${S}_{2}$ belongs to two different alpha-renamed copies of the program, we have $\operatorname{vars}\left( {S}_{1}\right)  \cap  \operatorname{vars}\left( {S}_{2}\right)  = \varnothing$ . Using lemma .2, we get $\vdash  \{ \Phi \} {S}_{1};{S}_{2}\{ \Psi \}$

- Rule (2).

By inductive hypothesis, $\vdash  \{ \Phi \} {S}_{1}$ ; skip; ${S}_{2}\{ \Psi \}$ . As ${S}_{1}$ ; skip is semantically equivalent to ${S}_{1}$ , we have $\vdash  \{ \Phi \} {S}_{1};{S}_{2}\{ \Psi \}$ .

- Rule (3).

By inductive hypothesis, $\vdash  \left\{  {\Phi }^{\prime }\right\}  {S}_{2};{S}_{3}\{ \Psi \}$ . Also, we know $\{ \Phi \} {S}_{1}\left\{  {\Phi }^{\prime }\right\}$ . Using the sequence rule in standard Hoare logic, we derive $\vdash  \{ \Phi \} {S}_{1};{S}_{2};{S}_{3}\{ \Psi \}$ .

- Rule (4).

By inductive hypothesis, $\vdash  \{ \Phi \} S\{ \Psi \}$ . As $S$ is semantically equivalent to $S;$ skip, we get $\{ \Phi \} S;\operatorname{skip}\{ \Psi \}$ .

- Rule (5).

By inductive hypothesis, $\vdash  \{ \Phi  \land  e\} {S}_{1};S;{S}_{3}\left\{  {\Psi }_{1}\right\}$ and $\vdash  \{ \Phi  \land$ $\neg e\} {S}_{2};S;{S}_{3}\left\{  {\Psi }_{2}\right\}$ . Since $\vDash  {\Psi }_{1} \rightarrow  {\Psi }_{1} \vee  {\Psi }_{2}$ and $\vDash  {\Psi }_{2} \rightarrow$ ${\Psi }_{1} \vee  {\Psi }_{2}$ , according to the consequence rule in standard Hoare logic we have $\{ \Phi  \land  e\} {S}_{1};S;{S}_{3}\left\{  {{\Psi }_{1} \vee  {\Psi }_{2}}\right\}$ and $\{ \Phi  \land$ $\neg e\} {S}_{2};S;{S}_{3}\left\{  {{\Psi }_{1} \vee  {\Psi }_{2}}\right\}$ . With the sequence rule in standard Hoare logic, assume

(1) $\vdash  \{ \Phi  \land  e\} {S}_{1}\left\{  {\Phi }_{1}\right\}$

(2) $\vdash  \left\{  {\Phi }_{1}\right\}  S;{S}_{3}\left\{  {{\Psi }_{1} \vee  {\Psi }_{2}}\right\}$

(3) $\vdash  \{ \Phi  \land  \neg e\} {S}_{2}\left\{  {\Phi }_{2}\right\}$

(4) $\vdash  \left\{  {\Phi }_{2}\right\}  S;{S}_{3}\left\{  {{\Psi }_{1} \vee  {\Psi }_{2}}\right\}$ .

Let ${\Phi }^{\prime } = {wp}\left( {{\Psi }_{1} \vee  {\Psi }_{2}}\right)$ . It follows immediately from (2) and (4) that ${\Phi }_{1} \rightarrow  {\Phi }^{\prime }$ and ${\Phi }_{2} \rightarrow  {\Phi }^{\prime }$ . We could apply the consequence rule again to (1) and (3) and derive $\vdash  \{ \Phi  \land  e\} {S}_{1}\left\{  {\Phi }^{\prime }\right\}$ and $\vdash  \{ \Phi  \land  \neg e\} {S}_{2}\left\{  {\Phi }^{\prime }\right\}$ . Using the condition rule in standard Hoare logic, we have $\{ \Phi \}$ if $e$ then ${S}_{1}$ else ${S}_{2}\left\{  {\Phi }^{\prime }\right\}$ . Combining (2), (4), sequence rule and the definition of ${wp}$ , we could finally derive $\vdash  \{ \Phi \}$ if $e$ then ${S}_{1}$ else ${S}_{2};S;{S}_{3}\left\{  {{\Psi }_{1} \vee  {\Psi }_{2}}\right\}$ .

- Rule (6).

By inductive hypothesis, $\vdash  \left\{  {\Psi }^{\prime }\right\}  S;{S}^{\prime }\{ \Psi \}$ . We also know that $\vdash  \{ \Phi \}$ while ${e}_{1}$ do ${S}_{1}\left\{  {\Phi }^{\prime }\right\}$ and $\vdash  \left\{  {\Phi }^{\prime }\right\}$ while ${e}_{2}$ do ${S}_{2}\left\{  {\Psi }^{\prime }\right\}$ . Applying the sequence rule in standard Hoare logic twice, we get $\vdash  \{ \Phi \}$ while ${e}_{1}$ do ${S}_{1}$ ; while ${e}_{2}$ do ${S}_{2};S;{S}^{\prime }\{ \Psi \}$ . Additionally, $S$ and while ${e}_{2}$ do ${S}_{2}$ comes from two different alpha-renamed copies so $\operatorname{vars}\left( S\right)  \cap  \operatorname{vars}\left( \right.$ while $\left. {{e}_{2}\text{do}{S}_{2}}\right)  =$ 0 . We could apply lemma . 2 and get

$\vdash  \{ \Phi \}$ while ${e}_{1}$ do ${S}_{1};S$ ; while ${e}_{2}$ do ${S}_{2};{S}^{\prime }\{ \Psi \}$

- Rule (7).

By inductive hypothesis, $\vdash  \left\{  {I \land  {e}_{1} \land  {e}_{2}}\right\}  {S}_{1};{S}_{2}\left\{  {I}^{\prime }\right\}$ and $\vdash$ $\left\{  {I\land \neg {e}_{1} \land  {e}_{2}}\right\}  S;{S}^{\prime }\{ \Psi \}$ . As $\vDash  {I}^{\prime } \rightarrow  I$ , we have $\vdash  \left\{  {I \land  {e}_{1} \land  }\right.$ $\left. {e}_{2}\right\}  {S}_{1};{S}_{2}\{ I\}$ due to consequence rule. Now we may apply the while rule in standard Hoare logic to obtain $\vdash  \{ I\}$ while ${e}_{1} \land$ ${e}_{2}$ do $\left( {{S}_{1};{S}_{2}}\right) \left\{  {I \land  \neg \left( {{e}_{1} \land  {e}_{2}}\right) }\right\}$ .

Now, as following two statements are semantically equivalent:

- while ${e}_{1} \land  {e}_{2}$ do $\left( {{S}_{1};{S}_{2}}\right)$

- while ${e}_{1} \land  {e}_{2}$ do $\left( {{S}_{1};{S}_{2}}\right)$ ; while ${e}_{1}$ do ${S}_{1}$ ; while ${e}_{2}$ do ${S}_{2}$ we could replace the former with the latter:

$\vdash  \{ I\}$ while ${e}_{1} \land  {e}_{2}$ do $\left( {{S}_{1};{S}_{2}}\right)$ ; while ${e}_{1}$ do ${S}_{1}$ ;

$$
\text{while}{e}_{2}\text{do}{S}_{2}\left\{  {I \land  \neg \left( {{e}_{1} \land  {e}_{2}}\right) }\right\}
$$

According to lemma .1, $\vDash  I \rightarrow  \left( {{e}_{1} \leftrightarrow  {e}_{2}}\right)$ . But we also know that the precondition $I \land  \neg \left( {{e}_{1} \land  {e}_{2}}\right)$ holds before the second loop while ${e}_{1}$ do ${S}_{1}$ . This implies $I\neg {e}_{1} \land  \neg {e}_{2}$ and therefore both of the two loops while ${e}_{1}$ do ${S}_{1}$ and while ${e}_{2}$ do ${S}_{2}$ would not execute, which means $\vdash  \left\{  {I\land \neg \left( {{e}_{1} \land  }\right. }\right.$ $\left. \left. {e}_{2}\right) \right\}$ while ${e}_{1}$ do ${S}_{1}$ ; while ${e}_{2}$ do ${S}_{2}\left\{  {I \land  \neg \left( {{e}_{1} \land  {e}_{2}}\right) }\right\}$ . Applying the consequence rule here we end up with

$\vdash  \{ I\}$ while ${e}_{1}$ do ${S}_{1}$ ; while ${e}_{2}$ do ${S}_{2}\left\{  {I\land \neg \left( {{e}_{1} \land  {e}_{2}}\right) }\right\}$ . Combining this with $\vDash  \Phi  \rightarrow  I$ and the second inductive hypothesis we finally get

$\vdash  \{ \Phi \}$ while ${e}_{1}$ do ${S}_{1};S$ ; while ${e}_{2}$ do ${S}_{2};{S}^{\prime }\{ \Psi \}$ .

THEOREM .4 (SOUNDNESS). Assuming soundness of taint environment $\sum$ , if $\sum  \vdash$ SideChannelFree $\left( {\lambda \overrightarrow{p}.S,\epsilon }\right)$ , then the program $\lambda \overrightarrow{p}.S$ does not have an $\epsilon$ -bounded resource side-channel.

Proof. We know that $\sum$ is sound and $\vDash  \Phi  \rightarrow  {\overrightarrow{{p}_{1}}}^{l} = {\overrightarrow{{p}_{2}}}^{l} \land$ ${\overrightarrow{{p}_{1}}}^{h} \neq  {\overrightarrow{{p}_{2}}}^{h}$ . Therefore, lemma . 3 applies, and we get $\vdash  \{ \Phi \} {S}_{1}^{\tau };{S}_{2}^{\tau }\{ \Psi \}$ . Additionally, $\left| { = \Psi  \rightarrow  \left| {{\tau }_{1} - {\tau }_{2}}\right|  \leq  \epsilon }\right|$ . Using the consequence rule in standard Hoare logic, we obtain $\vdash  \{ \Phi \} {S}_{1}^{\tau };{S}_{2}^{\tau }\left\{  {\left| {{\tau }_{1} - {\tau }_{2}}\right|  \leq  \epsilon }\right\}$ . By the soundness of Hoare logic, it follows that $\vDash  \{ \Phi \} {S}_{1}^{\tau };{S}_{2}^{\tau }\left\{  {\left| {{\tau }_{1} - {\tau }_{2}}\right|  \leq  \epsilon }\right\}$ . By the soundness of self-composition, this means that

$$
\forall \overrightarrow{{a}_{1}},\overrightarrow{{a}_{2}}.{\overrightarrow{{a}_{1}}}^{l} = {\overrightarrow{{a}_{2}}}^{l} \land  {\overrightarrow{{a}_{1}}}^{h} \neq  {\overrightarrow{{a}_{2}}}^{h} \Rightarrow  \left| {{\tau }_{1} - {\tau }_{2}}\right|  \leq  \epsilon
$$

By lemma 4.1, ${\tau }_{1} = {R}_{P}\left( \overrightarrow{{a}_{1}}\right)$ and ${\tau }_{2} = {R}_{P}\left( \overrightarrow{{a}_{2}}\right)$ . Substitute $\tau$ with ${R}_{P}$ we arrive at our conclusion

$$
\left. {\forall {\overrightarrow{a}}_{1},{\overrightarrow{a}}_{2}.\left( {{\overrightarrow{a}}_{1}{}^{l} = {\overrightarrow{a}}_{2}{}^{l} \land  {\overrightarrow{a}}_{1}{}^{h} \neq  {\overrightarrow{a}}_{2}{}^{h}}\right)  \Rightarrow  \left| {{R}_{P}\left( {\overrightarrow{a}}_{1}\right)  - {R}_{P}\left( {\overrightarrow{a}}_{2}\right) }\right|  \leq  \epsilon }\right)
$$