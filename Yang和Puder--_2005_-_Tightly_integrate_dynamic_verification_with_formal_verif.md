# Tightly Integrate Dynamic Verification With Formal Verification: A GSTE Based Approach

Jin Yang, Avi Puder

Design Technology, Intel Corporation

\{jin.yang, avi.puder\}@intel.com

December 2, 2004

## Abstract

GSTE (Generalized Symbolic Trajectory Evaluation) is a high capacity formal verifi cation technology that has been successfully applied to verifying complex Intel designs with tens of thousands of state elements. In this paper, we extend the use of GSTE by developing a dynamic checker that verifies a GSTE specifi cation against a scalar simulation trace. Unlike previous approaches, both the formal checker and the dynamic checker work directly on a GSTE specifi cation without the need for an intermediate monitor circuit. Our approach also offers a straight forward way to measure the quality (coverage) of a specification. The dynamic checker has been used in the real-life micro-processor design verifi cation.

## I. INTRODUCTION

Formal verifi cation and dynamic verifi cation are two complementary technologies for hardware verifi cation. Formal verifi cation fully proves the correctness of a design under ver-ifi cation (DUV) but often has the capacity limitation to handle complex designs. Dynamic verifi cation, on the other hand, can scale to the entire micro-processor but cannot cover every possible case. In recent years, researchers from both industry and academia have come to realize the importance of integrating these two technologies. Existing integration approaches are largely monitor based e.g. in $\left\lbrack  {8,{14},{11},3,9}\right\rbrack$ . In particular, a specification for the DUV is complied into a monitor circuit and connected with the DUV. During verifi cation, the monitor raises a red flag if and only if the DUV does not satisfy the specification. There are two drawbacks. First, the monitor construction is likely to incur additional complexity, which could further limit the capacity of formal verifi cation. Second, it is often harder to explain a bug in the context of the original specifi cation, as the monitor does not necessarily preserve the structure of the specification.

In this paper, we take a different approach. We start with GSTE, a formal verifi cation technology that combines the high capacity of symbolic simulation based STE with the expressive power of classic symbolic model checking [12]. GSTE has been successfully used in verifying complex industrial designs with tens of thousands of state elements (latches) [13, 4, 10].

Since GSTE is simulation based, it has a very natural connection to dynamic verifi cation. In particular, A GSTE specifi cation is given as an assertion graph, a sequential program that provides both sequential stimuli for simulating the DUV and the expected responses. We explore such a natural connection and develop a dynamic checker for GSTE. The checker directly verifi es an assertion graph against a dynamic simulation trace by moving tokens along paths in the graph. Such a direct approach makes it much easier to identify where and how the graph is violated when a failure is detected. Further, it does not restrict the capacity of the formal checker. Rather, it provides a fast prototyping and validation tool in the development of the specification, and offers a practical metric for measuring the quality of the specifi cation (rather than the simulation coverage with respect to the specification). The integration is seamless, as both checkers are based on the same simulation semantics. Finally, it allows an unlimited number of tokens, and thus provides an unconstrained runtime flexibility.

Being a software solution, however, our approach does have its own limitations. First, it cannot be emulated in hardware. Second, it cannot be used as an assumption in the formal veri-fi cation of another assertion graph in the context of [7].

The rest of the paper is organized as follows. In Section II, we describe the syntax and the simulation based semantics of an assertion graph. In Section III, we present the dynamic checker algorithm. We also show how to turn the algorithm into a stand-alone dynamic verifi cation solution. In Section IV, we discuss how to extend the algorithm to measure the coverage of an assertion graph with respect to a simulation trace and vice versa, and discuss several additional features. In Section V, we show the results from two real-life applications. We conclude the paper in Section VI.

II. ASSERTION GRAPH: SYNTAX AND SEMANTICS

An assertion graph in GSTE is a five-tuple $G =$ $\left( {V,{v}_{0}, E\text{, ant, cons}}\right)$ , where

1. $V$ is a set of vertices with ${v}_{0}$ being the initial vertex,

2. $E$ is a set of directed edges, and

3. ant and cons are functions that map each edge to a state predicate called antecedent and a state predicate called consequent, respectively.

A path in the graph of length $k$ is a consecutive edge sequence $\rho  = \left\lbrack  {{e}_{1},{e}_{2},\ldots ,{e}_{k}}\right\rbrack$ such that

1. source $\left( {e}_{1}\right)  = {v}_{0}$ , i.e., ${e}_{1}$ starts from ${v}_{0}$ , and

2. for all $1 \leq  i < k,\operatorname{sink}\left( {e}_{i}\right)  = \operatorname{source}\left( {e}_{i + 1}\right)$ , where $\operatorname{sink}\left( {e}_{i}\right)$ is the vertex edge ${e}_{i}$ ends at.

As an example, consider the following simple two-bit adder with a clock signal $c$ and an external stall signal $s$ .

![0196f2ec-33a0-7e10-b245-e712aae79dfe_1_296_568_440_217_0.jpg](images/0196f2ec-33a0-7e10-b245-e712aae79dfe_1_296_568_440_217_0.jpg)

Figure 1: Simple Adder

The specification for the adder is the assertion graph in Figure 2, where the predicate above an edge is the antecedent on the edge, and the one below is the consequent on the edge. The labels on the two bottom edges are both antecedents. The trivial predicate true is ignored from the graph. Symbols ${A}_{1}$ , ${A}_{0},{B}_{1},{B}_{0}$ in a predicate are rigid boolean variables (i.e., variables whose values never change) called symbolic constants. Intuitively, the specification says that if the two inputs of the adder ${a}_{1}{a}_{0}$ and ${b}_{1}{b}_{0}$ have values ${A}_{1}{A}_{0}$ and ${B}_{1}{B}_{0}$ when the stall signal $s$ is low, then the output of the adder ${g}_{1}{g}_{0}$ must be ${A}_{1}{A}_{0} + {B}_{1}{B}_{0}$ when the stall signal is low next time.

![0196f2ec-33a0-7e10-b245-e712aae79dfe_1_193_1294_647_317_0.jpg](images/0196f2ec-33a0-7e10-b245-e712aae79dfe_1_193_1294_647_317_0.jpg)

Figure 2: Adder Specifi cation

To give a more precise semantics of an assertion graph, we introduce a simple circuit model. A circuit consists of a set of signals $S$ . A state $s$ of the circuit is an assignment to all the signals in $S$ . For the purpose of this paper, we defi ne the behavior of the circuit as a suffix-closed infi nite set of simulation traces. A simulation trace (or simply trace) is an infi nite state sequence $\tau  = \left\lbrack  {{s}_{1},{s}_{2},\ldots }\right\rbrack$ . A set of traces is suffix-closed if every suffix of a trace in the set is also in the set.

Formally, we say trace $\tau$ satisfies a path $\rho  = \left\lbrack  {{e}_{1},{e}_{2},\ldots ,{e}_{k}}\right\rbrack$ in assertion graph $G$ , if for any given scalar assignment $V$ to the symbolic constants, $\tau$ satisfi es the antecedent sequence on $\rho$ under $V$ implies it also satisfi es the consequent sequence on $\rho$ under $V$ . Trace $\tau$ satisfies $G$ , if $\tau$ satisfi es every path in $G$ . A circuit satisfies $G$ , if every trace of the circuit satisfies $G$ .

[12] presented a model checking algorithm for verifying an assertion graph against a circuit modeled as a fi nite state machine. It also presented GSTE in a more powerful setting.

### III.A DYNAMIC CHECKER ALGORITHM

Although GSTE was developed as a high capacity formal verifi cation solution, there are many practical values to develop a semantically consistent dynamic verifi cation solution as we mentioned in Section I. For one, the dynamic solution provides a quick sanity check of the specifi cation in formal verifi cation. For another, it enables the re-use of the specifi cation when the capacity of GSTE becomes an issue.

Here we present an effi cient checker algorithm for directly validating an assertion graph against a simulation trace. Based on the satisfi ability semantics in the previous section, the algorithm extends and checks every path in the graph against the trace and each of its suffixes. It stops the action when either an antecedent violation or a consequent violation is discovered.

While checking an antecedent on the path against the corresponding state in the trace, a boolean constraint may be imposed to the symbolic constants. For instance, to satisfy

$$
c = 0 \land  s = 0 \land  {a}_{1}{a}_{0} = {A}_{1}{A}_{0} \land  {b}_{1}{b}_{0} = {B}_{1}{B}_{0}
$$

in state

$$
\left\lbrack  {c = 0, s = 0,{a}_{1} = 0,{a}_{0} = 1,{b}_{1} = 1,{b}_{0} = 0,\ldots }\right\rbrack  ,
$$

constraint $\neg {A}_{1} \land  {A}_{0} \land  {B}_{1} \land  \neg {B}_{0}$ must hold for the symbolic constants. Such constraints are accumulative, i.e., the global constraint for the symbolic constants on the last edge in the path is the conjunction of all the constraints on the edges along the path. The consequence on the last edge is only checked under the global constraint. In practice, these constraints are usually quite simple, often with a size linear to the number of symbolic constants.

Obviously, it would be practically impossible to explicitly construct all possible paths in the graph (e.g., by unfolding the graph in a tree). Fortunately, for each path being checked, the only things to be remembered from the history at any given simulation step are (1) the last edge on the path that has been checked, and (2) the constraint on the symbolic constants accumulative so far. Furthermore, since the graph has a fi nite structure, many paths will converge to the same vertex and from there expand along every outgoing edge. Therefore, the algorithm only keeps track of, at any given moment, the set of active vertices that are the sinks of the last checked edges. If a vertex appears on multiple paths, the global constraint on the symbolic constants it inherits from the paths is the disjunction of the global constraints on the last edges in these paths. Figure 3 lists the entire checker algorithm.

Some explanations of the algorithm are in order. For each active vertex $v, v$ .constr records the global constraint at the current moment. Variable edge_constr records the global constraint for the edge being checked. The evaluation function eval takes a state and a predicate, and substitutes each signal in the predicate by its value in the state, as illustrated in the example at the beginning of the section. As another example, the same predicate would be evaluated to false in state

$$
\left\lbrack  {c = 0, s = 1,\ldots }\right\rbrack  \text{.}
$$

---

Algorithm: dynamic_check $\left( {G,\tau ,{t}_{\max }}\right)$

	$t \mathrel{\text{:=}} 0$ ;

	${v}_{0}$ .constr :=true;

	. active $\mathrel{\text{:=}} \left\{  {v}_{0}\right\}$ ;

	while $t \leq  {t}_{\max }$

	next_active $\mathrel{\text{:=}} \left\{  {v}_{0}\right\}$ ;

		for each $v$ in active

		for each outgoing edge $e$ from $v$

		edge_constr := eval(τ[t], v.constr ∧ ant(e));

		if edge_constr $\neq$ false

			if $\operatorname{eval}\left( {\tau \left\lbrack  t\right\rbrack  ,\text{edge_constr}\land \neg \operatorname{cons}\left( e\right) }\right)  \neq$ false

				return failure;

			elseif $\operatorname{sink}\left( e\right)  \in$ next_active

					$\operatorname{sink}\left( e\right)$ .constr : $= \operatorname{sink}\left( e\right)$ .constr $\vee$ edge_constr;

			else

				sink(e).constr : $=$ edge_constr;

				add $\operatorname{sink}\left( e\right)$ to next_active;

			endif

		endif

19. endfor

20. endfor

21. active := next_active;

22. $t \mathrel{\text{:=}} t + 1$ ;

23. endwhile

	1. return(true);

end.

---

Figure 3: Dynamic Checker Algorithm for Assertion Graph

Initially, the initial vertex is the only one in the active set active (step 1 through step 3 in the algorithm), and the global constraint on the vertex is always true. The algorithm will go through every step in the simulation trace $\tau$ until either a failure is detected or a given step bound ${t}_{\max }$ is reached (step 4). At each simulation step (step 5 through step 22), the algorithm examines every vertex $v$ in the current active set. It checks every outgoing edge $e$ of the vertex. The antecedent on the edge is evaluated in the current state in $\tau$ at step 8, and the global constraint on the edge is computed. If the constraint is false, then an antecedent failure is detected and no further search is needed. Otherwise, if the negation of the consequent is satis-fi able in the state under the global constraint at step 10 , then a bug is discovered and the algorithm terminates. Otherwise, the sink vertex of $e$ is put into the active set for the next simulation step next_active at step 16 if it is visited first time, or its associated global constraint is properly updated at step 13 . Note that the initial vertex is always in the active set (step 5) so that every suffix of $\tau$ can be checked.

Note that in the algorithm the global constraint for each vertex is expressed in a symbolic form such as BDD (Binary Decision Diagram) or BED (Binary Expression Diagram). When used in a more traditional dynamic verifi cation framework, however, we can replace the vertex and its symbolic constraint by the copies (tokens) of the vertex, each of which carrying a scalar assignment to the symbolic constants.

This dynamic checking algorithm can be quite easily turned into a stand-alone dynamic verifi cation solution. Instead of checking the antecedent against an existing simulation trace at any given time, a stimulus is randomly generated by a BDD based constraint solver to satisfy the antecedent and injected into the circuit for on-the-fly simulation. The constraint (context) carried by a token makes sure that a global constraint over time is satisfi ed. In addition, a user can specify how paths in the graph should be explored, either by some probabilistic scheme or along a set of given paths.

## IV. QUALITY ANALYSIS AND ADDITIONAL FEATURES

In formal verifi cation, the quality of the specification being verifi ed is very important. There are two important quality measurement matrices. The first one is case coverage, i.e., whether the specification covers all possible interesting behaviors of the circuit under verifi cation [6]. The second is vacuity detection, i.e., whether some assumptions in the specification are vacuously true [2]. These two quality checks can be naturally defi ned for an assertion graph. Given a simulation trace, we say that an active vertex in the graph does not miss a case at a given time, if the antecedent on at least one of its outgoing edges is satisfi ed in the state of the trace at the given time (i.e., the state is being checked). We say that an edge in the graph is vacuously true, if the antecedent on the edge is never satisfi ed during the checking of the simulation trace.

The algorithm in Figure 3 can be readily extended to check these two quality measurements. For the case coverage, we simply check for every active vertex $v$ at every simulation step, there is at least one outgoing edge $e$ of $v$ that can be satis-fi ed in the current state of the trace. More precisely, we add a boolean attribute $v$ .cover to vertex $v$ , which is initialized to false (v.cover $=$ false) as the beginning of every loop at step 6. When an outgoing edge from $v$ is satisfi ed at step $9, v$ .cover is set to true (v.cover $=$ true). At the end of the loop right before step 20, we check if v.cover is true. If this is not the case, then a missing case is discovered for $v$ .

For the vacuity check, we add a counter attribute e.count to each edge $v$ in the graph. The attribute is initialized to 0 at the very beginning of the algorithm. Whenever the edge $v$ is examined and satisfi ed at step 9 , the counter increments its value by 1 (i.e., e.count $=$ e.count + 1). At the successful termination of the algorithm, edges with counter value 0 are reported as being possibly vacuously true.

The vacuity check can also be turned around to use in the coverage analysis and biased test generation in the dynamic verifi cation. For instance, an edge that is never satisfi ed indicates a coverage hole in the simulation trace, and the antecedent on the edge can be used to guide a on-the-fly biased test generation to cover the edge using any existing constraint solving technique. If no test can be generated to satisfy the antecedent, then a potential conflict (vacuity) is identified.

In addition to the quality coverage analysis, there are other extensions and features added to the dynamic checker algorithm which we shall only summarize in this paper. We have extended the algorithm to deal with a richer specifi cation language which allows a modularized specification using a set of assertion graphs to interact and constrain each other. We have also extended the vertex and edge based coverage analysis to paths in an assertion graph. Finally, we have added a counterexample generation capability that produces a path in the assertion graph that has been violated and a starting time in the simulation trace for the violation to happen.

## V. IMPLEMENTATION AND RESULTS

We have implemented a prototype of the dynamic checker in the FORTE environment using a functional language FL [1]. The checker runs lock-step with a dynamic simulation engine and reads the simulation trace generated by the engine on-the-fly. The state predicates and constraints are represented as BDDs [5]. The evaluation function eval is also implemented as BDD manipulations. The dynamic checker is tightly integrated with other components in FORTE. For instance, it uses the GUI and the debugging facility in FORTE to annotate and navigate an assertion graph with Edge/Path coverage results to ease the debugging process for large AG.

The dynamic checker has been applied to the real-life verifi - cation of micro-processor designs in Intel. In particular, it has been used for sanity check and quick validation of an assertion graph before serious effort is invested in complete formal verifi cation. Here we report the results from two such applications. Both were run on a ${2.8}\mathrm{{GHz}}$ Pentium(R) 4 processor-based computer with $1\mathrm{\;{GB}}$ memory running the Linux operating system. In both cases, missing cases were discovered and the specifications were augmented to cover them.

The first application was the verifi cation of the uop conservation property in a scheduler FIFO unit with roughly 5000 state elements (latches). The assertion graph for the property contains 46 vertices and 209 edges. 50 random dynamic simulation runs were conducted with the checker watching on the side. The average run time was 100 seconds for 10,000 cycles. The total coverage from the 50 runs was 50 percent of the edges in the assertion graph.

The second application was the verification of a property on a cache coherency logic, also with roughly 5000 state elements. The assertion graph for the property includes 10 vertices and 29 edges. 100 random dynamic simulation runs were conducted, which covered 96 percent of the edges in the assertion graph and more than 4000 different paths. The average run time was 160 seconds for 10,000 cycles.

## VI. CONCLUSION

In this paper, we presented a checker algorithm for directly verifying an assertion graph specifi cation against a dynamic simulation trace, and thus tightly integrated the dynamic ver-ifi cation capability into the GSTE based formal verifi cation framework. Such an integration is based on a single consistent simulation based semantics for assertion graphs, and enables specifi cation re-use. It preserves the efficiency and high capacity of the GSTE model checker for formal verifi cation and provides an effi cient native checker solution for dynamic verifi cation. Future work includes semi-formal methods for GSTE as well as improving the performance of the checker when needed.

## REFERENCES

[1] M. Aagaard, R. Jones, T. Melham, J. O'Leary, and C.-J. Seger. A methodology for large-scale hardware verifi cation. In FM-CAD'2000, November 2000.

[2] R. Armoni, L. Fix, A. Flaisher, O. Grumberg, N. Piterman, A. Tiemeyer, and M. Vardi. Enhanced vacuity detection in linear temporal logic. In IEEE International Conference on Computer-Aided Design, pages 368-380, 2003.

[3] L. Bening and H. Foster. Principles of Verifi able RTL Design: A Functional Coding Style Supporting Verifi cation Processes in Verilog. Kluwer Academic Publishers, 2001.

[4] B. Bentley. High level validation of next generation microprocessors. In IEEE International High-Level Design, Validation, and Test Workshop, 2002.

[5] R. Bryant. Graph-based algorithms for Boolean function manipulation. IEEE Trans. on Computers, C-35(8):677-691, August 1986.

[6] Y. Hoskote, T. Kam, and X. Z. P.-H. Ho. Coverage estimation for symbolic model checking. In Design Automation Conference, pages 300-305, 1999.

[7] A. Hu, J. Casas, and J. Yang. Reasoning about gste assertion graphs. In Formal Methods in CAD'2003, 2003.

[8] M. Kaufmann, A. Martin, and C. Pixley. Design constraints in symbolic model checking. In 10th International Conference on Computer-Aided Verifi cation (LNCS 1427), pages 477-487. Springer, 1998.

[9] M. Oliveira and A. Hu. High-level specifi cation and automatic generation of ip interface monitors. In 39th ACM/IEEE Design Automation Conference, pages 129-134, 2002.

[10] T. Schubert. High-level formal verifi cation of next generation micro-processors. In 40th ACM/IEEE Design Automation Conference, 2003.

[11] K. Shimizu, D. Dill, and A. Hu. Monitor-based formal specifi - cation of of pci. In Formal Methods in Computer-Aided Design (LNCS1954), pages 335-353. Springer, 2000.

[12] J. Yang and C.-J. Seger. Introduction to generalized symbolic trajectory evaluation. In Proc. of ICCD, September 2001.

[13] J. Yang and C.-J. Seger. Generalized symbolic trajectory evaluation - abstraction in action. In FMCAD'2002, pages 70-87, November 2002.

[14] J. Yuan, K. Shultz, C. Pixley, H. Miller, and A. Aziz. Modeling design constraints and biasing in simulation using bdds. In IEEE/ACM International Conference on Computer-Aided Designs, pages 584-589. Springer, 1999.