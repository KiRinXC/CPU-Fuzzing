# Multi-run side-channel analysis using Symbolic Execution and Max-SMT

Corina S. Păsăreanu

Carnegie Mellon University, NASA Ames

Moffet Field, CA, USA

corina.s.pasareanu@nasa.gov

Quoc-Sang Phan

Carnegie Mellon University

Moffet Field, CA, USA

sang.phan@sv.cmu.edu

Pasquale Malacaria

Queen Mary University of London

London, United Kingdom

p.malacaria@qmul.ac.uk

Abstract-Side-channel attacks recover confidential information from non-functional characteristics of computations, such as time or memory consumption. We describe a program analysis that uses symbolic execution to quantify the information that is leaked to an attacker who makes multiple side-channel measurements. The analysis also synthesizes the concrete public inputs (the "attack") that lead to maximum leakage, via a novel reduction to Max-SMT solving over the constraints collected with symbolic execution. Furthermore model counting and information-theoretic metrics are used to compute an attacker's remaining uncertainty about a secret after a certain number of side-channel measurements are made. We have implemented the analysis in the Symbolic PathFinder tool and applied it in the context of password checking and cryptographic functions, showing how to obtain tight bounds on information leakage under a small number of attack steps.

Index Terms-Side-Channel Attacks; Quantitative Information Flow; Cryptography; Multi-run Security; Symbolic Execution; Satisfiability Modulo Theories; Max-SMT

## I. INTRODUCTION

Side-channel attacks recover secret inputs to programs from non-functional characteristics of computations, such as time consumed, number of memory accesses or size of output files. Side-channel attacks have been shown to pose serious threats by recovering cryptographic keys, e.g. when using the well known RSA encryption/decryption algorithm [1], and private information about users, e.g. as with commonly used algorithms for data compression [2].

We propose a symbolic execution approach for the automatic analysis of software that computes quantitative bounds on the amount of information that can be leaked via side-channel attacks. Technically we use the fact that the amount of leaked information corresponds to the number of different possible side-channel observations, which we compute using symbolic execution and model counting via a reduction to symbolic quantitative information flow analysis (QIF)[3], [4]. The analysis is parametrized by a cost model which allows us to obtain side-channel measurements (time, memory, bytes written to a file, etc.) from the execution of bytecode instructions. The "observables" are the values for the cost computed for each path in the analyzed program. Furthermore we provide a method for automatically deriving the public user input that results into maximum leakage by using weighted Max-SMT solving for which efficient procedures exist [5].

Our key insight is that Max-SMT solving can be used to obtain the maximal assignment over the set of clauses obtained with symbolic execution (i.e. any other assignment satisfies less clauses) and this corresponds to the largest number of observables that can be reached by a particular public input, hence is the maximal leakage: any other choice of public input would result in less observables. We show experimentally that this new Max-SMT encoding can be more efficient than established approaches based on bounded model checking over program self-compositions [6] or on enumerating over the concrete values.

Our Max-SMT approach generalizes naturally over multiple-run side-channel attacks where we use symbolic execution to quantify the information revealed to an attacker after multiple channel measurements made. This corresponds to a typical scenario where an attacker makes multiple guesses by invoking and measuring the execution of the program multiple times on different public inputs to gradually uncover a secret that is constant across program runs (such as the secret key used in the RSA encryption/decryption algorithm). Max-SMT solving is used to compute a sequence of public inputs that lead to maximum leakage, exposing the vulnerability of the program to multi-run attacks.

Furthermore we show how to use an extension of symbolic execution, namely with quantitative reasoning [7], [8], to compute precise values for information theoretic metrics such as Shannon or Smith's min entropy [9]. The technique uses model counting over the constraints collected by symbolic execution, to compute the probability of executing different program paths (under a user-specified profile). Thus we can compute an attacker's remaining uncertainty about a secret after a number (k) of side-channel measurements made. We can also determine whether a secret is fully revealed after $\mathrm{k}$ runs or whether a program keeps leaking information after $\mathrm{k}$ runs etc.

We have implemented the analysis in the Symbolic PathFinder tool and show how to use it to measure side-channel vulnerabilities for Java bytecode programs in the presence of non-adaptive attacks. Our approach is general and can be implemented easily in other symbolic execution tools targeting other languages. We discuss the application of our approach in the context of password checking and cryptographic functions, showing how to obtain tight bounds on information leakage under a small number of attack steps.

![0196f76b-c033-7153-b805-019efb72b7bf_1_312_204_415_103_0.jpg](images/0196f76b-c033-7153-b805-019efb72b7bf_1_312_204_415_103_0.jpg)

Fig. 1. Attacker Model

The rest of the paper is organized as follows. In the next section we give background on quantitative information flow analysis, symbolic execution and constraint solving. We next discuss how to perform side-channel analysis using symbolic execution. Section IV outlines our Max-SMT encoding for finding the low input that maximizes leakage. Section V describes extending the analysis over multiple runs and Section VI discusses treatment of multi-threading. Section VII describes our implementation and experiments, Section VIII gives closely related work and Section IX gives our conclusions.

## II. BACKGROUND

## A. Quantitative Information Flow Analysis

Quantitative Information Flow (QIF) is a powerful approach to "measure" leaks of confidential information in a software system. Typically simple, qualitative, information flow analysis accepts programs as secure if confidential data cannot be inferred by an attacker through their observations of the system-this intuitive property is called non-interference. Although satisfying non-interference is a sound guarantee for a system to be secure, this requirement is too strict for most realistic programs: to leak some information is almost unavoidable. By quantifying leakage QIF addresses this limitation: not only programs with "zero interference" (non-interference) can be accepted as secure, but also programs with "small" interference.

Fig. 1 shows an example function that we use to illustrate QIF. It is a convention in the security literature to use the label $\mathrm{L}$ (low) to denote non-sensitive input, to use the label $\mathrm{H}$ (high) to denote sensitive private input, and to use the label $\mathrm{O}$ (output) to denote the output. A malicious user has access to the public data, $\mathrm{L}$ and $\mathrm{O}$ , and tries to infer the hidden secret, $\mathrm{H}$ , from that.

Similar to previous work[10] we assume that the malicious user can make one side-channel measurement per invocation of the program $P$ and that no measurement errors occur. Furthermore, we assume that the attacker has full knowledge about the implementation of $P$ .

A fundamental QIF result (the channel capacity theorem [11], [9]) shows that leakage for a program is always less than or equal to the log of the number of possible distinct observations that an attacker can make. By noting these observables with(NObs)and channel capacity with ${CC}$ , we have hence:

$$
\text{Information leaked} \leq  {\log }_{2}\left( \text{ NObs }\right)  = {CC}\left( P\right)
$$

The result states in essence that QIF reduces to counting the number of different observable outputs for the program. The result holds for different notions of leakage based on the probability of guessing the secret or the notion of leakage based on Shannon's information theory measuring the number of bits leaked. For these reasons counting the number of observables is the basis of state-of-the-art QIF analysis [6], [12], [13], [14], [15], [16].

## B. Symbolic Execution

Symbolic execution is a program analysis technique which executes programs on symbolic rather than concrete inputs, and it computes the program effects as functions in terms of the symbolic inputs [17]. The behavior of a program $P$ is determined by the values of its inputs and can be described by means of a symbolic execution tree where tree nodes are program states and tree edges are the program transitions as determined by the symbolic execution of program instructions.

The state $s$ of a program is defined by the tuple(IP, V, PC) where ${IP}$ represents the next instruction to be executed, $V$ is a mapping from each program variable $v$ to its symbolic value (i.e., a symbolic expression in terms of the symbolic inputs), and ${PC}$ is a path condition. ${PC}$ is a conjunction of constraints over the symbolic inputs that characterizes those inputs that follow the path from the program’s initial state to state $s$ . The current state $s$ and the next instruction ${IP}$ define the set of transitions from $s$ . The feasibility of the path conditions is checked using off-the-shelf constraint solvers [18], as the conditions are encountered during symbolic execution to detect infeasible paths and to generate test inputs that execute feasible paths. There are many tools that support symbolic execution for programs [19], [20], [21]. For our implementation we use Symbolic PathFinder - part of Java PathFinder tool set - described below.

## C. Java PathFinder and Symbolic PathFinder

Java PathFinder [22] (JPF) is an extensible run-time environment for the verification of Java bytecode, i.e., compiled Java programs. The analyses proposed here are incorporated Symbolic PathFinder (SPF) [23], which extends JPF with a symbolic execution mode. SPF uses JPF to systematically generate and execute the symbolic execution tree of the code under analysis and also to handle multi-threading. SPF uses a variety of off-the-shelf decision procedures and constraint solvers to solve the constraints generated by the symbolic execution of bytecode programs.

In recent work [7], [8] SPF was extended with quantitative reasoning which we leverage for quantifying the results. Specifically, SPF uses a combination of symbolic execution and model counting [24], [8] for computing the probability of successful termination (and alternatively the probability of failure) for a Java software component placed in a stochastic environment.

### D.SMT and weighted Max-SMT

A propositional variable $p$ is an assertion that must be true or false. A propositional variable $p$ or its negation $\neg p$ is called a literal. A clause is a disjunction of literals. A propositional formula is a conjunction of clauses. The problem of propositional satisfiability (SAT) is to determine, for a given formula, whether or not it is satisfiable, i.e. if it has a model: an assignment of Boolean values to variables that satisfies the formula.

---

int c = 32, low = 0, time = 0;

										while $\left( {\mathrm{c} >  = 0}\right) \{$

													int m = (int) Math.pow( 2 , c );

												if (high >= m) \{

																						low $= {low} + m$ ;

																					high = high - m ;

																						time ++;

									\}

														$\mathrm{c} = \mathrm{c} - 1$ ;

													time++;

	low $= 0$ ;

---

Fig. 2. Example of timing channel

An extension of SAT is the satisfiability modulo theories (SMT) problem [25]: which decides the satisfiability of a given quantifier-free first-order formula with respect to a background theory, such as linear arithmetic.

Max-SAT [26] is an extension of SAT to optimization. Given a weighted formula ${\varphi }_{w}$ where each clause ${C}_{i}$ has a weight ${\omega }_{i}$ (positive or infinity), find the assignment such that the sum of the weights of the falsified clauses is minimized.

The Max-SMT problem [5] is a generalization of the Max-SAT problem to first-order theories: given a weighted first-order formula ${\varphi }_{w}$ , find a model that minimizes the sum of the weights of the falsified clauses, or alternatively maximizes the sum of satisfied clauses; we use the latter formulation here. Several powerful SMT and Max-SMT solvers exist [18], [5] and have been shown to work on industrial strength examples.

## III. Symbolic Execution for Side-Channel ANALYSIS

Given a program $P$ with "high" inputs (secret) and "low" inputs (public) our goal is to compute information leakage via side-channels, i.e. time or memory consumption, writing to a file or socket etc. We base the computation of the leakage on Shannon’s Information theory where the observables $\mathcal{O}$ are the costs computed for each program path $\pi$ , denoted $\operatorname{cost}\left( \pi \right)$ i.e. number of instructions executed along $\pi$ , number of bytes allocated etc.

Thus we assume that each path can only give one observable. Our work is done in the context of a project that specifically targets side-channels but it is applicable to more general quantitative information flow analysis where this assumption holds.

This approach is illustrated by the example in Fig. 2. This program would leak all secrets (denoted by "high") into "low", but since "low" is set to 0 at the end of the program, the leakage into "low" is eliminated. However there is a side time channel which reveals the number of $1\mathrm{\;s}$ in the binary representation of the variable "high" (the side channels used to break RSA had this shape [27]). That side channel can be captured by introducing an auxiliary "time" variable as in Fig. 2. Intuitively, by introducing the variable "time" we simulate an adversary observing the timing channel. By observing the variable "time" at the end of the program we can deduce the timing leakage of "high" so to recover the side channel in terms of classical observables [3], [4]. Similarly, for memory allocation we keep a variable that counts the number of bytes allocated for example, according to the cost model yielding a unified framework for detecting time/memory side channels and for quantifying how many bits of information are being leaked.

Notice that in practice we do not introduce an auxiliary variable for time (or memory) but measure directly the cost of executing each bytecode instruction, using JPF listeners (see Section VII).

For the example in the Fig. 2, our analysis gives the result that the leakage measured in channel capacity is 5.459 bits.

## A. Computing Channel Capacity

To compute the Channel Capacity we perform a symbolic execution over the program where both "high" $h$ and "low" $l$ inputs are symbolic. We assume that the analyzed program is deterministic and that the input domain is finite.

We run symbolic execution to collect all symbolic paths of the program. Note that there is only a finite number of them. Each symbolic path summarizes multiple (possibly many) concrete program paths that follow the same instructions through the analyzed code. For simplicity we will assume that all the paths terminate within the prescribed bound. As this is not always the case in general, in practice we can use a notion of confidence (similar to [7]) that quantifies the impact of the execution bound on the quality of the analysis.

Let $P\left( {h, l}\right)$ be the (deterministic) program, and ${\pi }_{1},{\pi }_{2}..{\pi }_{n}$ be the symbolic paths of $P\left( {h, l}\right)$ . Let $P{C}_{1}\left( {h, l}\right) , P{C}_{2}\left( {h, l}\right) \ldots$ $P{C}_{n}\left( {h, l}\right)$ be the path conditions (disjoint by construction) computed with symbolic execution. Throughout the paper we use lower-case $h$ and $l$ to denote symbolic high, respectively low, inputs. Upper-case $H$ and $L$ denote concrete values. Assume every ${PC}$ contains at least one high variable (if not we eliminate all the PC's that only depend on low variables since they leak no information). Note that $h$ and $l$ may represent vectors of secrets and public values, respectively.

We can then extract the number of observables by counting the number of paths that have different cost. For each symbolic path ${\pi }_{i}$ we compute its cost $\operatorname{cost}\left( {\pi }_{i}\right)$ representing the side-channel measurement along that path according to a cost model.

Let $\mathcal{O} = \left\{  {{o}_{1},{o}_{2},\ldots {o}_{m}}\right\}$ , with $m \leq  n$ , be the set of observable costs. Note that different symbolic paths may have the same cost but a path can not have more than one cost. For the rest of the paper we use cost and observable interchangeably to refer to the side-channel measurement made along a path. Since each path condition $P{C}_{i}$ has a one-toone correspondence to a path ${\pi }_{i}$ , we also write $\operatorname{cost}\left( {P{C}_{i}}\right)$ to mean $\operatorname{cost}\left( {\pi }_{i}\right)$ . Furthermore, we write $\operatorname{cost}\left( {P\left( {H, L}\right) }\right)$ to denote the cost of the (concrete) path followed by the program on concrete values $H$ and $L$ . The Channel Capacity, i.e. the maximal possible leakage is then:

$$
{CC}\left( P\right)  = {\log }_{2}\left( \left| \mathcal{O}\right| \right)
$$

Note that this is an upper bound of the actual leakage of the program. We discuss how to obtain tighter bounds in the next section.

## B. Computing Shannon Entropy

The Shannon entropy $\mathcal{H}\left( P\right)$ gives a precise bound for the leakage, when the secret distribution is known. We can compute $\mathcal{H}\left( P\right)$ using probabilistic symbolic execution [7] as implemented in SPF. Let $\mathcal{O} = \left\{  {{o}_{1},{o}_{2},\ldots {o}_{m}}\right\}$ be the set of observables as defined before. For a uniform distribution over the secret, the probability of observing ${o}_{i}$ is:

$$
p\left( {o}_{i}\right)  = \frac{\mathop{\sum }\limits_{{\operatorname{cost}\left( {\pi }_{j}\right)  = {o}_{i}}}\sharp \left( {P{C}_{j}\left( {h, l}\right) }\right) }{\sharp D}
$$

Here $\sharp \left( {P{C}_{j}\left( {h, l}\right) }\right)$ denotes the number of solutions (i.e. high and low concrete values) of constraint $P{C}_{j}\left( {h, l}\right)$ (computed with an off-the-shelf model counting procedure such as Latte) and $\# D$ denotes the size of the input domain $D$ assumed to be (possible very large but) finite.

The Shannon entropy is then:

$$
\mathcal{H}\left( P\right)  =  - \mathop{\sum }\limits_{{i = 1, m}}p\left( {o}_{i}\right) {\log }_{2}\left( {p\left( {o}_{i}\right) }\right)
$$

For deterministic systems, the Shannon entropy also gives a measure of the leakage of the side-channel, corresponding also to the observation gain (on the secret) after one round of observation (see also Section V-B). Similar calculations can be used to compute other information-theoretic metrics, e.g. one can use the computed solution spaces to estimate the number of tries needed to guess a secret.

Note that SPF already incorporates Latte [24] for quantifying the solution spaces of mathematical linear-integer constraints; SPF also uses several optimizations such as simplifications, separate solving of independent constraints, caching and composition of results [8] to perform the model counting efficiently. For the example in the Fig. 2, our analysis computes the Shannon entropy: 3.925 bits (we assume the domain of variables is $0\ldots {10}^{4}$ , uniform distribution).

For the rest of the paper we focus on the computation of leakage using the channel capacity, since it gives worst-case bounds. However our implementation supports computing Shannon entropy as well.

## IV. SINGLE-RUN ANALYSIS (ALGORITHM MaxLeak)

In the previous section we presented a symbolic execution technique for computing the side-channel leakage after one program run. However, in the analysis we did not distinguish between high and low values, resulting in an over-approximation in the computed bounds. The problem is illustrated by the example in Fig. 3.

---

void example1(int 1, int h)\{

	if $\left( {1 < 0}\right) \{$

			if $\left( {\mathrm{h} < 0}\right) \operatorname{cost} = 1$ ;

			else if $\left( {h < 5}\right) \cos t = 2$ ;

			else $\cos t = 3$ ;

		\} else \{

				if $\left( {\mathrm{h} > 1}\right) \operatorname{cost} = 4$ ;

				else $\cos t = 5$ ;

	\}

---

Fig. 3. Example 1

---

	<img src="https://cdn.noedgeai.com/0196f76b-c033-7153-b805-019efb72b7bf_3.jpg?x=1275&y=209&w=347&h=255&r=0"/>

---

Fig. 4. Example 2

Here 1 denotes a public input while $\mathrm{h}$ denotes the secret; cost captures the observed cost along each path in the program. The number of possible observables for this program is 5, yielding a leakage of ${\log }_{2}\left( 5\right)$ according to the formulation from the previous section. However note that there is no public input that can possibly achieve this leakage in a single program run. Indeed, for $l < 0$ there are 3 possible observables (i.e. different costs), while for $l \geq  0$ there are 2 observables. Thus the maximum leakage of the program after one round of observations is only ${\log }_{2}\left( 3\right)$ .

Our goal is to compute automatically the value of the "low" public input that results in the maximum number of observables. This has a very important security meaning: it reveals the most vulnerable behaviors of a program in one run and the low input that trigger that behavior. Furthermore, that low input value can be used for building multiple-run attacks as described in next section.

The computation becomes more involved because for the same concrete low input, there may be multiple symbolic paths with different costs, corresponding to different values of the high input. For example, for any negative values of $l$ , there are three symbolic paths corresponding to $h < 0$ (with cost $1), h \geq  0 \land  h < 5$ (with cost 2) and $h \geq  5$ (with cost 3), respectively.

Intuitively picking such a low value translates in the maximum information about a secret that an attacker can get from observing one run of the program. Let us make this intuition more precise.

## A. Attackers knowledge

To find out the value of a secret $H$ , an attacker picks a value for the public input $L$ , invokes the program $P\left( {H, L}\right)$ and observes the cost obs. In general the attacker can not simply deduce $H$ from obs but she can infer the constraints on the secret that are coherent with the observation obs.

Following [10] we say that a secret $H$ is coherent with obs under $L$ whenever $P\left( {H, L}\right)$ has cost obs. Two secrets ${H}_{1}$ and ${H}_{2}$ are indistinguishable under $L$ if both $P\left( {{H}_{1}, L}\right)$ and $P\left( {{H}_{2}, L}\right)$ yield the same cost, i.e. $\operatorname{cost}\left( {P\left( {{H}_{1}, L}\right) }\right)  =$ $\operatorname{cost}\left( {P\left( {{H}_{2}, L}\right) }\right)$ . For every cost obs, the set of secret values that are coherent with obs under $L$ forms an equivalence class of indistinguishability under $L$ .

We propose to use symbolic execution to compute the constraints on the secret $h$ that are coherent with observation obs under $L$ . Notice that all the paths that lead to same cost obs under input $L$ satisfy the following constraint.

$$
\mathop{\bigvee }\limits_{{\operatorname{cost}\left( {P{C}_{i}}\right)  = {obs}}}P{C}_{i}\left( {h, L}\right)
$$

Here $P{C}_{i}\left( {h, L}\right)$ denotes the constraint $P{C}_{i}\left( {h, l}\right)$ where symbolic value $l$ was replaced with concrete value $L$ .

It follows that, for given $L$ , the set of observables (or equivalence classes) induced by $L$ is given by:

$$
{\mathcal{O}}_{L} = \left\{  {{obs} \in  \mathcal{O}\mid \exists H.\mathop{\bigvee }\limits_{{\operatorname{cost}\left( {P{C}_{i}}\right)  = {obs}}}P{C}_{i}\left( {H, L}\right) }\right\}
$$

The "most powerful attacker" will want to pick a value $L$ that induces the partitioning with the maximum number of equivalence classes (i.e. maximum number of observables), since that will reveal the most information about the secret. Such a maximal $L$ would need to satisfy the maximum number of clauses of the form $\mathop{\bigvee }\limits_{{\operatorname{cost}\left( {P{C}_{i}}\right)  = {obs}}}P{C}_{i}\left( {h, l}\right)$ , for ${obs} \in  \mathcal{O}$ .

One solution would be to enumerate all the possible values for the low input and for each low input $L$ to compute the cardinality of ${\mathcal{O}}_{L},\left| {\mathcal{O}}_{L}\right|$ . The low input that yields the maximum number of observables is then returned to the user. Although the input domain is assumed to be finite in practice this explicit enumeration approach would be inefficient. We present a symbolic approach below.

## B. Max-SMT formulation

We first rename each path condition as follows: for each $P{C}_{i}\left( {h, l}\right)$ , we define $P{C}_{i}\left( {{h}_{i}, l}\right)$ where all the high symbolic values $h$ have been renamed to fresh symbolic values ${h}_{i}$ . Intuitively the renamed path conditions define constraints on low variables (while the high variables are left unconstrained) and the goal is to find the low input value that leads to the maximum number of observations for any value of $\mathrm{h}$ . This intuition has similarities with self composition [28], [29].

For example, for path condition $P{C}_{1}\left( {h, l}\right)  : l < 0 \land  h < 0$ , we re-write it as $P{C}_{1}\left( {{h}_{1}, l}\right)  : l < 0 \land  {h}_{1} < 0$ while path condition $P{C}_{2}\left( {h, l}\right)  : l < 0 \land  h \geq  0 \land  h < 5$ , we re-write it as $P{C}_{2}\left( {{h}_{2}, l}\right)  : l < 0 \land  {h}_{2} \geq  0 \land  {h}_{2} < 5$ etc.

We formulate a Max-SMT problem by building a clause for each cost, where the clause is the disjunction of the PCs that lead to same cost, and the weight is 1 . Intuitively each clause defines an indistinguishability equivalence relation as defined above. We then let Max-SMT solver pick the value $L$ that satisfies most clauses, meaning that it induces the partitioning on the secret with the maximum number of equivalence indistinguishability classes. Algorithm 1 outlines our approach, where the input is a program $P\left( {h, l}\right)$ and a function cost that defines the cost for each program path.

Theorem 1: The solution returned by Algorithm 1 yields maximum leakage for program $\mathrm{P}$ .

Proof 1: Let $L$ be the solution for $l, w$ be the total weight and ${C}_{{i}_{1}},{C}_{{i}_{2}}..{C}_{{i}_{w}}$ be the set of satisfiable clauses returned by Max-SMT. $L$ induces a partitioning where each equivalence class contains the secret values that satisfy one of ${C}_{{i}_{1}},{C}_{{i}_{2}}..{C}_{{i}_{w}}$ . Since each clause has weight 1 and Max-SMT returns the set of clauses that maximizes the weight it follows that $L$ induces the partitioning with the maximum number of equivalence classes and hence maximum leakage.

Algorithm 1 MaxLeak: Single-Run Side-Channel Analysis

---

Inputs: Program $P\left( {h, l}\right)$ and cost function cost.

Perform symbolic execution over $P\left( {h, l}\right)$ .

Compute path conditions $P{C}_{1}\left( {h, l}\right) , P{C}_{2}\left( {h, l}\right) ..P{C}_{n}\left( {h, l}\right)$ .

Compute observables $\mathcal{O} = \left\{  {\operatorname{cost}\left( {P{C}_{i}\left( {h, l}\right) }\right)  \mid  i = 1, n}\right\}$ .

for each cost ${o}_{i} \in  \mathcal{O}$ do

	Build Max-SMT clause with weight $1 : {C}_{i} :  : \left( {\mathop{\bigvee }\limits_{{\operatorname{cost}\left( {P{C}_{j}}\right)  = {o}_{i}}}P{C}_{j}\left( {{h}_{j}, l}\right) }\right)$

end for

Solve ${C}_{1},{C}_{2}..{C}_{m}$ with Max-SMT solver.

Let $L$ be the solution returned by Max-SMT with weight $w$

return Low input value: $L$ and max number of observables: $w$

---

## C. Examples

Let us now analyze Example 1 using the Max-SMT formulation. The example has 5 symbolic paths, with the following (renamed) path conditions:

$P{C}_{1} : l < 0 \land  {h}_{1} < 0$ with cost 1,

$P{C}_{2} : l < 0 \land  {h}_{2} \geq  0 \land  {h}_{2} < 5$ with cost 2,

$P{C}_{3} : l < 0 \land  {h}_{3} \geq  5$ with cost 3,

$P{C}_{4} : l \geq  0 \land  {h}_{4} > 1$ with cost 4,

$P{C}_{5} : l \geq  0 \land  {h}_{5} \leq  1$ with cost 5 .

Its encoding as a Max-SMT problem is as the set of the following clauses, each with weight 1:

${C}_{1} :  : \left( {l < 0 \land  {h}_{1} < 0}\right)$

${C}_{2} :  : \left( {l < 0 \land  {h}_{2} \geq  0 \land  {h}_{2} < 5}\right)$

${C}_{3} :  : \left( {l < 0 \land  {h}_{3} \geq  5}\right)$

${C}_{4} :  : \left( {l \geq  0 \land  {h}_{4} > 1}\right)$

${C}_{5} :  : \left( {l \geq  0 \land  {h}_{5} \leq  1}\right)$

Max-SMT reports that ${C}_{1},{C}_{2},{C}_{3}$ are satisfiable with $l =  - 1$ as a solution (Max-SMT also gives solutions for ${h}_{1},{h}_{2},{h}_{3},{h}_{4},{h}_{5}$ which we ignore here) resulting in maximum weight 3. Thus the low input that achieves maximum leakage is $l =  - 1$ and the leakage is ${\log }_{2}\left( 3\right)  = {1.58}$ bits.

Consider another example (see Figure 4). The example has 4 symbolic paths with following path conditions and costs:

$P{C}_{1} : {h}_{1} > 0 \land  l < 0$ with cost 1,

$P{C}_{2} : {h}_{2} > 0 \land  l \geq  0$ with cost 2,

$P{C}_{3} : {h}_{3} \leq  0 \land  l < 5$ with cost 1,

$P{C}_{4} : {h}_{4} \leq  0 \land  l \geq  5$ with cost 4 .

Note that the paths that have conditions $P{C}_{1}$ and $P{C}_{3}$ both have same cost (1). The Max-SMT clauses are as follows, each with weight $1 : {C}_{1} :  : \left( {\left( {{h}_{1} > 0 \land  l < 0}\right)  \vee  \left( {{h}_{3} \leq  0 \land  l < 5}\right) }\right)$ ${C}_{2} :  : \left( {{h}_{2} > 0 \land  l \geq  0}\right)$ ${C}_{3} :  : \left( {{h}_{4} \leq  0 \land  l \geq  5}\right)$

The result of Max-SMT solving is that for $l = 0$ , the first two clauses are satisfied with maximum weight 2 , hence maximum leakage is ${\log }_{2}\left( 2\right)  = 1$ bit.

## V. MULTI-RUN ANALYSIS (ALGORITHM MaxLeak)

We consider now the case where a malicious agent performs a multi-run attack to gather information for deducing $h$ or narrowing down its possible values. Such an attack consists of a sequence of attack steps, where each step consists in querying the program on a chosen low input and measuring the cost of the side-channel for that program run. An attack ends if either the secret changes or if the attacker stops querying the system. We consider the case of non-adaptive attacks, where the attacker does not have access to the system's responses until the end of the attack. Thus, when choosing a message, she cannot take into account the outcomes of her previous queries.

Let us analyze the attacker’s knowledge after $k$ rounds of observation. Suppose that the attacker picked values ${L}_{1},{L}_{2}$ .. ${L}_{k}$ and observed the program $k$ times, by running $P\left( {H,{L}_{i}}\right)$ , for $i = 1\ldots k$ . Suppose each $P\left( {H,{L}_{i}}\right)$ leads to an observation ${o}_{i}$ . Intuitively, with each new observation, the attacker can infer more constraints on the secret, i.e., after $k$ observations, the attacker learns that the secret is coherent with ${o}_{1}$ under ${L}_{1}$ , with ${o}_{2}$ under ${L}_{2}$ ,.. and with ${o}_{k}$ under ${L}_{k}$ .

Similar to the single-run analysis, we say that two secret values ${H}_{1}$ and ${H}_{2}$ are indistinguishable under input sequence $\left\langle  {{L}_{1},{L}_{2}..{L}_{k}}\right\rangle$ if they lead to the same observable sequence $\left\langle  {{o}_{1},{o}_{2},..{o}_{k}}\right\rangle$ .

Proposition 1: The indistinguishability relation under $\left\langle  {{L}_{1},{L}_{2},..{L}_{k}}\right\rangle$ forms an equivalence relation on the secret values.

Proof 2: Reflexivity, symmetry and transitivity are easy to check because the program is deterministic. For example for transitivity we need to show ${H}_{1} \approx  {H}_{2}$ and ${H}_{2} \approx  {H}_{3}$ implies ${H}_{1} \approx  {H}_{3}$ .

${H}_{1} \approx  {H}_{2}$ means that ${H}_{2}$ and ${H}_{1}$ under input sequence $\left\langle  {{L}_{1},{L}_{2}..{L}_{k}}\right\rangle$ lead to the same observable sequence $s$ , and ${H}_{2} \approx  {H}_{3}$ means that ${H}_{2}$ and ${H}_{3}$ under input sequence $\left\langle  {{L}_{1},{L}_{2}..{L}_{k}}\right\rangle$ lead to the same observable sequence ${s}^{\prime }$ . Hence, because the program is deterministic, $s = {s}^{\prime }$ and ${H}_{1}$ and ${H}_{3}$ under input sequence $\left\langle  {{L}_{1},{L}_{2}..{L}_{k}}\right\rangle$ lead to the same observable sequence $s$ .

The approach described in the previous section generalizes naturally for a multi-run analysis, where in the case of a $k$ -step attack, we analyze the $k$ -composition of the program, denoted ${P}_{k}\left( {h,{l}_{1},{l}_{2}..{l}_{k}}\right)$ :

$$
P\left( {h,{l}_{1}}\right) ;P\left( {h,{l}_{2}}\right) ;..P\left( {h,{l}_{k}}\right)
$$

In other words, we consider running the same program $k$ times, with different symbolic inputs: ${l}_{1},{l}_{2}..{l}_{k}$ . Note that $h$ remains the same across the $k$ runs (since the secret is the same across the runs). Each path $\pi$ in ${P}_{k}\left( {h,{l}_{1},{l}_{2}..{l}_{k}}\right)$ is a composition of paths ${\pi }_{1};{\pi }_{2};..{\pi }_{k}$ , where each ${\pi }_{i}$ is a path in the i-th program version, $P\left( {h,{l}_{i}}\right)$ . We define the cost ${\operatorname{cost}}_{k}\left( \pi \right)  = \left\langle  {{o}_{1},{o}_{2},..{o}_{k}}\right\rangle$ , where each ${o}_{i}$ is the cost of ${\pi }_{i}$ , i.e., an observable for the composite program is the sequence of $k$ side-channel measurements made during the attack. With the new formulation of the observables, we can apply Algorithm 1 to ${P}_{k}$ (where the low input $l$ is now the sequence $\left\langle  {{l}_{1},\ldots ,{l}_{k}}\right\rangle$ ) to compute the sequence of low inputs that lead to the maximum number of different observable tuples in $k$ steps. We denote this as ${\mathbf{{MaxLeak}}}_{k}$ .

Theorem 2: The solution returned by Algorithm 1 for ${P}_{k}\left( {h,{l}_{1},{l}_{2}..{l}_{k}}\right)$ and ${\operatorname{cost}}_{k}$ yields maximum leakage for program $P$ after $k$ runs.

Theorem 2 follows directly from Theorem 1 (the program being analyzed is the k -composition of $P$ and the cost function is ${\operatorname{cost}}_{k}$ ).

Consider again Example 1 in Fig. 3. In two runs, Max-SMT returns 4 satisfiable clauses out of 13 distinct clauses. Hence, there are 4 observables and the maximum leakage is 2 bits. The low inputs are ${l}_{1} = 0$ and ${l}_{2} =  - 1$ . For Example 2 in Fig. 4, in two runs, Max-SMT reports 2 satisfiable clauses out of 7 clauses. Therefore the number of observables is 2, the same as in the first run, and this is the maximum leakage possible.

We also formalize here the intuition that the attacker obtains more information about the secret with each program run. The attacker obtains more information (or equivalently reduces the uncertainty) about the secret as it refines the partition induced by the input with each program run. If two values ${H}_{1}$ and ${H}_{2}$ of the secret $h$ are indistinguishable, then they are in the same equivalence class in the equivalence relation. When the attacker runs the program one more time, she can distinguish more values of the secret $h$ , by splitting some equivalence classes (in other word, the equivalence relation is refined). This results in more equivalence classes (likely of smaller sizes) so the amount of information gained increases.

Proposition 2: Let ${w}_{k}$ and ${w}_{k + 1}$ be the number of observ-ables returned by Algorithm 1 for a k-run and k+1-run analysis respectively. Then ${w}_{k} \leq  {w}_{k + 1}$ .

As a corollary we have that if the leakage is the same at steps $k$ and $k + 1$ , then that corresponds to maximum leakage of the program [30].

## A. Greedy Multi-run Analysis (Algorithm GreedyLeak ${}_{k}$ )

While in the previous section we have shown how to compute maximal leakage over $k$ runs, the above algorithm doesn't guarantee a specific leakage ordering, so for example for a program with maximal leakage of 7 bits in two runs Algorithm 1 could return an assignment where 2 bits are leaked in the first run and 5 bits are leaked in the second run.

An important leakage ordering is the one corresponding to an attacker at each step picking the low input returning the maximal leakage for that run. We can easily adapt our previous algorithm to capture this attacker. To do so we should proceed as follows: find the maximal ${L}_{1}$ over one run using Algorithm 1: consider then the maximal low input over two runs where we have "blocked" the clauses satisfied by ${L}_{1}$ , we have now ${L}_{1},{L}_{2}$ and we repeat until we have found ${L}_{1},\ldots ,{L}_{k}$ .

The above algorithm is greedy, so it doesn't necessarily return the sequence with the maximal possible leakage at each step. For example consider the case where ${L}_{1}$ and ${L}_{1}^{\prime }$ both return the same maximal leakage in the first run but after ${L}_{1}$ the next maximal ${L}_{2}$ returns a higher maximal leakage than the maximal leakage that ${L}_{2}^{\prime }$ returns after ${L}_{1}^{\prime }$ . Max-SMT could choose ${L}_{1}^{\prime }$ over ${L}_{1}$ and so the sequence chosen by the algorithm starting with ${L}_{1}^{\prime },{L}_{2}^{\prime },\ldots$ would have a lower leakage in the second element that the sequence starting with ${L}_{1},{L}_{2},\ldots$ . We call this approach ${\mathbf{{GreedyLeak}}}_{k}$ .

Other possible attackers can be modelled in a similar fashion, e.g. an adaptive attacker that chooses the next low input following the observables from the previous rounds. In this case the attacker can be modelled as a function from sequence of observables (the history) to low inputs (the next input).

## B. Information-theoretic metrics

Let us consider a generic function $\mathcal{L}$ measuring leakage in bits, e.g. channel capacity or Shannon entropy, and suppose $\mathcal{L}$ is capable to measure leakage over multiple runs of a program $P$ given low inputs ${l}_{i}$ . Let’s denote $\mathcal{L}\left( {P\left( {H,{L}_{1},\ldots ,{L}_{k}}\right) }\right)$ the leakage over $k$ runs with low inputs ${L}_{1},\ldots ,{L}_{k}$ .

We can then define the information gain at the $k + 1$ run according to $\mathcal{L}$ and low inputs ${L}_{1},\ldots ,{L}_{k},{L}_{k + 1}$ as $\mathcal{L}\left( {{P}_{k + 1}\left( {H,{L}_{1},\ldots ,{L}_{k + 1}}\right) }\right)  - \mathcal{L}\left( {{P}_{k}\left( {H,{L}_{1},\ldots ,{L}_{k}}\right) }\right)$ , that is the difference between the leakage after $k + 1$ and $k$ runs: this is the leakage revealed by the $k + 1$ run of the program $P$ as measured by $\mathcal{L}$ . It is the secret revealed by $P\left( {H,{L}_{k + 1}}\right)$ which had not been previously been revealed by ${\left( {P}_{j}\left( H,{L}_{1},\ldots ,{L}_{j}\right) \right) }_{1 \leq  j \leq  k}$

The remaining secrecy at the $k$ -th run according to $\mathcal{L}$ is defined as the difference between the initial secret and the leakage according to $\mathcal{L}$ as measured after $k$ runs: this is the secret that has not been leaked in the $k$ runs.

In the present work we concentrated on channel capacity as the leakage measure $\mathcal{L}$ , that is the maximal leakage as measured according to Shannon entropy or Smith's min entropy [9]. In this setting we have that the information gain at the $k + 1$ run is the maximal leakage revealed by the $k + 1$ run as measured by Shannon or Min entropy. The remaining secrecy at the $k$ -th run is the secret that cannot have been leaked in the $k$ runs. This is the minimum amount of secrecy guaranteed after $k$ runs.

If we were to take as measure of leakage $\mathcal{L}$ Shannon entropy, computed for example with model counting over the constraints as outlined in section III-B, then the information gain at the first step reduces to just the leakage (i.e. $\mathcal{H}\left( P\right)  - 0 = \mathcal{H}\left( P\right)  = \mathcal{L}\left( P\right)$ ) and the remaining secrecy at the 1st step is just the remaining uncertainty, i.e. the posterior entropy

$$
\mathcal{H}\left( h\right)  - \mathcal{L}\left( P\right)  = \mathcal{H}\left( h\right)  - \left( {\mathcal{H}\left( h\right)  - \mathcal{H}\left( {h \mid  P}\right) }\right)  = \mathcal{H}\left( {h \mid  P}\right)
$$

The above generalize to $k$ runs, e.g. the remaining secrecy at the $k$ -th run would then be the remaining uncertainty about the secret given $k$ runs and the information gain at the $k + 1$ run is the reduction in uncertainty about the secret induced by the $k + 1$ run of the program. As Shannon entropy is, when we know the distribution of the secret, a more precise measure of leakage than channel capacity, the induced measures of information gain and remaining secrecy will also be more precise. For example since channel capacity can be a big overestimation of leakage this will make remaining secrecy according to channel capacity a big underestimation of the non leaked secret. As a concrete example consider a password check over 1000 elements: the remaining secrecy according to channel capacity after one step will be $\log \left( {1000}\right)  - 1 \simeq  {8.9}$ whereas the real remaining secrecy is closer to 9.9 bits.

Notice that in terms of our algorithm an information gain of 0 at $k + 1$ means that the number of equivalence classes (maximal number of satisfied clauses) at round $k$ and $k + 1$ is equal, i.e. nothing new about the secret can be revealed at round $k + 1$ . A particular case of information gain being 0 is when the maximal number of satisfied clauses is $d$ where $d$ is the size of the secret. In that case all secret has been revealed.

Note that our algorithms find the low inputs that maximize the number of equivalence classes. Our tool can compute the sizes of each partition (using model counting) to calculate various measures such as Shannon entropy. Finding the input that maximizes entropy for a specific distribution is future work.

## VI. DISCUSSION

## A. Multi-threading

So far we assumed that the program under analysis is sequential. We discuss here how to extend the analysis to multi-threaded programs using maximal linear schedules (i.e. thread interleavings) [7]. Suppose we have no knowledge about the next-choice distribution or the specific scheduling policy for the thread scheduler.

Intuitively each thread scheduling induces a sequential program on which we can apply the leakage analysis as described above. As there is a finite number of schedules for the current exploration depths, we can therefore compute the low input and maximum leakage for each one of them and report the worst case to the user.

To account for multi-threading we need to extend the definition of symbolic execution as well as the attacker model, since an input may result in multiple program executions, corresponding to different thread schedules. We replace IP with a set of pairs $\left( {{t}_{i}, I{P}_{i}}\right)$ , where each ${t}_{i}$ identifies an active thread and $I{P}_{i}$ represents the next instruction to be executed by thread ${t}_{i}$ . A schedule $\mathcal{S}$ is a sequence ${t}_{i},{t}_{j},\ldots {t}_{k}$ defining the order of access to the CPU for all the active threads. The result of such an execution is a set of pairs $\left( {{\mathcal{S}}_{i}, P{C}_{i}}\right)$ where $P{C}_{i}$ are path conditions and ${\mathcal{S}}_{i}$ is the schedule associated with the specific execution. We can record the schedules produced by symbolic execution into a prefix tree and define the maximal schedules:

Definition 1: A thread schedule is maximal if it is not a prefix of any other schedule in the paths reported by a (bounded) symbolic execution.

For a thread schedule $\mathcal{S}$ that is maximal, we define a set of path conditions

$$
{\Pi }_{\mathcal{S}} = \left\{  {P{C}_{i} \mid  {\mathcal{S}}_{i} \in  \operatorname{prefix}\left( \mathcal{S}\right) }\right\}
$$

where $\operatorname{prefix}\left( \mathcal{S}\right)$ is the set of all the prefixes of $\mathcal{S}$ including $\mathcal{S}$ . Intuitively ${\Pi }_{\mathcal{S}}$ contains all the path conditions for the paths that "follow" the same schedule $\mathcal{S}$ , accounting for possible early termination.

For a maximal schedule the path conditions corresponding to ${\Pi }_{\mathcal{S}}$ cover the entire domain input [7]. Thus, ${\Pi }_{\mathcal{S}}$ is the result of symbolically executing program $P\left( {h, l}\right)$ restricted to schedule $\mathcal{S}$ which can be seen as executing a sequential program ${P}_{\mathcal{S}}\left( {h, l}\right)$ . We can therefore compute the value of low that maximizes leakage by applying Algorithm 1 to ${P}_{\mathcal{S}}\left( {h, l}\right)$ (essentially the path conditions computed by Algorithm 1 will be the path conditions corresponding to each path in ${\Pi }_{\mathcal{S}}$ ). Maximal schedules can then be ordered according to their worst leakage and the worst result is reported to the user. The approach extends to multi-run analysis as well.

As an example, consider two threads:

${T}_{1} :  :$ example $1\left( {l, h}\right)$ ;

${T}_{2} :  : l =  - l$

As before, $\mathrm{h}$ and 1 are inputs. Method example1 is described in Figure 3. Assume for simplicity that thread 1 invokes method example 1 atomically (e.g. in a synchronized block). Then $1 =  - 1$ gives 3 observables for thread schedule " ${T}_{1};{T}_{2}$ " but only 2 observables for " ${T}_{2};{T}_{1}$ ". Thus, there may be more than one program execution associated with an input, each with different observations, and only some executions give the worst leakage.

Note that the number of possible thread interleavings may be very large and ennumerating all of them may be infeasible. The problem can be addressed by partial order reduction (POR), supported by Java Pathfinder. POR exploits the commutativity of concurrently executed instructions, which result in the same state when executed in different orders.

Note also that the non-determinism introduced by multi-threading produces some "noise" that makes the leakage smaller [31], [32] and this is not captured by our analysis. However, if the attacker is allowed to observe the scheduling sequence, the leakage increases as in the example. Furthermore, our analysis produces a thread schedule which can be analyzed by developers for debugging and can thus be quite useful in practice.

Further there are more powerful notions of tree-like [33] and probabilistic schedulers [31], [32] that would allow us to compute more precise information on the leakage. We plan to explore them in depth for future work.

## B. Garbage Collection

The view of the adversaries as defined in this paper disregards the memory management handled by garbage collection whose execution is unpredictable and may affect the side channel measurements. Our analysis can be adapted to more refined cost models and different garbage collection policies by modifying and controlling the scheduler and the garbage collection inside JPF's custom JVM.

Nondeterminism and multi-threading also provide a powerful mechanism for studying the effects of the garbage collection, whose role is to find the unnecessary (garbage) objects and to remove them. We can view the garbage collector as a separate thread that interferes with the execution of the program analysis, affecting both its execution time and memory execution costs. Our tool implements a custom VM which allows to experiment with different garbage collection policies. We leave this for future work.

---

boolean check( byte[] secret, byte[] input)\{

	for ( int i = 0; i < SIZE; i++ )\{

		if ( secret[i] != input[i])\{

					return false;

		\}

		Thread.sleep(25L);

	\}

	return true;

\}

---

Fig. 5. Password Check

## VII. IMPLEMENTATION AND EXPERIENCE

We implemented our analysis in Symbolic PathFinder [23]. We use Z3 [18] (bit-vector theory) for SMT and Max-SMT solving. We implemented JPF listeners to monitor the bytecode instructions executed by the program, and to perform the analysis on the following possible type of side-channels: timing (by measuring execution time of each instruction according to a cost model), memory-usage (by computing the number of live heap objects along a path), network and file communication (by providing models for network and file interactions and computing number of bytes written to an output stream or file via methods write of java.io.OutputStream and java.io.FileOutputStream respectively). For experiments we used a simple timing model that allows easy comparison with ACSAC.

We evaluated our implementation based on two sets of experiments.

- We compared the single-run analysis (i.e. Algorithm MaxLeak, various configurations) with another symbolic approach, ACSAC [6] (described in detail below) and with brute-force enumeration (the baseline for our approach).

- We also evaluated the multi-run analysis (MaxLeak ${}_{k}$ ) for increasing number of $k$ . We compared the "full approach" ${\operatorname{MaxLeak}}_{k}$ , which computes the input sequence all at once using Max-SMT solving, with the "greedy" approach GreedyLeak (described in Section V-A) which computes the inputs one by one.

We analyzed a set of Java examples for the different side channels described above. Here we describe in detail the analysis of two representative examples: password checking and cryptographic functions (which we could convert easily into $\mathrm{C}$ allowing a comparison with $\mathrm{{ACSAC}}$ ).

All the experiments were run on a standard MacBook Pro with 2.2 GHz Intel Core i7 and 16 GB 1600 MHz DDR3.

## A. Examples

Timing channel in password check Fig. 5 shows the code of a simple segmented password checking program. Here secret represents the "high" value and input is the "low" value; both secret and input have the same SIZE (i.e. the same length as strings). The code checks the password and the input character by character and returns false as soon a mismatching character is found. This implementation is insecure against an attacker measuring the time taken by the method to return "true" or "false". We analyzed this example for different values of SIZE and element range.

Timing channel in cryptographic functions Fig. 6 shows the implementation of fast modular exponentiation [27] - an operation that is typically found in asymmetric cryptographic algorithms such as RSA used by modern computers to encrypt and decrypt messages. Here $e$ is the secret, num is the public input, and $\mathrm{m}$ is a constant (the product of two prime numbers). Method modPow1 iterates over the bits in the secret e (the while loop) and performs different computations based on whether the bit is 1 or 0 . The timing channels in this kind of functions result from the fact that modular multiplication is not constant time: for some operands it takes longer than for others (because a so-called extra reduction step). With the reduction step one gets a subtler dependency between low input num, key e, and timing. By varying the low input, an attacker can in some circumstances guess the key [34]. To study this phenomenon, we modified the algorithm to include a simple reduction step (see Fig. 7; second if statement has a dependency on low input that can be exploited for an attack). Other more involved methods are treated similarly. We analyzed the program for different values of the modulo $\mathrm{m}$ (where num is bound by $\mathrm{m}$ ) and different $\mathrm{e}$ lengths.

While the password check involves only simple computations, the analysis of modular exponentiation results in nonlinear constraints that are hard to solve with existing constraint solvers, and hence it is good to stress-test our proposed technique.

## B. MaxLeak (default) vs. MaxLeak (No Solver)

We note that in our approach there is some redundancy between the solving performed by SPF and the solving performed by Max-SMT. We therefore compared MaxLeak with two configurations: running SPF with and without the solver. With the first option (default), SPF uses constraint solving to rule out infeasible PCs, so only the feasible PCs are used in the Max-SMT calculation. With the second option (No Solver), SPF performs no solving, and collects all the PCs (including the infeasible ones) and sends them to Max-SMT for solving (which implicitly also rules out infeasible PCs).

### C.The ACSAC [6] approach

Most of the previous work on automated QIF (Section VIII) performs the analysis assuming the "low" input is given. While this is a sound assumption, since the low input is controlled or at least is known by the attacker, those techniques cannot synthesize the low inputs that maximize the leakage. An exception is the work in [6]: the technique is capable, in the case of a single run, to determine if the leakage is at least $k$ bits, and provide a low input that makes the program leak $k$ bits. As such ACSAC [6] is a good candidate to benchmark the performance of our approach for the single run analysis. At a high level, ACSAC computes the self-composition of $k$ copies of the program, and computes an assertion that these $k$ copies cannot create $k$ different observables. The bounded model checker CBMC [35] is then used to verify this assertion.

To perform the comparison with ACSAC we manually translated the programs in $\mathrm{C}$ code (the input language of CBMC) and instrumented the code by adding a time variable to simulate an adversary observing the timing channel (as described in Section III). The values of the time variable at the end of the program constitute the "observables" for the ACSAC approach. Note also that CBMC uses a different solver than Z3, which is better for bitvectors [35].

For ACSAC we report the time it takes in the last step to validate the assertion (note that in reality ACSAC performs an iterative approach, for an increasing number of observables, until the assertion becomes valid, so the overall time is higher than the one we report).

## D. Results and Discussion

The results of the single-run experiments are shown in Figures 8 and 9, while the results for the multi-run experiments are shown in Figures 10 and 11.

In the tables, $k$ is the number of steps (for the multi-run analysis), maxObs is the maximum number of observables reported by Max-SMT, #PCs is the number of PCs reported by SPF, #Obs is the number of clauses, time SPF is time to run SPF in seconds, and time Max-SMT is the time to run Max-SMT in seconds. A "-" means analysis timed out in 1 hour.

For single-run, MaxLeak (No Solver) performs better than MaxLeak(default). The time for running SPF is much smaller (since it uses no solving) but the number of generated paths may be very large (since it also includes infeasible paths) up to the point that Max-SMT can no longer solve the generated clauses (last line in Figure 9).

The results also show that ACSAC does not scale well for the single-run analysis ${}^{1}$ . The reason is that for a single run, ACSAC requires the composition of NObs copies of the program to validate the assertion (where ${NObs}$ is the number of possible observations in one run). In contrast, MaxLeak uses only one copy of the program for the single-run analysis. Thus, although we use different tools and different solvers for the comparison, we believe there is an inherent complexity problem with the ACSAC approach (supported by our experiments).

For multi-run analysis, the results show, as expected, that MaxLeak is more expensive than GreedyLeak, both in the number of PCs generated and the analysis time (where the Max-SMT solving time is dominant). On the other hand GreedyLeak does not always returns the maximum leakage: Fig. 11 shows the number of observables returned by

---

${}^{1}$ The time-out at small configurations in Figure 9 may be due to some hard to solve constraints that are unsat at smaller configurations but become sat when $m$ is larger.

---

---

int modPowl( int num, int e, int m)\{

	int s = 1, y = num, res = 0 ;

	while $\left( {\mathrm{e} > 0}\right) \{$

		if (e % 2 == 1) \{

			res $= \left( {\mathrm{s} * \mathrm{y}}\right) \% \mathrm{m}$ ;

		\} else \{

			res = s ;

		\}

			= (res * res) % m;

		e $l = 2$ ;

	\}

	return res ;

\}

---

Fig. 6. Modular Exponentiation

---

int modPow2( int num, int e, int m )\{

	int s = 1, y = num, res = 0 ;

	while $\left( {\mathrm{e} > 0}\right) \{$

		if (e % 2 == 1) \{

			// reduction :

			int tmp = s * y ;

				if $\left( {{tmp} > m}\right) \{$

				tmp $=$ tmp $- \mathrm{m}$ ;

			\}

			res $=$ tmp $\% \mathrm{\;m}$ ;

		\} else \{

				res = s;

		\}

		s = (res * res) % m;

		e $l = 2$ ;

	\}

	return res ;

\}

---

Fig. 7. Simple Reduction GreedyLeak vs MaxLeak or brute force, when they can complete. Adding a backtracking mechanism to GreedyLeak could address the problem.

<table><tr><td rowspan="2">SIZE</td><td rowspan="2">maxObs</td><td colspan="4">MaxLeak (default)</td><td colspan="4">MaxLeak (No solver)</td><td rowspan="2">ACSAC</td></tr><tr><td>#PC</td><td>#Obs</td><td>time SPF</td><td>time Max-SMT</td><td>#PC</td><td>#Obs</td><td>time SPF</td><td>time Max-SMT</td></tr><tr><td>10</td><td>11</td><td>11</td><td>11</td><td>0.561</td><td>0.046</td><td>11</td><td>11</td><td>0.333</td><td>0.047</td><td>2m33.349</td></tr><tr><td>50</td><td>51</td><td>51</td><td>51</td><td>8.662</td><td>4.958</td><td>51</td><td>51</td><td>0.566</td><td>5.07</td><td>-</td></tr><tr><td>100</td><td>101</td><td>101</td><td>101</td><td>59.852</td><td>36.061</td><td>101</td><td>101</td><td>0.932</td><td>36.983</td><td>-</td></tr><tr><td>200</td><td>201</td><td>201</td><td>201</td><td>7m50.156</td><td>16m29.761</td><td>201</td><td>201</td><td>3.086</td><td>16m37.846</td><td>-</td></tr></table>

Fig. 8. Single-run analysis of password check (range 1..62). Brute force times out in all configurations

<table><tr><td rowspan="2">Modulo</td><td rowspan="2">Len</td><td rowspan="2">maxObs</td><td colspan="4">MaxLeak (default)</td><td colspan="4">MaxLeak (No solver)</td><td rowspan="2">ACSAC</td><td rowspan="2">BruteForce</td></tr><tr><td>#PC</td><td>#Obs</td><td>time SPF</td><td>time Max-SMT</td><td>#PC</td><td>#Obs</td><td>time SPF</td><td>time Max-SMT</td></tr><tr><td rowspan="5">1717</td><td>3</td><td>6</td><td>13</td><td>6</td><td>8.830</td><td>0.483</td><td>40</td><td>9</td><td>0.417</td><td>0.763</td><td>1 m22.537</td><td>0.103</td></tr><tr><td>4</td><td>9</td><td>38</td><td>9</td><td>1m9.719</td><td>1.675</td><td>121</td><td>12</td><td>0.569</td><td>4.264</td><td>-</td><td>0.099</td></tr><tr><td>5</td><td>12</td><td>107</td><td>12</td><td>6m10.376</td><td>7.585</td><td>364</td><td>15</td><td>0.899</td><td>27.448</td><td>-</td><td>0.097</td></tr><tr><td>6</td><td>15</td><td>285</td><td>15</td><td>27m26.593</td><td>34.665</td><td>1093</td><td>18</td><td>1.967</td><td>3m59.985</td><td>-</td><td>0.110</td></tr><tr><td>7</td><td>18</td><td>-</td><td>-</td><td>-</td><td>-</td><td>3280</td><td>21</td><td>6.367</td><td>${43}\mathrm{\;m}{5.840}$</td><td>-</td><td>0.122</td></tr><tr><td rowspan="5">834443</td><td>3</td><td>6</td><td>13</td><td>6</td><td>4.810</td><td>0.398</td><td>40</td><td>9</td><td>0.421</td><td>0.811</td><td>-</td><td>0.328</td></tr><tr><td>4</td><td>9</td><td>40</td><td>9</td><td>24.222</td><td>2.151</td><td>121</td><td>12</td><td>0.557</td><td>5.491</td><td>-</td><td>0.661</td></tr><tr><td>5</td><td>12</td><td>121</td><td>12</td><td>$1\mathrm{m}{59.615}$</td><td>10.387</td><td>364</td><td>15</td><td>0.866</td><td>25.362</td><td>-</td><td>1.621</td></tr><tr><td>6</td><td>15</td><td>364</td><td>15</td><td>8m50.549</td><td>1 m12.780</td><td>1093</td><td>18</td><td>1.997</td><td>$3\mathrm{\;m}{50.015}$</td><td>-</td><td>4.402</td></tr><tr><td>7</td><td>18</td><td>1093</td><td>18</td><td>37m49.129</td><td>6m6.093</td><td>3280</td><td>21</td><td>6.487</td><td>41m20.159</td><td>-</td><td>9.739</td></tr><tr><td rowspan="5">1964903306</td><td>3</td><td>6</td><td>13</td><td>6</td><td>5.138</td><td>0.509</td><td>40</td><td>9</td><td>0.436</td><td>1.010</td><td>5.211</td><td>6m52.263</td></tr><tr><td>4</td><td>9</td><td>40</td><td>9</td><td>50.067</td><td>3.119</td><td>121</td><td>12</td><td>0.604</td><td>5.525</td><td>1m5.912</td><td>18m19.167</td></tr><tr><td>5</td><td>12</td><td>121</td><td>12</td><td>7m58.086</td><td>1m5.797</td><td>364</td><td>15</td><td>0.980</td><td>1m0.471</td><td>10m47.589</td><td>47m18.512</td></tr><tr><td>6</td><td>15</td><td>-</td><td>-</td><td>-</td><td>-</td><td>1093</td><td>18</td><td>2.002</td><td>31m42.575</td><td>-</td><td>-</td></tr><tr><td>7</td><td>18</td><td/><td>-</td><td>-</td><td>-</td><td>3280</td><td>21</td><td>8.455</td><td>-</td><td>-</td><td>-</td></tr></table>

Fig. 9. Single-run analysis of modPow2 (Len is e's length in bits).

The "no solver" option performs well for the password check but for modPow this option generates so many paths that we could not obtain any result from Max-SMT. This may be due to some bottleneck in the front-end of the solver which we hope to solve in the future. We are talking to the Z3 developers to explore a better integration between the Z3 solving on the SPF side and Max-SMT.

As expected, the brute-force approach performs well for small configurations, but it becomes intractable for larger sizes. In our experiments with modulo exponentiation, brute force did outperform MaxLeak in single-run analysis for small values of $\mathrm{m}$ . However, for a large $\mathrm{m}$ , which is often the case in cryptographic systems, MaxLeak outperformed brute force. This is even clearer in the multi-run analysis when brute force failed to synthesize any 3-run attacks, even for a small value of $\mathrm{m}$ .

Nevertheless, all compared approaches suffer from scala-bility issues as the length of the secret increases. However, we noticed that the experiments expose some regularity in the results. For example, for single-run, the maximum number of observables for the password check is ${SIZE} + 1$ while for modPow it is $3 * \left( {\text{Length} - 1}\right)$ . This suggests that one can extrapolate from these results and get the vulnerability for larger input configurations, even if they can not be analyzed directly using symbolic execution. We plan to investigate this further.

<table><tr><td rowspan="2">RANGE</td><td rowspan="2">SIZE</td><td rowspan="2">$\mathrm{k}$</td><td colspan="5">${\operatorname{MaxLeak}}_{k}$ (No solver)</td><td colspan="5">GreedyLeak ${}_{k}$ (No solver)</td></tr><tr><td>#PC</td><td>#Obs</td><td>maxObs</td><td>time SPF</td><td>time Max-SMT</td><td>#PC</td><td>#Obs</td><td>maxObs</td><td>time SPF</td><td>time Max-SMT</td></tr><tr><td rowspan="8">2</td><td>2</td><td>2</td><td>9</td><td>9</td><td>4</td><td>0.652</td><td>0.042</td><td>9</td><td>9</td><td>4</td><td>0.104</td><td>0.030</td></tr><tr><td rowspan="3">3</td><td>2</td><td>16</td><td>16</td><td>6</td><td>0.685</td><td>0.115</td><td>16</td><td>16</td><td>6</td><td>0.113</td><td>0.088</td></tr><tr><td>3</td><td>64</td><td>64</td><td>7</td><td>0.736</td><td>2.797</td><td>64</td><td>64</td><td>7</td><td>0.151</td><td>1.154</td></tr><tr><td>4</td><td>256</td><td>256</td><td>8</td><td>1.008</td><td>1m33.259</td><td>256</td><td>256</td><td>8</td><td>0.202</td><td>15.425</td></tr><tr><td rowspan="4">4</td><td>2</td><td>25</td><td>25</td><td>8</td><td>0.695</td><td>0.382</td><td>25</td><td>25</td><td>8</td><td>0.154</td><td>0.306</td></tr><tr><td>3</td><td>125</td><td>125</td><td>10</td><td>0.884</td><td>18.147</td><td>125</td><td>125</td><td>10</td><td>0.211</td><td>9.495</td></tr><tr><td>4</td><td>625</td><td>625</td><td>12</td><td>1.156</td><td>17m41.007</td><td>625</td><td>625</td><td>12</td><td>0.404</td><td>3m28.896</td></tr><tr><td>5</td><td>3125</td><td>3125</td><td>-</td><td>2.555</td><td>-</td><td>3125</td><td>3125</td><td>-</td><td>1.321</td><td>-</td></tr><tr><td rowspan="10">3</td><td rowspan="5">2</td><td>2</td><td>9</td><td>9</td><td>5</td><td>0.680</td><td>0.036</td><td>9</td><td>9</td><td>5</td><td>0.099</td><td>0.031</td></tr><tr><td>3</td><td>27</td><td>27</td><td>6</td><td>0.693</td><td>0.272</td><td>27</td><td>27</td><td>6</td><td>0.114</td><td>0.140</td></tr><tr><td>4</td><td>81</td><td>81</td><td>7</td><td>0.809</td><td>4.255</td><td>81</td><td>81</td><td>7</td><td>0.130</td><td>0.593</td></tr><tr><td>5</td><td>243</td><td>243</td><td>8</td><td>0.937</td><td>1 m5.058</td><td>243</td><td>243</td><td>8</td><td>0.182</td><td>4.867</td></tr><tr><td>6</td><td>729</td><td>729</td><td>9</td><td>1.229</td><td>13m9.257</td><td>729</td><td>729</td><td>9</td><td>0.307</td><td>57.014</td></tr><tr><td rowspan="5">3</td><td>2</td><td>16</td><td>16</td><td>7</td><td>0.678</td><td>0.115</td><td>16</td><td>16</td><td>7</td><td>0.111</td><td>0.088</td></tr><tr><td>3</td><td>64</td><td>64</td><td>9</td><td>0.761</td><td>2.464</td><td>64</td><td>64</td><td>9</td><td>0.147</td><td>1.304</td></tr><tr><td>4</td><td>256</td><td>256</td><td>11</td><td>0.951</td><td>1 m6.886</td><td>256</td><td>256</td><td>11</td><td>0.222</td><td>28.445</td></tr><tr><td>5</td><td>1024</td><td>1024</td><td>13</td><td>1.400</td><td>33m31.888</td><td>1024</td><td>1024</td><td>13</td><td>0.441</td><td>5m52.209</td></tr><tr><td>6</td><td>4096</td><td>4096</td><td>-</td><td>2.837</td><td>-</td><td>4096</td><td>4096</td><td>-</td><td>1.417</td><td>-</td></tr></table>

Fig. 10. Multi-run analysis of password check. Brute force takes a couple of seconds.

<table><tr><td rowspan="2">Modulo</td><td rowspan="2">Len</td><td rowspan="2">$\mathrm{k}$</td><td colspan="5">${\text{MaxLeak}}_{k}$ (default)</td><td colspan="5">GreedyLeak ${}_{k}$ (default)</td><td colspan="2">BruteForce</td></tr><tr><td>#PC</td><td>#Obs</td><td>maxObs</td><td>time SPF</td><td>time Max-SMT</td><td>#PC</td><td>#Obs</td><td>maxObs</td><td>time SPF</td><td>time Max-SMT</td><td>maxObs</td><td>time</td></tr><tr><td rowspan="10">1717</td><td>3</td><td>2</td><td>31</td><td>14</td><td>7</td><td>1m49.327</td><td>6.101</td><td>13</td><td>10</td><td>7</td><td>11.072</td><td>3.901</td><td>7</td><td>0.144</td></tr><tr><td rowspan="2">4</td><td>2</td><td>128</td><td>29</td><td>15</td><td>17m28.882</td><td>47.972</td><td>38</td><td>19</td><td>14</td><td>$1\mathrm{m}{15.528}$</td><td>6.614</td><td>15</td><td>4.792</td></tr><tr><td>3</td><td/><td/><td/><td/><td/><td>38</td><td>30</td><td>15</td><td>1 m46.254</td><td>18.051</td><td/><td/></tr><tr><td rowspan="3">5</td><td>2</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>107</td><td>31</td><td>24</td><td>8m11.855</td><td>13.077</td><td>27</td><td>11.996</td></tr><tr><td>3</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>107</td><td>64</td><td>29</td><td>9m32.251</td><td>1 m16.879</td><td>-</td><td>-</td></tr><tr><td>4</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>107</td><td>75</td><td>31</td><td>9m49.246</td><td>1 m16.903</td><td>-</td><td>-</td></tr><tr><td rowspan="4">6</td><td>2</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>285</td><td>46</td><td>35</td><td>38m05.420</td><td>51.766</td><td>40</td><td>28.820</td></tr><tr><td>3</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>285</td><td>108</td><td>52</td><td>41m14.786</td><td>3m34.222</td><td>-</td><td>-</td></tr><tr><td>4</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>285</td><td>157</td><td>59</td><td>${43}\mathrm{\;m}{48.355}$</td><td>3m36.526</td><td>-</td><td>-</td></tr><tr><td>5</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>285</td><td>178</td><td>63</td><td>46m08.591</td><td>3m55.498</td><td>-</td><td>-</td></tr><tr><td rowspan="6">834443</td><td>3</td><td>2</td><td>31</td><td>14</td><td>7</td><td>40.518</td><td>2.067</td><td>13</td><td>11</td><td>7</td><td>8.078</td><td>0.384</td><td>-</td><td>-</td></tr><tr><td rowspan="2">4</td><td>2</td><td>156</td><td>29</td><td>15</td><td>7m04.709</td><td>36m49.891</td><td>40</td><td>18</td><td>14</td><td>45.272</td><td>11.351</td><td>-</td><td>-</td></tr><tr><td>3</td><td/><td/><td/><td/><td/><td>40</td><td>26</td><td>15</td><td>1m01.474</td><td>4.973</td><td>-</td><td>-</td></tr><tr><td rowspan="2">5</td><td>2</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>121</td><td>32</td><td>26</td><td>3m42.725</td><td>1m53.156</td><td>-</td><td>-</td></tr><tr><td>3</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>121</td><td>58</td><td>-</td><td>4m34.153</td><td>-</td><td>-</td><td>-</td></tr><tr><td>6</td><td>2</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>364</td><td>55</td><td>-</td><td>15m20.872</td><td>-</td><td>-</td><td>-</td></tr><tr><td rowspan="4">1964903306</td><td>3</td><td>2</td><td>31</td><td>14</td><td>7</td><td>47.246</td><td>2.945</td><td>13</td><td>9</td><td>7</td><td>8.276</td><td>1.183</td><td>-</td><td>-</td></tr><tr><td rowspan="2">4</td><td>2</td><td>156</td><td>29</td><td>-</td><td>${15}\mathrm{\;m}{29.818}$</td><td>-</td><td>40</td><td>20</td><td>14</td><td>1m13.667</td><td>32.551</td><td>-</td><td>-</td></tr><tr><td>3</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>40</td><td>29</td><td>-</td><td>$1\mathrm{m}{15.041}$</td><td>-</td><td>-</td><td>-</td></tr><tr><td>5</td><td>2</td><td>-</td><td>-</td><td>-</td><td>-</td><td>-</td><td>121</td><td>31</td><td>-</td><td>${12}\mathrm{\;m}{18.071}$</td><td>-</td><td>-</td><td>-</td></tr></table>

Fig. 11. Multi-run analysis of modPow2.

Note also that the password check is a special case where the choice of the low input in a single run is irrelevant. Therefore the results for a single run are not very illuminating. However the results (both for single- and multi-run analysis) do support the fact that our symbolic techniques perform well when only simple linear constraints are involved, and can potentially scale to large programs. Note also that in the case of linear constraints, other solvers can perform much better than Z3bitvector. However, solvers struggle in the presence of non-linear constraints (as for the modPow example).

We also performed some preliminary experiments with an adaptive greedy approach (see Section V-A) which creates a new Max-SMT problem with each new observation made. Our preliminary results indicate that the adaptive approach could scale much better than the non-adaptive one: although we need to solve more Max-SMT problems than in the non-adaptive case, each problem is smaller and therefore easier to solve. Furthermore adaptive strategies may result in smaller "attack" sequences and may be more appropriate for examples such as the password check (e.g. to model an attack that guesses the password character by character). We plan to explore this topic in depth in the future.

## E. Security relevance of experiments

Let us give some intuition on what these experiments mean from a security point of view.

Consider the password experiments shown in Figure 10 and consider for example the case: range 2, size 3, i.e. we have a 3 bits secret. For 3 runs $\left( {\mathrm{k} = 3}\right)$ the results indicate 7 maxObs hence the channel capacity is ${\log }_{2}\left( 7\right)  \simeq  {2.8}$ bits which means that all the secret is at most leaked in the three runs. It is easy to see that this is a tight bound.

Similarly, consider the modulus exponentiation in Figure 11 with modulus 1717 and a 6 bits secret; then in 5 runs using GreedyLeak we found ${63}\operatorname{maxObs}$ i.e. at most ${5.9}\left( { = {\log }_{2}\left( {63}\right) }\right)$ bits can be leaked, which is basically all the secret. This is also a tight bound, which is rather surprising by looking at the code.

It is in fact easy to demonstrate that the bounds obtained by our analysis are tight: by proposition 1 the indistinguishability relation under $\left\langle  {{L}_{1},{L}_{2},..{L}_{k}}\right\rangle$ forms an equivalence relation on the secret values, and each observable represents an equivalence class on the secret. There are 63 observables and 63 secrets (this is because the secret cannot take the value 0 ), which implies each class has only one element, which means that each observables is associated to a unique secret, and so all the secret is leaked.

Note also that the GreedyLeak ${}_{k}$ approach is capable of obtaining tight bounds on the leakage, but, as expected, the attack sequence may be larger than for MaxLeak ${}_{k}$ . For example, in Figure 11, modulus 1717 and password length 4 bits, the greedy approach gives 3 runs to obtain the maximum number of observables (15) while the same information can be obtained only with two runs (shown with ${\operatorname{MaxLeak}}_{k}$ ).

Finally note that Max-SMT also generates the concrete solutions for the low input sequence, showing the developers how to precisely obtain the program executions that lead to maximum leakage.

## VIII. RELATED WORK

Side-channel attacks have been well studied in the literature [10], [1], [27] however there are few automated approaches that are able to quantify information leakage over multiple runs [10], [36].

Köpf and Basin [10] developed an automated approach for multi-run analysis in a more general setting, addressing adaptive attacks. However the technique is based on an enumeration algorithm (doubly-exponential in the number of attack steps); they also present a greedy heuristic to compute the remaining entropy of the secret after $k$ steps in the context of adaptive attacks. We propose here a symbolic approach that leverages Max-SMT solving to avoid an explicit search of best attacks. Our experiments show the technique has merits compared with brute-force enumeration, indicating its potential for synthesizing adaptive attacks as well. We leave this for future work.

Heusser and Malacaria [6] use the model checker CBMC to determine the public input that results in maximal leakage in the more general context of quantitative flow analysis. The technique requires an $n$ -step composition of the program, where $\mathrm{n}$ is the number of different observations one can make in one run of the program, and does not address multi-run analysis. Our technique can determine the same information from the analysis of only one program run but it is limited to the specific context of side-channel analysis. We showed experimentally that the Max-SMT approach can be much better in practice.

Backes et al. [37] use symbolic techniques and model counting for quantitative information flow analysis but they assume a-priori knowledge of the public inputs. Previous work [12], [15], [16] has used symbolic execution for quantitative flow analysis in the simpler context of single-run attacks where only upper-bounds on the leakage are computed. That work does not compute the public user input that maximizes the leakage, it does not address multi-run attacks and it does not use probabilistic symbolic execution for computing information theoretic metrics - all our contributions here.

Mardziel et al. [36] generalizes the work of Köpf and Basin [10] by considering probabilistic systems to account for secrets that change over time. They use probabilistic programming to implement a model of information flow for probabilistic, interactive systems with adaptive adversaries and to compute the leakage. This suggests a possible connection with the probabilistic capabilities of SPF that we plan to explore in future work.

Cache side channels are studied extensively in [13], [14] although not in a multi-run setting. Our tool is built on top of JPF, a custom VM with its own memory model, and can thus analyze some memory side-channels but we do not currently model architecturespecific cacheside channels as in [13], [14] which we leave for future work.

## IX. Conclusion

We described a symbolic execution approach to side-channels detection and quantification. We defined a new application of Max-SMT solving to identify the low input triggering the most vulnerable behavior of the program and analyze leakage of multi-run attacks. We implemented the analysis in Symbolic PathFinder and showed its merit on several examples.

We made few attempts to optimizing our implementation and at present it does not scale when the number of time steps is large. However we note that the research area of Max-SMT solving is still very young and we are optimistic that it will increase in the future attaining the level of maturity of SMT solving that has grown explosively in the past decade. Further our analysis may benefit from a tighter integration between the symbolic exploration and the Max-SMT solving and from distributing the analysis, e.g. by creating parallel versions of the symbolic execution engine and creating a new parallel job with each new observation made. Other areas for future work include investigating the use of qCoral [8] for quantifying solution spaces over non-linear constraints and extending our analysis for leakage computation in the presence of noisy observations.

## ACKNOWLEDGMENT

We are grateful to Nikolaj Bjorner for answering many of our questions about $\mathrm{Z}3$ and Max-SMT. We also thank the anonymous reviewers for their constructive comments. This material is based on research sponsored by DARPA under agreement number FA8750-15-2-0087. The U.S. Government is authorized to reproduce and distribute reprints for Governmental purposes notwithstanding any copyright notation thereon. Malacaria research was supported by EPSRC grant EP/K032011/1.

## REFERENCES

[1] D. Brumley and D. Boneh, "Remote Timing Attacks Are Practical," in Proceedings of the 12th Conference on USENIX Security Symposium - Volume 12, SSYM'03, (Berkeley, CA, USA), pp. 1-1, USENIX Association, 2003.

[2] J. Kelsey, "Compression and information leakage of plaintext," in Revised Papers from the 9th International Workshop on Fast Software Encryption, FSE '02, (London, UK, UK), pp. 263-276, Springer-Verlag, 2002.

[3] D. Clark, S. Hunt, and P. Malacaria, "A static analysis for quantifying information flow in a simple imperative language," J. Comput. Secur., vol. 15, pp. 321-371, Aug. 2007.

[4] P. Malacaria, "Assessing security threats of looping constructs," in Proceedings of the 34th annual ACM SIGPLAN-SIGACT symposium on Principles of programming languages, POPL '07, (New York, NY, USA), pp. 225-235, ACM, 2007.

[5] R. Nieuwenhuis and A. Oliveras, "On SAT Modulo Theories and Optimization Problems," in Proceedings of the 9th International Conference on Theory and Applications of Satisfability Testing, SAT'06, (Berlin, Heidelberg), pp. 156-169, Springer-Verlag, 2006.

[6] J. Heusser and P. Malacaria, "Quantifying information leaks in software," in Proceedings of the 26th Annual Computer Security Applications Conference, ACSAC '10, (New York, NY, USA), pp. 261-269, ACM, 2010.

[7] A. Filieri, C. S. Păsăreanu, and W. Visser, "Reliability analysis in symbolic pathfinder," in Proceedings of the 2013 International Conference on Software Engineering, ICSE '13, (Piscataway, NJ, USA), pp. 622- 631, IEEE Press, 2013.

[8] M. Borges, A. Filieri, M. d'Amorim, C. S. Păsăreanu, and W. Visser, "Compositional Solution Space Quantification for Probabilistic Software Analysis," in Proceedings of the 35th ACM SIGPLAN Conference on Programming Language Design and Implementation, PLDI '14, (New York, NY, USA), pp. 123-132, ACM, 2014.

[9] G. Smith, "On the Foundations of Quantitative Information Flow," in Proceedings of the 12th International Conference on Foundations of Software Science and Computational Structures, FOSSACS '09, (Berlin, Heidelberg), pp. 288-302, Springer-Verlag, 2009.

[10] B. Köpf and D. Basin, "An Information-theoretic Model for Adaptive Side-channel Attacks," in Proceedings of the 14th ACM Conference on Computer and Communications Security, CCS '07, (New York, NY, USA), pp. 286-296, ACM, 2007.

[11] P. Malacaria and H. Chen, "Lagrange multipliers and maximum information leakage in different observational models," in Proceedings of the third ACM SIGPLAN workshop on Programming languages and analysis for security, PLAS '08, (New York, NY, USA), pp. 135-146, ACM, 2008.

[12] Q.-S. Phan, P. Malacaria, O. Tkachuk, and C. S. Păsăreanu, "Symbolic Quantitative Information Flow," SIGSOFT Softw. Eng. Notes, vol. 37, pp. 1-5, Nov. 2012.

[13] B. Köpf, L. Mauborgne, and M. Ochoa, "Automatic quantification of cache side-channels," in Proceedings of the 24th international conference on Computer Aided Verification, CAV'12, (Berlin, Heidelberg), pp. 564-580, Springer-Verlag, 2012.

[14] G. Doychev, D. Feld, B. Köpf, L. Mauborgne, and J. Reineke, "CacheAudit: A Tool for the Static Analysis of Cache Side Channels," in Proceedings of the 22Nd USENIX Conference on Security, SEC'13, (Berkeley, CA, USA), pp. 431-446, USENIX Association, 2013.

[15] Q.-S. Phan and P. Malacaria, "Abstract Model Counting: A Novel Approach for Quantification of Information Leaks," in Proceedings of the 9th ACM Symposium on Information, Computer and Communications Security, ASIA CCS '14, (New York, NY, USA), pp. 283-292, ACM, 2014.

[16] Q.-S. Phan, P. Malacaria, C. S. Păsăreanu, and M. d'Amorim, "Quantifying Information Leaks Using Reliability Analysis," in Proceedings of the 2014 International SPIN Symposium on Model Checking of Software, SPIN 2014, (New York, NY, USA), pp. 105-108, ACM, 2014.

[17] J. C. King, "Symbolic execution and program testing," Commun. ACM, vol. 19, pp. 385-394, July 1976.

[18] L. De Moura and N. Bjørner, "Z3: an efficient SMT solver," in Proceedings of the 14th international conference on Tools and algorithms for the construction and analysis of systems, TACAS'08, (Berlin, Heidelberg), pp. 337-340, Springer-Verlag, 2008.

[19] E. Bounimova, P. Godefroid, and D. Molnar, "Billions and Billions of Constraints: Whitebox Fuzz Testing in Production," in Proceedings of the 2013 International Conference on Software Engineering, ICSE '13, (Piscataway, NJ, USA), pp. 122-131, IEEE Press, 2013.

[20] T. Avgerinos, A. Rebert, S. K. Cha, and D. Brumley, "Enhancing Symbolic Execution with Veritesting," in Proceedings of the 36th International Conference on Software Engineering, ICSE 2014, (New York, NY, USA), pp. 1083-1094, ACM, 2014.

[21] L. Ciortea, C. Zamfir, S. Bucur, V. Chipounov, and G. Candea, "Cloud9: A Software Testing Service," SIGOPS Oper. Syst. Rev., vol. 43, pp. 5-10, Jan. 2010.

[22] "Java PathFinder." http://babelfish.arc.nasa.gov/trac/jpf/.

[23] C. S. Păsăreanu, W. Visser, D. Bushnell, J. Geldenhuys, P. Mehlitz, and N. Rungta, "Symbolic PathFinder: integrating symbolic execution with model checking for Java bytecode analysis," Automated Software Engineering, pp. 1-35, 2013.

[24] J. A. D. Loera, R. Hemmecke, J. Tauzer, and R. Yoshida, "Effective lattice point counting in rational convex polytopes," Journal of Symbolic Computation, vol. 38, no. 4, pp. 1273 - 1302, 2004. Symbolic Computation in Algebra and Geometry.

[25] L. De Moura and N. Bjørner, "Satisfiability modulo theories: introduction and applications," Commun. ACM, vol. 54, pp. 69-77, Sept. 2011.

[26] R. Karp, "Reducibility among combinatorial problems," in Complexity of Computer Computations (R. Miller, J. Thatcher, and J. Bohlinger, eds.), The IBM Research Symposia Series, pp. 85-103, Springer US, 1972.

[27] P. C. Kocher, "Timing Attacks on Implementations of Diffie-Hellman, RSA, DSS, and Other Systems," in Proceedings of the 16th Annual International Cryptology Conference on Advances in Cryptology, CRYPTO '96, (London, UK, UK), pp. 104-113, Springer-Verlag, 1996.

[28] A. Darvas, R. Hähnle, and D. Sands, "A theorem proving approach to analysis of secure information flow," in Proceedings of the Second international conference on Security in Pervasive Computing, SPC'05, (Berlin, Heidelberg), pp. 193-209, Springer-Verlag, 2005.

[29] G. Barthe, P. R. D'Argenio, and T. Rezk, "Secure Information Flow by Self-Composition," in Proceedings of the 17th IEEE workshop on Computer Security Foundations, CSFW '04, (Washington, DC, USA), IEEE Computer Society, 2004.

[30] P. Malacaria, "Algebraic foundations for quantitative information flow," Mathematical Structures in Computer Science, vol. 25, pp. 404-428, 2 2015.

[31] H. Chen and P. Malacaria, "The optimum leakage principle for analyzing multi-threaded programs," in Information Theoretic Security (K. Kurosawa, ed.), vol. 5973 of Lecture Notes in Computer Science, Springer Berlin Heidelberg, 2010.

[32] H. Chen and P. Malacaria, "Quantitative Analysis of Leakage for Multi-threaded Programs," in Proceedings of the 2007 Workshop on Programming Languages and Analysis for Security, PLAS '07, (New York, NY, USA), pp. 31-40, ACM, 2007.

[33] K. S. Luckow, C. S. Pasareanu, M. B. Dwyer, A. Filieri, and W. Visser, "Exact and approximate probabilistic symbolic execution for nondeterministic programs," in ACM/IEEE International Conference on Automated Software Engineering, ASE '14, Vasteras, Sweden - September 15 - 19, 2014, pp. 575-586, 2014.

[34] J.-F. Dhem, F. Koeune, P.-A. Leroux, P. Mestr, J.-J. Quisquater, and J.- L. Willems, "A practical implementation of the timing attack," in Smart Card Research and Applications (J.-J. Quisquater and B. Schneier, eds.), vol. 1820 of Lecture Notes in Computer Science, pp. 167-182, Springer Berlin Heidelberg, 2000.

[35] "CBMC." http://www.cprover.org/cbmc/.

[36] P. Mardziel, M. S. Alvim, M. W. Hicks, and M. R. Clarkson, "Quantifying information flow for dynamic secrets," in 2014 IEEE Symposium on Security and Privacy, SP 2014, Berkeley, CA, USA, May 18-21, 2014, pp. 540-555, 2014.

[37] M. Backes, B. Kopf, and A. Rybalchenko, "Automatic Discovery and Quantification of Information Leaks," in Proceedings of the 2009 30th IEEE Symposium on Security and Privacy, SP '09, (Washington, DC,

USA), pp. 141-153, IEEE Computer Society, 2009.