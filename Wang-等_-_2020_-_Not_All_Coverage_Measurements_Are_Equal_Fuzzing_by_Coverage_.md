# Not All Coverage Measurements Are Equal: Fuzzing by Coverage Accounting for Input Prioritization

Yanhao Wang ${}^{1,2}$ Xiangkun Jia ${}^{3}$ Yuwei Liu ${}^{1,4}$ Kyle Zeng ${}^{5}$

Tiffany ${\mathrm{{Bao}}}^{5}$ Dinghao Wu ${}^{3}$

${}^{1}$ TCA/SKLCS, Institute of Software, Chinese Academy of Sciences ${}^{2}$ QiAnXin Technology Research Institute

bennsylvania State University ${}^{4}$ School of Cyber Security, University of Chinese Academy of Sciences

a State University ${}^{6}$ Cyberspace Security Research Center, Peng Cheng Laboratory

\{wangyanhao, liuyuwei\}@tca.iscas.ac.cn purui@iscas.ac.cn \{tbao, zengyhkyle\}@asu.edu \{xxj56, duw12\}@psu.edu

Abstract-Coverage-based fuzzing has been actively studied and widely adopted for finding vulnerabilities in real-world software applications. With coverage information, such as statement coverage and transition coverage, as the guidance of input mutation, coverage-based fuzzing can generate inputs that cover more code and thus find more vulnerabilities without prerequisite information such as input format. Current coverage-based fuzzing tools treat covered code equally. All inputs that contribute to new statements or transitions are kept for future mutation no matter what the statements or transitions are and how much they impact security. Although this design is reasonable from the perspective of software testing that aims at full code coverage, it is inefficient for vulnerability discovery since that 1) current techniques are still inadequate to reach full coverage within a reasonable amount of time, and that 2) we always want to discover vulnerabilities early so that it can be fixed promptly. Even worse, due to the non-discriminative code coverage treatment, current fuzzing tools suffer from recent anti-fuzzing techniques and become much less effective in finding vulnerabilities from programs enabled with anti-fuzzing schemes.

To address the limitation caused by equal coverage, we propose coverage accounting, a novel approach that evaluates coverage by security impacts. Coverage accounting attributes edges by three metrics based on three different levels: function, loop and basic block. Based on the proposed metrics, we design a new scheme to prioritize fuzzing inputs and develop TortoiseFuzz, a greybox fuzzer for finding memory corruption vulnerabilities. We evaluated TortoiseFuzz on 30 real-world applications and compared it with 6 state-of-the-art greybox and hybrid fuzzers: AFL, AFLFast, FairFuzz, MOPT, QSYM, and Angora. Statistically, TortoiseFuzz found more vulnerabilities than 5 out of 6 fuzzers (AFL, AFLFast, FairFuzz, MOPT, and Angora), and it had a comparable result to QSYM yet only consumed around $2\%$ of QSYM’s memory usage on average. We also compared coverage accounting metrics with two other metrics, AFL-Sensitive and LEOPARD, and TortoiseFuzz performed significantly better than both metrics in finding vulnerabilities. Furthermore, we applied the coverage accounting metrics to QSYM and noticed that coverage accounting helps increase the number of discovered vulnerabilities by 28.6% on average. TortoiseFuzz found 20 zero-day vulnerabilities with 15 confirmed with CVE identifications.

## I. INTRODUCTION

Fuzzing has been extensively used to find real-world software vulnerabilities. Companies such as Google and Apple have deployed fuzzing tools to discover vulnerabilities, and researchers have proposed various fuzzing techniques [4, 6, 7, ${12},{18},{29},{33},{43},{45},{47},{56},{60},{61}\rbrack$ . Specifically, coverage-guided fuzzing [4, 6, 12, 18, 33, 43, 45, 56, 61] has been actively studied in recent years. In contrast to generational fuzzing, which generates inputs based on given format specifications $\left\lbrack  {2,3,{16}}\right\rbrack$ , coverage-guided fuzzing does not require knowledge such as input format or program specifications. Instead, coverage-guided fuzzing mutates inputs randomly and uses coverage to select and prioritize mutated inputs.

AFL [60] leverages edge coverage (a.k.a. branch coverage or transition coverage), and libFuzzer [47] supports both edge and block coverage. Specifically, AFL saves all inputs with new edge coverage, and it prioritizes inputs by size and latency while guaranteeing that the prioritized inputs cover all edges. Based on AFL, recent work advances the edge coverage metrics by adding finer-grained information such as call context [12], memory access addresses, and more preceding basic blocks [53].

However, previous work treats edges equally, neglecting that the likelihoods of edge destinations being vulnerable are different. As a result, for all the inputs that lead to new coverage, those that execute the newly explored code that is less likely to be vulnerable are treated as important as the others and are selected for mutation and fuzzing.

Although such design is reasonable for program testing which aims at full program coverage, it delays the discovery of a vulnerability. VUzzer [43] mitigates the issue by de-prioritizing inputs that lead to error-handling code, but it depends on taint analysis and thus is expensive. CollAFL [18] proposes alternative input prioritization algorithms regarding the execution path, but it cannot guarantee that prioritized inputs cover all security-sensitive edges and it may cause the fuzzer to be trapped in a small part of the code. AFL-Sensitive [53] and Angora [12] add more metrics complementary to edges, but edges are still considered equally, so the issue still exists for the inputs with the same value in the complementary metrics. LEOPARD [15] considers function coverage instead of edges and it weights functions differently, but it requires static analysis to preprocess, which causes extra performance overhead. Even worse, these approaches are all vulnerable to anti-fuzzing techniques [23, 28] (See II-D). Therefore, we need a new input prioritization method that finds more vulnerabilities and is less affected by anti-fuzzing techniques.

In this paper, we propose coverage accounting, a new approach for input prioritization. Our insight is that, any work that adds additional information to edge representation will not be able to defeat anti-fuzzing since the fundamental issue is that current edge-guided fuzzers treat coverage equally. Moreover, memory corruption vulnerabilities are closely related to sensitive memory operations, and sensitive memory operations can be represented at different granularity of function, loop, and basic block [27]. To find memory corruption vulnerabilities effectively, we should cover and focus on edges associated with sensitive memory operations only. Based on this observation, our approach assesses edges from function, loop, and basic block levels, and it labels edges security-sensitive based on the three metrics of different levels. We prioritize inputs by new security-sensitive coverage, and cull the prioritized inputs by the hit count of security-sensitive edges and meanwhile guarantee the selected inputs cover all visited security-sensitive edges.

Based on the proposed approach, we develop TortoiseFuzz, a greybox coverage-guided fuzzer ${}^{1}$ . TortoiseFuzz does not rely on taint analysis or symbolic execution; the only addition to AFL is the coverage accounting scheme inserted in the step of queue culling (See II).

TortoiseFuzz is simple yet powerful in finding vulnerabilities. We evaluated TortoiseFuzz on 30 popular real-world applications and compared TortoiseFuzz with 6 state-of-the-art greybox $\left\lbrack  {7,{31},{36},{60}}\right\rbrack$ and hybrid fuzzers $\left\lbrack  {{12},{59}}\right\rbrack$ . We calculated the number of discovered vulnerabilities, and conducted Mann-Whitney U test to justify statistical significance between TortoiseFuzz and the compared fuzzers. TortoiseFuzz performed better than 5 out of 6 fuzzers (AFL, AFLFast, FairFuzz, MOPT, and Angora), and it had a comparable result to QSYM yet only consumed, on average, around $2\%$ of the memory resourced used by QSYM. TortoiseFuzz found 20 zero-day vulnerabilities with 15 confirmed with CVE identifications.

We also compared coverage accounting metrics against AFL-Sensitive and LEOPARD, and the experiment showed that our coverage accounting metrics performed significantly better in finding vulnerabilities than both metrics. Furthermore, we applied the coverage accounting metrics to QSYM, and we noticed that coverage accounting boosted the number of discovered vulnerabilities by 28.6% on average.

To foster future research, we will release the prototype of TortoiseFuzz open-sourced at https://github.com/TortoiseFuzz/ as well as the experiment data for reproducibility.

Contribution. In summary, this paper makes the following contributions.

- We propose coverage accounting, a novel approach for input prioritization with metrics that evaluates edges in terms of the relevance of memory corruption vulnerabilities. Our approach is lightweighted without expensive analyses, such as taint analysis and symbolic execution, and is less affected by anti-fuzzing techniques.

![0196f76d-bb93-7ae3-9d9a-7b87ec3c97c0_1_941_152_774_515_0.jpg](images/0196f76d-bb93-7ae3-9d9a-7b87ec3c97c0_1_941_152_774_515_0.jpg)

Fig. 1: The framework of AFL.

- We design and develop TortoiseFuzz, a greybox fuzzer based on coverage accounting. We will release Tortoise-Fuzz with source code.

- We evaluated TortoiseFuzz on 30 real-world programs and compared it with 4 greybox fuzzers and 2 hybrid fuzzers. As a greybox fuzzer, TortoiseFuzz outperformed all the 4 greybox fuzzers and 1 hybrid fuzzers. Tortoise-Fuzz achieved a comparable result to the other hybrid fuzzer, QSYM, yet only spent 2% of the memory resource costed by QSYM. TortoiseFuzz also found 20 zero-day vulnerabilities, with 15 confirmed with CVE IDs.

## II. BACKGROUND

In this section, we present the background of coverage-guided fuzzing techniques. We first introduce the high-level design of coverage-guided fuzzing, and then explain the details of input prioritization and input mutation in fuzzing.

## A. Coverage-guided Fuzzing

Fuzzing is an automatic program testing technique for generating and testing inputs to find software vulnerabilities [38]. It is flexible and easy to apply to different programs, as it does not require the understanding of programs, nor manual generation of testing cases.

At a high level, coverage-guided fuzzing takes an initial input (seed) and a target program as input, and produces inputs triggering program error as outputs. It works in a loop, where it repeats the process of selecting an input, running the target program with the input, and generating new inputs based on the current input and its running result. In this loop, coverage is used as the fundamental metric to select inputs, which is the reason why such techniques are called coverage-guided fuzzing.

Figure 1 shows the architecture of AFL [60], a reputable coverage-guided fuzzer based on which many other fuzzers are developed. AFL first reads all the initial seeds and moves them to a testcase queue(1), and then gets a sample from the queue (2). For each sample, AFL mutates it with different strategies (3) and sends the mutated samples to a forked server where the testing program will be executed with every mutated sample (4). During the execution, the fuzzer collects coverage information and saves the information in a global data structure. AFL uses edge coverage, which is represented by a concatenation of the unique IDs for source and destination basic blocks, and the global data structure is a bitmap (5). If the testing program crashes, the fuzzer marks and reports it as a proof of concept of a vulnerability(6). If the sample is interesting under the metrics, the fuzzer puts it into the queue and labels it as "favored" if it satisfies the condition of being favored (7).

---

${}^{1}$ The name comes from a story of Aesop’s Fables. A tortoise is ridiculed by a rabbit at first in a race, crawls slowly but steadily, and beats the rabbit finally. As American Fuzzy Lop is a kind of rabbit, TortoiseFuzz wins.

---

## B. Input Prioritization

Input prioritization is to select inputs for future mutation and fuzzing. Coverage-guided fuzzers leverage the coverage information associated with the executions to select inputs. Different fuzzers apply different criteria for testing coverage, including block coverage, edge coverage, and path coverage. Comparing to block coverage, edge coverage is more delicate and sensitive as it takes into account the transition between blocks. It is also more scalable than path coverage as it avoids path explosion.

AFL and its descendants use edge coverage for input prior-itization. In particular, AFL's input prioritization is composed of two parts: input filtering (step 7 in Figure 1) and queue culling (step 1 in Figure 1). Input filtering is to filter out inputs that are not "interesting", which is represented by edge coverage and hit counts. Queue culling is to rank the saved inputs for future mutation and fuzzing. Queue culling does not discard yet re-organizes inputs. The inputs with lower ranks will have less chance to be selected for fuzzing. Input filtering happens along with each input execution. Queue culling, on the other hand, happens after a certain number of input executions which is controlled by mutation energy.

## 1) Input Filtering

AFL keeps a new input if the input satisfies either of the following conditions: - The new input produces new edges between basic blocks. - The hit count of an existing edge achieves a new scale.

Both conditions require the representation of edge. To balance between effectiveness and efficiency, AFL represents an edge of two basic blocks by combining the IDs of the source and destination basic blocks by shift and xor operations.

---

		cur_location = <coMPILE_TIME_RANDOM>;

bitmap[cur_location ⊕ prev_location]++;

prev_location = cur_location » 1;

---

For each edge, AFL records whether it is visited, as well as the times of visit for each previous execution. AFL defines multiple ranges for the times of visit (i.e., bucketing). Once the times of visit of the current input achieves a new range, AFL will update the record and keep the input.

The data structure for such record is a hash map, and thus is vulnerable for the hash collision. CollAFL [18] points out a new scheme that mitigates the hash collision issue, which is complementary to our proposed approach for input prioritization.

## 2) Queue Culling

The goal of queue culling is to concise the inputs while maintaining the same amount of edge coverage. Inputs that remained from the input filtering process may be repetitive in terms of edge coverage. In this process, AFL selects a subset of inputs that are more efficient than other inputs while still cover all edges that are already visited by all inputs.

Specifically, AFL prefers inputs with less size and less execution latency. To this end, AFL will first mark all edges as not covered. In the next, AFL iteratively selects an edge that is not covered, chooses the input that covers the edge and meanwhile has the smallest size and execution latency (which is represented as a score proportional to these two elements), and marks all edges that the input visits as covered. AFL repeats this process until all edges are marked as covered.

Note that in AFL's implementation, finding the best input for each edge occurs in input filtering rather than in queue culling. AFL uses a map top-rate with edges as keys and inputs as values to maintain the best input for each edge. In the process of input filtering, if AFL decides to keep an input, it will calculate the score proportional to size and execution time, and update the top-rate. For each edge along with the input's execution path, if its associated input in top-rate is not as good as the current input in terms of size and execution time, AFL will replace the value of the edge with the current input. This is just for ease of implementation: in this way, AFL does not need a separate data structure to store the kept inputs in the current energy cycle with their size and latency. For the details of the algorithm, please refer to Algorithm 1 in Section IV.

## 3) Advanced Input Prioritization Approaches

Edge coverage, although well balances between code coverage and path coverage, is insufficient for input prioritization because it does not consider the finer-grained context. Under such circumstances, previous work proposes to include more information to coverage representation. Angora [12] proposes to add a calling stack, and AFL-Sensitive [53] presents multiple additional information such as memory access address (memory-access-aware branch coverage) and n-basic block execution path (n-gram branch coverage).

This advancement improves typical edge coverage to be finer-grained, but it still suffers from the problem that inputs may fall into a "cold" part of a program which is less likely to have memory corruption vulnerabilities yet contributes to new coverage. For example, error-handling codes typically do not contain vulnerabilities, and thus fuzzer should avoid to spend overdue efforts in fuzzing around error-handling code. VUzzer [43] de-prioritizes the inputs that lead to error handling codes or frequent paths. However, it requires extra heavyweight work to identify error-handling codes, which makes fuzzing less efficient.

CollAFL [18] proposes new metrics that are directly related to the entire execution path rather than single or a couple of edges. Instead of queue culling, it takes the total number of instructions with memory access as metrics for input prioritization. However, CollAFL cannot guarantee that the prioritized inputs cover all the visited edges. As a consequence, it may fall into a code snippet that involves intensive memory operations yet is not vulnerable, e.g., a loop with a string assignment.

LEOPARD [15] keeps queue culling yet add an additional step, prioritizing the selected inputs from queue culling by a function-level coverage metrics, rather than choosing randomly in AFL. The approach is able to cover all visited edge in each fuzzing loop, but it requires to preprocess the targeting programs for function complexity analysis and thus brings performance overhead.

## C. Input Mutation and Energy Assignment

Generally, input mutation can also be viewed as input prioritization: if we see the input space as all the combinations of bytes, then input mutation prioritizes a subset of inputs from the input space by mutation. Previous work design comprehensive mutation strategies [12, 26, 31, 58] and optimal mutation scheduling approaches [36]. These input mutation approaches are all complementary to our proposed input prioritization scheme.

Similarly, energy assignment approaches such as AFLFast [7], AFLGo [6], FairFuzz [31] also prioritizes inputs by deciding the number of children inputs mutated from a father input. AFLFast [7] assigns more energy to the seeds with low frequency based on the Markov chain model of transition probability. While AFLGo [6] becomes a directed fuzzer, which allocates more energy on targeted vulnerable code. FairFuzz [31] marks branches that are hit fewer times than a pre-defined rarity-cutoff value as rare branches, and optimizes the distribution of fuzzing energy to produce inputs to hit a given rare branch.

## D. Anti-Fuzzing Techniques

Current anti-fuzzing techniques [23, 28] defeat coverage-guided fuzzers by two design deficiencies: 1) most coverage-guided fuzzers do not differentiate the coverage of different edges, and 2) hybrid fuzzers use heavyweight taint analysis or symbolic execution. Anti-fuzzing techniques deceive fuzzers by inserting fake paths, adding a delay in error-handling code, and obfuscating codes to slow down dynamic analyses.

Current anti-fuzzing techniques make coverage-guided fuzzers much less effective in vulnerability discovery, causing 85%+ performance decrease in exploring paths. Unfortunately, many of the presented edge-coverage-based fuzzers [12, 15, ${43},{53},{60}\rbrack$ suffer from the current anti-fuzzing techniques. VUzzer is affected due to the use of concolic execution. LEOPARD, which considers function-level code complexity as a metric for input prioritization, is vulnerable to fake paths insertion. As path insertion increases the complexity of the function with inserted paths, LEOPARD will mistakenly prioritize the inputs that visit these functions while does not prioritize the inputs that skip the inserted paths in the function. As a consequence, inputs that execute the inserted path will be more likely to be prioritized.

AFL, Angora, and AFL-Sensitive are also affected by fake paths because fake paths contribute to more code coverage. More generally, any approach that adds more information to edge representation yet still treat edge equally will be affected by anti-fuzzing. Essentially, this is because that edge coverage is treated equally despite the fact that edges have different likelihoods in leading to vulnerabilities.

## III. COVERAGE ACCOUNTING

Prior coverage-guided fuzzers [4, 6, 7, 12, 33, 45, 47, 56, ${60},{61}\rbrack$ are limited as they treat all blocks and edges equally. As a result, these tools may waste time in exploring the codes that are less likely to be vulnerable, and thus are inefficient in finding vulnerabilities. Even worse, prior work can be undermined by current anti-fuzzing techniques [23, 28] which exploit the design deficiency in current coverage measurement.

To mitigate this issue, we propose coverage accounting, a new approach to measure edges for input prioritization. Coverage accounting needs to meet two requirements. First, coverage accounting should be lightweighted. One purpose of coverage accounting is to shorten the time to find a vulnerability by prioritizing inputs that are more likely to trigger vulnerabilities. If coverage accounting takes long, it will not be able to shorten the time.

Second, coverage accounting should not rely on taint analysis or symbolic execution. This is because that coverage accounting needs to defend against anti-fuzzing. Since current anti-fuzzing techniques are capable of defeating taint analysis and symbolic execution, we should avoid using these two analyses in coverage accounting.

Based on the intuition that memory corruption vulnerabilities are directly related to memory access operations, we design coverage accounting for memory errors as the measurement of an edge in terms of future memory access operations. Furthermore, inspired by HOTracer [27], which treats memory access operations at different levels, we present the latest and future memory access operations from three granularity: function calls, loops, and basic blocks.

Our design is different from known memory access-related measurements. CollAFL [18] counts the total number of memory access operations throughout the execution path, which implies the history memory access operations. Wang et al. [53] apply the address rather than the count of memory access. Type-aware fuzzers such as Angora [12], TIFF [26], and ProFuzzer [58] identify inputs that associated to specific memory operations and mutate towards targeted programs or patterns, but they cause higher overhead due to type inference, and that input mutation is separate from input prioritization in our context that could be complementary to our approach.

## 1) Function Calls

On the function call level, we abstract memory access operations as the function itself. Intuitively, if a function was involved in a memory corruption, appearing in the call stack of the crash, then it is likely that the function will be involved again due to patch incompleteness or developers' repeated errors, and we should prioritize the inputs that will visit this function.

TABLE I: Top 20 vulnerability involved functions.

<table><tr><td>Function</td><td>Number</td><td>Function</td><td>Number</td></tr><tr><td>memcpy</td><td>80</td><td>vsprintf</td><td>9</td></tr><tr><td>strlen</td><td>35</td><td>GET_COLOR</td><td>7</td></tr><tr><td>ReadImage</td><td>17</td><td>read</td><td>7</td></tr><tr><td>malloc</td><td>15</td><td>load_bmp</td><td>6</td></tr><tr><td>memmove</td><td>12</td><td>huffcode</td><td>6</td></tr><tr><td>free</td><td>12</td><td>strcmp</td><td>6</td></tr><tr><td>memset</td><td>12</td><td>new</td><td>5</td></tr><tr><td>delete</td><td>11</td><td>getName</td><td>5</td></tr><tr><td>memcmp</td><td>10</td><td>strncat</td><td>5</td></tr><tr><td>getString</td><td>9</td><td>png_load</td><td>5</td></tr></table>

Inspired by VCCFinder [41], we check the information of disclosed vulnerabilities on the Common Vulnerabilities and Exposures ${}^{2}$ in the latest 4 years to find the vulnerability-involved functions. We crawl the reference webpages on CVE descriptions and the children webpages, extract the call stacks from the reference webpages and synthesize the involved functions. Part of them are shown in Table I (the top 20 vulnerability functions). We observe from this table that the top frequent vulnerability-involved functions are mostly from libraries, especially libc, which matches with the general impression that memory operation-related functions in libc such as strlen and memcpy are more likely to be involved in memory corruptions.

Given the vulnerability-involved functions, we assess an edge by the number of vulnerability-involved functions in the destination basic block. Formally, let $\mathcal{F}$ denote for the set of vulnerability-involved functions, let ${ds}{t}_{e}$ denote for the destination basic block of edge $e$ , and let $C\left( b\right)$ denote for the calling functions in basic block $b$ . For an $e$ , we have:

$$
\operatorname{Func}\left( e\right)  = \operatorname{card}\left( {C\left( {{ds}{t}_{e}}\right)  \cap  \mathcal{F}}\right)  \tag{1}
$$

where $\operatorname{Func}\left( e\right)$ represents the metric, and $\operatorname{card}\left( \cdot \right)$ represents for the cardinality of the variable as a set.

## 2) Loops

Loops are widely used for accessing data and are closely related to memory errors such as overflow vulnerabilities. Therefore, we introduce the loop metric to incentivize inputs that iterate a loop, and we use the back edge to indicate that. To deal with back edges, we introduce CFG-level instrumentation to track this information, instead of the basic block instrumentation. We construct CFGs for each module of the target program, and analyze the natural loops by detecting back edges [1]. Let function IsBackEdge(e)be a boolean function outputting whether or not edge $e$ is a back edge. Given an edge indicated by $e$ , we have the loop metric $\operatorname{Loop}\left( e\right)$ as follows:

$$
\operatorname{Loop}\left( e\right)  = \left\{  \begin{array}{l} 1,\text{ if }\operatorname{IsBackEdge}\left( e\right)  = \text{ True } \\  0,\text{ otherwise } \end{array}\right.  \tag{2}
$$

## 3) Basic Blocks

The basic block metric abstracts the memory operations that will be executed immediately followed by the edge. As a basic block has only one exit, all instructions will be executed, and the memory access in this basic block will also be enforced. Therefore, it is reasonable to consider the basic block metric as the finest granularity for coverage accounting.

Specifically, we evaluate an edge by the number of instructions that involve memory operations. Let IsContainMem(i) be a boolean function for whether or not an instruction $i$ contains memory operations. For edge $e$ with destination basic block ${ds}{t}_{e}$ , we evaluate the edge by the basic block metric $\mathrm{{BB}}\left( e\right)$ as follows:

$$
\mathrm{{BB}}\left( e\right)  = \operatorname{card}\left( \left\{  {i \mid  i \in  {ds}{t}_{e} \land  \operatorname{IsContainMem}\left( i\right) }\right\}  \right)  \tag{3}
$$

Discussion: Coverage accounting design. One concern is that the choice of the vulnerable is too specific and heuristic-based. We try to make it more general by selecting based on commit history and vulnerability reports, which is also acceptable by related papers [34, 35, 40, 44]. One may argue that this method cannot find vulnerabilities associated with functions that were not involved in any vulnerabilities before or custom functions. This concern is valid, but can be resolved by the other two metrics in a finer granularity as the three coverage accounting metrics are complementary. Moreover, all three metrics contribute to the final results of finding vulnerabilities (More details in subsection "Coverage accounting metrics" in Section VI-D).

Another way to select vulnerable functions is code analysis and there is a recent relevant paper LEOPARD [15] which is according to the complexity score and vulnerability score of functions. The score is calculated based on several code features including loops and memory accesses, which is more like the combination of all three coverage accounting metrics we propose. As the paper of LEOPARD mentioned that it could be used for fuzzing, we set an experiment to compare with it on our data set (More details in subsection "Coverage accounting vs. other metrics" in Section VI-D).

### IV.The Design of TortoiseFuzz

On the high-level, the goal of our design is to prioritize the inputs that are more likely to lead to vulnerable code, and meanwhile ensure the prioritized inputs cover enough code to mitigate the issue that the fuzzer gets trapped or misses vulnerabilities. There are three challenges for the goal. The first challenge is how to properly define the scope of the code to be covered and select a subset of inputs that achieve the complete coverage. Basically AFL's queue culling algorithm guarantees that the selected inputs will cover all visited edges. Our insight is that, since memory operations are the prerequisite of memory errors, only security-sensitive edges matter for the vulnerabilities and thus should be fully covered by selected input. Based on this insight, we re-scope the edges from all visited edges to security-sensitive only, and we apply AFL's queue culling algorithm on the visited security-sensitive edges. In this way, we are able to select a subset of inputs that cover all visited security-sensitive edges.

The following challenge is how to define security-sensitive with coverage accounting. It is intuitive to set a threshold for the metrics, and then define edges exceeding the threshold as security sensitive. We set the threshold conservatively: edges are security sensitive as long as the value of the metrics is above 0 . We leave the investigation on the threshold as future work (see Section VII).

---

${}^{2}$ For prototype, we use CVE dataset, https://cve.mitre.org/

---

The last challenge is how to make fuzzing evolve towards vulnerabilities. Our intuition is that, the more an input hits a security-sensitive edge, the more likely the input will evolve to trigger a vulnerability. We prioritize an input with hit count, based on the proposed metrics for coverage accounting.

Based on the above considerations, we decide to design TortoiseFuzz based upon AFL, and remain the combination of input filtering and queue culling for input prioritization. TortoiseFuzz, as a greybox coverage-guided fuzzer with coverage accounting for input prioritization, is lightweighted and robust to anti-fuzzing. For the ease of demonstration, we show the algorithm of AFL in Algorithm 1, and explain our design (marked in grey) based on AFL's algorithm.

Algorithm 1 Fuzzing algorithm with coverage accounting

---

function FUZZING(Program, Seeds)

	$\mathrm{P} \leftarrow$ INSTRUMENT(Program, CovFb, AccountingFb) $\;\mathrm{D}$

Phase

	// AccountingFb is FunCallMap, LoopMap, or InsMap

	INITIALIZE(Queue, CrashSet, Seeds)

	INITIALIZE(CovFb, accCov, TopCov)

	INITIALIZE(AccountingFb, accAccounting, TopAccounting)

	// accAccounting is MaxFunCallMap, MaxLoopMap, or MaxInsMap

	repeat $\; \vartriangleright$ Fuzzing Loop Phase

		input $\leftarrow$ NEXTSEED(Queue)

		NumChildren $\leftarrow$ MUTATEENERGY(input)

		for $\mathrm{i} = 0 \rightarrow$ NumChildren do

			child $\leftarrow$ MUTATE(input)

			IsCrash, CovFb, AccountingFb $\leftarrow$ RUN(P, child)

			if IsCrash then

				CrashSet $\leftarrow$ CrashSet $\cup$ child

			else if SAVE_IF_INTERESTING(CovFb, accCov) then

				TopCov, TopAccounting $\leftarrow$

					UPDATE(child, CovFb, AccountingFb, accAccounting)

				Queue $\leftarrow$ Queue $\cup$ child

			end if

		end for

		CULL_QUEUE(Queue, TopCov, TopAccounting)

	until time out

end function

---

## A. Framework

The process of TortoiseFuzz is shown in Algorithm 1. TortoiseFuzz consists of two phases: instrumentation phase and fuzzing loop phase. In the instrumentation phase (Section IV-B), the target program is instrumented with codes for preliminary analysis and runtime execution feedback. In the fuzzing loop phase (Section IV-C), TortoiseFuzz iteratively executes the target program with testcases, appends interesting samples to the fuzzing queue based on the execution feedback, and selects inputs for future iterations.

## B. Instrumentation Phase

The instrumentation phase is to insert runtime analysis code into the program. For source code, we add the analysis code during compilation; otherwise, we rewrite the code to insert the instrumentation. If the target requires specific types of inputs, we modify the I/O interface with instrumentation. The inserted runtime analysis code collects the statistics for coverage and security sensitivity evaluation.

## C. Fuzzing Loop Phase

The fuzzing loop is described from line 8 to 23 in Algorithm 1. Before the loop starts, TortoiseFuzz first creates a sample queue Queue from the initial seeds and a set of crashes CrashSet (line 4). The execution feedback for each sample is recorded in the coverage feedback map (i.e., CovFb at line 5) and accounting feedback map (i.e., AccountingFb at line 6). The corresponding maps accCov (line 5) and accAccounting (line 6) are global accumulated structures to hold all covered transitions and their maximum hit counts. The TopCov and TopAccounting are used to prioritize samples.

For each mutated sample, TortoiseFuzz feeds it to the target program and reports if the return status is crashed. Otherwise, it uses the function Save_If_Interesting to append it to the sample queue Queue if it matches the input filter conditions (new edges or hit bucket change) (line 16). It will also update the structure accCov.

For the samples in Queue, the function NextSeed selects a seed for the next test round according to the probability (line 9), which is determined by the favor attribute of the sample. If the value of favor is 1 , then the probability is 100%; otherwise it is $1\%$ . The origin purpose of favor is to have a minimal set of samples that could cover all edges seen so far, and turn to fuzz them at the expense of the rest. We improve the mechanism to prioritize mutated samples with two steps, Update (line 18) and Cull_Queue (line 22). More specifically, Update will update the structure accAccounting and return the top rated lists TopCov and TopAccounting, which are used in the following step of function Cull_Queue.

## 1) Updating Top Rated Candidates

To prioritize the saved interesting mutations, greybox fuzzers (e.g., AFL) maintain a list of entries TopCov for each edge ${\text{edge}}_{i}$ to record the best candidates, sample ${}_{j}$ , that are more favorable to explore. As shown in Formula 4, sample $j$ is "favor" for ${\operatorname{edge}}_{i}$ as the sample can cover ${\operatorname{edge}}_{i}$ and there are no previous candidates, or if it has less cost than the previous ones (i.e., execution latency multiplied by file size).

$$
\operatorname{TopCov}\left\lbrack  {\operatorname{edge}}_{i}\right\rbrack   = \begin{cases} {\operatorname{sample}}_{j}, & {\operatorname{CovFb}}_{j}\left\lbrack  {\operatorname{edge}}_{i}\right\rbrack   > 0 \\   &  \land  \left( {\operatorname{TopCov}\left\lbrack  {\operatorname{edge}}_{i}\right\rbrack   = \varnothing }\right. \\   & \left. {\vee \operatorname{IsMin}\left( {{\operatorname{exec}}_{ - }{\operatorname{time}}_{j} * {\operatorname{size}}_{j}}\right) }\right) \\  0, & \text{ otherwise } \end{cases}
$$

(4)

Cost-favor entries are not sufficient for the fuzzer to keep the sensitivity information to the memory operations; hence, TortoiseFuzz maintains a list of entries for each memory-related edge to record the "memory operation favor". As Formula 5 shows, if there is no candidate for edg ${e}_{i}$ , or if the ${\text{sample}}_{j}$ could max the hit count of edge edge ${e}_{i}$ , we mark it as "favor". If the hit count is the same with the previous saved one, we mark it as "favor" if the cost is less. The AccountingFb

and accAccounting are determined by coverage accounting.

$$
\text{TopAccounting}\left\lbrack  {\text{edge}}_{i}\right\rbrack   = \left\{  \begin{matrix} {\text{sample }}_{j}, & \left( {\text{TopAccounting}\left\lbrack  {\text{edge}}_{i}\right\rbrack   =  = \varnothing  \land  {\text{Coverb}}_{j}\left\lbrack  {\text{edge}}_{i}\right\rbrack   > 0}\right) \\   &  \vee  {\text{AccountingFb}}_{i}\left\lbrack  {\text{edge}}_{i}\right\rbrack   = \text{accAccounting}\left\lbrack  {\text{edge}}_{i}\right\rbrack  \\   &  \vee  \left( {\text{AccountingFb}\left\lbrack  {\text{edge}}_{i}\right\rbrack   = \text{accAccounting}\left\lbrack  {\text{edge}}_{i}\right\rbrack  }\right) \\   &  \land  \text{ISMM}\left( {\text{neve}\_ {\text{cuting}}_{i} * {\text{step}}_{i}}\right) \\  0, & \text{otherwise} \end{matrix}\right.
$$

(5)

## 2) Queue Culling

The top-rated candidates recorded by TopAccounting are a superset of samples that can cover all the security-sensitive edges seen so far. To optimize the fuzzing effort, as shown in Algorithm 2, TortoiseFuzz re-evaluates all top rated candidates after each round of testing to select a quasi-minimal subset of samples that cover all of the accumulated memory related edges and have not been fuzzed. First, we create a temporal structure Temp_map to hold all the edges seen up to now. During traversing the seed queue ${Queue}$ , if a sample is labeled with "favor", we will choose it as final "favor" (line 9). Then the edges covered by this sample is computed and the temporal Temp_map (line 10) is updated. The process proceeds until all the edges seen so far are covered. With this algorithm, we select favorable seeds for the next generation and we expect they are more dangerous to memory errors (line 13).

However, TortoiseFuzz prefers to exploring program states with less breadth than the original path coverage. This may result in a slow increase of samples in Queue and fewer samples with the favor attributes. To solve this problem, TortoiseFuzz uses the original coverage-sensitive top rated entries TopCov to re-cull the Queue while there is no favor sample in the Queue (line 15-24). Also, whenever the TopAccounting is changed (line 5), TortoiseFuzz will switch back to the security-sensitive strategy.

Algorithm 2 Cull Queue

---

function CULL_QUEUE(Queue, TopCov, TopAccounting)

	for $\mathrm{q} =$ Queue.head $\rightarrow$ Queue.end do

		q.favor $= 0$

	end for

	if IsChanged(TopAccounting) then

		Temp_map $\leftarrow$ accCov[MapSize]

		for $\mathrm{i} = 0 \rightarrow$ MapSize do

			if TopAccounting[i] && TopAccounting[i].unfuzzed then

				TopAccounting[i].favor = 1

					UPDATE_MAP(TopAccounting[i], Temp_map)

			end if

		end for

		Syn(Queue, TopAccounting)

	else

		// switch back to TopCov with coverage-favor

		for $\mathrm{i} = 0 \rightarrow$ MapSize do

			Temp_map $\leftarrow$ accCov[MapSize]

			if TopCov[i] && Temp_map[i] then

					TopCov[i].favor $= 1$

					UPDATE_MAP(TopCov[i], Temp_map)

				end if

		end for

		SYN(Queue, TopCov)

	end if

end function

---

Discussion: Defending against anti-fuzzing. Current anti-fuzzing techniques defeat prior fuzzing tools by inserting fake paths that trap fuzzers, adding a delay in error-handling code, and obfuscating code to slow down taint analysis and symbolic execution. TortoiseFuzz, along with coverage accounting, is robust to code obfuscation as it does not require taint analysis or symbolic execution. It is also not highly affected by anti-fuzzing because input prioritization helps to avoid the execution of error-handling code since error-handling code do not typically contain intensive memory operation. Also coverage accounting is robust to inserted fake branches created by Fuzzification [28], which the branches are composed of pop and ret. As Coverage accounting does not consider pop and ret as security-sensitive operations, it will not prioritize inputs that visit fake branches.

One may argue that a simple update for anti-fuzzing will defeat TortoiseFuzz, such as adding memory operations in fake branches. However, since memory access costs much more than other operations such as arithmetic operations, adding memory access operations in fake branches may cause slowdown and affect the performance for normal inputs, which is not acceptable for real-world software. Therefore, one has to carefully design fake branches that defeat TortoiseFuzz and keep reasonable performance, which is much harder than the current anti-fuzzing methods. Therefore, we argue that although TortoiseFuzz is not guaranteed to defend against all anti-fuzzing techniques now and future, it will significantly increase the difficulty of successful anti-fuzzing.

## V. IMPLEMENTATION

TortoiseFuzz is implemented based on AFL [60]. Besides the AFL original implementation, TortoiseFuzz consists of about 1400 lines of code including instrumentation ( $\sim  {700}$ lines in C++) and fuzzing loop ( $\sim  {700}$ lines in C). We also wrote a python program of 34 lines to crawl the vulnerability reports. For the function call level coverage accounting, we get the function names in instructions by calling getCalled-Function(   ) in the LLVM pass, and calculate the weight value by matching with the list of high-risk functions in Table I. For the loop level coverage accounting, we construct the CFG with adjacency matrix and then use the depth-first search algorithm to traverse the CFG and mark the back edges. For the basic block level coverage accounting, we mark the memory access characteristics of the instructions with the mayReadFromMemory(   ) and mayWriteToMemory(   ) functions from LLVM [30].

## VI. EVALUATION

In this section, we evaluate coverage accounting by testing TortoiseFuzz on real-world applications. We will answer the following research questions:

- RQ1: Is TortoiseFuzz able to find real-world zero-day vulnerabilities?

- RQ2: How do the results of TortoiseFuzz compare to previous greybox or hybrid fuzzers in real-world programs?

- RQ3: How do the results of the three coverage accounting metrics compare to other coverage metrics or input prioritization approaches?

- RQ4: Is coverage accounting able to cooperate with other fuzzing techniques and help improve vulnerability discovery?

- RQ5: Is coverage accounting robust against the current anti-fuzzing technique?

TABLE II: Compared fuzzers.

<table><tr><td>Fuzzer</td><td>Year</td><td>Type</td><td>Open</td><td>Target</td><td>Select</td></tr><tr><td>AFL</td><td>2016</td><td>greybox</td><td>Y</td><td>$\mathrm{S}/{\mathrm{B}}^{1}$</td><td>✓</td></tr><tr><td>AFLFast</td><td>2016</td><td>greybox</td><td>Y</td><td>S/B</td><td>✓</td></tr><tr><td>Steelix</td><td>2017</td><td>greybox</td><td>$\mathrm{N}$</td><td>S</td><td/></tr><tr><td>VUzzer</td><td>2017</td><td>hybrid</td><td>${\mathrm{Y}}^{2}$</td><td>B</td><td/></tr><tr><td>CollAFL</td><td>2018</td><td>greybox</td><td>$\mathrm{N}$</td><td>S</td><td/></tr><tr><td>FairFuzz</td><td>2018</td><td>greybox</td><td>Y</td><td>S</td><td>✓</td></tr><tr><td>T-fuzz</td><td>2018</td><td>hybrid</td><td>${\mathrm{Y}}^{3}$</td><td>B</td><td/></tr><tr><td>QSYM</td><td>2018</td><td>hybrid</td><td>Y</td><td>S</td><td>✓</td></tr><tr><td>Angora</td><td>2018</td><td>hybrid</td><td>Y</td><td>S</td><td>✓</td></tr><tr><td>MOPT</td><td>2019</td><td>greybox</td><td>Y</td><td>S</td><td>✓</td></tr><tr><td>DigFuzz</td><td>2019</td><td>hybrid</td><td>$\mathrm{N}$</td><td>S</td><td/></tr><tr><td>ProFuzzer</td><td>2019</td><td>hybrid</td><td>$\mathrm{N}$</td><td>S</td><td/></tr></table>

${}^{1}$ S: target source code, B: target binary.

${}^{2}$ VUzzer depends on external IDA Pro and pintool, and instrumenting with real-world program is too expensive to be scalable in our experiment environment. Also, VUzzer did not perform as well as other hybrid tools such as QSYM. Under such condition, we did not include VUzzer in real-world program experiment.

${}^{3}$ Some components of these tools cannot work for all binaries.

## A. Experiment Setup

Dataset. We collected 30 applications from the papers published from 2016 to 2019. The applications include image parsing and processing libraries, text parsing tools, an assembly tool, multimedia file processing libraries, language translation tools, and so on. We selected the latest version of the testing applications by the time we ran the experiment.

One may argue that the experiment dataset is lack of other test suites such as LAVA-M. We find that LAVA-M does not fit for evaluating fuzzer effectiveness since it does not reflect the real-world scenarios. We will further discuss the choice of dataset in Section VII.

In our evaluation, we found that there are 18 applications from which no fuzzers found any vulnerabilities. For ease of demonstration, we will only show the results for the 12 applications with found vulnerabilities throughout the rest of the evaluation.

Compared fuzzers. We collected recent fuzzers published from 2016 to 2019 as the candidate for comparison, as shown in Table II. We consider each fuzzer regarding whether or not it is open-source, and whether or not it can run on our experiment dataset. We filtered tools that are not open-source or do not scale to our test real-world programs and thus cannot be compared. The remained compared fuzzers are 4 greybox fuzzers (AFL, AFLFast, FairFuzz and MOPT) and 2 hybrid fuzzers (QSYM and Angora). For detailed explanations for each fuzzer, we refer readers to the footnote in Table II.

Environment and process. We ran our experiments on eight identical servers with 32 CPU Intel(R) Xeon(R) CPU E5- 2630 V3@2.40GHZ cores, 64GB RAM, and 64-bits Ubuntu 16.04.3 TLS. For each target application, we configured all fuzzers with the same seed ${}^{3}$ and dictionary set ${}^{4}$ . We ran each fuzzer with each target program for 140 hours according to CollAFL [18], and we repeated all experiments for 10 times as advised by Klees et al. [29].

We identified vulnerabilities in two steps. Given reported crashes, we first filtered out invalid and redundant crashes using a self-written script with ASan [46] then manually inspected the remaining crashes and reported those that are security related. For code coverage, we used gcov [20], which is a well-known coverage analysis tool.

## B. RQ1: Finding Zero-day Vulnerabilities

Table III shows the union of the real-world vulnerabilities identified by TortoiseFuzz in the 10 times of the experiment. The table presents each discovered vulnerability with its ID, the corresponding target program, the vulnerability type and whether it is a zero-day vulnerability. In total, TortoiseFuzz found 56 vulnerabilities in 10 different types, including stack buffer overflow, heap buffer overflow, use after free, and double free vulnerabilities, many of which are critical and may lead to severe consequences such as arbitrary code execution. Among the 56 found vulnerabilities, 20 vulnerabilities are zero-day vulnerabilities, and 15 vulnerabilities have been confirmed with a CVE identification ${}^{5}$ . The result indicates that TortoiseFuzz is able to identify a reasonable number of zero-day vulnerabilities from real-world applications.

## C. RQ2: TortoiseFuzz vs. Other Fuzzers in Real-world Ap- plications

In this experiment, we tested and compared TortoiseFuzz with greybox fuzzers and hybrid fuzzers on real-world applications. We evaluated each fuzzer by three metrics: discovered vulnerabilities, code coverage, and performance.

Discovered vulnerabilities. Table IV shows the average and the maximum numbers of vulnerabilities found by each fuzzer among 10 repeated runs. The table also shows the p-value of the Mann-Whitney U test between TortoiseFuzz and a comparing fuzzer regarding the total number of vulnerabilities found from all target programs in each of the 10 repeated experiment.

Our experiment shows that TortoiseFuzz is more effective than the other testing greybox fuzzers in finding vulnerabilities from real-world applications. TortoiseFuzz detected 41.7 vulnerabilities on average and 50 vulnerabilities in maximum, which outperformed all the other greybox fuzzers. Comparing to FairFuzz, the second best-performed greybox fuzzers, TortoiseFuzz found 43.9% more vulnerabilities on average and 31.6% more in maximum. Additionally, we compared the set of vulnerabilities in the best run of each fuzzer, and we observed that TortoiseFuzz covered all but 1 vulnerability found by the other fuzzer, and TortoiseFuzz found 10 more vulnerabilities that none of the other fuzzers discovered.

For hybrid fuzzers, TortoiseFuzz showed a better result than Angora and a comparable result with QSYM. Tortoise-Fuzz outperformed Angora by ${61.6}\%$ on the sum of average and by 56.3% on the sum of the maximum numbers of vulnerabilities for each target programs among the 10 times of the experiment. TortoiseFuzz also detected more or equal vulnerabilities from 91.7% (11/12) of the target applications. TortoiseFuzz found slightly more vulnerabilities than QSYM on average and equal vulnerabilities in maximum. For each target program, specifically, TortoiseFuzz performed better than QSYM in 7 programs, equally in 2 programs, and worse in 3 programs. Based on the Mann-Whitney U test, the result between TortoiseFuzz and QSYM was not statistically significant, and thus we consider TortoiseFuzz comparable to QSYM in finding vulnerabilities from real-world programs.

---

${}^{3}$ We select an initial seed randomly from the testcase set provided by the target program.

${}^{4}$ We do not use the dictionary in the experiment.

${}^{5}$ CVE-2018-17229 and CVE-2018-17230 are contained in both exiv2 and new_exiv2.

---

TABLE III: Real-world Vulnerabilities found by TortoiseFuzz.

<table><tr><td>Program CVE-2019-20169 heap-use-after-free ✓ CVE-2019-16347heap-buffer-overflow✓</td><td>Version</td><td>ID</td><td>Vulnerability Type</td><td>New</td></tr><tr><td rowspan="14">exiv2</td><td rowspan="14">0.26</td><td>CVE-2018-16336</td><td>heap-buffer-overflow</td><td>✓</td></tr><tr><td>CVE-2018-17229 CVE-2018-17230</td><td>heap-buffer-overflow heap-buffer-overflow</td><td>✓ ✓</td></tr><tr><td>issue_400</td><td>heap-buffer-overflow</td><td>✓</td></tr><tr><td>issue_460</td><td>stack-buffer-overflow</td><td>-</td></tr><tr><td>CVE-2017-11336</td><td>heap-buffer-overflow</td><td>-</td></tr><tr><td>CVE-2017-11337</td><td>invalid free</td><td>-</td></tr><tr><td>CVE-2017-11339 CVE-2017-14857</td><td>heap-buffer-overflow invalid free</td><td>- -</td></tr><tr><td>CVE-2017-14858</td><td>heap-buffer-overflow</td><td>-</td></tr><tr><td>CVE-2017-14861</td><td>stack-buffer-overflow</td><td>-</td></tr><tr><td>CVE-2017-14865</td><td>heap-buffer-overflow</td><td>-</td></tr><tr><td>CVE-2017-14866</td><td>heap-buffer-overflow</td><td>-</td></tr><tr><td>CVE-2017-17669</td><td>heap-buffer-overflow</td><td>-</td></tr><tr><td>issue_170</td><td>heap-buffer-overflow</td><td>-</td></tr><tr><td>CVE-2018-10999</td><td>heap-buffer-overflow</td><td>-</td></tr><tr><td rowspan="5">new_exiv2</td><td rowspan="5">0.26</td><td>CVE-2018-17229</td><td>heap-buffer-overflow</td><td>✓</td></tr><tr><td>CVE-2018-17230</td><td>heap-buffer-overflow</td><td>✓</td></tr><tr><td>CVE-2017-14865</td><td>heap-buffer-overflow</td><td>-</td></tr><tr><td>CVE-2017-14866</td><td>heap-buffer-overflow</td><td>-</td></tr><tr><td>CVE-2017-14858</td><td>heap-buffer-overflow</td><td>-</td></tr><tr><td>exiv2_9.17</td><td>0.26</td><td>CVE-2018-17282</td><td>null pointer dereference</td><td>✓</td></tr><tr><td rowspan="4">nasm</td><td rowspan="4">2.14rc4</td><td>CVE-2018-8882</td><td>stack-buffer-under-read</td><td>-</td></tr><tr><td>CVE-2018-8883</td><td>stack-buffer-over-read</td><td>-</td></tr><tr><td>CVE-2018-16517 CVE-2018-19209</td><td>null pointer dereference null pointer dereference</td><td>- -</td></tr><tr><td>CVE-2018-19213 CVE-2019-20165</td><td>memory leaks null pointer dereference</td><td>- ✓</td></tr><tr><td rowspan="8">gpac</td><td rowspan="8">0.7.1</td><td>CVE-2018-21017</td><td>memory leaks</td><td>✓</td></tr><tr><td>CVE-2018-21015</td><td>Segment Fault</td><td>✓</td></tr><tr><td>CVE-2018-21016</td><td>heap-buffer-overflow</td><td>✓</td></tr><tr><td>issue_1340</td><td>heap-use-after-free</td><td>-</td></tr><tr><td>issue_1264</td><td>heap-buffer-overflow</td><td>-</td></tr><tr><td>CVE-2018-13005</td><td>heap-buffer-over-read</td><td>-</td></tr><tr><td>issue_1077</td><td>heap-use-after-free</td><td>-</td></tr><tr><td>issue_1090 CVE-2018-15209</td><td>double-free heap-buffer-overflow</td><td>- ✓</td></tr><tr><td>libtiff</td><td>4.0.9</td><td>CVE-2018-16335 CVE-2018-11440</td><td>heap-buffer-over-read stack-buffer-overflow</td><td>✓ -</td></tr><tr><td>liblouis</td><td>3.7.0</td><td>issue_315</td><td>memory leaks</td><td>-</td></tr><tr><td rowspan="3">ngiflib</td><td rowspan="3">0.4</td><td>issue_10 CVE-2019-16346</td><td>stack-buffer-overflow heap-buffer-overflow</td><td>✓ ✓</td></tr><tr><td>CVE-2018-11575</td><td>stack-buffer-overflow</td><td>-</td></tr><tr><td>CVE-2018-11576</td><td>heap-buffer-over-read</td><td>-</td></tr><tr><td>libming</td><td>0_4_8</td><td>CVE-2018-13066</td><td>memory leaks</td><td>-</td></tr><tr><td/><td/><td>(2 similar crashes)</td><td>memory leaks</td><td>-</td></tr><tr><td>catdoc</td><td>0_95</td><td>crash crash</td><td>memory leaks Segment Fault</td><td>- -</td></tr><tr><td/><td/><td>CVE-2017-11110</td><td>heap-buffer-underflow</td><td>-</td></tr><tr><td>tcpreplay</td><td>4.3</td><td>CVE-2018-20552 CVE-2018-20553</td><td>heap-buffer-overflow heap-buffer-overflow</td><td>✓ ✓</td></tr><tr><td>flvmeta</td><td>1.2.1</td><td>issue_13</td><td>null pointer dereference</td><td>✓</td></tr><tr><td/><td/><td>issue_12</td><td>heap-buffer-overflow</td><td>-</td></tr></table>

Furthermore, we compared the union set of the discovered vulnerabilities across 10 repeated runs between TortoiseFuzz and QSYM. While TortoiseFuzz missed 9 vulnerabilities found by QSYM, only one of them is a zero-day vulnerability. The zero-day vulnerability is in the parse_mref function of nasm. We analyzed the missing cases and found that, some of the vulnerabilities are protected by conditional branches related to the input file, which does not belong to any of our metrics and thus the associated inputs cannot be prioritized.

Overall, our experiment results show that TortoiseFuzz outperformed AFL, AFLFast, FairFuzz, MOPT, and Angora, and it is comparable to QSYM in finding vulnerabilities.

Code coverage. In addition to the number of found vulnerabilities, we measured the code coverage of the testing fuzzers across different target programs. Although coverage accounting does not aim to improve code coverage, measuring code coverage is still meaningful, as it is an import metrics for evaluating program testing techniques. We also want to investigate if coverage accounting affects code coverage in the fuzzing process.

To investigate the impact on code coverage caused by coverage accounting, we compare the code coverage between TortoiseFuzz and AFL, the fuzzer based on which coverage accounting is implemented. Table V shows the average code coverage of all fuzzers performing on the target programs, and Table VI shows the p-value of the Mann-Whitney U test between TortoiseFuzz and other fuzzers. Based on Table V, we observe that TortoiseFuzz had a better coverage than AFL on average for ${75}\% \left( {9/{12}}\right)$ of the target programs. In terms of statistical significance, 3 out of 12 programs are statistically different in coverage, and TortoiseFuzz has a higher average value for all the three cases, which implies that TortoiseFuzz is statistically better than AFL in code coverage. Therefore, coverage accounting does not affect code coverage in fuzzing process.

Comparing TortoiseFuzz to other fuzzers, we observe that although TortoiseFuzz does not aim to high coverage, its performance is fair among all fuzzers. Most of the results are not statistically significant between TortoiseFuzz and AFL, AFLFast, and FairFuzz. Between TortoiseFuzz and MOPT, TortoiseFuzz performed statistically better in three cases, whereas MOPT performed statistically better in two cases. TortoiseFuzz's results are statistically higher than that of Angora in most cases, not as good as that of QSYM.

Furthermore, we study the variability of the code coverage of each testing fuzzer and target program, shown in Figure 2 and Table VII. We observe from the figure that the performance of TortoiseFuzz in code coverage is stable in most of the testing cases: the interquartile range is below 2% for 11 out of 12 of the testing programs.

Performance. Given that TortoiseFuzz had a comparable result with QSYM, we compare the resource performance of TortoiseFuzz to that of the hybrid fuzzer QSYM. For each of the 10 repeated experiment, we logged the memory usage of QSYM and TortoiseFuzz every five seconds, and we show the memory usage of each fuzzer and each targeting program in Figure 3. The figure indicates that TortoiseFuzz spent less memory resources than QSYM, which reflects the fact that hybrid fuzzers need more resources to execute heavy-weighted analyses such as taint analysis, concolic execution, and constraint solving.

Case study. To better understand the internal of why Tortoise-Fuzz managed to find zero-days vulnerabilities, we conduct a case study to investigate the fuzzing process and compare TortoiseFuzz to other fuzzers such as AFL. Figure 4 shows the fuzzing process of TortoiseFuzz and AFL for finding CVE- 2018-16335 ${}^{6}$ . The Seed ID indicates the ID of the testing inputs generated by each fuzzer. The lines in the figure shows the evolution of the generated seeds along with the fuzzing loop. The label of the nodes in the line of TortoiseFuzz shows the metric causing the seed to be prioritized.

TABLE IV: The Average Number of Vulnerabilities Identified by Each Fuzzer.

<table><tr><td rowspan="3">Program</td><td colspan="10">Grey-box Fuzzers</td><td colspan="4">Hybrid Fuzzers</td></tr><tr><td colspan="2">TortoiseFuzz</td><td colspan="2">AFL</td><td colspan="2">AFLFast</td><td colspan="2">FairFuzz</td><td colspan="2">MOPT</td><td colspan="2">Angora</td><td colspan="2">QSYM</td></tr><tr><td>Average</td><td>Max</td><td>Average</td><td>Max</td><td>Average</td><td>Max</td><td>Average</td><td>Max</td><td>Average</td><td>Max</td><td>Average</td><td>Max</td><td>Average</td><td>Max</td></tr><tr><td>exiv2</td><td>9.7</td><td>12</td><td>9.0</td><td>12</td><td>5.4</td><td>9</td><td>7.1</td><td>10</td><td>8.7</td><td>11</td><td>10.0</td><td>13</td><td>8.3</td><td>10</td></tr><tr><td>new_exiv2</td><td>5.0</td><td>5</td><td>0.0</td><td>0</td><td>0.0</td><td>0</td><td>0.0</td><td>0</td><td>0.0</td><td>0</td><td>0.0</td><td>0</td><td>6.5</td><td>9</td></tr><tr><td>exiv2_9.17</td><td>1.0</td><td>1</td><td>0.9</td><td>1</td><td>0.4</td><td>1</td><td>0.7</td><td>1</td><td>0.8</td><td>1</td><td>0.0</td><td>0</td><td>1.8</td><td>2</td></tr><tr><td>gpac</td><td>7.6</td><td>9</td><td>3.5</td><td>6</td><td>4.8</td><td>7</td><td>6.6</td><td>9</td><td>4.0</td><td>6</td><td>6.0</td><td>8</td><td>7.5</td><td>9</td></tr><tr><td>liblouis</td><td>1.2</td><td>2</td><td>0.3</td><td>1</td><td>0.0</td><td>0</td><td>0.9</td><td>2</td><td>0.1</td><td>1</td><td>0.0</td><td>0</td><td>2.3</td><td>3</td></tr><tr><td>libming</td><td>3.0</td><td>3</td><td>2.9</td><td>3</td><td>3.0</td><td>3</td><td>3.0</td><td>3</td><td>0.0</td><td>0</td><td>3.0</td><td>3</td><td>3.0</td><td>3</td></tr><tr><td>libtiff</td><td>1.2</td><td>2</td><td>0.0</td><td>0</td><td>0.0</td><td>0</td><td>0.0</td><td>0</td><td>0.0</td><td>0</td><td>0.0</td><td>0</td><td>0.4</td><td>1</td></tr><tr><td>nasm</td><td>3.0</td><td>4</td><td>1.7</td><td>2</td><td>2.2</td><td>3</td><td>2.2</td><td>3</td><td>3.8</td><td>5</td><td>1.8</td><td>2</td><td>2.8</td><td>4</td></tr><tr><td>ngiflib</td><td>4.7</td><td>5</td><td>4.4</td><td>5</td><td>3.2</td><td>5</td><td>4.0</td><td>5</td><td>2.7</td><td>5</td><td>3.0</td><td>4</td><td>4.5</td><td>5</td></tr><tr><td>flvmeta</td><td>2.0</td><td>2</td><td>2.0</td><td>2</td><td>2.0</td><td>2</td><td>2.0</td><td>2</td><td>2.0</td><td>2</td><td>2.0</td><td>2</td><td>2.0</td><td>2</td></tr><tr><td>tcpreplay</td><td>1.2</td><td>2</td><td>0.0</td><td>0</td><td>0.0</td><td>0</td><td>0.5</td><td>1</td><td>0.0</td><td>0</td><td>-*</td><td>- *</td><td>0.0</td><td>0</td></tr><tr><td>catdoc</td><td>2.1</td><td>3</td><td>1.3</td><td>2</td><td>1.2</td><td>2</td><td>2.0</td><td>2</td><td>0.3</td><td>1</td><td>0.0</td><td>0</td><td>2.0</td><td>2</td></tr><tr><td>SUM</td><td>41.7</td><td>50</td><td>26.0</td><td>34</td><td>22.2</td><td>32</td><td>29.0</td><td>38</td><td>22.4</td><td>32</td><td>25.8</td><td>32</td><td>41.1</td><td>50</td></tr><tr><td colspan="3">p-value of the Mann-Whitney U test</td><td colspan="2">0.0001668</td><td colspan="2">0.0001668</td><td colspan="2">0.0001649</td><td colspan="2">0.0001659</td><td colspan="2">0.0001668</td><td colspan="2">1.0</td></tr></table>

* Angora run abnormally. For all 360 pcap files included in the test suit, Angora reported the error log "There is none constraint in the seeds".

![0196f76d-bb93-7ae3-9d9a-7b87ec3c97c0_9_133_767_1537_391_0.jpg](images/0196f76d-bb93-7ae3-9d9a-7b87ec3c97c0_9_133_767_1537_391_0.jpg)

Fig. 2: The variability of the code coverage of different fuzzers with different target programs.

TABLE V: Code coverage (average in 10 runs) of the target applications achieved by various fuzzers. The highest values among fuzzers are highlighted in blue.

<table><tr><td rowspan="2">Program</td><td colspan="5">Grey-box Fuzzers</td><td colspan="2">Hybrid Fuzzers</td></tr><tr><td>TortoiseFuzz</td><td>AFL</td><td>AFLFast</td><td>FairFuzz</td><td>MOPT</td><td>Angora</td><td>QSYM</td></tr><tr><td>exiv2</td><td>19.77%</td><td>16.65%</td><td>15.00%</td><td>20.02%</td><td>19.53%</td><td>23.65%</td><td>22.23%</td></tr><tr><td>new_exiv2</td><td>19.83%</td><td>14.71%</td><td>11.67%</td><td>19.44%</td><td>15.86%</td><td>8.29%</td><td>$\mathbf{{21.40}\% }$</td></tr><tr><td>exiv2_9.17</td><td>19.16%</td><td>15.41%</td><td>13.27%</td><td>18.37%</td><td>20.53%</td><td>8.03%</td><td>23.38%</td></tr><tr><td>gpac</td><td>3.92%</td><td>3.86%</td><td>3.31%</td><td>4.64%</td><td>3.47%</td><td>7.07 %</td><td>5.63%</td></tr><tr><td>liblouis</td><td>29.44%</td><td>26.65%</td><td>29.53%</td><td>28.30%</td><td>29.47%</td><td>23.41%</td><td>31.42%</td></tr><tr><td>libming</td><td>21.00%</td><td>20.85%</td><td>21.01%</td><td>21.08%</td><td>21.05%</td><td>19.66%</td><td>21.10%</td></tr><tr><td>libtiff</td><td>39.00%</td><td>41.62%</td><td>36.27%</td><td>40.09%</td><td>38.42%</td><td>37.75%</td><td>42.77%</td></tr><tr><td>nasm</td><td>30.64%</td><td>29.83%</td><td>30.71%</td><td>31.98%</td><td>33.27%</td><td>27.70%</td><td>32.22%</td></tr><tr><td>ngiflib</td><td>76.13%</td><td>76.40%</td><td>76.31%</td><td>75.67%</td><td>76.15%</td><td>75.59%</td><td>76.52%</td></tr><tr><td>flvmeta</td><td>12.16%</td><td>12.10%</td><td>12.10%</td><td>12.16%</td><td>12.13%</td><td>12.05%</td><td>12.10%</td></tr><tr><td>tcpreplay</td><td>20.17%</td><td>17.89%</td><td>$\mathbf{{21.34}\% }$</td><td>18.20%</td><td>11.71%</td><td>-</td><td>17.61%</td></tr><tr><td>catdoc</td><td>48.12%</td><td>49.98%</td><td>39.90%</td><td>49.98%</td><td>29.74%</td><td>47.10%</td><td>62.80%</td></tr></table>

TABLE VI: p-values of the Mann-Whitney U test on the code coverage in 10 runs.

<table><tr><td>Program</td><td>AFL</td><td>AFLFast</td><td>FairFuzz</td><td>MOPT</td><td>Angora</td><td>QSYM</td></tr><tr><td>exiv2</td><td>6.23E-01</td><td>7.04E-01</td><td>1.40E-01</td><td>2.25E-01</td><td>1.80E-04</td><td>1.79E-04</td></tr><tr><td>exiv2_new</td><td>2.30E-02</td><td>1.65E-03</td><td>2.40E-01</td><td>1.31E-03</td><td>1.57E-04</td><td>1.30E-01</td></tr><tr><td>exiv2_9.17</td><td>6.23E-01</td><td>3.62E-01</td><td>${2.56}\mathrm{E} - {01}$</td><td>4.56E-03</td><td>1.71E-04</td><td>1.78E-04</td></tr><tr><td>gpac</td><td>4.48E-01</td><td>1.72E-01</td><td>${2.89}\mathrm{E} - {01}$</td><td>9.54E-02</td><td>1.75E-04</td><td>2.76E-04</td></tr><tr><td>liblouis</td><td>2.75E-04</td><td>6.15E-01</td><td>3.23E-01</td><td>${5.69}\mathrm{E} - {01}$</td><td>1.54E-04</td><td>2.03E-04</td></tr><tr><td>libming</td><td>${1.68}\mathrm{E} - {01}$</td><td>${3.68}\mathrm{E} - {01}$</td><td>4.41E-04</td><td>1.37E-02</td><td>${5.94}\mathrm{E} - {05}$</td><td>1.59E-05</td></tr><tr><td>libtiff</td><td>7.61E-01</td><td>${2.49}\mathrm{E} - {02}$</td><td>7.90E-01</td><td>9.90E-02</td><td>${1.10}\mathrm{E} - {01}$</td><td>${1.83}\mathrm{E} - {01}$</td></tr><tr><td>nasm</td><td>1.01E-01</td><td>6.21E-01</td><td>${8.50}\mathrm{E} - {01}$</td><td>1.63E-03</td><td>2.07E-03</td><td>5.61E-03</td></tr><tr><td>ngiflib</td><td>4.39E-01</td><td>5.18E-01</td><td>9.32E-01</td><td>${1.43}\mathrm{E} - {01}$</td><td>${2.13}\mathrm{E} - {01}$</td><td>${2.60}\mathrm{E} - {01}$</td></tr><tr><td>flvmeta</td><td>5.02E-03</td><td>5.02E-03</td><td>${1.00}\mathrm{E} + {00}$</td><td>${2.04}\mathrm{E} - {01}$</td><td>1.35E-03</td><td>5.02E-03</td></tr><tr><td>tcpreplay</td><td>9.29E-02</td><td>4.24E-01</td><td>4.02E-01</td><td>${2.02}\mathrm{E} - {02}$</td><td>-</td><td>6.75E-01</td></tr><tr><td>catdoc</td><td>${2.18}\mathrm{E} - {01}$</td><td>6.56E-01</td><td>9.67E-02</td><td>7.46E-02</td><td>1.55E-02</td><td>1.19E-04</td></tr></table>

Based on the figure, we find that TortoiseFuzz and AFL and TortoiseFuzz deviated in the second round of fuzzing. AFL prioritized other seeds over Seed 147, since Seed 147 is memory-intensive and results in longer execution time. However, we consider memory operations as a coverage accounting metric, and thus TortoiseFuzz prioritized the seed. As a result, Seed 147 evolved and finally became the input that triggers the vulnerability. This case indicates that memory operations, although cost longer execution time, helps to generate inputs that triggering memory corruption errors and thus should be taken as a metric for input prioritization.

---

${}^{6}$ https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-16335

---

![0196f76d-bb93-7ae3-9d9a-7b87ec3c97c0_10_141_149_1539_435_0.jpg](images/0196f76d-bb93-7ae3-9d9a-7b87ec3c97c0_10_141_149_1539_435_0.jpg)

Fig. 3: Memory resource usage of QSYM and TortoiseFuzz.

TABLE VII: The variance of the code coverage.

<table><tr><td rowspan="2">Program</td><td colspan="5">Grey-box Fuzzers</td><td colspan="2">Hybrid Fuzzers</td></tr><tr><td>TortoiseFuzz</td><td>AFL</td><td>AFLFast</td><td>Fairfuzz</td><td>MOPT</td><td>Angora</td><td>QSYM</td></tr><tr><td>exiv2</td><td>7.61E-06</td><td>3.18E-03</td><td>4.47E-03</td><td>2.71E-04</td><td>4.00E-04</td><td>6.52E-05</td><td>1.08E-04</td></tr><tr><td>exiv2_new</td><td>3.29E-04</td><td>4.11E-03</td><td>3.53E-03</td><td>1.16E-04</td><td>1.71E-04</td><td>2.27E-05</td><td>9.88E-04</td></tr><tr><td>exiv2_9.17</td><td>3.20E-05</td><td>3.32E-03</td><td>4.10E-03</td><td>4.99E-04</td><td>6.90E-05</td><td>3.61E-06</td><td>${8.00}\mathrm{E} - {05}$</td></tr><tr><td>gpac</td><td>5.58E-05</td><td>3.51E-04</td><td>5.15E-05</td><td>1.18E-04</td><td>1.43E-04</td><td>2.78E-05</td><td>3.21E-06</td></tr><tr><td>liblouis</td><td>1.10E-05</td><td>3.55E-04</td><td>3.01E-06</td><td>8.41E-04</td><td>6.52E-05</td><td>9.53E-05</td><td>1.10E-04</td></tr><tr><td>libming</td><td>7.70E-34</td><td>1.51E-05</td><td>9.00E-08</td><td>1.60E-07</td><td>2.50E-07</td><td>${3.60}\mathrm{E} - {05}$</td><td>7.70E-34</td></tr><tr><td>libtiff</td><td>7.72E-04</td><td>1.63E-03</td><td>2.94E-05</td><td>1.10E-03</td><td>4.02E-03</td><td>1.50E-04</td><td>1.86E-03</td></tr><tr><td>nasm</td><td>1.90E-03</td><td>1.70E-03</td><td>1.93E-03</td><td>1.12E-04</td><td>2.28E-05</td><td>4.31E-03</td><td>2.32E-03</td></tr><tr><td>ngiflib</td><td>2.13E-04</td><td>1.96E-04</td><td>1.96E-04</td><td>3.38E-05</td><td>2.21E-05</td><td>1.35E-04</td><td>${1.83}\mathrm{E} - {04}$</td></tr><tr><td>flvmeta</td><td>2.40E-07</td><td>0.00E+00</td><td>0.00E+00</td><td>2.40E-07</td><td>2.10E-07</td><td>2.50E-07</td><td>0.00E+00</td></tr><tr><td>tcpreplay</td><td>3.65E-03</td><td>2.68E-03</td><td>2.31E-03</td><td>4.74E-03</td><td>4.98E-04</td><td>-</td><td>4.21E-03</td></tr><tr><td>catdoc</td><td>1.32E-03</td><td>3.60E-07</td><td>${2.41}\mathrm{E} - {02}$</td><td>5.22E-05</td><td>2.75E-02</td><td>0.00E+00</td><td>4.00E-07</td></tr></table>

![0196f76d-bb93-7ae3-9d9a-7b87ec3c97c0_10_172_1175_670_330_0.jpg](images/0196f76d-bb93-7ae3-9d9a-7b87ec3c97c0_10_172_1175_670_330_0.jpg)

Fig. 4: Test case generation process of CVE-2018-16335

## D. RQ3: Coverage Metrics

Recall that we proposed three metrics for coverage accounting: Function calls, loops, and basic blocks. In this subsection, we evaluate the effectiveness of each metric, and we compare our combination of the metrics with other coverage-related metrics and input prioritization approach. We ran the three metrics separately for 140 hours and repeated for 10 times, and we represent the result based on these separate runs.

Internal investigation of the three metrics of coverage accounting. In this experiment, we investigate the effectiveness of each metric and their individual contribution to the overall metrics. We ran the three metrics separately for 140 hours and repeated for 10 times, and we represent the result based on these separate runs. Table VIII shows the code coverage and the number of discovered vulnerabilities of each coverage accounting metric. In the table, func represents the function call metric, loop represents the loop metric, and ${bb}$ represents the basic block metric. Running with the real-world programs, the loop, func, and ${bb}$ metrics found 28.2,24.6, and 28.3 vulnerabilities on average, respectively. All metrics found a non-negligible number of vulnerabilities exclusively, as shown in Figure 5. This implies that the three metrics complement each other and are all necessary for coverage accounting.

TABLE VIII: Code coverage and vulnerabilities of the three strategies of TortoiseFuzz.

<table><tr><td rowspan="3">Program</td><td colspan="3" rowspan="2">Code coverage</td><td colspan="6">Vulnerabilities</td></tr><tr><td colspan="3">Average</td><td colspan="3">Max</td></tr><tr><td>func</td><td>bb</td><td>loop</td><td>func</td><td>bb</td><td>loop</td><td>func</td><td>bb</td><td>loop</td></tr><tr><td>exiv2</td><td>17.39%</td><td>14.64%</td><td>17.56%</td><td>6.0</td><td>5.7</td><td>6.6</td><td>10</td><td>8</td><td>12</td></tr><tr><td>new_exiv2</td><td>18.66%</td><td>19.20%</td><td>19.71%</td><td>3.0</td><td>0.0</td><td>2.5</td><td>5</td><td>0</td><td>5</td></tr><tr><td>exiv2_9.17</td><td>16.75%</td><td>15.70%</td><td>14.80%</td><td>0.4</td><td>0.8</td><td>0.3</td><td>1</td><td>1</td><td>1</td></tr><tr><td>gpac</td><td>3.73%</td><td>3.77%</td><td>3.77%</td><td>4.6</td><td>4.2</td><td>4.3</td><td>6</td><td>7</td><td>9</td></tr><tr><td>liblouis</td><td>26.92%</td><td>25.64%</td><td>24.11%</td><td>1.0</td><td>0.5</td><td>0.7</td><td>1</td><td>1</td><td>2</td></tr><tr><td>libming</td><td>20.62%</td><td>20.72%</td><td>20.02%</td><td>3.0</td><td>3.0</td><td>3.0</td><td>3</td><td>3</td><td>3</td></tr><tr><td>libtiff</td><td>37.37%</td><td>38.79%</td><td>37.33%</td><td>0.0</td><td>0.8</td><td>0.3</td><td>0</td><td>2</td><td>1</td></tr><tr><td>nasm</td><td>29.07%</td><td>29.48%</td><td>28.80%</td><td>1.9</td><td>2.5</td><td>2.3</td><td>3</td><td>3</td><td>3</td></tr><tr><td>ngiflib</td><td>76.04%</td><td>76.04%</td><td>76.04%</td><td>3.9</td><td>3.6</td><td>3.8</td><td>5</td><td>5</td><td>5</td></tr><tr><td>flvmeta</td><td>12.10%</td><td>12.10%</td><td>12.10%</td><td>2.0</td><td>2.0</td><td>2.0</td><td>2</td><td>2</td><td>2</td></tr><tr><td>tcpreplay</td><td>17.24%</td><td>16.98%</td><td>17.56%</td><td>0.4</td><td>0.0</td><td>0.8</td><td>1</td><td>0</td><td>2</td></tr><tr><td>catdoc</td><td>48.07%</td><td>48.04%</td><td>47.98%</td><td>2.0</td><td>1.5</td><td>1.7</td><td>2</td><td>2</td><td>3</td></tr></table>

![0196f76d-bb93-7ae3-9d9a-7b87ec3c97c0_10_1129_1214_359_421_0.jpg](images/0196f76d-bb93-7ae3-9d9a-7b87ec3c97c0_10_1129_1214_359_421_0.jpg)

Fig. 5: The set of the bugs found by the 3 coverage accounting metrics from real-world programs (in the best run).

TABLE IX: The number of found vulnerabilities associated with the metrics of coverage accounting and AFL-Sensitive.

<table><tr><td rowspan="3">Program</td><td colspan="12">Vulnerabilities</td></tr><tr><td colspan="4">TortoiseFuzz</td><td colspan="8">AFL-Sensitive</td></tr><tr><td>func</td><td>bb</td><td>loop</td><td>ALL</td><td>bc</td><td>ct</td><td>ma</td><td>mw</td><td>n2</td><td>n4</td><td>n8</td><td>ALL</td></tr><tr><td>objdump</td><td>0</td><td>1</td><td>0</td><td>1</td><td>0</td><td>0</td><td>0</td><td>0</td><td>1</td><td>1</td><td>0</td><td>1</td></tr><tr><td>readelf</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td></tr><tr><td>strings</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td></tr><tr><td>nm</td><td>0</td><td>1</td><td>0</td><td>1</td><td>0</td><td>1</td><td>0</td><td>1</td><td>0</td><td>0</td><td>1</td><td>1</td></tr><tr><td>size</td><td>1</td><td>1</td><td>1</td><td>1</td><td>0</td><td>1</td><td>0</td><td>0</td><td>1</td><td>1</td><td>1</td><td>1</td></tr><tr><td>file</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td></tr><tr><td>gzip</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td></tr><tr><td>tiffset</td><td>0</td><td>1</td><td>1</td><td>1</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td></tr><tr><td>tiff2pdf</td><td>1</td><td>0</td><td>0</td><td>1</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td></tr><tr><td>gif2png</td><td>3</td><td>5</td><td>5</td><td>5</td><td>4</td><td>4</td><td>5</td><td>4</td><td>4</td><td>4</td><td>5</td><td>5</td></tr><tr><td>info2cap</td><td>7</td><td>5</td><td>10</td><td>10</td><td>9</td><td>7</td><td>5</td><td>5</td><td>10</td><td>7</td><td>7</td><td>10</td></tr><tr><td>jhead</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td><td>0</td></tr><tr><td>SUM</td><td>12</td><td>14</td><td>17</td><td>20</td><td>13</td><td>13</td><td>10</td><td>10</td><td>16</td><td>13</td><td>14</td><td>18</td></tr></table>

TABLE X: The code coverage associated with the metrics of coverage accounting and AFL-Sensitive. The highest values are highlighted in blue.

<table><tr><td rowspan="3">Program</td><td colspan="10">Code coverage(%)</td></tr><tr><td colspan="3">TortoiseFuzz</td><td colspan="7">AFL-Sensitive</td></tr><tr><td>func</td><td>bb</td><td>loop</td><td>bc</td><td>ct</td><td>ma</td><td>mw</td><td>n2</td><td>n4</td><td>n8</td></tr><tr><td>objdump</td><td>6.30</td><td>9.00</td><td>8.50</td><td>7.80</td><td>5.60</td><td>6.00</td><td>5.70</td><td>7.80</td><td>7.90</td><td>6.90</td></tr><tr><td>readelf</td><td>21.50</td><td>35.60</td><td>37.40</td><td>34.70</td><td>33.10</td><td>25.70</td><td>28.90</td><td>33.00</td><td>33.30</td><td>35.00</td></tr><tr><td>strings</td><td>0.20</td><td>0.20</td><td>0.20</td><td>0.20</td><td>0.20</td><td>0.20</td><td>0.20</td><td>0.20</td><td>0.20</td><td>0.20</td></tr><tr><td>nm</td><td>4.60</td><td>12.20</td><td>11.20</td><td>10.80</td><td>9.50</td><td>5.40</td><td>5.20</td><td>6.30</td><td>10.50</td><td>9.90</td></tr><tr><td>size</td><td>5.70</td><td>5.70</td><td>5.60</td><td>6.30</td><td>5.40</td><td>4.70</td><td>4.50</td><td>6.10</td><td>6.20</td><td>5.50</td></tr><tr><td>file</td><td>34.20</td><td>34.50</td><td>34.30</td><td>22.20</td><td>22.90</td><td>23.30</td><td>23.40</td><td>22.80</td><td>24.60</td><td>22.90</td></tr><tr><td>gzip</td><td>38.60</td><td>37.70</td><td>37.70</td><td>38.20</td><td>38.60</td><td>38.90</td><td>36.20</td><td>38.60</td><td>38.60</td><td>38.70</td></tr><tr><td>tiffset</td><td>30.10</td><td>30.40</td><td>30.50</td><td>10.00</td><td>10.00</td><td>10.50</td><td>10.00</td><td>10.00</td><td>10.00</td><td>10.00</td></tr><tr><td>tiff2pdf</td><td>40.70</td><td>40.50</td><td>38.70</td><td>31.30</td><td>33.40</td><td>33.10</td><td>29.40</td><td>30.80</td><td>33.40</td><td>31.60</td></tr><tr><td>gif2png</td><td>73.60</td><td>73.20</td><td>73.20</td><td>72.80</td><td>72.80</td><td>69.10</td><td>64.40</td><td>72.70</td><td>73.40</td><td>73.40</td></tr><tr><td>info2cap</td><td>41.20</td><td>40.90</td><td>42.10</td><td>40.70</td><td>41.00</td><td>34.30</td><td>35.60</td><td>41.30</td><td>40.70</td><td>39.20</td></tr><tr><td>jhead</td><td>21.00</td><td>21.00</td><td>21.00</td><td>21.00</td><td>21.00</td><td>21.00</td><td>21.00</td><td>21.00</td><td>21.00</td><td>21.00</td></tr></table>

The comparison to other coverage metrics. In this experiment, we compare our proposed metrics with two other coverage metrics for input prioritization: AFL-Sensitive [53] and LEOPARD [15].

AFL-Sensitive [53] presents 7 coverage metrics such as memory access address (memory-access-aware branch coverage) and n-basic block execution path (n-gram branch coverage). We ran our metrics and the 7 coverage metrics of AFL-Sensitive on the same testing suite and equal amount of time reported in the paper [53], and we compared them with regard to the number of discovered vulnerabilities and code coverage.

Table IX and Table X show the number of discovered vulnerabilities and the code coverage associated with the metrics of coverage accounting and AFL-Sensitive. Per AFL-Sensitive [53], a metric should be included if it achieves the top of all metrics in the associated found vulnerabilities or code coverage on a target program. Based on our experiment, we see that all metrics are necessary for coverage accounting and AFL-Sensitive. Taking all metrics into account, we observed that coverage accounting reported a few more vulnerabilities than AFL-Sensitive. Coverage accounting also slightly outperformed in code coverage, given that it achieved a higher coverage in ${66.7}\% \left( {8/{12}}\right)$ and a lower coverage in ${16.7}\%$ $\left( {2/{12}}\right)$ of the target programs. The results indicate coverage account performed slightly better than AFL-Sensitive in the number of discovered vulnerabilities and code coverage with fewer metrics, and that the metrics of coverage accounting are more effective than those of AFL-Sensitive.

TABLE XI: The Number of vulnerabilities found by LEOPARD and TortoiseFuzz (10 runs).

<table><tr><td rowspan="2">Program</td><td colspan="2">TortoiseFuzz</td><td colspan="2">LEOPARD</td></tr><tr><td>Average</td><td>Max</td><td>Average</td><td>Max</td></tr><tr><td>exiv2</td><td>9.7</td><td>12</td><td>6.0</td><td>11</td></tr><tr><td>new_exiv2</td><td>5.0</td><td>5</td><td>0.0</td><td>0</td></tr><tr><td>exiv2_9.17</td><td>1.0</td><td>1</td><td>0.7</td><td>1</td></tr><tr><td>gpac</td><td>7.6</td><td>9</td><td>5.8</td><td>8</td></tr><tr><td>liblouis</td><td>1.2</td><td>2</td><td>0.0</td><td>0</td></tr><tr><td>libming</td><td>3.0</td><td>3</td><td>3.0</td><td>3</td></tr><tr><td>libtiff</td><td>1.2</td><td>2</td><td>0.0</td><td>0</td></tr><tr><td>nasm</td><td>3.0</td><td>4</td><td>1.9</td><td>3</td></tr><tr><td>ngiflib</td><td>4.7</td><td>5</td><td>3.7</td><td>5</td></tr><tr><td>flvmeta</td><td>2.0</td><td>2</td><td>2.0</td><td>2</td></tr><tr><td>tepreplay</td><td>1.2</td><td>2</td><td>0.0</td><td>0</td></tr><tr><td>catdoc</td><td>2.1</td><td>3</td><td>2.0</td><td>2</td></tr><tr><td>SUM</td><td>41.7</td><td>50</td><td>25.1</td><td>35</td></tr><tr><td colspan="3">p-value of the Mann-Whitney U test</td><td colspan="2">0.0001707</td></tr></table>

LEOPARD [15] proposed a function-level coverage accounting scheme for input prioritization. Given a target program, it first identifies potentially vulnerable functions in the program, and then it calculates a score for the identified functions. The score of a function is defined based on code complexity properties, such as loop structures and data dependency. Finally, LEOPARD prioritized inputs by the sum of the score of the potentially vulnerable functions executed by the target program with each input. On the contrary of TortoiseFuzz which prioritize inputs by basic block-level metrics, LEOPARD assesses the priority of inputs based on the level of functions and in particular, pre-identified potentially vulnerable functions.

Since LEOPARD integrates metrics such as code complexity and vulnerable functions internally, we compare the result of TortoiseFuzz as the integration of code coverage to that of the corresponding LEOPARD fuzzer. As the LEOPARD fuzzer or metric implementation is not open-sourced, we contacted the authors and received from them the identified potentially vulnerable functions with computed scores. We then wrote a fuzzer, per their suggestion of the design, deployed the computed scores to the fuzzer, and ran the fuzzer with the LEOPARD metrics. We kept the total amount of time 140 hours for each experiment, and compared the number of discovered vulnerabilities between code coverage and LEOPARD metrics, shown in Table XI.

We observe that TortoiseFuzz found more vulnerabilities than LEOPARD from ${83}\% \left( {{10}/{12}}\right)$ applications on average and an equal number of vulnerabilities from the other 2 applications. The p-value is 0.0001707, which demonstrates statistical significance between TortoiseFuzz and LEOPARD and thus the coverage accounting metrics, which is on basic block level, perform better than the LEOPARD metrics, which is on function level, in identifying vulnerabilities from real-world applications.

TABLE XII: The number of vulnerabilities detected by QSYM with and without coverage accounting (5 runs).

<table><tr><td rowspan="3">Program</td><td colspan="10">Vulnerabilities</td></tr><tr><td colspan="2">QSYM(+AFL)</td><td colspan="2">QSYM+func</td><td colspan="2">QSYM+bb</td><td colspan="2">QSYM+loop</td><td colspan="2">QSYM+CA</td></tr><tr><td>AVG</td><td>Max</td><td>AVG</td><td>Max</td><td>AVG</td><td>Max</td><td>AVG</td><td>Max</td><td>AVG</td><td>Max</td></tr><tr><td>exiv2</td><td>8.2</td><td>10</td><td>11.2</td><td>12</td><td>8.4</td><td>11</td><td>10.6</td><td>13</td><td>13.0</td><td>15</td></tr><tr><td>new_exiv2</td><td>7.4</td><td>9</td><td>5.5</td><td>8</td><td>3.8</td><td>8</td><td>5.0</td><td>8</td><td>7.3</td><td>9</td></tr><tr><td>exiv2_9.17</td><td>2.0</td><td>2</td><td>1.5</td><td>2</td><td>1.5</td><td>2</td><td>1.8</td><td>3</td><td>1.8</td><td>3</td></tr><tr><td>gpac</td><td>6.0</td><td>8</td><td>7.0</td><td>10</td><td>8.4</td><td>10</td><td>7.6</td><td>9</td><td>9.2</td><td>11</td></tr><tr><td>liblouis</td><td>2.0</td><td>3</td><td>2.6</td><td>3</td><td>2.8</td><td>3</td><td>2.4</td><td>3</td><td>3.0</td><td>3</td></tr><tr><td>libming</td><td>3.0</td><td>3</td><td>3.0</td><td>3</td><td>3.0</td><td>3</td><td>3.0</td><td>3</td><td>3.0</td><td>3</td></tr><tr><td>libtiff</td><td>0.8</td><td>1</td><td>0.4</td><td>1</td><td>0.6</td><td>1</td><td>0.8</td><td>1</td><td>0.8</td><td>1</td></tr><tr><td>nasm</td><td>2.2</td><td>3</td><td>2.6</td><td>3</td><td>2.8</td><td>3</td><td>2.8</td><td>4</td><td>3.2</td><td>4</td></tr><tr><td>ngiflib</td><td>4.2</td><td>5</td><td>5.0</td><td>5</td><td>5.0</td><td>5</td><td>4.8</td><td>5</td><td>5.0</td><td>5</td></tr><tr><td>flvmeta</td><td>2.0</td><td>2</td><td>2.0</td><td>2</td><td>2.0</td><td>2</td><td>2.0</td><td>2</td><td>2.0</td><td>2</td></tr><tr><td>tcpreplay</td><td>0.0</td><td>0</td><td>0.0</td><td>0</td><td>0.4</td><td>1</td><td>0.6</td><td>1</td><td>0.6</td><td>1</td></tr><tr><td>catdoc</td><td>2.0</td><td>2</td><td>2.0</td><td>2</td><td>2.0</td><td>2</td><td>2.0</td><td>2</td><td>2.0</td><td>2</td></tr><tr><td>SUM</td><td>39.8</td><td>48</td><td>43.6</td><td>51</td><td>41.4</td><td>51</td><td>44.0</td><td>54</td><td>51.2</td><td>59</td></tr><tr><td colspan="3">p-value of the U test</td><td colspan="2">0.2477059</td><td colspan="2">0.5245183</td><td colspan="2">0.1387917</td><td colspan="2">0.0119252</td></tr></table>

### E.RQ4. Improving the State-of-the-art with Coverage Ac- counting.

As an input prioritization mechanism, coverage accounting is able to cooperate with other types of fuzzing improvement such as input generation. In this experiment, we study the question that whether coverage accounting, as an extension to the state-of-art fuzzer, helps improve the effectiveness in vulnerability discovery.

Recall that QSYM was best-performed compared fuzzers in our experiment, and that it found 9 vulnerabilities that are missed by TortoiseFuzz. Therefore, we selected QSYM and compared the number of vulnerabilities found by QSYM with and without coverage metrics.

In this experiment, we compared QSYM to the other four variations: QSYM with the function metric (QSYM+func), the basic block metric (QSYM+bb), the loop metric (QSYM+loop), and with the full coverage accounting metrics (QSYM+CA). We ran all tools for 140 hours and repeated each experiment for 5 times.

Table XII shows the number of vulnerabilities discovered by each fuzzer. We find that all the metrics help to improve QSYM in vulnerability discovery. In particular, QSYM with full coverage accounting (QSYM+CA) is able to find 28.6% more vulnerabilities on average, and 22.9% more in the sum of the best per-program performance. This indicates that coverage accounting is able to cooperate with the state-of-the-art fuzzers and significantly improve the effectiveness in vulnerability discovery.

## F. RQ5: Defending against Anti-fuzzing

Recent work $\left\lbrack  {{23},{28}}\right\rbrack$ show that current fuzzing schemes are vulnerable to anti-fuzzing techniques. Fuzzification [28], for example, proposes three methods to hinder greybox fuzzers and hybrid fuzzers. Fuzzification effectively reduced the number of discovered paths by 70.3% for AFL and QSYM on real-world programs.

To test the robustness of coverage accounting against Fuzzification, we implemented TortoiseFuzz-Bin, a version of TortoiseFuzz to test binary programs based on AFL Qemu mode and IDA Pro. We got 4 testing binaries from Fuzzifi-cation compiled with all three anti-fuzzing methods. In our experiment, we ran TortoiseFuzz-Bin for 72 hours aligned with the setup of Fuzzification. We selected the initial seeds based on the suggestion of the author of Fuzzification. We also used Fuzzification's method to get the numbers of the path discovered. Additionally, we measured the code coverage of TortoiseFuzz-Bin and AFL.

![0196f76d-bb93-7ae3-9d9a-7b87ec3c97c0_12_947_151_762_618_0.jpg](images/0196f76d-bb93-7ae3-9d9a-7b87ec3c97c0_12_947_151_762_618_0.jpg)

Fig. 6: Paths discovered by TortoiseFuzz from real-world programs. Each program is compiled with eight settings: func, bb, loop, afl (without protection), func-, bb-, loop-, afl-(with all protections of Fuzzification).

Table XIII and Table show the number of discovered paths and the code coverage of each testing cases after 72 hours of fuzzing process. Based on the tables, we find that AFL decreased much more than all the metrics in coverage accounting. Additionally, we did the statistics of number of discovered paths over time, shown in Figure 6. The figure indicates that the coverage accounting metrics consistently performed better than AFL since 4 hours after the experiment starts, which indicates that coverage accounting is more robust than AFL over time.

## VII. DISCUSSION

## A. Coverage Accounting Metrics

TortoiseFuzz prioritizes inputs by a combination of coverage and security impact. The security impact is represented by the memory operations on three different types of granularity at function, loop, and instruction level. These are empirical heuristics inspired by Jia et al. [27]. We see our work as a first step to investigate how to comprehensively account coverage quantitatively and adopt the quantification to coverage-guided fuzzing. In the future, we plan to study the quantification in a more systematic way. A possible direction is to consider more heuristics and apply machine learning to recognize the feasible features for effective fuzzing.

TABLE XIII: The number of discovered paths of the Anti-fuzz experiment. Each program is compiled with eight settings: func, bb, loop, afl (without Fuzzification protection), func-, bb-, loop-, afl- (with full Fuzzifucation protection). The blue values denote the highest decrease rate.

<table><tr><td rowspan="3">Program</td><td colspan="8">#Paths discovered</td></tr><tr><td colspan="6">TortoiseFuzz-Bin</td><td colspan="2">AFL-Qemu</td></tr><tr><td>func</td><td>func-</td><td>bb</td><td>bb-</td><td>loop</td><td>loop-</td><td>afl</td><td>afl-</td></tr><tr><td>nm</td><td>4626</td><td>3120 (-32.5%)</td><td>5449</td><td>4040 (-25.8%)</td><td>5490</td><td>3888 (-29.1%)</td><td>5043</td><td>2884 (-42.8%)</td></tr><tr><td>objcopy</td><td>9518</td><td>7942 (-16.5%)</td><td>9823</td><td>7837 (-20.2%)</td><td>9869</td><td>7417 (-24.8%)</td><td>10165</td><td>6005 (-40.9%)</td></tr><tr><td>objdump</td><td>11353</td><td>10951 (-3.5%)</td><td>10618</td><td>10508 (-1.0%)</td><td>11272</td><td>10448 (-7.3%)</td><td>11569</td><td>10688 (-7.6%)</td></tr><tr><td>readelf</td><td>8761</td><td>6737 (-23.1%)</td><td>8673</td><td>6378 (-26.4%)</td><td>9075</td><td>6560 (-27.7%)</td><td>9863</td><td>5006 (-49.2%)</td></tr></table>

TABLE XIV: The code coverage result of the Anti-fuzz experiment. Each program is compiled with eight settings: func, bb, loop, afl (without Fuzzification protection), func-, bb-, loop-, afl- (with full Fuzzification protection). The numbers in parentheses are the decrease rates cased by Fuzzification. The blue values denote the highest decrease rate.

<table><tr><td rowspan="3">Program</td><td colspan="8">Code coverage</td></tr><tr><td colspan="6">TortoiseFuzz-Bin</td><td colspan="2">AFL-Qemu</td></tr><tr><td>func</td><td>func-</td><td>bb</td><td>bb-</td><td>loop</td><td>loop-</td><td>afl</td><td>afl-</td></tr><tr><td>nm</td><td>5.2%</td><td>4.0% (-1.2%)</td><td>5.2%</td><td>4.8% (-0.4%)</td><td>5.3%</td><td>4.5% (-0.8%)</td><td>5.3%</td><td>3.9% (-1.4%)</td></tr><tr><td>objcopy</td><td>8.2%</td><td>7.5% (-0.7%)</td><td>8.3%</td><td>7.4% (-0.9%)</td><td>8.5%</td><td>7.0% (-1.5%)</td><td>8.8%</td><td>6.2% (-2.6%)</td></tr><tr><td>objdump</td><td>7.5%</td><td>7.3% (-0.2%)</td><td>7.3%</td><td>7.3% (-0.0%)</td><td>7.5%</td><td>7.5% (-0.0%)</td><td>7.8%</td><td>7.4% (-0.4%)</td></tr><tr><td>readelf</td><td>18.9%</td><td>15.8% (-3.1%)</td><td>18.4%</td><td>15.7% (-2.7%)</td><td>18.7%</td><td>15.50% (-3.2%)</td><td>20.5%</td><td>12.6% (-7.9%)</td></tr></table>

### B.The Threshold for Security-sensitivity

We currently set a unified threshold for deciding security-sensitive edges: an edge is security-sensitive if one of the three metrics' value is above 0 . Ideally, the threshold should be specific to programs. Future work would be to design approaches to automatically generate the threshold for each program through static analysis or during the fuzzing execution.

## C. LAVA-M vs. Real-world Data Set

In our experiment, we observed that the LAVA-M test suite is different from the real-world program test suite in two aspects. First, LAVA-M has more test cases involving magic words, which makes the test suite biased in testing exploit generation tools. Second, the base binaries for LAVA-M are not as complex as the real-world programs we tested. We acknowledge the value of the data set; yet a future direction is to systematically compare the inserted bugs in LAVA-M against a large number of real-world programs, understand the difference and the limitation of the current test suite, and build a test suite that is more comprehensive and more representative to real-world situations.

## D. Statistical Significance of #Vulnerabilities per Program

In our evaluation, we did not report the statistical significance between TortoiseFuzz and other fuzzers in terms of the number of vulnerabilities found per program. On the contrary to the per-program p-value in code coverage shown in Table VI and represented in REDQUEEN [4], vulnerabilities are very sparse in one vulnerabilities. Therefore, it is hard to tell the effectiveness difference among fuzzers from a single program. For example, for the flvmet a program, the statistical significance among all tools are inconclusive, since all tools found 2 vulnerabilities across all runs. Therefore, we report the p-value of the total number of vulnerabilities found from all target programs. This also indicates the necessity of having a comprehensive data suite with a sufficient number of vulnerabilities.

## VIII. RELATED WORK

Plenty of techniques have been presented to improve fuzzing in different aspects since the concept of fuzzing was developed in 1990s [38]. In this section, we introduce some of the presented fuzzing techniques. For a more comprehensive study on fuzzing techniques, please refer to recent surveys such as Chen et al. [10], Li et al. [32], and Manès et al [37].

Fuzzing specific program types. Some fuzzing techniques focus on specific types of programs, based on which they propose more effective fuzzing techniques. These studies include fuzzing on protocol [5, 14], firmware [11, 17, 39], and OS kernel [13, 45, 51, 57].

Hardware-assisted fuzzing. Previous work proposes various approaches to increase the efficiency of the fuzzing process. Xu et al. [56] designs three new operating primitives to remove execution redundancy and improve the performance of AFL in running testing inputs. Hardware-based fuzzing, such as kAFL [45] and PTfuzz [61], leverages hardware features such as Intel Processor Trace [25] to guide fuzzing without the overhead caused by instrumentation. These techniques decreases the time spent on program execution and information extraction, so that fuzzers can explore more inputs within a given amount of time.

Generational fuzzing. Besides mutation fuzzing, generational fuzzers also play an important role to test programs and find security vulnerabilities. Fuzzers such as Peach [16], Sulley [3], and SPIKE [2], generate samples based on a pre-defined configuration which specifies the input format. As generational fuzzers requires a format configuration which typically is generated manually, these fuzzers are less generic and depend on human efforts.

Machine learning-assisted fuzzing. Recent works, such as Skyfire [52], Learn&fuzz [22], and NEUZZ [48], combines fuzzing with machine learning and artificial intelligence. They learn the input formats or the relationships between input and program execution, and use the learned result to guide the generation of testing inputs. Such fuzzers, though, usually cause high overhead, because of the involvement of machine learning processing.

Static analysis and its assistance on fuzzing. Dowser [24] performs static analysis at compile time to find vulnerable code like loops and pointers, so as QTEP [54]. Sparks et al. [49] extracts the control flow graphs from the target to help input generation. Steelix [33] and VUzzer [43] analyze magic values, immediate values, and strings that can affect control flow.

Dynamic analysis and its assistance on fuzzing. Dynamic analysis including symbolic execution and taint analysis help enhance fuzzing. Taint analysis is capable of showing the relationship between input and program execution. BuzzFuzz [19], TaintScope [55], and VUzzer [43] use this technique to find relevant bytes and reduce the mutation space. Symbolic execution helps exploring program states. KLEE [8], SAGE [21], MoWF [42], and Driller [50] use this technique to execute into deeper logic. To solve the problems such as path explosion and constraint complexity, SYMFUZZ [9] reduces the symbolized input bytes by taint analysis, while Angora [12] performs a search inspired by gradient descent algorithm during constraint solving. Another problem is that program analysis will cause extra overhead. REDQUEEN [4] leverages a lightweighted taint tracking and symbolic execution method for optimization.

## IX. Conclusion

In this paper, we propose TortoiseFuzz, an advanced coverage-guided fuzzer with a novel technique called coverage accounting for input prioritization. Based on the insight that the security impact on memory corruption vulnerabilities can be represented with memory operations, and that memory operations can be abstracted at different levels, we evaluate edges based on three levels, function call, loop, and basic block. We combine the evaluation with coverage information for input prioritization. In our experiments, we tested TortoiseFuzz with 6 greybox and hybrid fuzzers on 30 real-world programs, and the results showed that TortoiseFuzz outperformed all but one hybrid fuzzers, yet spent only $2\%$ of memory resources. Our experiments also showed that coverage accounting was able to defend against current anti-fuzzing techniques. In addition, TortoiseFuzz identified 20 zero-day vulnerabilities, 15 of which have been confirmed and released with CVE IDs.

## ACKNOWLEDGEMENT

We thank the anonymous reviewers of this work for their helpful feedback. We also thank Peng Chen, Xiaoning Du, Jinho Jung, Cornelius Aschermann and other authors of Angora, LEAPORD, Fuzzification and Redqueen for their help with experiments. This research is supported, in part, by National Natural Science Foundation of China (Grant No. U1736209), Peng Cheng Laboratory Project of Guangdong Province PCL2018KP004. Dinghao Wu's research was supported in part by the PNC Technologies Career Development Professorship. All opinions expressed in this paper are solely those of the authors. REFERENCES

[1] A. V. Aho, R. Sethi, and J. D. Ullman, "Compilers: Principles, techniques, and tools," Addison wesley, 1986.

[2] D. Aitel, "An introduction to spike, the fuzzer creation kit," presentation slides, Aug, 2002.

[3] P. Amini and A. Portnoy, "Sulley fuzzing framework," http://www.fuzzing.org/wp-content/SulleyManual.pdf, [2019-6-1].

[4] C. Aschermann, S. Schumilo, T. Blazytko, R. Gawlik, and T. Holz, "REDQUEEN: Fuzzing with input-to-state correspondence," in Proceedings of the Network and Distributed System Security Symposium, 2019.

[5] G. Banks, M. Cova, V. Felmetsger, K. Almeroth, R. Kemmerer, and G. Vigna, "SNOOZE: Toward a stateful network protocol fuzzer," in Proceedings of the 9th International Conference on Information Security. Springer-Verlag, 2006.

[6] M. Böhme, V.-T. Pham, M.-D. Nguyen, and A. Roy-choudhury, "Directed greybox fuzzing," in Proceedings of the 2017 ACM SIGSAC Conference on Computer and Communications Security. ACM, 2017.

[7] M. Böhme, V.-T. Pham, and A. Roychoudhury, "Coverage-based greybox fuzzing as Markov chain," in Proceedings of the 2016 ACM SIGSAC Conference on Computer and Communications Security. ACM, 2016.

[8] C. Cadar, D. Dunbar, D. R. Engler et al., "KLEE: Unassisted and automatic generation of high-coverage tests for complex systems programs," in Proceedings of the 8th USENIX Conference on Operating Systems Design and Implementation. USENIX Association, 2008.

[9] S. K. Cha, M. Woo, and D. Brumley, "Program-adaptive mutational fuzzing," in Proceedings of the 2015 IEEE Symposium on Security and Privacy. IEEE, 2015.

[10] C. Chen, B. Cui, J. Ma, R. Wu, J. Guo, and W. Liu, "A systematic review of fuzzing techniques," Computers & Security, 2018.

[11] J. Chen, W. Diao, Q. Zhao, C. Zuo, Z. Lin, X. Wang, W. C. Lau, M. Sun, R. Yang, and K. Zhang, "IoTFuzzer: Discovering memory corruptions in IoT through app-based fuzzing," in NDSS, 2018.

[12] P. Chen and H. Chen, "Angora: Efficient fuzzing by principled search," in Proceedings of the 2018 IEEE Symposium on Security and Privacy. IEEE Computer Society, 2018.

[13] J. Corina, A. Machiry, C. Salls, Y. Shoshitaishvili, S. Hao, C. Kruegel, and G. Vigna, "DIFUZE: Interface aware fuzzing for kernel drivers," in Proceedings of the 2017 ACM SIGSAC Conference on Computer and Communications Security. ACM, 2017.

[14] J. De Ruiter and E. Poll, "Protocol state fuzzing of TLS implementations," in Proceedings of the 24th USENIX Conference on Security Symposium. USENIX Association, 2015.

[15] X. Du, B. Chen, Y. Li, J. Guo, Y. Zhou, Y. Liu, and Y. Jiang, "Leopard: Identifying vulnerable code for vulnerability assessment through program metrics," in Proceedings of the 41st International Conference on Software Engineering. IEEE Press, 2019.

[16] M. Eddington, "Peach fuzzing platform," https:// www.peach.tech/products/peach-fuzzer/peach-platform/, [2019-6-1].

[17] Q. Feng, R. Zhou, C. Xu, Y. Cheng, B. Testa, and H. Yin, "Scalable graph-based bug search for firmware images," in Proceedings of the 2016 ACM SIGSAC Conference on Computer and Communications Security. ACM, 2016.

[18] S. Gan, C. Zhang, X. Qin, X. Tu, K. Li, Z. Pei, and Z. Chen, "Collafl: Path sensitive fuzzing," in Proceedings of the 2018 IEEE Symposium on Security and Privacy. IEEE, 2018.

[19] V. Ganesh, T. Leek, and M. Rinard, "Taint-based directed whitebox fuzzing," in Proceedings of the 31st International Conference on Software Engineering. IEEE Computer Society, 2009.

[20] gcov, "a test coverage program," https://gcc.gnu.org/onlinedocs/gcc/Gcov.html#Gcov, fetched 2020.

[21] P. Godefroid, M. Y. Levin, and D. Molnar, "SAGE: Whitebox fuzzing for security testing," Queue, 2012.

[22] P. Godefroid, H. Peleg, and R. Singh, "Learn&#38;Fuzz: Machine learning for input fuzzing," in Proceedings of the ${32}\mathrm{{Nd}}$ IEEE/ACM International Conference on Automated Software Engineering. Piscataway, NJ, USA: IEEE Press, 2017.

[23] E. Güler, C. Aschermann, A. Abbasi, and T. Holz, "An-tiFuzz: Impeding fuzzing audits of binary executables," in Proceedings of the 28th USENIX Security Symposium, 2019.

[24] I. Haller, A. Slowinska, M. Neugschwandtner, and H. Bos, "Dowsing for overflows: A guided fuzzer to find buffer boundary violations," in Proceedings of the 22Nd USENIX Conference on Security. USENIX Association, 2013.

[25] Intel, "Processor Tracing," https://software.intel.com/en-us/blogs/2013/09/18/processor-tracing, 2013.

[26] V. Jain, S. Rawat, C. Giuffrida, and H. Bos, "TIFF: Using input type inference to improve fuzzing," in Proceedings of the 34th Annual Computer Security Applications Conference. ACM, 2018.

[27] X. Jia, C. Zhang, P. Su, Y. Yang, H. Huang, and D. Feng, "Towards efficient heap overflow discovery," in Proceedings of the 26th USENIX Security Symposium. USENIX Association, 2017.

[28] J. Jung, H. Hu, D. Solodukhin, D. Pagan, K. H. Lee, and T. Kim, "Fuzzification: Anti-fuzzing techniques," in Proceedings of the 28th USENIX Security Symposium, 2019.

[29] G. Klees, A. Ruef, B. Cooper, S. Wei, and M. Hicks, "Evaluating fuzz testing," in Proceedings of the 2018 ACM SIGSAC Conference on Computer and Communications Security. ACM, 2018.

[30] C. Lattner and V. Adve, "LLVM: A compilation framework for lifelong program analysis & transformation," in Proceedings of the international symposium on Code generation and optimization: feedback-directed and runtime optimization. IEEE Computer Society, 2004.

[31] C. Lemieux and K. Sen, "FairFuzz: a targeted mutation strategy for increasing greybox fuzz testing coverage," in Proceedings of the 33rd ACM/IEEE International Conference on Automated Software Engineering, 2018.

[32] J. Li, B. Zhao, and C. Zhang, "Fuzzing: A survey," Cybersecurity, 2018.

[33] Y. Li, B. Chen, M. Chandramohan, S.-W. Lin, Y. Liu, and A. Tiu, "Steelix: program-state based binary fuzzing," in Proceedings of the 2017 11th Joint Meeting on Foundations of Software Engineering. ACM, 2017.

[34] Z. Li, D. Zou, S. Xu, H. Jin, H. Qi, and J. Hu, "VulPecker: An automated vulnerability detection system based on code similarity analysis," in Proceedings of the 32nd Annual Conference on Computer Security Applications, 2016.

[35] Z. Li, D. Zou, S. Xu, X. Ou, H. Jin, S. Wang, Z. Deng, and Y. Zhong, "Vuldeepecker: A deep learning-based system for vulnerability detection," in 25th Annual Network and Distributed System Security Symposium, NDSS 2018, San Diego, California, USA, February 18-21, 2018, 2018.

[36] C. Lyu, S. Ji, C. Zhang, Y. Li, W.-H. Lee, Y. Song, and R. Beyah, "MOPT: Optimized mutation scheduling for fuzzers," in Proceedings of the 28th USENIX Security Symposium. USENIX Association, 2019.

[37] V. J. M. Manès, H. Han, C. Han, S. K. Cha, M. Egele, E. J. Schwartz, and M. Woo, "Fuzzing: Art, science, and engineering," CoRR, 2018.

[38] B. P. Miller, L. Fredriksen, and B. So, "An empirical study of the reliability of UNIX utilities," Comm. ACM, 1990.

[39] M. Muench, J. Stijohann, F. Kargl, A. Francillon, and D. Balzarotti, "What you corrupt is not what you crash: Challenges in fuzzing embedded devices," in Proceedings of the Network and Distributed System Security Symposium (NDSS), 2018.

[40] S. Neuhaus, T. Zimmermann, C. Holler, and A. Zeller, "Predicting vulnerable software components," in Proceedings of the 14th ACM Conference on Computer and Communications Security, 2007.

[41] H. Perl, S. Dechand, M. Smith, D. Arp, F. Yamaguchi, K. Rieck, S. Fahl, and Y. Acar, "Vccfinder: Finding potential vulnerabilities in open-source projects to assist code audits," in Proceedings of the 22nd ACM SIGSAC Conference on Computer and Communications Security, 2015.

[42] V.-T. Pham, M. Böhme, and A. Roychoudhury, "Model-based whitebox fuzzing for program binaries," in Proceedings of the 31st IEEE/ACM International Conference on Automated Software Engineering. ACM, 2016.

[43] S. Rawat, V. Jain, A. Kumar, L. Cojocar, C. Giuffrida, and H. Bos, "VUzzer: Application-aware evolutionary fuzzing," in Proceedings of the 24th Network and Distributed System Security Symposium. The Internet Society, 2017.

[44] R. Scandariato, J. Walden, A. Hovsepyan, and W. Joosen, "Predicting vulnerable software components via text mining," IEEE Transactions on Software Engineering, 2014.

[45] S. Schumilo, C. Aschermann, R. Gawlik, S. Schinzel, and T. Holz, "kAFL: Hardware-assisted feedback fuzzing for OS kernels," in 26th USENIX Security Symposium (USENIX Security 17). USENIX Association, 2017.

[46] K. Serebryany, D. Bruening, A. Potapenko, and D. Vyukov, "AddressSanitizer: A fast address sanity checker," in Usenix Conference on Technical Conference, 2012.

[47] K. Serebryany, "Continuous fuzzing with libfuzzer and addresssanitizer," IEEE, 2016.

[48] D. She, K. Pei, D. Epstein, J. Yang, B. Ray, and S. Jana, "NEUZZ: Efficient fuzzing with neural program smoothing," in Proceedings of the 2018 IEEE Symposium on Security and Privacy. IEEE, 2018.

[49] S. Sparks, S. Embleton, R. Cunningham, and C. Zou, "Automated vulnerability analysis: Leveraging control flow for evolutionary input crafting," in Twenty-Third Annual Computer Security Applications Conference (AC-SAC 2007), 2007.

[50] N. Stephens, J. Grosen, C. Salls, A. Dutcher, R. Wang, J. Corbetta, Y. Shoshitaishvili, C. Kruegel, and G. Vigna, "Driller: Augmenting fuzzing through selective symbolic execution," in Proceedings of the 23rd Network and Distributed Systems Security Symposium. The Internet Society, 2016.

[51] D. Vyukov, "syzkaller," https://github.com/google/ syzkaller, fetched 2020.

[52] J. Wang, B. Chen, L. Wei, and Y. Liu, "Skyfire: Data-driven seed generation for fuzzing," in 2017 IEEE Symposium on Security and Privacy (SP), 2017.

[53] J. Wang, Y. Duan, W. Song, H. Yin, and C. Song, "Be sensitive and collaborative: Analyzing impact of coverage metrics in greybox fuzzing," in 22nd International Symposium on Research in Attacks, Intrusions and Defenses (RAID 2019). USENIX Association, 2019.

[54] S. Wang, J. Nam, and L. Tan, "QTEP: Quality-aware test case prioritization," in Proceedings of the 2017 11th Joint Meeting on Foundations of Software Engineering. ACM, 2017.

[55] T. Wang, T. Wei, G. Gu, and W. Zou, "TaintScope: A checksum-aware directed fuzzing tool for automatic software vulnerability detection," in Proceedings of the 2010 IEEE Symposium on Security and Privacy. IEEE Computer Society, 2010.

[56] W. Xu, S. Kashyap, C. Min, and T. Kim, "Designing new operating primitives to improve fuzzing performance," in Proceedings of the 2017 ACM SIGSAC Conference on Computer and Communications Security. ACM, 2017.

[57] W. Xu, H. Moon, S. Kashyap, P.-N. Tseng, and T. Kim, "Fuzzing file systems via two-dimensional input space exploration," in IEEE Symposium on Security and Privacy, 2019.

[58] W. You, X. Wang, S. Ma, J. Huang, X. Zhang, X. Wang, and B. Liang, "Profuzzer: On-the-fly input type probing for better zero-day vulnerability discovery," in Pro-Fuzzer: On-the-fly Input Type Probing for Better Zero-Day Vulnerability Discovery. IEEE, 2019.

[59] I. Yun, S. Lee, M. Xu, Y. Jang, and T. Kim, "QSYM: A practical concolic execution engine tailored for hybrid fuzzing," in Proceedings of the 27th USENIX Security Symposium. USENIX Association, 2018.

[60] M. Zalewski, "American fuzzy lop (AFL) fuzzer," http: //lcamtuf.coredump.cx/afl/technical_details.t, 2013.

[61] G. Zhang, X. Zhou, Y. Luo, X. Wu, and E. Min, "PTfuzz: Guided fuzzing with processor trace feedback," IEEE Access, 2018.