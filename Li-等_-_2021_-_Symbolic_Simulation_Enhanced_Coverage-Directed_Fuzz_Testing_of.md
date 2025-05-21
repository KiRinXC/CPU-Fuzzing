# Symbolic Simulation Enhanced Coverage-Directed Fuzz Testing of RTL Design

Tun Li, Hongji Zou, Dan Luo, Wanxia Qu

College of Computer Science and Technology, National University of Defense Technology,

Changsha, Hunan, China, 410073

Email: \{tunli, zouhongji, luodan, quwanxia\}@nudt.edu.cn

Abstract-With the ending of Moore's Law and Dennard scaling, modern System-on-a-Chip (SoC) trends to incorporate a large and growing number of specialized modules for specific applications. Verification is vital to the RTL design and faces new challenges due to the growing design complexities. In this paper, we proposed a symbolic simulation enhanced coverage-directed dynamic verification technique for RTL designs. We proposed novel Full Multiplexer Toggle Coverage (FMTC) to trace and provide feedback to the verification process. The proposed method is a hybrid between symbolic simulation and mutation based fuzz testing that offsets the disadvantages of both. The achievement of high coverage is obtained by interleaved symbolic simulation and fuzz testing passes. The symbolic simulation pass is used to generate tests that direct the testing to untouched corners. While the mutation based fuzz testing pass is used to leverage test generation tasks and to enable the method to deal with large scale designs. The empirical evaluation of the method shows promising results on archiving high coverage for practical designs.

## I. INTRODUCTION

With the ending of Moore's Law and Dennard scaling, modern $\mathrm{{SoC}}$ trends to incorporate a large and growing number of specialized modules for specific applications. This growing complexity has led to a design productivity crisis. Some novel hardware description languages (HDLs), which are usually embedded in some modern programming languages as domain-specific languages for hardware (HDSLs), are developed to improve design productivity, such as Chisel[2] on Scala, PyRTL[3] and PyMTL[4] on Python, and $\mathrm{C}\lambda \mathrm{{aSH}}\left\lbrack  5\right\rbrack$ on Haskell. HDSLs are also a kind of hardware construction language (HCL), which means the designs are elaborated-through-execution. To implement elaborate-through-execution mechanics, HDSLs usually defines an intermediate representation (IR) for RTL modeling which contains a complete set of operations and structures for the description and manipulation of hardware. The designs described in HDSLs are usually converted to some predefined intermediate representations (IR) for further optimization and transformations. Therefore, verifying the correctness of the RTL design in IR form has become a new challenge.

All the current works on verification for RTL design in IR form could be classified into two types: formal (static) verification and simulation-based (dynamic) verification. For static verification, CoSA [6] integrates with CoreIR [7] to provide formal analyses by reducing all analyses to symbolic model-checking problems, which is solved by integrating with K-Induction[8]/Interpolation[9] and K-Liveness techniques[10]. For dynamic verification, Kevin et al proposed [11] a coverage-guided mutational fuzz testing method for designs in FIRRTL [12] IR format, which combines mutation-based test generation and FPGA-accelerated RTL simulation techniques. The method finally implemented as an open source fuzz testing tool called RFUZZ[13]. However, RFUZZ only implements blindly fuzz techniques which make it difficult to achieve high testing coverage. Vladimir et al [14] proposed a coverage-guided fuzzing technique for verifying instruction set simulator (IIS). However, IIS is more like a software than a RTL design. To the best of our knowledge, the RFUZZ is the first work that applying fuzz testing to RTL design constructed by HCLs.

In this paper, we enhance coverage-directed dynamic verification techniques of RTL designs presented in HDSL IR format in several aspects. First, we define Full Multiplexer Toggle Coverage (FMTC) to enhance the coverage criterion of testing. Second, the seed inputs generation process is enhanced with symbolic simulation and cone-of-influence (COI) analysis[15]. Finally, the achievement of high coverage is enhanced by iteratively interleaved symbolic and fuzz testing passes, which makes best use of the results of concrete simulation and direct test generation to cover the uncovered corners.

To the best of our knowledge, there is no previous work on combining symbolic simulation and fuzz testing for RTL designs in HDSL IR form. We believe that the proposed method is a novel and interesting design point in the space of solution to the novel verification problem. Although the method is proposed and implemented for PyRTL IR form, it could be easily applied to other IR formats such as FIRRTL and CoreIR.

## II. BACKGROUND

## A. Overview of PyRTL and Its IR format

PyRTL is a Python embedded RTL hardware design language for fast implementation and simulation of RTL designs. PyRTL defines a collection of python classes such as WireVector, LogicNet, which allows to model RTL designs. Fig. 1 shows a simple PyRTL example simplified from "example3" in PyRTL 0.8.7 distribution, which is a state machine with 2 inputs, 2 outputs, a 2-bit register and an implied clock. Fig. 2 shows the IR form constructed by executing the program in Fig.1. Readers could refer to [3] for detailed explanation.

---

This work was supported in part by the National Natural Science Foundation of China under Grant U19A2062 and the National Key R&D Program of China (2018YFB0204301). (Corresponding author: Tun Li)

---

![0196f2c4-8003-735e-8e2f-96cce13b0774_1_303_311_404_394_0.jpg](images/0196f2c4-8003-735e-8e2f-96cce13b0774_1_303_311_404_394_0.jpg)

Fig. 1. A Sample PyRTL design.

![0196f2c4-8003-735e-8e2f-96cce13b0774_1_187_812_644_410_0.jpg](images/0196f2c4-8003-735e-8e2f-96cce13b0774_1_187_812_644_410_0.jpg)

Fig. 2. The Corresponding PyRTL IR.

In PyRTL, all the WireVector and LogicNet objects are organized in a Block object. As shown in Fig. 2, WireVector objects are used to represent wires that connect LogicNet objects. Each LogicNet object has a primitive operation, which takes some WireVector objects as its inputs and produces output as the computation result of primitive operation.

The major data structures used to represent a circuit are WireVectors and LogicNet classes. The WireVector objects are just used to represent connecting lines between LogicNet objects, whereas LogicNet objects are used to represent various logic modules at RTL. Each LogicNet object is a 4-tuple and is defined as follow.

$$
\text{(operation, parameters, args, dests)} \tag{1}
$$

Here operation and parameters define the function and the set of any additional parameters of a LogicNet object, respectively, whereas args and dests define the set of WireVector objects connected as inputs and driven as outputs for the operation, respectively. For example, the tuple for the multiplex with output denoted by tmp24 is defined as follow, where ${}^{\prime }{tmp}{3}^{\prime }$ is the selection signal:

$$
\left( {{}^{\prime }{x}^{\prime },\text{ None },\left( {{}^{\prime }{\text{ tmp 3 }}^{\prime }{,}^{\prime }{\text{ tmp 23 }}^{\prime },1}\right) {,}^{\prime }{\text{ tmp 24 }}^{\prime }}\right)  \tag{2}
$$

All the multiplexers are organized at the bottom of Fig. 2. Note that a multiplexer can take both word-level and bit-level signals as its inputs and outputs. However, its select signal is always at bit-level.

## B. Coverage-Directed Testing and Motivations

In coverage-directed testing, the execution of a design under testing (DUT) is driven by concrete test inputs (stimuli). While at the same time, a feedback mechanism is attached to record the testing executions and check whether coverage goal is achieved. If no new coverage is achieved, new inputs are generated by using some strategies, such as various mutation techniques when embedding fuzz testing.

We have implemented the mutation based fuzz testing proposed in [11] and carried out experiments with several intuition examples. We observe in our experiments that there are several limitations of the technique which motivates the work presented in this paper.

Algorithm 1 Enhanced Coverage-Directed Testing Approach

---

Require:

		The design under testing $d$

		The coverage target $T$

## Ensure:

		A set of generated test inputs

		$S \leftarrow$ GenerateSeeds(d)

		totalCoverage $\leftarrow  \varnothing$

		target $\leftarrow  T$

		testGenerated $\leftarrow  \varnothing$

		repeat

			for input in $S$ do

					candidate $\leftarrow$ Mutate(input) + \{input\}

					for cincandidate do

						covered, partialCovered $\leftarrow  {RUN}\left( {d, c}\right)$

						if coverage not in totalCoverage then

							testGenerated $\leftarrow$ testGenerated $+ \{$ input $\}$

							totalCoverage $\leftarrow$ totalCoverage + coverage

						end if

						uncovered $\leftarrow$ target - totalCoverage

						$S \leftarrow$ generateNewInputs $(d$ ,

											partialCovered, uncovered, $S$ )

					end for

			end for

			until (given time budget expired) or (uncovered $=  = \varnothing$ )

		return testGenerated

---

First, the initial seed inputs are randomly generated, which may waste much simulation effort to get to the coverage target. Second, although [11] claimed that the mutation is directed by coverage feedbacks, we found that the mutations are applied blindly without taking the coverage feedbacks, such as the uncovered multiplexers, into account.

Finally, the Mux Toggle Coverage (MTC) coverage metrics only requires the control signal value of a multiplexer to be a 0-1 or 1-0 toggle. We believe that traversing an edge twice instead of once is novel and interesting. Therefore, we enhance the MTC by defining Full Multiplexer Toggle Coverage (FMTC): A multiplexer achieves a FMTC in a single testing means that its control signal value should experience a 0-1-0 or a 1-0-1 toggle.

### III.The Proposed Method

The proposed method is listed in Algorithm 1 with the enhancements highlighted in gray.

The first improvement (line 1) is seed inputs generation according to the circuit structure of a DUT. First the COI of each multiplexer is computed. Then, multiplexers with the greatest COI set are selected, and tests that could cover the selected multiplexers are generated using symbolic simulation and constraint solving techniques.

The second improvement (line 7) is generating tests from each seed input by mutation to get lots of tests with a lightweight method. We adopt two types of mutators from AFL [16]: deterministic and non-deterministic havoc. Deterministic mutators mutate every position in the input, whereas nondeterministic mutators mutate random position.

The third part (lines 9, 15) is generating tests for uncovered or partially covered multiplexers using symbolic simulation techniques. A uncovered multiplexer is the one that whose control signal is always 0 or 1 during a single testing, whereas a partially covered multiplexer (denoted by partialCovered) is the one whose control signal has toggled from 0 to 1 , or from 1 to 0 but not toggled to 0 or 1 during a single testing, respectively.

In follows, we will introduce the key aspects of the proposed method in details.

## A. Cycle-Based Symbolic Simulation of PyRTL IR

The cycle-based symbolic simulation algorithm is implemented by unrolling the DUT to the desired time steps. Then at each time step, the symbols of primary inputs at current time step and the expression of the state signals at the previous time step form the inputs of the current input expressions. Therefore, at the last time step, the expressions of the primary outputs is formed by all the input symbols at all the time steps and state expressions at all the previous time steps.

Therefore, the key concern is how to construct expressions for a design in PyRTL IR form. To accomplish this, we first set up a set of transformation rules to transform LogicNet objects into constraint expressions. For example, for the multiplexer in Formula (2), the corresponding SMT constraint is as follow:

(assert (ite $\left( { = \text{tmp3 #b0)}\left( { = \text{tmp24 tmp23}}\right) \left( { = \text{tmp24 #b1}}\right) }\right)$ )

Then, the DUT is unrolled and transformed to a pure combinational logic circuit by assigning each signal with a time step index. Therefore, the result of each symbolic simulation pass is a set of SMTLIB2 constraints reflecting the connection relation among LogicNet objects of the unrolled design according to the transformation rules.

## B. Seed Inputs Generation

Here we try to generate seed inputs to stimulate as many multiplexers as possible with as few seed inputs as possible. We proposed a method to statically evaluate and search the multiplexers which will impact the most other multiplexers when being stimulated. The evaluation is based on COI calculation. and the result is a set of LogicNet objects.

Let $V, I$ and $U$ be the sets of internal signals, primary inputs and logic units of a DUT, respectively. For the given set of signals $\bar{V} \subseteq  V$ and for signal ${v}_{i} \in  \bar{V}$ is the control signal of a multiplexer that is of interest, the COI of $\bar{V}$ (denoted by $\operatorname{COI}\left( \bar{V}\right)$ ) is the minimal set of signals such that

- $\bar{V} \subseteq  {COI}\left( \bar{V}\right)$ ;

- If for some ${v}_{i} \in  {COI}\left( \bar{V}\right)$ , the set of inputs signals of $u \in  U$ that driven ${v}_{i}$ as outputs, denoted by ${DEP}\left( {v}_{i}\right)$ , Then ${DEP}\left( {v}_{i}\right)  \subseteq  {COI}\left( \bar{V}\right)$ .

By making $\bar{V}$ initially contain the control signal of a multiplexer $m$ , the $\operatorname{COIM}\left( m\right)$ is obtained by applying a mapping operation on the set $U$ with the criterion that for any $u \in  U$ whose inputs are in $\operatorname{COI}\left( \bar{V}\right)$ and $u$ is a multiplexer. This type of COI is denoted by $\operatorname{COIM}\left( m\right)$ , where $m$ is a multiplexer.

We (randomly) select a multiplexer $m$ with the top ${25}\%$ greatest $\left| {{COIM}\left( m\right) }\right|$ for seed generation, where $\left| \cdot \right|$ returns the number of elements in the set $\left| {{COIM}\left( m\right) }\right|$ . Next, the DUT is symbolically simulated for $k$ cycles, where $k$ is the level of $m$ or is set by testers.

For the selected multiplexer, we construct constraints that hope its control signal to take a "0-1-0" or "1-0-1" toggle. The final constraint is then constructed by combining the toggle constraints and the expression of symbolic simulation output. Finally, the combined constraint is applied to Z3 [17] solver and the solutions on primary inputs of the DUT is the seed inputs for the consequent testing. The seed inputs generation pass will be ended when one solution is returned.

In practice, the toggle may have much combinations, such as several 0s followed by several 1s and finally flipped to 0 . To reduce the call of Z3 solver, for the given unrolling time step $k$ , where $k \geq  3$ , we restrict the toggle to be $k - 2$ patterns such as ${}^{\prime }{00}\ldots {10}^{\prime },{00}\ldots {100}^{\prime },\ldots$ , and ${}^{\prime }{0100}\ldots {0}^{\prime }$ , or ${}^{\prime }{11}\ldots {01}^{\prime },{11}\ldots {011}^{\prime },\ldots$ , and ${}^{\prime }{1011}.{.1}^{\prime }$ . Finally, all the pattern-constraints will be connected with an "or" operation to let $\mathrm{Z}3$ solver explore all the possible solutions.

The seed is for multiple cycles. For the convenience of applying mutational fuzz testing, we concatenate the solution for each cycle in sequence. When applying the concatenated seed to concrete simulation, an inner split operation is used to assign each input with correct value at each time step. The split is just determined by the number of test input bits divided by the number of cycles of a particular test runs.

## C. Coverage Directed Mutation Testing

In the mutational fuzzing process, the seed inputs generated in section III-B is first applied to run a concrete testing. Our concatenated organization of inputs for multiple cycles allows us to directly implement the mutation heuristics from the successful AFL fuzz testing tool. Similar to AFL, every new entry in our test set is mutated sequentially with all the deterministic mutation and non-deterministic havoc mutators. For any mutation that changes the size of the test input, we pad with zeros in order to maintain an input size that is necessary for a single testing.

TABLE I

BENCHMARKS

<table><tr><td>$\mathbf{{Name}}$</td><td>Input Width</td><td>Number of Mux</td><td>Lines of $\mathbf{{PyRTL}}$</td><td>Number of LogicNets</td><td>Number of Cycles</td><td>Our method</td><td>[11]</td><td>Number of Mutations</td><td>Our method</td><td>[11]</td></tr><tr><td>state machine</td><td>2</td><td>7</td><td>30</td><td>57</td><td>3</td><td>28.6%</td><td>14.3%</td><td>108</td><td>71.4%</td><td>57.1%</td></tr><tr><td>simple multiplier</td><td>65</td><td>6</td><td>27</td><td>35</td><td>3</td><td>66.7%</td><td>16.7%</td><td>9922</td><td>100%</td><td>100%</td></tr><tr><td>complex multiplier</td><td>65</td><td>6</td><td>26</td><td>40</td><td>3</td><td>66.7%</td><td>16.7%</td><td>9922</td><td>100%</td><td>100%</td></tr><tr><td>AES</td><td>257</td><td>28</td><td>42</td><td>496</td><td>3</td><td>60.7%</td><td>46.4%</td><td>14564</td><td>75%</td><td>0%</td></tr><tr><td>CPU core</td><td>96</td><td>8</td><td>57</td><td>84</td><td>3</td><td>25%</td><td>0%</td><td>40306</td><td>64.2%</td><td>46.4%</td></tr><tr><td>RISC-V Core</td><td>142</td><td>185</td><td>1700</td><td>5532</td><td>20</td><td>36.4%</td><td>0.16%</td><td>56800</td><td>72.9%</td><td>14.1%</td></tr><tr><td>OpenCore 1200</td><td>293</td><td>654</td><td>6000</td><td>13045</td><td>15</td><td>27.3%</td><td>0.61%</td><td>87900</td><td>54.7%</td><td>10.2%</td></tr></table>

Along each mutation and concrete simulation for a seed input, we record the coverage information for each multiplexer. Besides recording the multiplexer which experienced an FMTC and the one experienced a half toggle is also recorded. All the multiplexers experienced half toggle are kept in a set, denoted by partialCovered. At the same time, the circuit states at the end of the simulation is recorded for further usage.

After all the seed inputs are mutated and simulated, we select a multiplexer from partialCovered for further exploration. At first, the elements in partialCovered are sorted according to $\left| {{COIM}\left( m\right) }\right|$ in descending order. Then, we take the first multiplexer and the corresponding circuit states to initialize a new symbolic simulation. The constraint produced by the symbolic simulation is try to make the selected multiplexer accomplish a full toggle in the next $n$ time steps, which could be set by designers. Then the DUT is initialized with the recorded circuit states and simulated with the newly generated tests derived from Z3 solutions.

The newly generated tests will go through the mutation pass, and if new coverage achieved, the new test is pad to the end of the test corresponding to the recorded states. Finally, if there are multiplexers which have never been toggled. We apply the previous techniques for seed generation for the uncovered multiplexer, and the only difference is that we start symbolic simulation with recorded circuit states to shorten the test time.

## IV. EXPERIMENTAL RESULTS

We implement the proposed method based on PyRTL 0.8.7 [3] and Z3 4.8.6 in about 2000 lines of Python code. There are some utils in PyRTL that could help to manipulate a DUT in PyRTL IR form. With the working_block function, we could easily access all the LogicNets especially all the multiplexers, monitor and record the states of a DUT during simulation, set and reset the states of a DUT, and symbolic simulate a DUT.

We evaluate the proposed method on a range of designs and the evaluation results are shown in Table I. The ${2}^{nd}$ to ${5}^{th}$ columns show the design characteristics of each design. The state machine is the design presented in Fig. 1. The simple and complex multipliers implement multiply using slow and fast shift-and-add algorithm, respectively. The AES is a 128- bit AES encryption/decryption design. While the last 3 designs are a simple CPU core with 7 instructions, a 32-bit RISC-V core with 36 instructions and an 32-bit OpenCore 1200[19] design with 53 instructions.

We compared our method with the method presented in [11] on coverage achievement. The comparison experiments are designed to show the enhancement of our method. The ${6}^{th}$ column of Table I shows the number of cycles symbolic simulation takes. At the same time, we randomly generate seed for the same cycles to make a fair comparison. The ${7}^{th}$ and ${8}^{th}$ columns show the average coverage achieved by seed generated by the techniques presented in III-B and generated randomly [11], respectively. Here we only select the top 25% multiplexers with the highest $\left| {COIM}\right|$ to generate seed by symbolic simulation. The length of the seed inputs generated are for 3 cycles for the first 5 benchmarks, and 20 cycles for RISC-V and 15 cycles for OpenCore 1200. We can conclude that for most of the benchmarks, the testing could achieve higher coverage quickly with seed inputs generated by our method. We can also conclude that the calculation of $\left| {COIM}\right|$ for each multiplexer is necessary to achieve high coverage.

The last 2 columns of Table I show the coverage achieved by the techniques proposed in this paper and presented in [11] with the seed inputs generated using the corresponding techniques, respectively. Base on the experimental results, we can conclude that the proposed method could obtain more promising results on coverage achievement than the one in [11].

## V. CONCLUSION

In this paper, we propose an enhanced coverage-directed testing method for RTL designs. The experimental results show consistent improvements over the state-of-the-art mutational fuzz testing methods on high coverage achievement. In the near future, we will focus on how to make the best use of structure and applications of circuits to further improve the method, and apply it to more real designs.

[1] Y. Lee et al., "An Agile Approach to Building RISC-V Microprocessors," IEEE Micro, vol. 36, no. 2, pp. 8-20, Mar.-Apr. 2016.

[2] J. Bachrach et al., "Chisel: Constructing Hardware in a Scala Embedded Language," DAC'12, pp. 1216-1225, 2012.

[3] J. Clow, e. al., "A pythonic approach for rapid hardware prototyping and instrumentation," FPL'17, pp. 1-7, 2017.

[4] D. Lockhart, G. Zibrat, and C. Batten, "PyMTL: A Unified Framework for Vertically Integrated Computer Architecture Research," MICRO'14, pp. 280-292, 2014.

[5] C. Baaij, "Cλash: From Haskell to hardware," Masters thesis, University of Twente, 2009.

[6] C. Mattarei, M. Mann, C. Barrett, R. G. Daly, D. Huff, P. Hanrahan, "CoSA: Integrated Verification for Agile Hardware Design," Procs. FM-CAD, pp. 1-5, 2018.

[7] R. Daly. CoreIR: A simple LLVM-style hardware compiler. https: //github.com/rdaly525/coreir, 2020.

[8] M. Sheeran, S. Singh, and G. Stalmarck, "Checking safety properties using induction and a sat-solver," In International conference on formal methods in computer-aided design, pp. 127-144, 2000.

[9] K. L. McMillan, "Interpolation and sat-based model checking," CAV'03, pp. 1-13, 2003.

[10] K. Claessen and N. Sörensson, "A liveness checking algorithm that counts," In Formal Methods in Computer-Aided Design (FMCAD), pp. 52-59, 2012.

[11] K Laeufer, J Koenig, D Kim, J Bachrach, and K Sen, "RFUZZ: coverage-directed fuzz testing of RTL on FPGAs," ICCAD'18, Article 28, pp. 1-8, 2018.

[12] Adam M. Izraelevitz, et.al., "Reusability is FIRRTL ground: Hardware construction languages, compiler frameworks, and transformations," IC-CAD 2017, pp. 209-216, 2017.

[13] Kevin Laeufer, RFuzz, https://github.com/ekiwi/rfuzz, accessed 2020.11.04.

[14] Vladimir Herdt, Daniel Groe, Hoang M. Le, and Rolf Drechsler, "Verifying Instruction Set Simulators using Coverage-guided Fuzzing," In Design, Automation & Test in Europe (DATE), pp. 360-365, 2019.

[15] E. M. Clarke, O. Grumberg, and D. A. Peled, Model Checking, The MIT Press, 2000.

[16] M. Zalewski, "American Fuzzy Lop Technical Details," http://lcamtuf.coredump.cx/afl/technical details.txt. (2014). Accessed April, 2019.

[17] Leonardo deMoura and Nikolaj Bjrner, "Z3: An efficient SMT solver," TACAS'08, LNCS vol.4963, pp. 337-340, 2008.

[18] The Satisfiability Modulo Theories Library, http://smtlib.cs.uiowa.edu.Accessed Apr. 2020.

[19] OpenRISC 1200 IP Core Specification, https://opencores.org/projects/or1k_old/openrisc 1200, accessed 2020.11.04.