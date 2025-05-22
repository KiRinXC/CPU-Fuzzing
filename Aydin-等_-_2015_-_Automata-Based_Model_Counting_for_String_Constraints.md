# Automata-based Model Counting for String Constraints*

Abdulbaki Aydin, Lucas Bang, Tevfik Bultan

University of California, Santa Barbara

Abstract. Most common vulnerabilities in Web applications are due to string manipulation errors in input validation and sanitization code. String constraint solvers are essential components of program analysis techniques for detecting and repairing vulnerabilities that are due to string manipulation errors. For quantitative and probabilistic program analyses, checking the satisfiability of a constraint is not sufficient, and it is necessary to count the number of solutions. In this paper, we present a constraint solver that, given a string constraint, 1) constructs an automaton that accepts all solutions that satisfy the constraint, 2) generates a function that, given a length bound, gives the total number of solutions within that bound. Our approach relies on the observation that, using an automata-based constraint representation, model counting reduces to path counting, which can be solved precisely. We demonstrate the effectiveness of our approach on a large set of string constraints extracted from real-world web applications.

## 1 Introduction

Since many computer security vulnerabilities are due to errors in string manipulating code, string analysis has become an active research area in the last decade $\left\lbrack  {9,{39},{36},{17},{31},3,{12},{38}}\right\rbrack$ . Symbolic execution is a well-known automated bug detection technique which has been applied to vulnerability detection [28]. In order to apply symbolic execution to analysis of string manipulating programs, it is necessary to check satisfiability of string constraints [6]. Several string constraint solvers have been proposed in recent years to address this problem [18, ${21},{19},{40},{23},1,{24},{32}\rbrack$ .

There are two recent research directions that aim to extend symbolic execution beyond assertion checking. One of them is quantitative information flow, where the goal is to determine how much secret information is leaked from a given program $\left\lbrack  {{10},{26},{29},{27}}\right\rbrack$ , and another one is probabilistic symbolic execution where the goal is to compute probability of the success and failure paths in order to establish reliability of the given program $\left\lbrack  {{13},7}\right\rbrack$ . Interestingly, both of these approaches require the same basic extension to constraint solving: They require a model-counting constraint solver that not only determines if a constraint is satisfiable, but it also computes the number of satisfying instances.

---

* This material is based on research sponsored by NSF under grant CCF-1423623 and by DARPA under agreement number FA8750-15-2-0087. The U.S. Government is authorized to reproduce and distribute reprints for Governmental purposes notwithstanding any copyright notation thereon. The views and conclusions contained herein are those of the authors and should not be interpreted as necessarily representing the official policies or endorsements, either expressed or implied, of DARPA or the U.S. Government. Part of this research was conducted while Tevfik Bultan was visiting Koç University in Istanbul, Turkey, supported by a research fellowship from TÜBITAK under the BIDEB 2221 program.

---

In this paper, we present an automata-based model-counting technique for string constraints that consists of two main steps: 1) Given a string constraint and a variable, we construct an automaton that accepts all the string values for that variable for which the string constraint is satisfiable. 2) Given an automaton we generate a function that takes a length bound as input and returns the total number of strings that are accepted by the automaton that have a length that is less than or equal to the given bound.

Our constraint language can handle regular language membership queries, word equations that involve concatenation and replacement, and arithmetic constraints on string lengths. For a class of constraints that we call pseudo-relational, our approach gives the precise model-count. For constraints that are not in this class our approach computes an upper bound. We implemented a tool called Automata-Based model Counter for string constraints (ABC) using the approach we present in this paper. Our experiments demonstrate that $\mathrm{{ABC}}$ is effective and efficient when applied to thousands of string constraints extracted from real-world web applications.

Related Work: Our inspiration for this work was the recently proposed model-counting string constraint solver SMC [25]. Similar to SMC, we also utilize generating functions in model-counting. However, due to some significant differences in how we utilize generating functions, our approach is strictly more precise than the approach used in SMC. For example, SMC cannot determine the precise model count for a regular expression constraint such as $x \in  {\left( a \mid  b\right) }^{ * } \mid  {ab}$ , whereas our approach is precise for all regular expressions. More importantly, SMC cannot propagate string values across logical connectives which reduces its precision. For example, for a simple constraint such as $\left( {x \in  a \mid  b}\right)  \vee  \left( {x \in  a\left| b\right| c \mid  d}\right)$ SMC will generate a model-count range which consists of an upper bound of 6 and a lower bound of 2 , whereas our approach will generate the exact count which is 4 . Moreover, SMC always generates a lower bound of 0 for conjunctions that involve the same variable. So, the range generated for $\left( {x \in  a \mid  b}\right)  \land  \left( {x \in  a\left| b\right| c \mid  d}\right)$ would be 0 to 2 , whereas our approach generates the exact count which is 2 . The set of constraints we handle is also larger than the constraints that SMC can handle. In particular, we can handle constraints with replace operations which is common in server-side input sanitization code.

There has been significant amount of work on string constraint solving in recent years $\left\lbrack  {{18},{21},{19},{28},{15},{40},{23},1,{24},{32}}\right\rbrack$ . Some of these constraints solvers bound the string length $\left\lbrack  {{21},{28},{23}}\right\rbrack$ whereas our approach handles strings of arbitrary length. None of these string constraint solvers provide model-counting functionality. Our modal-counting constraint solver, ABC, builds on the automata-based string analysis tool Stranger $\left\lbrack  {{39},{36},{38}}\right\rbrack$ , which was determined to be the best in terms of precision and efficiency in a recent empirical study for evaluating string constraint solvers for symbolic execution of Java programs [20]. In addition to checking satisfiability, ABC also generates an automaton that accepts all possible solutions and provides model-counting capability. To the best of our knowledge, ABC is the only tool that supports all of these. In addition to enabling quantitative and probabilistic analysis by model counting, our constraint solver also enables automated program repair synthesis by generating a characterization of all solutions $\left\lbrack  {{37},2}\right\rbrack$ .

## 2 Automata Construction for String Constraints

In this section, we discuss how to construct automata for string constraints. Given a constraint and a variable, our goal is to construct an automaton that accepts all strings, which, when assigned as the value of the variable in the given constraint, results in a satisfiable constraint.

### 2.1 String Constraint Language

We define the set of string constraints using the following abstract grammar:

$$
F \rightarrow  C\left| {\neg F}\right| F \land  F \mid  F \vee  F \tag{1}
$$

$$
C \rightarrow  S \in  R \tag{2}
$$

$$
S = S \tag{3}
$$

$$
S = S \cdot  S \tag{4}
$$

$$
\text{LEN}\left( S\right) {On} \tag{5}
$$

$$
\text{LEN}\left( S\right) O\operatorname{LEN}\left( S\right)  \tag{6}
$$

$$
\text{CONTAINS}\left( {S, s}\right)  \tag{7}
$$

$$
\text{BEGINS}\left( {S, s}\right)  \tag{8}
$$

$$
\operatorname{ENDS}\left( {S, s}\right)  \tag{9}
$$

$$
n = \operatorname{INDEXOF}\left( {S, s}\right)  \tag{10}
$$

$$
S = \operatorname{REPLACE}\left( {S, s, s}\right)  \tag{11}
$$

$$
S \rightarrow  v \mid  s \tag{12}
$$

$$
R \rightarrow  s\left| \varepsilon \right| {RR}\left| R\right| R\left| {R}^{ * }\right|  \tag{13}
$$

$$
O \rightarrow   < \left|  = \right|  >  \tag{14}
$$

where $C$ denotes the basic constraints, $n$ denotes integer values, $s \in  {\sum }^{ * }$ denotes string values, $\varepsilon$ is the empty string, $v$ denotes string variables,. is the string concatenation operator, $\operatorname{LEN}\left( v\right)$ denotes the length of the string value that is assigned to variable $v$ , and the string functions are defined as follows:

---

- CONTAINS $\left( {v, s}\right)  \Leftrightarrow  \exists {s}_{1},{s}_{2} \in  {\sum }^{ * } : v = {s}_{1}s{s}_{2}$

- BEGINS $\left( {v, s}\right)  \Leftrightarrow  \exists {s}_{1} \in  {\sum }^{ * } : v = s{s}_{1}$

- $\operatorname{ENDS}\left( {v, s}\right)  \Leftrightarrow  \exists {s}_{1} \in  {\sum }^{ * } : v = {s}_{1}s$

$- n = \operatorname{INDEXOF}\left( {v, s}\right)  \Leftrightarrow  (\operatorname{CONTAINS}\left( {v, s}\right)  \land  \left( {\exists {s}_{1},{s}_{2} \in  {\sum }^{ * } : \operatorname{LEN}\left( {s}_{1}\right)  = n \land  v = }\right.$

	$\left. \left. {{s}_{1}s{s}_{2}}\right) \right)  \land  \left( {\forall i < n : \neg \left( {\exists {s}_{1},{s}_{2} \in  {\sum }^{ * } : \operatorname{LEN}\left( {s}_{1}\right)  = i \land  v = {s}_{1}s{s}_{2}}\right) }\right)  \vee$

	$\left( {\neg \operatorname{coNTAINS}\left( {v, s}\right)  \land  n =  - 1}\right)$

---

$- v = \operatorname{REPLACE}\left( {{v}^{\prime },{s}_{1},{s}_{2}}\right)  \Leftrightarrow  \left( {\exists {s}_{3},{s}_{4},{s}_{5} \in  {\sum }^{ * } : {v}^{\prime } = {s}_{3}{s}_{1}{s}_{4} \land  v = {s}_{3}{s}_{2}{s}_{5} \land  }\right.$

${s}_{5} = \operatorname{REPLACE}\left( {{s}_{4},{s}_{1},{s}_{2}}\right)  \land  \left( {\forall {s}_{6},{s}_{7} \in  {\sum }^{ * } : {v}^{\prime } = {s}_{6}{s}_{1}{s}_{7} \Rightarrow  \operatorname{LEN}\left( {s}_{6}\right)  \geq  \operatorname{LEN}\left( {s}_{3}\right) }\right) ) \vee$ $\left( {\neg \operatorname{coNTAINS}\left( {{v}^{\prime },{s}_{1}}\right)  \land  v = {v}^{\prime }}\right)$

and the definitions of these functions when the string variable $v$ is replaced with a string constant are similar.

Given a constraint $F$ , let ${V}_{F}$ denote the set of variables that appear in $F$ . Let $F\left\lbrack  {s/v}\right\rbrack$ denote the constraint that is obtained from $F$ by replacing all appearances of $v \in  {V}_{F}$ with the string constant $s$ . We define the truth set of the formula $F$ for variable $v$ as $\llbracket F, v\rrbracket  = \{ s \mid  F\left\lbrack  {s/v}\right\rbrack$ is satisfiable $\}$ .

We identify three classes of constraints: 1) Single-variable constraints are constructed using at most one string variable (i.e., ${V}_{F} = \{ v\}$ or ${V}_{F} = \varnothing$ ), they do not contain constraints of type (4), (6), and (11), and have a single variable on the left hand side of constraints of type (3). 2) Pseudo-relational constraints: are a set of constraints that we define in the next section, for which the truth sets are regular (i.e., each $\llbracket F, v\rrbracket$ is a regular set). 3) Relational constraints are the constraints that are not pseudo-relational constraints (truth sets of relational constraints can be non-regular).

### 2.2 Mapping Constraints to Automata

A Deterministic Finite Automaton (DFA) $A$ is a 5-tuple $\left( {Q,\sum ,\delta ,{q}_{0}, F}\right)$ , where $Q = \{ 1,2,\ldots , n\}$ is the set of $n$ states, $\sum$ is the input alphabet, $\delta  \subseteq  Q \times  Q \times  \sum$ is the state transition relation set, ${q}_{0} \in  Q$ is the initial state, and $F \subseteq  Q$ is the set of final, or accepting, states.

Given an automaton $A$ , let $\mathcal{L}\left( A\right)$ denote the set of strings accepted by $A$ . Given a constraint $F$ and a variable $v$ , our goal is to construct an automaton $A$ , such that $\mathcal{L}\left( A\right)  = \llbracket F, v\rrbracket$ .

Automata Construction for Single-Variable Constraints: Let us define an automata constructor function $\mathcal{A}$ such that, given a formula $F$ and a variable $v$ , $\mathcal{A}\left( {F, v}\right)$ is an automaton where $\mathcal{L}\left( {\mathcal{A}\left( {F, v}\right) }\right)  = \llbracket F, v\rrbracket$ . In this section we discuss how to implement the automata constructor function $\mathcal{A}$ .

Consider the following string constraint $F \equiv  \neg \left( {x \in  {\left( {01}\right) }^{ * }}\right)  \land  \operatorname{LEN}\left( x\right)  \geq  1$ over the alphabet $\sum  = \{ 0,1\}$ . Let us name the sub-constraints of $F$ as ${C}_{1} \equiv  x \in  {\left( {01}\right) }^{ * }$ , ${C}_{2} \equiv  \operatorname{LEN}\left( x\right)  \geq  1,{F}_{1} \equiv  \neg {C}_{1}$ , where $F \equiv  {F}_{1} \land  {C}_{2}$ . The automata construction algorithm starts from the basic constraints at the leaves of the syntax tree $\left( {C}_{1}\right.$ and ${C}_{2}$ ), and constructs the automata for them. Then it traverses the syntax tree towards the root by constructing an automaton for each node using the automata constructed for its children (where the automaton for ${F}_{1}$ is constructed using the automaton for ${C}_{1}$ and the automaton for $F$ is constructed using the automata for ${F}_{1}$ and ${C}_{2}$ ). Figure 1 demonstrates the automata construction algorithm on our running example.

Let $\mathcal{A}\left( {\sum }^{ * }\right) ,\mathcal{A}\left( {\sum }^{n}\right) ,\mathcal{A}\left( s\right)$ , and $\mathcal{A}\left( \varnothing \right)$ denote automata that accept the languages ${\sum }^{ * },{\sum }^{n},\{ s\}$ , and $\varnothing$ , respectively. We construct the automaton $\mathcal{A}\left( {F, v}\right)$ recursively on the structure of the single-variable constraint $F$ as follows:

- case ${V}_{F} = \varnothing$ (i.e., there are no variables in $F$ ): Evaluate the constraint $F$ . If $F \equiv$ true then $\mathcal{A}\left( {F, v}\right)  = \mathcal{A}\left( {\sum }^{ * }\right)$ , otherwise $\mathcal{A}\left( {F, v}\right)  = \mathcal{A}\left( \varnothing \right)$ .

![0196f760-828a-79a3-b5a0-90589a4fd3c5_4_406_281_989_486_0.jpg](images/0196f760-828a-79a3-b5a0-90589a4fd3c5_4_406_281_989_486_0.jpg)

Fig.1. (a) The syntax tree for the string constraint $\neg \left( {x \in  {\left( {01}\right) }^{ * }}\right)  \land  \operatorname{LEN}\left( x\right)  \geq  1$ and (b) the automata construction that traverses the syntax tree from the leaves towards the root.

- case $F \equiv  \neg {F}_{1} : \mathcal{A}\left( {F, v}\right)$ is constructed using $\mathcal{A}\left( {{F}_{1}, v}\right)$ and it is an automaton that accepts the complement language ${\sum }^{ * } - \mathcal{L}\left( {\mathcal{A}\left( {{F}_{1}, v}\right) }\right)$ .

- case $F \equiv  {F}_{1} \land  {F}_{2}$ or $F \equiv  {F}_{1} \vee  {F}_{2} : \mathcal{A}\left( {F, v}\right)$ is constructed using $\mathcal{A}\left( {{F}_{1}, v}\right)$ and $\mathcal{A}\left( {{F}_{2}, v}\right)$ using automata product, and it accepts the language $\mathcal{A}\left( {{F}_{1}, v}\right)  \cap  \mathcal{A}\left( {{F}_{2}, v}\right)$ or $\mathcal{A}\left( {{F}_{1}, v}\right)  \cup  \mathcal{A}\left( {{F}_{2}, v}\right)$ , respectively.

- case $F \equiv  v \in  R : \mathcal{A}\left( {F, v}\right)$ is constructed using regular expression to automata conversion algorithm and accepts all strings that match the regular expression $R$ .

- case $F \equiv  v = s : \mathcal{A}\left( {F, v}\right)  = \mathcal{A}\left( s\right)$ .

- case $F \equiv  \operatorname{LEN}\left( v\right)  = n : \mathcal{A}\left( {F, v}\right)  = \mathcal{A}\left( {\sum }^{n}\right)$ .

- case $F \equiv  \operatorname{LEN}\left( v\right)  < n : \mathcal{A}\left( {F, v}\right)$ is an automaton that accepts the language $\{ \varepsilon \}  \cup$ ${\mathit{Σ}}^{1} \cup  {\mathit{Σ}}^{2} \cup  \ldots  \cup  {\mathit{Σ}}^{n - 1}.$

- case $F \equiv  \operatorname{LEN}\left( v\right)  > n : \mathcal{A}\left( {F, v}\right)$ is constructed using $\mathcal{A}\left( {\sum }^{n + 1}\right)$ and $\mathcal{A}\left( {\sum }^{ * }\right)$ and then constructing an automaton that accepts the concatenation of those languages, i.e., ${\sum }^{n + 1}{\sum }^{ * }$ .

- case $F \equiv  \operatorname{CONTAINS}\left( {v, s}\right)  : \mathcal{A}\left( {F, v}\right)$ is an automaton that is constructed using $\mathcal{A}\left( {\sum }^{ * }\right)$ and $\mathcal{A}\left( s\right)$ and it accepts the language ${\sum }^{ * }s{\sum }^{ * }$ .

- case $F \equiv  \operatorname{BEGINS}\left( {v, s}\right)  : \mathcal{A}\left( {F, v}\right)$ is constructed using $\mathcal{A}\left( {\sum }^{ * }\right)$ and $\mathcal{A}\left( s\right)$ , and it accepts the language $s{\sum }^{ * }$ .

- case $F \equiv  \operatorname{ENDS}\left( {v, s}\right)  : \mathcal{A}\left( {F, v}\right)$ is constructed using $\mathcal{A}\left( {\sum }^{ * }\right)$ and $\mathcal{A}\left( s\right)$ , and it accepts the language ${\sum }^{ * }s$ .

- case $F \equiv  n = \operatorname{INDEXOF}\left( {v, s}\right)$ : Let ${L}_{i}$ denote the language ${\sum }^{i}s{\sum }^{ * }$ . Automata that accept the languages ${L}_{i}$ can be constructed using $\mathcal{A}\left( {\sum }^{i}\right) ,\mathcal{A}\left( s\right)$ , and $\mathcal{A}\left( {\sum }^{ * }\right)$ . Then $\mathcal{A}\left( {F, v}\right)$ is the automaton that accepts the language ${\sum }^{n}s{\sum }^{ * } - \left( {\{ \varepsilon \}  \cup  {L}_{1} \cup  {L}_{2} \cup  \ldots  \cup  }\right.$ $\left. {L}_{n - 1}\right)$ which can be constructed using $\mathcal{A}\left( {\sum }^{n}\right) ,\mathcal{A}\left( s\right) ,\mathcal{A}\left( {\sum }^{ * }\right)$ , and the automata that accept ${L}_{i}$ .

Pseudo-Relational Constraints: Pseudo-relational constraints are multi-variable constraints. Note that, using multiple variables, one can specify constraints with non-regular truth sets. For example, given the constraint $F \equiv  x = y.y,\llbracket F, x\rrbracket$ is not a regular set, so we cannot construct an automaton precisely recognizing its truth set. Below, we define a class of constraints called pseudo-relational constraints for which $\llbracket F, v\rrbracket$ is regular.

We assume that constraint $F$ is converted to DNF form where $F \equiv  { \vee  }_{i = 1}^{n}{F}_{i}$ , ${F}_{i} \equiv   \land  {}_{j = 1}^{m}{C}_{ij}$ , and each ${C}_{ij}$ is either a basic constraint or negation of a basic constraint. The constraint $F$ is pseudo-relational if each ${F}_{i}$ is pseudo-relational.

Given $F \equiv  {C}_{1} \land  {C}_{2} \land  \ldots  \land  {C}_{n}$ , where each ${C}_{i}$ is either a basic constraint or negation of a basic constraint, for each ${C}_{i}$ , let ${V}_{{C}_{i}}$ denote the set of variables that appear in ${C}_{i}$ . We call $F$ pseudo-relational if the following conditions hold:

1. Each variable $v \in  {V}_{F}$ appears in each ${C}_{i}$ at most once.

2. There is only one variable, $v \in  {V}_{F}$ , that appears in more than one constraint ${C}_{i}$ where $v \in  {V}_{{C}_{i}} \land  \left| {V}_{{C}_{i}}\right|  > 1$ , and in each ${C}_{i}$ that $v$ appears in, $v$ is on the left hand side of the constraint. We call $v$ the projection variable.

3. For all variables ${v}^{\prime } \in  {V}_{F}$ other than the projection variable, there is a single constraint ${C}_{i}$ where ${v}^{\prime } \in  {V}_{{C}_{i}} \land  \left| {V}_{{C}_{i}}\right|  > 1$ and the projection variable $v$ appears in ${C}_{i}$ , i.e., $v \in  {V}_{{C}_{i}}$ .

4. For all constraints ${C}_{i}$ where $\left| {V}_{{C}_{i}}\right|  > 1,{C}_{i}$ is not negated in the formula $F$ .

Many string constraints extracted from programs via symbolic execution are pseudo-relational constraints, or can be converted to pseudo-relational constraints. The projection variable represents either the variable that holds the value of the user's input to the program (for example, user input to a web application that needs to be validated), or the value of the string expression at a program sink. A program sink is a program point (such as a security sensitive function) for which it is necessary to compute the set of values that reach to that program point in order to check for vulnerabilities.

For example, following constraint is a pseudo-relational constraint extracted from a web application (regular expressions are simplified):

$$
\left( {x = y.z}\right)  \land  \left( {\operatorname{LEN}\left( y\right)  = 0}\right)  \land  \neg \left( {z \in  \left( {0 \mid  1}\right) }\right)  \land  \left( {x = t}\right)  \land  \neg \left( {t \in  {0}^{ * }}\right)
$$

Automata Construction for Pseudo-Relational Constraints: Given a pseudo-relational constraint $F$ and the projection variable $v$ , we now discuss how to construct the automaton $\mathcal{A}\left( {F, v}\right)$ that accepts $\llbracket F, v\rrbracket$ . As above, we assume that $F$ is converted to DNF form where $F \equiv   \vee  {}_{i = 1}^{n}{F}_{i},{F}_{i} \equiv   \land  {}_{j = 1}^{m}{C}_{ij}$ , and each ${C}_{ij}$ is either a basic constraint or negation of a basic constraint.

In order to construct the automaton $\mathcal{A}\left( {F, v}\right)$ we first construct the automata $\mathcal{A}\left( {{F}_{i}, v}\right)$ for each ${F}_{i}$ where $\mathcal{A}\left( {{F}_{i}, v}\right)$ accepts the language $\llbracket {F}_{i}, v\rrbracket$ . Then we combine the $\mathcal{A}\left( {{F}_{i}, v}\right)$ automata using automata product such that $\mathcal{A}\left( {F, v}\right)$ accepts the language $\left\lbrack  {{F}_{1}, v}\right\rbrack   \cup  \left\lbrack  {{F}_{2}, v}\right\rbrack   \cup  \ldots  \cup  \left\lbrack  {{F}_{m}, v}\right\rbrack$ .

Since we discussed how to handle disjunction, from now on we focus on constraints of the form $F \equiv  {C}_{1} \land  {C}_{2} \land  \ldots  \land  {C}_{n}$ where each ${C}_{i}$ is either a basic constraint or negation of a basic constraint. For each ${C}_{i}$ , let ${V}_{{C}_{i}}$ denote the set of variables that appear in ${C}_{i}$ . If ${V}_{{C}_{i}}$ is a singleton set, then we refer to the variable in it as ${v}_{{C}_{i}}$ .

First, for each single-variable constraint ${C}_{i}$ that is not negated, we construct an automaton that accepts the truth set of the constraint ${C}_{i},\left\llbracket  {{C}_{i},{v}_{{C}_{i}}}\right\rrbracket$ , using the techniques we discussed above for single-variable constraints. If ${C}_{i}$ is negated, then we construct the automaton that accepts the complement language ${\sum }^{ * } -$ $\left\lbrack  {{C}_{i},{v}_{{C}_{i}}}\right\rbrack$ (note that, only single-variable constraints can be negated in pseudo-relational constraints). Let us call these automata $\mathcal{A}\left( {{C}_{i},{v}_{{C}_{i}}}\right)$ (some of which may correspond to negated constraints).

Then, for any variable ${v}^{\prime } \in  {V}_{F}$ that is not the projection variable, we construct an automaton $\mathcal{A}\left( {F,{v}^{\prime }}\right)$ which accepts the intersection of the languages $\mathcal{A}\left( {{C}_{i},{v}^{\prime }}\right)$ for all single-variable constraints that ${v}^{\prime }$ appears in, i.e., $\mathcal{L}\left( {\mathcal{A}\left( {F,{v}^{\prime }}\right) }\right)  =$ $\mathop{\bigcap }\limits_{{{V}_{{C}_{i}} = \left\{  {v}^{\prime }\right\}  }}\mathcal{L}\left( {\mathcal{A}\left( {{C}_{i},{v}^{\prime }}\right) }\right) .$

Next, for each multi-variable constraint ${C}_{i}$ we construct an automaton that accepts the language $\llbracket {C}_{i}, v\rrbracket$ where $v$ is the projection variable as follows:

- case ${C}_{i} \equiv  v = {v}^{\prime } : \mathcal{A}\left( {{C}_{i}, v}\right)  = \mathcal{A}\left( {F,{v}^{\prime }}\right)$ .

- case ${C}_{i} \equiv  v = {v}_{1}.{v}_{2} : \mathcal{A}\left( {{C}_{i}, v}\right)$ is constructed using the automata $\mathcal{A}\left( {F,{v}_{1}}\right)$ and $\mathcal{A}\left( {F,{v}_{2}}\right)$ and it accepts the concatenation of the languages $\mathcal{L}\left( {\mathcal{A}\left( {F,{v}_{1}}\right) }\right)$ and $\mathcal{L}\left( {\mathcal{A}\left( {F,{v}_{2}}\right) }\right)$ .

- case ${C}_{i} \equiv  \operatorname{LEN}\left( v\right)  = \operatorname{LEN}\left( {v}^{\prime }\right)$ : Given the automaton $\mathcal{A}\left( {F,{v}^{\prime }}\right)$ , we construct an automaton ${A}_{\operatorname{LEN}\left( {F,{v}^{\prime }}\right) }$ such that $s \in  \mathcal{L}\left( {A}_{\operatorname{LEN}\left( {F,{v}^{\prime }}\right) }\right)  \Leftrightarrow  \exists {s}^{\prime } : \operatorname{LEN}\left( s\right)  = \operatorname{LEN}\left( {s}^{\prime }\right)  \land  {s}^{\prime } \in$ $\mathcal{L}\left( {\mathcal{A}\left( {F,{v}^{\prime }}\right) }\right)$ . Then, $\mathcal{A}\left( {{C}_{i}, v}\right)  = {A}_{\operatorname{LEN}\left( {F,{v}^{\prime }}\right) }$ .

- case ${C}_{i} \equiv  \operatorname{LEN}\left( v\right)  < \operatorname{LEN}\left( {v}^{\prime }\right)$ : Given the automaton $\mathcal{A}\left( {F,{v}^{\prime }}\right)$ we find the length of the maximum word accepted by $\mathcal{A}\left( {F,{v}^{\prime }}\right)$ , which is infinite if $\mathcal{A}\left( {F,{v}^{\prime }}\right)$ has a loop that can reach an accepting state. If it is infinite then $\mathcal{A}\left( {{C}_{i}, v}\right)  = A\left( {\sum }^{ * }\right)$ . If not, then given the maximum length $m,\mathcal{A}\left( {{C}_{i}, v}\right)$ is the automaton that accepts the language $\{ \varepsilon \}  \cup  {\sum }^{1} \cup  {\sum }^{2} \cup  \ldots  \cup  {\sum }^{m - 1}$ . Note that if $m = 0$ then $\mathcal{A}\left( {{C}_{i}, v}\right)  = A\left( \varnothing \right)$ .

- case ${C}_{i} \equiv  \operatorname{LEN}\left( v\right)  > \operatorname{LEN}\left( {v}^{\prime }\right)$ : Given the automaton $\mathcal{A}\left( {F,{v}^{\prime }}\right)$ we find the length of the minimum word accepted by $\mathcal{A}\left( {F,{v}^{\prime }}\right)$ . Given the minimum length $m,\mathcal{A}\left( {{C}_{i}, v}\right)$ is the automaton that accepts the concatenation of the languages accepted by $\mathcal{A}\left( {\sum }^{m + 1}\right)$ and $\mathcal{A}\left( {\sum }^{ * }\right)$ , i.e, ${\sum }^{m + 1}{\sum }^{ * }$ .

- case ${C}_{i} \equiv  v = \operatorname{REPLACE}\left( {{v}^{\prime }, s, s}\right)$ : Given the automaton $\mathcal{A}\left( {F,{v}^{\prime }}\right)$ we use the construction presented in $\left\lbrack  {{39},{38}}\right\rbrack$ for language based replacement to construct the automaton $\mathcal{A}\left( {{C}_{i}, v}\right)$ .

The final step of the construction is to construct $\mathcal{A}\left( {F, v}\right)$ using the automata $\mathcal{A}\left( {{C}_{i}, v}\right)$ where $\mathcal{L}\left( {\mathcal{A}\left( {F, v}\right) }\right)  = \mathop{\bigcap }\limits_{{v \in  {V}_{{C}_{i}}}}\mathcal{L}\left( {\mathcal{A}\left( {{C}_{i}, v}\right) }\right)$ .

For pseudo-relational constraints, the automaton $\mathcal{A}\left( {F, v}\right)$ ) constructed based on the above construction accepts the truth set of the formula $F$ for the projected variable, i.e., $\mathcal{L}\left( {\mathcal{A}\left( {F, v}\right) }\right)  = \llbracket F, v\rrbracket$ . However, the replace function has different variations in different programming languages (such as first-match versus longest-match replace) and the match pattern can be given as a regular expression. The language-based replace automata construction we use $\left\lbrack  {{39},{38}}\right\rbrack$ over-approximates the replace operation in some cases, which would then result in over-approximation of the truth set: $\mathcal{L}\left( {\mathcal{A}\left( {F, v}\right) }\right)  \supseteq  \llbracket F, v\rrbracket$ .

Automata Construction for Relational Constraints: For constraints that are not pseudo-relational, we extend the above algorithm to compute an over approximation of $\llbracket F, v\rrbracket$ . In relational constraints, more than one variable can be involved in multi-variable constraints which creates a cycle in constraint evaluation.

Given a relational constraint in the form $F \equiv  {C}_{1} \land  {C}_{2} \land  \ldots  \land  {C}_{n}$ , we start with initializing each $\mathcal{A}\left( {F, v}\right)$ to $\mathcal{A}\left( {\sum }^{ * }\right)$ , i.e., initially variables are unconstrained. Then, we process each constraint as we described above to compute new automata for the variables in that constraint using the automata that are already available for each variable. We can stop this process at any time, and, for each variable $v$ , we would get an over-approximation of the truth-set $\mathcal{A}\left( {F, v}\right)  \supseteq  \llbracket F, v\rrbracket$ . We can state this algorithm as follows:

Algorithm 1 AUTOMATAFORCONSTRAINT $\left( {F \equiv  {C}_{1} \land  {C}_{2} \land  \ldots  \land  {C}_{n}}\right)$

---

for $v \in  {V}_{F}$ do

	$\mathcal{A}\left( {F, v}\right)  = \mathcal{A}\left( {\sum }^{ * }\right) ;$

end for

count $= 0$ ; done $=$ false;

while count $<$ bound $\land  \neg$ done do

	for each $C \in  F$ and $v \in  {V}_{C}$ do

		construct ${A}^{\prime }$ where $\mathcal{L}\left( {A}^{\prime }\right)  = \mathcal{L}\left( {\mathcal{A}\left( {F, v}\right) }\right)  \cap  \mathcal{L}\left( {\mathcal{A}\left( {C, v}\right) }\right)$ ;

		$\mathcal{L}\left( {\mathcal{A}\left( {F, v}\right) }\right)  = {A}^{\prime }$ ;

	end for

	if none of the $\mathcal{L}\left( {\mathcal{A}\left( {F, v}\right) }\right)$ changed during the current iteration of the while loop

	then

		done $=$ true;

	end if

	count $=$ count +1 ;

end while

---

In order to improve the efficiency of the above algorithm, we first build a constraint dependency graph where,1) a multi-variable constraint ${C}_{i}$ depends on a single variable constraint ${C}_{j}$ if ${V}_{{C}_{j}} \subseteq  {V}_{{C}_{i}}$ , and 2) a multi-variable constraint ${C}_{i}$ depends on a multi-variable constraint ${C}_{j}$ if ${V}_{{C}_{j}} \cap  {V}_{{C}_{i}} \neq  \varnothing$ . We traverse the constraints based on their ordering in the dependency graph and iteratively refine the automata in case of cyclic dependencies. Note that, in the constructions we described above we only constructed automaton for the variable on the left-hand-side of a relational constraint using the automata for the variables on the right-hand-side of the constraint. In the general case we need to construct automata for variables on the right-hand-side of the relational constraints too. We do this using techniques similar to the ones we described above. Constructing automata for the right-hand-side variables is equivalent to the pre-image computations used during backward symbolic analysis as discussed in [35] and we use the constructions given there. Finally, unlike pseudo-relational constraints, a relational constraint can contain negation of a basic constraint ${C}_{i}$ where $\left| {V}_{{C}_{i}}\right|  > 1$ . In such cases, in constructing the truth set of $\neg {C}_{i}$ we can use the complement language ${\sum }^{ * } - \left\llbracket  {{C}_{i}, v}\right\rbrack$ only if $\left\llbracket  {{C}_{i}, v}\right\rrbracket$ is a singleton set. Otherwise, we construct an over approximation of the truth set of $\neg {C}_{i}$ .

## 3 Automata-based Model Counting

Once we have translated a set of constraints into an automaton we employ algebraic graph theory [5] and analytic combinatorics [14] to perform model counting. In our method, model counting corresponds exactly to counting the accepting paths of the constraint DFA up to a given length bound $k$ . This problem can be solved using dynamic programming techniques in $O\left( {k \cdot  \left| \delta \right| }\right)$ time where $\delta$ is the DFA transition relation $\left\lbrack  {{11},{16}}\right\rbrack$ . However, for each different bound, the dynamic programming technique requires another traversal of the DFA graph.

A preferable solution is to derive a symbolic function that given a length bound $k$ outputs the number of solutions within bound $k$ . To achieve this, we use the transfer matrix method $\left\lbrack  {{30},{14}}\right\rbrack$ to produce an ordinary generating function which in turn yields a linear recurrence relation that is used to count constraint solutions. We will briefly review the necessary background and then describe the model counting algorithm.

Given a DFA $A$ , consider its corresponding language $\mathcal{L}$ . Let ${\mathcal{L}}_{i} = \{ w \in  \mathcal{L}$ : $\left| w\right|  = i\}$ , the language of strings in $\mathcal{L}$ with length $i$ . Then $\mathcal{L} = \mathop{\bigcup }\limits_{{i > 0}}{\mathcal{L}}_{i}$ . Define $\left| {\mathcal{L}}_{i}\right|$ to be the cardinality of ${\mathcal{L}}_{i}$ . The cardinality of $\mathcal{L}$ can be computed by the sum of a series ${a}_{0},{a}_{1},\ldots ,{a}_{i},\ldots$ where each ${a}_{i}$ is the cardinality of the corresponding language ${\mathcal{L}}_{i}$ , i.e., ${a}_{i} = \left| {\mathcal{L}}_{i}\right|$ .

For example, recall the automaton in Fig. 1. Let ${\mathcal{L}}^{x}$ be the language over $\sum  = \{ 0,1\}$ that satisfies the formula $\left( {x \notin  {\left( {01}\right) }^{ * } \land  \operatorname{LEN}\left( x\right)  \geq  1}\right)$ . Then ${\mathcal{L}}^{x}$ is described by the expression ${\sum }^{ * } - {\left( {01}\right) }^{ * }$ . In the language ${\mathcal{L}}^{x}$ , we have zero strings of length $0\left( {\varepsilon  \notin  {\mathcal{L}}^{x}}\right)$ , two strings of length $1\left( {\{ 0,1\} }\right)$ , three strings of length $3\left( {\{ {00},{10},{11}\} }\right)$ , and so on. The sequence is then ${a}_{0} = 0,{a}_{1} = 2,{a}_{2} = 3,{a}_{3} =$ $8,{a}_{4} = {15}$ , etc. For any length $i,\left| {\mathcal{L}}_{i}^{x}\right|$ , is given by a ${3}^{rd}$ order linear recurrence relation:

$$
{a}_{0} = 0,{a}_{1} = 2,{a}_{2} = 3 \tag{15}
$$

$$
{a}_{i} = 2{a}_{i - 1} + {a}_{i - 2} - 2{a}_{i - 3}\text{for}i \geq  3
$$

In fact, using standard techniques for solving linear homogeneous recurrences, we can derive a closed form solution to determine that

$$
\left| {\mathcal{L}}_{i}^{x}\right|  = \left( {1/2}\right) \left( {{2}^{i + 1} + {\left( -1\right) }^{i + 1} - 1}\right) . \tag{16}
$$

In the following discussion we give a general method based on generating functions for deriving a recurrence relation and closed form solution that we can use for model counting.

Generating Functions: Given the representation of the size of a language $\mathcal{L}$ as a sequence $\left\{  {a}_{i}\right\}$ we can encode each $\left| {\mathcal{L}}_{i}\right|$ as the coefficients of a polynomial, an ordinary generating function (GF). The ordinary generating function of the sequence ${a}_{0},{a}_{1},\ldots ,{a}_{i},\ldots$ is the infinite polynomial $\left\lbrack  {{30},{14}}\right\rbrack$

$$
g\left( z\right)  = \mathop{\sum }\limits_{{i \geq  0}}{a}_{i}{z}^{i} \tag{17}
$$

Although $g\left( z\right)$ is an infinite polynomial, $g\left( z\right)$ can be interpreted as the Taylor series of a finite rational expression. I.e., we can also write $g\left( z\right)  = p\left( z\right) /q\left( z\right)$ , where $p\left( z\right)$ and $q\left( z\right)$ are finite degree polynomials. If $g\left( z\right)$ is given as a finite rational expression, each ${a}_{i}$ can be computed from the Taylor expansion of $g\left( z\right)$ :

$$
{a}_{i} = \frac{{g}^{\left( i\right) }\left( 0\right) }{i!} \tag{18}
$$

where ${g}^{\left( i\right) }\left( z\right)$ is the ${i}^{th}$ derivative of $g\left( z\right)$ . We write $\left\lbrack  {z}^{i}\right\rbrack  g\left( z\right)$ for the ${i}^{th}$ Taylor series coefficient of $g\left( z\right)$ . Returning to our example, we can write the generating function for $\left| {\mathcal{L}}_{i}^{x}\right|$ both as a rational function and as an infinite Taylor series polynomial. The reader can verify the following equivalence by computing the right hand side coefficients via equation (18).

$$
g\left( z\right)  = \frac{{2z} - {z}^{2}}{1 - {2z} - {z}^{2} + 2{z}^{3}} = 0{z}^{0} + 2{z}^{1} + 3{z}^{2} + 8{z}^{3} + {15}{z}^{4} + \ldots  \tag{19}
$$

![0196f760-828a-79a3-b5a0-90589a4fd3c5_9_381_281_1031_302_0.jpg](images/0196f760-828a-79a3-b5a0-90589a4fd3c5_9_381_281_1031_302_0.jpg)

Fig. 2. (a) The original DFA $A$ , and (b) the augmented DFA ${A}^{\prime }$ used for model counting (sink state omitted).

Generating Function for a DFA: Given a DFA $A$ and length $k$ we can compute the generating function ${g}_{A}\left( z\right)$ such that the ${k}^{th}$ Taylor series coefficient of ${g}_{A}\left( z\right)$ is equal to $\left| {{\mathcal{L}}_{k}\left( A\right) }\right|$ using the transfer-matrix method $\left\lbrack  {{30},{14}}\right\rbrack$ .

We first apply a transformation and add an extra state, ${s}_{n + 1}$ . The resulting automaton is a DFA ${A}^{\prime }$ with $\lambda$ -transitions from each of the accepting states of $A$ to ${s}_{n + 1}$ where $\lambda$ is a new padding symbol that is not in the alphabet of $A$ . Thus, $\mathcal{L}\left( {A}^{\prime }\right)  = \mathcal{L}\left( A\right)  \cdot  \lambda$ and furthermore $\left| {{\mathcal{L}}_{i}\left( A\right) }\right|  = \left| {{\mathcal{L}}_{i + 1}\left( {A}^{\prime }\right) }\right|$ . That is, the augmented DFA ${A}^{\prime }$ preserves both the language and count information of $A$ . Recalling the automaton from Fig. 1, the corresponding augmented DFA is shown in Fig. 2(b). (Ignore the dashed $\lambda$ transition for the time being.)

From ${A}^{\prime }$ we construct the $\left( {n + 1}\right)  \times  \left( {n + 1}\right)$ transfer matrix $T.{A}^{\prime }$ has $n + 1$ states ${s}_{1},{s}_{2},\ldots {s}_{n + 1}$ . The matrix entry ${T}_{i, j}$ is the number of transitions from state ${s}_{i}$ to state ${s}_{j}$ . Then the generating function for $A$ is

$$
{g}_{A}\left( z\right)  = {\left( -1\right) }^{n}\frac{\det \left( {I - {zT} : n + 1,1}\right) }{z\det \left( {I - {zT}}\right) }, \tag{20}
$$

where $\left( {M : i, j}\right)$ denotes the matrix obtained by removing the ${i}^{th}$ row and ${j}^{th}$ column from $M, I$ is the identity matrix, det $M$ is the matrix determinant, and $n$ is the number of states in the original DFA $A$ . The number of accepting paths of $A$ with length exactly $k$ , i.e. $\left| {{\mathcal{L}}_{k}\left( A\right) }\right|$ , is then given by $\left\lbrack  {z}^{k}\right\rbrack  {g}_{A}\left( z\right)$ which can be computed through symbolic differentiation via equation 18 .

For our running example, we show the transition matrix $T$ and the terms (I - zT)and $\left( {I - {zT} : n,1}\right)$ . Here, ${T}_{1,2}$ is 1 because there is a single transition from state 1 to state $2,{T}_{3,3}$ is 2 because there are two transitions from state 3 to itself, ${T}_{2,4}$ is 1 because there is a single $\left( \lambda \right)$ transition from state 2 to state 4, and so on for the remaining entries.

$$
T = \left\lbrack  \begin{array}{llll} 0 & 1 & 1 & 0 \\  1 & 0 & 1 & 1 \\  0 & 0 & 2 & 1 \\  0 & 0 & 0 & 1 \end{array}\right\rbrack  , I - {zT} = \left\lbrack  \begin{matrix} 1 &  - z &  - z & 0 \\   - z & 1 &  - z &  - z \\  0 & 0 & 1 - {2z} - z & \\  0 & 0 & 0 & 1 \end{matrix}\right\rbrack  ,\left( {I - {zT} : n,1}\right)  = \left\lbrack  \begin{matrix}  - z &  - z & 0 \\  1 &  - z &  - z \\  0 & 1 - {2z} - z &  \end{matrix}\right\rbrack
$$

Applying equation (20) results in the same GF that counts ${\mathcal{L}}_{i}\left( A\right)$ given in (19).

$$
{g}_{{A}^{\prime }}\left( z\right)  =  - \frac{\det \left( {I - {zT} : n,1}\right) }{z\det \left( {I - {zT}}\right) } = \frac{{2z} - {z}^{2}}{1 - {2z} - {z}^{2} + 2{z}^{3}}. \tag{21}
$$

Suppose we now want to know the number of solutions of length six. We compute the sixth Taylor series coefficient to find that $\left| {{\mathcal{L}}_{6}^{x}\left( A\right) }\right|  = \left\lbrack  {z}^{6}\right\rbrack  g\left( z\right)  = {63}$ .

Deriving a Recurrence Relation: We would like a way to compute $\left\lbrack  {z}^{i}\right\rbrack  g\left( z\right)$ that is more direct than symbolic differentiation. We describe how a linear recurrence for $\left\lbrack  {z}^{i}\right\rbrack  g\left( z\right)$ can be extracted from the GF. Before we describe how to accomplish this in general, we demonstrate the procedure for our example. Combining equations (17) and (21) and multiplying by the denominator, we have

$$
{2z} - {z}^{2} = \left( {1 - {2z} - {z}^{2} + 2{z}^{3}}\right) \mathop{\sum }\limits_{{i \geq  0}}{a}_{i}{z}^{i}.
$$

Expanding the sum for $0 \leq  i < 3$ and collecting terms,

$$
{2z} - {z}^{2} = {a}_{0} + \left( {{a}_{1} - 2{a}_{0}}\right) z + \left( {{a}_{2} - 2{a}_{1} - {a}_{0}}\right) {z}^{2} + \mathop{\sum }\limits_{{i \geq  3}}\left( {{a}_{i} - 2{a}_{i - 1} - {a}_{i - 2} + 2{a}_{i - 3}}\right) {z}^{i}.
$$

Comparing each coefficient of ${z}^{i}$ on the left side to the coefficient of ${z}^{i}$ on the right side, we have the set of equations

$$
{a}_{0} = 0
$$

$$
{a}_{1} - 2{a}_{0} = 2
$$

$$
{a}_{2} - 2{a}_{1} - {a}_{0} =  - 1
$$

$$
{a}_{i} - 2{a}_{i - 1} - {a}_{i - 2} + 2{a}_{i - 3} = 0\text{, for}i \geq  3
$$

One can see that this results in the same solution given in equation (15).

This idea is easily generalized. Recall that $g\left( z\right)  = p\left( z\right) /q\left( z\right)$ for finite degree polynomials $p$ and $q$ . Suppose that the maximum degree of $p$ and $q$ is $m$ . Then

$$
g\left( z\right)  = \frac{{b}_{m}{z}^{m} + \ldots  + {b}_{1}z + {b}_{0}}{{c}_{m}{z}^{m} + \ldots  + {c}_{1}z + {c}_{0}} = \mathop{\sum }\limits_{{i \geq  0}}{a}_{i}{z}^{i}.
$$

Multiplying by the denominator, expanding the sum up to $m$ terms, and comparing coefficients we have the resulting system of equations which can be solved for $\left\{  {{a}_{i} : 0 \leq  i \leq  m}\right\}$ using standard linear algebra:

$$
\mathop{\sum }\limits_{{j = 0}}^{i}{c}_{j}{a}_{i - j} = \left\{  \begin{array}{l} {b}_{i},\text{ for }0 \leq  i \leq  m \\  0,\text{ for }i > m \end{array}\right.
$$

For any DFA $A$ , since each coefficient ${a}_{i}$ is associated with $\left| {{\mathcal{L}}_{k}\left( A\right) }\right|$ , the recurrence gives us an $O\left( {kn}\right)$ method to compute $\left| {{\mathcal{L}}_{k}\left( A\right) }\right|$ for any string length bound $k$ . In addition, standard techniques for solving linear homogeneous recurrence relations can be used to derive a closed form solution for $\left| {{\mathcal{L}}_{i}\left( A\right) }\right|$ [22].

Counting All Solutions within a Given Bound: The above described method gives a generating function that encodes each $\left| {{\mathcal{L}}_{i}\left( A\right) }\right|$ separately. Instead, we seek a generating function that encodes the number of all solutions within a bound. To this end we define the automata model counting function

$$
{\mathcal{{MC}}}_{A}\left( k\right)  = \mathop{\sum }\limits_{{i \geq  0}}^{k}\left| {{\mathcal{L}}_{i}\left( A\right) }\right|  \tag{22}
$$

In order to compute $\mathcal{M}{\mathcal{C}}_{A}\left( k\right)$ we make a simple adjustment. All that is needed is to add a single $\lambda$ -cycle (the dashed transition in Fig. 2(b)) to the accepting state of the augmenting DFA ${A}^{\prime }$ . Then ${\mathcal{L}}_{k + 1}\left( {A}^{\prime }\right)  = \mathop{\bigcup }\limits_{{i = 0}}^{k}{\mathcal{L}}_{i}\left( A\right)  \cdot  {\lambda }^{k - i}$ and the accepting paths of strings in ${\mathcal{L}}_{k + 1}\left( {A}^{\prime }\right)$ are in one-to-one correspondence with the accepting paths of strings in $\mathop{\bigcup }\limits_{{i = 0}}^{k}{\mathcal{L}}_{i}\left( A\right)$ . Consequently, $\left| {{\mathcal{L}}_{k + 1}\left( {A}^{\prime }\right) }\right|  = \mathop{\sum }\limits_{{i = 0}}^{k}\left| {{\mathcal{L}}_{i}\left( A\right) }\right|$ and so $\mathcal{M}{\mathcal{C}}_{A}\left( k\right)  = \left| {{\mathcal{L}}_{k + 1}\left( {A}^{\prime }\right) }\right|$ . Hence, we can compute $\mathcal{M}{\mathcal{C}}_{A}$ using the recurrence for $\left| {{\mathcal{L}}_{i}\left( {A}^{\prime }\right) }\right|$ with the additional $\lambda$ -cycle.

## 4 Implementation

We implemented Automata-Based model Counter for string constraints (ABC) using the symbolic string analysis library provided by the Stranger tool $\lbrack {39},{36}$ , 38]. We used the symbolic DFA representation of the MONA DFA library [8] to implement the constructions described in Section 2. In MONA's DFA library, the transition relation of the DFA is represented as a Multi-terminal Binary Decision Diagram (MBDD) which results in a compact representation of the transition relation. ABC supports more operations (such as TRIM, SUBSTRING) than the ones listed in Section 2 using constructions similar to the ones given in that section.

ABC supports the SMT-LIB 2 language syntax. We specifically added support for CVC4 string operations [24]. In string constraint benchmarks provided by CVC4, boolean variables are used to assert the results of subformulas. In our automata-based constraint solver, we check the satisfiability of a formula by checking if its truth set is empty or not. We eliminated the boolean variables that are only used to check the results of string operations (such as string equivalence, string membership) and instead substituted the corresponding expressions directly. We converted if-then-else structures into disjunctions. We also searched for several patterns between length equations and word equations to infer the values of the string variables whenever possible (for example when we see the constraint $\operatorname{LEN}\left( x\right)  = 0$ we can infer that the string variable $x$ must be equal to the empty string). These transformations allow us to convert some constraints to pseudo-relational constraints that we can precisely solve. If these transformations do not resolve all the cyclic dependencies in a constraint then the resulting DFA may recognize an over-approximation of all possible solutions.

We implemented the automata-based model counting algorithm of Section 3 by passing the automaton transfer matrix to Mathematica for computing the generating function, corresponding recurrence relation, and the model count for a specific bound. Because the DFAs we encountered in our experiments typically have sparse transition graphs, we make use of Mathematica's powerful and efficient implementations of symbolic sparse matrix determinant functions [33].

## 5 Experiments

To evaluate ABC we experimented with a set of Java application benchmarks, SMT-LIB 2 translation of Kaluza JavaScript benchmarks, and several examples from the SMC distribution. In our experiments we compared ABC to SMC [25] and CVC4 [24]. We ran all the experiments on an Intel I5 machine with 2.5GHz X 4 processors and ${32}\mathrm{{GB}}$ of memory running Ubuntu ${14.04}^{1}$ .

Table 1. Constraint Characteristics

<table><tr><td rowspan="2">Benchmarks</td><td rowspan="2">#Constraints</td><td colspan="10">Frequency of Operations Per 1000 Formulas</td></tr><tr><td>E</td><td>-</td><td>=</td><td>LEN</td><td>REPLACE</td><td>INDEXOF</td><td>CONTAINS</td><td>BEGINS</td><td>ENDS</td><td>SUBSTRING</td></tr><tr><td>ASE</td><td>116164</td><td>0.42</td><td>386.10</td><td>129.39</td><td>382.54</td><td>639.28</td><td>4.11</td><td>7.52</td><td>16.91</td><td>7.51</td><td>41.17</td></tr><tr><td>Kaluza Small</td><td>368433</td><td>30.29</td><td>93.89</td><td>224.87</td><td>46.84</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td></tr><tr><td>Kaluza Big</td><td>5138323</td><td>38.12</td><td>129.53</td><td>164.64</td><td>60.46</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td></tr></table>

Table 1 shows the frequency of string operations from our string constraint grammar that are contained in the ASE, Kaluza Small, and Kaluza Big benchmark sets. ASE benchmarks are from Java programs and represent server-side code [20]. The Kaluza benchmarks are taken from JavaScript programs and represent client-side code [28]. All three benchmarks have regular expression membership $\left(  \in  \right)$ , concatenation $\left( \text{.}\right) ,$ string equality $\left(  = \right)$ , and length constraints. However, the ASE benchmark contains additional string operations that are typically used for input sanitization, like REPLACE and SUBSTRING.

Java Benchmarks. String constraints in these benchmarks are extracted from 7 real-world Java applications: Jericho HTML Parser, jxml2xql (an xml-to-sql converter), MathParser, MathQuizGame, Natural CLI (a natural language command line tool), Beasties (a command line game), HtmlCleaner, and iText (a PDF library) [20]. These benchmarks represent server-side code and employ many input-sanitizing string operators such as REPLACE and SUBSTRING as seen in Table 1. These string constraints were generated by extracting program path constraints through dynamic symbolic execution [20].

In [20], an empirical evaluation of several string constraint solvers is presented. As a part of this empirical evaluation, the authors use the symbolic string analysis library of Stranger $\left\lbrack  {{39},{36},{38}}\right\rbrack$ to construct automata for path constraints on strings. In order to evaluate the model counting component of $\mathrm{{ABC}}$ , we ran their tool on the 7 benchmark sets and output the resulting automata whenever the constraint is satisfiable. Out of 116,164 string path constraints, 66,236 were found to be satisfiable and we performed model counting on those cases. The constraints in Java benchmarks are all single-variable or pseudo-relational constraints. The resulting automata do not have any over-approximation caused by relational constraints. As a measure of the size of the resulting automata, we give the number of BDD nodes used in the symbolic transition relation representation of MONA. The average number of BDD nodes for the satisfiable path constraints is 69.51 and the size of the each BDD node is 16 bytes. For these benchmarks our model-counter is efficient; the average running time of model counting per path constraint is 0.0015 seconds and the resulting model-counting recurrence is precise, i.e., gives the exact count for any given bound.

SMC and CVC4 are not able to handle the constraints in this data set since they do not support sanitization operations such as REPLACE.

---

${}^{1}$ Results of our experiments are available at http://www.cs.ucsb.edu/v-vlab/ABC/

---

SMC Examples. For a comparative evaluation of our tool with SMC, we used the examples that are listed on SMC's web page. We translated the 6 example constraints listed in table 2 into SMT-LIB2 language format that we support. We inspected the examples to confirm that they are pseudo-relational, i.e., our analysis generates a precise model-counting function for these constraints. We compare our results with the results reported in SMC's web page. The first column of the Table 2 shows the file names of these example constraints. The second column shows the bounds used for obtaining the model counts. The next two columns show the log-scale SMC lower and upper bound values for the model counts. The last column shows the log-scale model count produced by ABC. We omit the decimal places of the numbers to fit them on the page. For all the cases ABC generates a precise count given the bound. ABC's count is exactly equal to SMC's upper bound for four of the examples and is exactly equal to SMC's lower bound for one example. For the last example ABC reports a count that is between the lower and upper bound produced by SMC. Note that these are log scaled values and actual differences between a lower and an upper-bound values are huge. Although SMC is unable to produce an exact answer for any of these examples, ABC produces an exact count for each of them.

Table 2. Log scaled comparison between SMC and ABC

<table><tr><td/><td>bound</td><td>SMC lower bound</td><td>SMC upper bound</td><td>ABC count</td></tr><tr><td>nullhttpd</td><td>500</td><td>3752</td><td>3760</td><td>3760</td></tr><tr><td>ghttpd</td><td>620</td><td>4880</td><td>4896</td><td>4896</td></tr><tr><td>csplit</td><td>629</td><td>4852</td><td>4921</td><td>4921</td></tr><tr><td>grep</td><td>629</td><td>4676</td><td>4763</td><td>4763</td></tr><tr><td>WC</td><td>629</td><td>4281</td><td>4284</td><td>4281</td></tr><tr><td>obscure</td><td>6</td><td>0</td><td>3</td><td>2</td></tr></table>

JavaScript Benchmarks. We also experimented with Kaluza benchmarks which were extracted from JavaScript code via dynamic symbolic execution [28]. These benchmarks are divided to a small and large set based on the sizes of the constraints. These benchmarks have been used by both SMC and CVC4 tools. ABC handles 19,731 benchmark constraints in the satisfiable small set with an average of 0.32 seconds per constraint for model counting, whereas SMC handles 17,559 constraints with an average of 0.26 seconds per constraint. ABC handles 1,587 benchmark constraints in satisfiable big set with an average of 0.34 seconds per constraint for model counting, whereas SMC handles 1,342 constraints with an average of 5.29 seconds per constraint. We were not able to do a one-to-one timing and precision comparison between ABC and SMC for each constraint due to an error in the SMC data file (the mapping between file names and results is incorrect).

Satisfiability Checking Evaluation. We ran ABC on SMT-LIB 2 translation of the full set of JavaScript benchmarks. We put a 20-second CPU timeout limit on $\mathrm{{ABC}}$ for each benchmark constraint. Table 3 shows the comparison between $\mathrm{{ABC}}$ and the CVC4 [24] constraint solver based on the CVC4 results that are available online. The first column shows the initial satisfiability classification of the data set by the creators of the benchmarks [28]. The next two columns show the number of results that ABC and CVC4 agree. The last three columns show the cases where ABC and CVC4 differ. Note that, since ABC over-approximates the solution set, if the given constraint is not single-valued or pseudo-relational, it is possible for ABC to classify a constraint as satisfiable even if it is unsatisfiable. However, it is not possible for ABC to classify a constraint unsatisfiable if it is satisfiable. Out of 47,284 benchmark constraints ABC and CVC4 agree on 41,793 of them. As expected ABC never classifies a constraint as unsatisfiable if CVC4 classifies it as satisfiable. However, due to over-approximation of relational constraints, ABC classifies 2,459 constraints as satisfiable although CVC4 classifies them as unsatisfiable. A practical approach would be to use ABC together with a satisfiability solver like CVC4, and, given a constraint, first use the satisfiability solver to determine the satisfiability of the formula, and then use $\mathrm{{ABC}}$ to generate its truth set and the model counting function.

Table 3. Constraint-Solver Comparison

<table><tr><td rowspan="2"/><td>ABCCVC4</td><td>$\mathrm{{ABC}}$CVC4</td><td>$\mathrm{{ABC}}$CVC4</td><td>ABCCVC4</td><td>$\mathrm{{ABC}}$CVC4</td></tr><tr><td>sat - sat</td><td>unsat-unsat</td><td>sat-unsat</td><td>unsat-sat</td><td>sat-timeout</td></tr><tr><td>sat/small</td><td>19728</td><td>3</td><td>0</td><td>0</td><td>0</td></tr><tr><td>sat/big</td><td>1587</td><td>0</td><td>0</td><td>0</td><td>0</td></tr><tr><td>unsat/small</td><td>8139</td><td>3013</td><td>74</td><td>0</td><td>0</td></tr><tr><td>unsat/big</td><td>3419</td><td>5904</td><td>2385</td><td>0</td><td>2359</td></tr></table>

The average automata construction time for big benchmark constraints is 0.44 seconds and for small benchmark constraints it is 0.01 seconds. CVC4 average running times are 0.18 seconds and 0.015 seconds respectively (excluding timeouts). CVC4 times out for 2359 constraints, whereas ABC never times out. For those 2359 constraints, ABC reports satisfiable. ABC is unable to handle 672 constraints; the automata package we use (MONA) is unable to handle the resulting automata and we believe that these cases can be solved by modifying MONA. For these 672 constraints; CVC4 times out for 29 of them, reports unsat for 246 of them, and reports sat for 397 of them. There are also a few thousand constraints from the Kaluza benchmarks that CVC4 is unable to handle.

## 6 Conclusions and Future Work

We presented a model-counting string constraint solver that, given a constraint, generates: 1) An automaton that accepts all solutions to the given string constraint; 2) A model-counting function that, given a length bound, returns the number of solutions within that bound. Our experiments on thousands of constraints extracted from real-world web applications demonstrates the effectiveness and efficiency of the proposed approach. Our string constraint solver can be used in quantitative information flow, probabilistic analysis and automated repair synthesis. We plan to extend our automata-based model-counting approach to Presburger arithmetic constraints using an automata-based representation for Presburger arithmetic constraints $\left\lbrack  {{34},4}\right\rbrack$ .

## References

1. Parosh Aziz Abdulla, Mohamed Faouzi Atig, Yu-Fang Chen, Lukás Holík, Ahmed Rezine, Philipp Rümmer, and Jari Stenman. String constraints for verification. In Proceedings of the 26th International Conference on Computer Aided Verification (CAV), pages 150-166, 2014.

2. Muath Alkhalaf, Abdulbaki Aydin, and Tevfik Bultan. Semantic differential repair for input validation and sanitization. In Proceedings of the International Symposium on Software Testing and Analysis (ISSTA), pages 225-236, 2014.

3. Muath Alkhalaf, Tevfik Bultan, and Jose L. Gallegos. Verifying client-side input validation functions using string analysis. In Proceedings of the 34th International Conference on Software Engineering (ICSE), pages 947-957, 2012.

4. Constantinos Bartzis and Tevfik Bultan. Efficient symbolic representations for arithmetic constraints in verification. Int. J. Found. Comput. Sci., 14(4):605-624, 2003.

5. N. Biggs. Algebraic Graph Theory. Cambridge Mathematical Library. Cambridge University Press, 1993.

6. Nikolaj Bjørner, Nikolai Tillmann, and Andrei Voronkov. Path feasibility analysis for string-manipulating programs. In Proceeding of the 15th International Conference on Tools and Algorithms for the Construction and Analysis of Systems (TACAS), pages 307-321, 2009.

7. Mateus Borges, Antonio Filieri, Marcelo d'Amorim, Corina S. Pasareanu, and Willem Visser. Compositional solution space quantification for probabilistic software analysis. In Proceedigns of the ACM SIGPLAN Conference on Programming Language Design and Implementation (PLDI), 2014.

8. BRICS. The MONA project. http://www.brics.dk/mona/.

9. Aske Simon Christensen, Anders Møller, and Michael I. Schwartzbach. Precise analysis of string expressions. In Proceedings of the 10th International Static Analysis Symposium (SAS), pages 1-18, 2003.

10. David Clark, Sebastian Hunt, and Pasquale Malacaria. A static analysis for quantifying information flow in a simple imperative language. J. Comput. Secur., 15(3), 2007.

11. Thomas H. Cormen, Clifford Stein, Ronald L. Rivest, and Charles E. Leiserson. Introduction to Algorithms. McGraw-Hill Higher Education, 2nd edition, 2001.

12. Loris DAntoni and Margus Veanes. Static analysis of string encoders and decoders. In Proceedings of the 14th International Conference on Verification, Model Checking, and Abstract Interpretation (VMCAI), pages 209-228, 2013.

13. Antonio Filieri, Corina S. Pasareanu, and Willem Visser. Reliability analysis in symbolic pathfinder. In Proceedings of the 35th International Conference on Software Engineering (ICSE), pages 622-631, 2013.

14. Philippe Flajolet and Robert Sedgewick. Analytic Combinatorics. Cambridge University Press, New York, NY, USA, 1 edition, 2009.

15. Vijay Ganesh, Mia Minnes, Armando Solar-Lezama, and Martin C. Rinard. Word equations with length constraints: What's decidable? In Proceedings of the 8th International Haifa Verification Conference (HVC), pages 209-226, 2012.

16. Jonathan L. Gross, Jay Yellen, and Ping Zhang. Handbook of Graph Theory, Second Edition. Chapman & Hall/CRC, 2nd edition, 2013.

17. Pieter Hooimeijer, Benjamin Livshits, David Molnar, Prateek Saxena, and Margus Veanes. Fast and precise sanitizer analysis with bek. In Proceedings of the 20th USENIX Conference on Security, 2011.

18. Pieter Hooimeijer and Westley Weimer. A decision procedure for subset constraints over regular languages. In Proceedings of the ACM SIGPLAN Conference on Programming Language Design and Implementation (PLDI), pages 188-198, 2009.

19. Pieter Hooimeijer and Westley Weimer. Solving string constraints lazily. In Proceedings of the 25th IEEE/ACM International Conference on Automated Software Engineering (ASE), pages 377-386, 2010.

20. Scott Kausler and Elena Sherman. Evaluation of string constraint solvers in the context of symbolic execution. In Proceedings of the 29th ACM/IEEE International Conference on Automated software engineering (ASE), pages 259-270, 2014.

21. Adam Kiezun, Vijay Ganesh, Philip J. Guo, Pieter Hooimeijer, and Michael D. Ernst. Hampi: a solver for string constraints. In Proceedings of the 18th International Symposium on Software Testing and Analysis (ISSTA), pages 105-116, 2009.

22. Donald E. Knuth. The Art of Computer Programming, Volume I: Fundamental Algorithms. Addison-Wesley, 1968.

23. Guodong Li and Indradeep Ghosh. PASS: string solving with parameterized array and interval automaton. In Proceedings of the 9th International Haifa Verification Conference (HVC), pages 15-31, 2013.

24. Tianyi Liang, Andrew Reynolds, Cesare Tinelli, Clark Barrett, and Morgan Deters. A DPLL(T) theory solver for a theory of strings and regular expressions. In Proceedings of the 26th International Conference on Computer Aided Verification (CAV), pages 646-662, 2014.

25. Loi Luu, Shweta Shinde, Prateek Saxena, and Brian Demsky. A model counter for constraints over unbounded strings. In Proceedings of the ACM SIGPLAN Conference on Programming Language Design and Implementation (PLDI), page 57, 2014.

26. Stephen McCamant and Michael D. Ernst. Quantitative information flow as network flow capacity. In Proceedings of the ACM SIGPLAN Conference on Programming Language Design and Implementation (PLDI), pages 193-205, 2008.

27. Quoc-Sang Phan, Pasquale Malacaria, Oksana Tkachuk, and Corina S. Päsäreanu. Symbolic quantitative information flow. SIGSOFT Softw. Eng. Notes, 37(6):1-5, 2012.

28. Prateek Saxena, Devdatta Akhawe, Steve Hanna, Feng Mao, Stephen McCamant, and Dawn Song. A symbolic execution framework for javascript. In Proceedings of the 31st IEEE Symposium on Security and Privacy, 2010.

29. Geoffrey Smith. On the foundations of quantitative information flow. In Proceedings of the 12th International Conference on Foundations of Software Science and Computational Structures (FOSSACS), pages 288-302, 2009.

30. Richard P. Stanley. Enumerative Combinatorics: Volume 1. Cambridge University Press, New York, NY, USA, 2nd edition, 2011.

31. Takaaki Tateishi, Marco Pistoia, and Omer Tripp. Path- and index-sensitive string analysis based on monadic second-order logic. In Proceedings of the International Symposium on Software Testing and Analysis (ISSTA), pages 166-176, 2011.

32. Minh-Thai Trinh, Duc-Hiep Chu, and Joxan Jaffar. S3: A symbolic string solver for vulnerability detection in web applications. In Proceedings of the ACM SIGSAC Conference on Computer and Communications Security (CCS), pages 1232-1243, 2014.

33. Inc. Wolfram Research. Mathematica, 2014.

34. Pierre Wolper and Bernard Boigelot. On the construction of automata from linear arithmetic constraints. In Proceedings of the 6th International Conference on Tools and Algorithms for Construction and Analysis of Systems (TACAS), pages 1-19, 2000.

35. Fang Yu. Automatic Verification of String Manipulating Programs. PhD thesis, University of California, Santa Barbara, 2010.

36. Fang Yu, Muath Alkhalaf, and Tevfik Bultan. Stranger: An automata-based string analysis tool for php. In Proceedings of the 16th International Conference on Tools and Algorithms for the Construction and Analysis of Systems (TACAS), pages 154-157, 2010.

37. Fang Yu, Muath Alkhalaf, and Tevfik Bultan. Patching vulnerabilities with sanitization synthesis. In Proceedings of the 33rd International Conference on Software Engineering (ICSE), pages 131-134, 2011.

38. Fang Yu, Muath Alkhalaf, Tevfik Bultan, and Oscar H. Ibarra. Automata-based symbolic string analysis for vulnerability detection. Formal Methods in System Design, 44(1):44-70, 2014.

39. Fang Yu, Tevfik Bultan, Marco Cova, and Oscar H. Ibarra. Symbolic string verification: An automata-based approach. In Proceedings of the 15th International SPIN Workshop on Model Checking Software (SPIN), pages 306-324, 2008.

40. Yunhui Zheng, Xiangyu Zhang, and Vijay Ganesh. Z3-str: A z3-based string solver for web application analysis. In Proceedings of the 9th Joint Meeting on Foundations of Software Engineering (ESEC/FSE), pages 114-124, 2013.