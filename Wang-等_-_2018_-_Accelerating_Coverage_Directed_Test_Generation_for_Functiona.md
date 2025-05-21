# Accelerating Coverage Directed Test Generation for Functional Verification: A Neural Network-based Framework

Fanchao Wang, Hanbin Zhu, Pranjay Popli, Yao Xiao, Paul Bodgan and Shahin Nazarian

University of Southern California

Los Angeles, CA, USA

\{fanchaow, hanbinzh, ppopli, xiaoyao, pbogdan, shahin.nazarian\}@usc.edu

## Abstract

With increasing design complexity, the correlation between test transactions and functional properties becomes non-intuitive, hence impacting the reliability of test generation. This paper presents a modified coverage directed test generation based on an Artificial Neural Network (ANN). The ANN extracts features of test transactions and only those which are learned to be critical, will be sent to the design under verification. Furthermore, the priority of coverage groups is dynamically learned based on the previous test iterations. With ANN-based screening, low-coverage or redundant assertions will be filtered out, which helps accelerate the verification process. This allows our framework to learn from the results of the previous vectors and use that knowledge to select the following test vectors. Our experimental results confirm that our learning-based framework can improve the speed of existing function verification techniques by ${24.5}\mathrm{x}$ and also also deliver assertion coverage improvement, ranging from ${4.3}\mathrm{x}$ to ${28.9}\mathrm{x}$ , compared to traditional coverage directed test generation, implemented in UVM.

## CCS CONCEPTS

- Computing methodologies $\rightarrow$ Machine learning; - Hardware $\rightarrow$ Simulation and emulation; Coverage metrics; Assertion checking;

## KEYWORDS

Coverage Directed Test Generation; Neural Networks; UVM

## ACM Reference format:

Fanchao Wang, Hanbin Zhu, Pranjay Popli, Yao Xiao, Paul Bodgan and Shahin Nazarian. 2018. Accelerating Coverage Directed Test Generation for Functional Verification: A Neural Network-based Framework. In Proceedings of 2018 Great Lakes Symposium on VLSI, Chicago, IL, USA, May 23-25, 2018 (GLSVLSI '18), 6 pages.

https://doi.org/10.1145/3194554.3194561

## 1 INTRODUCTION

Gordon Moore's [15] observation that the number of transistors in modern chips doubles approximately every 2 years, has also resulted in exponential escalation of the complexity of modern technologies and systems. Furthermore, with the increasing customer needs for ASIC, design specifications are becoming elusive, which leads to disagreement on implementation of the functional models in HDL. Upon consensus of a system design, RTL designers translate specifications into RTL, which makes it difficult to verify all possible corner cases. In addition to misinterpretation of spec, the design flow, namely synthesis and physical design and optimization steps may introduce more functional bugs.

As the heterogeneity and complexity of state-of-the-art systems, such as those in IoT and cloud systems, grow [5][6][10][17], systematic functional verification plays a more significant role in making chips functionally reliable for users. It is not feasible to exhaustively simulate every possible case and state of the system. For example, a typical 64-bit adder calls for ${2}^{129}$ possible test vectors to cover all corner cases. Assuming a $2\mathrm{{GHz}}$ processor is used to run it and one stimulus per 16 cycles since the depth of modern pipeline is 15-20, roughly ${2}^{76}$ years are needed to verify a simple adder. On the other hand, formal verification techniques, necessitate a detailed understanding of the design well ahead of time and also may suffer from state explosion. Therefore, compared to the design of an actual chip, verification adds more energy, cost, complexity, and especially time [8]. Traditionally, a combination of constrained-random simulation-based and formal verification has been used in literature and industry to mitigate the cost of verification and achieve high coverage. However the increasing complexity of systems with larger number of inputs, states, interfaces and protocols, as well as tighter time-to-market enforced by market, have led the designers to seek beyond traditional verification algorithms towards machine learning algorithms to mitigate the verification challenges.

In the simulation-based verification, test stimuli are randomly generated under spec constraints and applied to DUV (Design Under Verification). CDG (Coverage-directed Test Generation) can be incorporated to increase the coverage rate within the limited time. However the traditional CDG can under-perform in progressively complex environments due to manual bias. For example, in state-of-the-art micro-architectures, features such as out-of-order executions would make it difficult to predict what constraints would produce tests with high coverage [11][12].

In this work, we aim to revisit the problem of verification of ASIC blocks that are typically complex RTL blocks with high chance of error bugs, due to complex functional specifications and parameters associated with them. We understand many of those parameters may turn out to be irrelevant, therefore we aim to design a verification platform that automatically learns the important dependencies and utilizes the knowledge it gains from previous test failures in order to increase the coverage and also the speed of verification. To guide test generation toward high coverage rate, we introduce an ANN (Artificial Neural Network) module into CDG with the goal of increasing the probability of selecting high coverage test cases. We use assertions coverage as the main metric of coverage rate. The assertions are grouped based on their functional and structural dependencies. Next the randomly generated transactions (i.e., test vectors) are classified using the ANN. Only the transactions with high probability of high coverage will be sent to the DUV. To further enhance the verification process, the assertion groups are prioritized and the priorities of the groups are dynamically changed. Also the low-coverage or redundant assertions will be filtered out. Given the limited verification time, our framework achieves significantly higher assertion coverage than that of the traditional CDG-based frameworks.

---

classroom use is granted without fee provided that copies are not made or distributed for profit or commercial advantage and that copies bear this notice and the full citation on the first page. Copyrights for components of this work owned by others than ACM must be honored. Abstracting with credit is permitted. To copy otherwise, or republish, to post on servers or to redistribute to lists, requires prior specific permission and/or a fee. Request permissions from permissions@acm.org.GLSVLSI '18, May 23-25, 2018, Chicago, IL, USA © 2018 Association for Computing Machinery. ACM ISBN 978-1-4503-5724-1/18/05...\$15.00 https://doi.org/10.1145/3194554.3194561

---

The rest of the paper is organized as follows. Section 2 presents the related work on verification acceleration. Section 3 describes the proposed methodology including coverage-directed test generation, assertion grouping, and the ANN setup. Section 4 provides the experimental results on performance improvement of our framework against traditional UVM. Section 5 concludes this paper.

## 2 PRIOR WORK

Several research works have utilized machine learning for formal verification based on BDDs [14] (Binary Decision Diagrams [1][2]) and SAT solvers [3][7]. Simulation-based verification has been used as a complement to formal verification, especially in cases that lack detailed understanding of design specifications.

A great amount of effort has been dedicated to improving simulation performance, using simulator optimizations such as GPG-PUs, emulators, and sampling. Event-driven gate level simulation, proposed in [4], was accelerated by general-purpose graphics processing units (GPGPUs). Also a gate level design is leveraged to exploit advantages of low switching activity under large hardware designs. New algorithms for netlist partitioning were developed and optimized for GPGPUs by first doing levelization to make parallelization of the netlist easier and then executing one level in parallel only when at least one of its input values have changed, which is event-driven. Finally, Nvidia's CUDA architecture was performed to offer a programming interface which could enable users to write software applications based on the underlying GPGPUs.

The CDG-based verification, proposed in [9], uses Bayesian networks to model coverage information and testbench directives. A dependency graph based on causal relationships between testbench directives (cause) and coverage parameters (effect) is created, which is then trained and evaluated. It can achieve full coverage faster as compared to manual test directives. However, it is more suitable for automatic creation of test directives as compared to human users.

## 3 OUR VERIFICATION FRAMEWORK

### 3.1 Coverage-Directed Test Generation

This section proposes a new test generation process to replace manual generation of transactions. Since the relationship between the test transactions and the assertions are non-intuitive, we propose an ANN-based framework that captures the non-intuitive relations and patterns, which ultimately enhances the quality of verification in terms of speed and coverage both. In addition to the relations between test transactions and assertions, there exist patterns among various design bugs (e.g., because designers tend to make similar mistakes in different parts of the design). Also considering the high complexity of the verification flow, AI can be very effective in capturing dependencies and filtering the parameters that turn out to have no impact on the outcome. We have found ANN as an effective learning methodology for the CDG process. Before elaborating the detail in each step, several terms are needed to be clarified. The transaction represents random generated test vectors under design specs. Based on the previous simulation, bias is generated by scoreboard to control the direction in following simulation. The steps of our ANN-based CDG process are as follows in Figure 1.

![0196f2a9-e799-7593-8302-26c64f93c424_1_936_256_686_403_0.jpg](images/0196f2a9-e799-7593-8302-26c64f93c424_1_936_256_686_403_0.jpg)

Figure 1: ANN-based CDG process. In the simulation phase, randomly generated transactions will be classified. Next, based on the previous simulation, the ANN eliminates irrelevant transactions and the remaining transactions will be treated as stimuli for the next round. This procedure is repeated several times to meet overall coverage requirement.

Step 1: This step is performed to prepare the training data for the ANN. Randomly generated vectors are sent to DUT. Depending on which assertions they trigger, they will be labeled. Noted that a single test vector is possibly labeled by multiple groups because it tests different aspects of the design. The design properties are written using assertions. As explained in the next section, the assertions are clustered (grouped) using the assertion grouping model.

Step 2: The ANN is trained with labeled transactions generated in the Step 1. Once the ANN is well-trained, it will be embedded as the ANN module into the modified CDG process for the rest of the verification process. Note that to efficiently use the limited verification process, our framework utilizes a conventional CDG, at start while the ANN is being trained, and switches to ANN-based CDG when the training is complete.

Step 3: As shown in the test generation block, the randomly generated test transactions are fed into the trained ANN to predict which assertion group they may target. In case they trigger assertion groups with high priorities (criticality) they will be forwarded to the DUV. Otherwise they will be filtered out. One main key to the efficiency of our approach is that our ANN would be significantly faster than the DUV simulator or emulator.

Step 4: In this step, the stimuli, chosen in step 3, is applied to the DUV (c.f. figure 1). The simulation results are monitored. if there exists any assertion group which hasn't been tested, the scoreboard will give a bias to the filter logic to skew future tests focusing on that assertion group.

Step 5: If overall coverage achieves the test plan expectation, terminate the CDG process, otherwise jump to step 3 with the bias generated in step 4 .

Theoretically, in the worst case scenario the number of labels is too large for the ANN to handle, because each assertion can in the worst case be treated as a separate category (i.e., assertion group.) However in practice the number of assertion groups can be very low. This is thanks to the fact that many assertions are related and therefore own common patterns that can be recognized by the ANN. We will show that only 6 assertion groups are sufficient for a dual-core processor with various tricky bugs, to provide significantly higher verification quality.

### 3.2 Assertion Grouping

Typically, different assertions may overlap with each other in terms of the design specifications they target. We refer to those assertions as related, and we aim to cluster them into the same group. The reason for multiple assertions to be related, is that same design structures are tested, which means some control signals, execution paths, and arithmetic/memory resources are common. Another reason for related assertions is that the designer tends to make similar mistakes, leading to assertions having overlaps, i.e., one failing would imply that the related assertions have a high probability of failing too. Clustering these similar assertions to the same group can significantly increase the efficiency of verification in terms of verification time and coverage. Assertion grouping has the following benefits: (1) It helps reducing the overall simulation time by shrinking the assertion set. During the functional verification, coverage is the main metric to evaluate the completeness of the test. Each assertion should be triggered by test stimuli to ensure the design meets its specification. If assertion set is too large, enormous time will be consumed to find stimuli which can trigger assertions. Therefore, shrinking the overall assertion set helps engineers avoid redundancy and save efforts and time. (2) In case a certain assertion fails in simulation, design bugs may exist in any components related to this assertion, therefore it is reasonable to put extra effort in verifying those components. With assertion grouping, since assertions in the same group are interrelated, each time one assertion fails, testing other assertions in the same group can help verify the related resources (datapath, etc.) with high failure probabilities.

In our grouping method, there are three factors contributing to the calculation of correlation, namely number of common structural signals (denoted by $\left. \left\{  {f}_{0}\right\}  \right)$ , length of the shared datapath $\left( \left\{  {f}_{1}\right\}  \right)$ , and common resources $\left( \left\{  {f}_{2}\right\}  \right)$ . All 3 factors or a subset of those could be used for assertion grouping. We must however address this concern: Is the process of assertion grouping based on the $\left\{  {{f}_{0},{f}_{1},{f}_{2}}\right\}$ limited to designs for which the designer knows exactly which structure would relate to which assertion? In other words, if an assertion is written based on a high level behavioral view of the design without clear understanding of how the structure of the corresponding behavior would turn out post-synthesis, would assertion grouping fail? The answer is No. Even for those assertions based on the design behavior, the common control signals can be used to group the assertions. Each factor generates its own grouping results. Since there are limited number of resources in a design, every resource can have its own assertion group, i.e., size of f2 is small. However the scenario space of $\mathrm{f}0$ and $\mathrm{f}1$ is vast for complex designs, therefore it is infeasible to create one group per scenario. We as part of our assertion group method, propose a clustering technique based on edge-labeled graph modeling to convey the correlation between assertions and efficiently cluster the assertions into groups. Similar ideas are also used in [16] and [13] to reduce the overall assertion size and clustering hardware checker. As shown in Figure 2, the edge-labeled weighted graph is defined as the Assertion Graph $\left( {AG}\right)  = \{ V, E\}$ . The set of vertices, defined as $V = \left\{  {{v}_{0},{v}_{1},{v}_{2},\ldots ,{v}_{n}}\right\}$ represents all assertions required to be inserted for the DUV, with ${v}_{n}$ , representing assertion $\mathrm{n}$ . The weight of edge ${w}_{a\;b}^{i}$ denotes the correlation between two assertions a and b when considering factor i.

To make each edge weight into a dimensionless quantity, ${w}_{a\_ b}^{i}$ denotes the standard weight between assertions a and b, while considering factor i. We apply the following normalization function to make the weight scale consistent across different factors:

$$
{w}_{a\_ b}^{i} = \frac{{w}_{a\_ b}^{\prime } - {\mu }_{i}}{{\sigma }_{i}}
$$

$$
{\mu }_{i} = \frac{1}{{N}_{i}}\mathop{\sum }\limits_{{ab}}{w}_{a\_ b}^{\prime }
$$

$$
{\sigma }_{i} = \sqrt{\frac{\mathop{\sum }\limits_{{ab}}{\left( {w}_{a\_ b}^{\prime i} - {\mu }_{i}\right) }^{2}}{{N}_{i} - 1}}
$$

where ${N}_{i}$ is the number of weights given the factor $\mathrm{i},{w}_{a\_ b}^{\prime }$ is the raw weight between two vertices $\left\{  {{v}_{a},{v}_{b}}\right\}  ,{\mu }_{i}$ is the mean and ${\sigma }_{i}$ is the standard deviation of all edge raw weights for the factor i. After calculating each edge weight, two adjacent nodes with weight higher than threshold ${\delta }_{\text{corr }}$ will be grouped together, edge weight lower than ${\delta }_{\text{corr }}$ pair will be considered as belonging to two distinct group. All assertions get grouping by three different factors separately. After group merging, the final grouping result will be a assertion group set ${G}_{\text{assertion }} = \left\{  {{g}_{1},{g}_{2},{g}_{3},\cdots ,{g}_{n}}\right\}$ and each individual group contains all related assertion vertices ${g}_{i} = \left\{  {{v}_{1},{v}_{2},{v}_{3},\cdots ,{v}_{n}}\right\}$ . Note that factors $\mathrm{f}0,\mathrm{f}1$ , and $\mathrm{f}2$ impact the relation between assertions, i.e., the edge weight of two adjacent nodes depends on them. The following is an example of assertion grouping for a pipelined processor with 5 assertions groups:

$$
{A}_{1} : I{D}_{op} =  = {R}_{\text{type }}\& {AL}{U}_{\text{src }} =  = {ADD} \mapsto  \# \left\lbrack  {3 : 4}\right\rbrack  W{B}_{\text{out }} =  = {AD}{D}_{\text{ref }}
$$

$$
{A}_{2} : I{D}_{op} =  = {R}_{\text{type }}\& \& {AL}{U}_{\text{src }} =  = {SUB} \mapsto  \# \left\lbrack  {3 : 4}\right\rbrack  W{B}_{\text{out }} =  = {SU}{B}_{\text{ref }}
$$

$$
{A}_{3} : I{D}_{op} =  = {M}_{\text{type }}\& {ME}{M}_{sl} =  = {SD} \mapsto  \# \left\lbrack  {2 : 3}\right\rbrack  {ME}{M}_{\text{wen }}
$$

$$
{A}_{4} : I{D}_{op} =  = {M}_{\text{type }}\& {ME}{M}_{sl} =  = {LD} \mapsto  \# \lbrack 2 : 3\rbrack {ME}{M}_{en}
$$

$$
{A}_{5} : I{D}_{op} =  = {BEZ}\& \& {\operatorname{Reg}}_{0} =  = 0 \mapsto  W{B}_{\text{out }} =  = {\text{ branch }}_{\text{taken }}
$$

The following describes each factor and shows how they should be considered towards calculating the raw weights:

3.2.1 Number of Overlapping Control Signals ${f}_{0}$ . It aims to capture the similarity between two assertions when considering their control signals. The ratio of the number of shared signals with average number of two assertion signals is defined by the raw weight. For example, for assertion ${A}_{3}$ and ${A}_{4}$ , there are one common control signal $\left( {I{D}_{op} =  = {M}_{\text{type }}}\right)$ and two different signals $\left( {{ME}{M}_{sl} =  = {SD}}\right.$ and ${ME}{M}_{sl} =  = {LD}$ ). Therefore the raw weight is $1/2$ .

![0196f2a9-e799-7593-8302-26c64f93c424_3_156_249_1486_415_0.jpg](images/0196f2a9-e799-7593-8302-26c64f93c424_3_156_249_1486_415_0.jpg)

Figure 2: Overview of assertion grouping procedure. First two weighted graphs are constructed based on the structural similarities between assertions. Next, any edge with a weight lower than ${\delta }_{corr}$ is considered as irrelevant, which means the corresponding pair of vertices, i.e., assertions are irrelevant. Finally only the relevant edges remains which means the corresponding pairs of vertices are group. Also the ”common resources" would be directly used for grouping. Finally the 3 grouping results are merged into distinct groups.

3.2.2 Locality ${f}_{1}$ . This factor is used to merge two assertions into a group if their majority execution data paths are overlapped. The ratio of the shared data path with the average individual total data path is calculated as the raw weight for two adjacent assertion vertices. E.g., ${A}_{1}$ relates to four stages and ${A}_{3}$ relates to three stages. The common data path between ${A}_{1}$ and ${A}_{3}$ is three stages. Therefore, the edge weight between ${A}_{1}$ and ${A}_{3}$ is $3/{3.5}$ .

3.2.3 Common Resource ${f}_{2}$ . We assume the number of resources for a given design (DUV) is limited, therefore each resource has its own assertion group. Here, ${A}_{1}$ and ${A}_{2}$ are sharing ALU, ${A}_{3}$ and ${A}_{4}$ are sharing memory block and ${A}_{5}$ uses branch predictor exclusively.

As the last step of assertion group algorithm, the groups are checked to see whether any two or more groups perfectly match each other, i.e., they have identical assertions, which means they should merge into one group. Note that groups will have common assertions, therefore groups have overlaps, however merging would occur, if and only if the exact set of assertions appear in two or more groups.

### 3.3 ANN Setting

Considering the non-trivial relations between the input vectors and the assertion groups, we utilize ANN to identify patterns within transactions. The first layer of ANN focuses more on the local patterns whereas the following layers are set to capture the patterns from the former layers on a higher perspective. The final layer therefore makes decisions from a higher perspective, based on local and more global patterns. This relation between layers and property patterns is advantageous to realize the relationship between input test vectors and the assertion groups. For instance, for the CPU design, the opcode is straightforward and we can anticipate that if certain ALU operations have bugs, then it may relate to certain opcodes. However, the relationship between certain opcodes and whether they will lead to the data dependency problem is not straightforward. Therefore, the ANN can be used to find the correlation in terms of above questions.

We have examined two configurations: In the first configuration, 32-bit vectors are sent directly to the ANN, which means the input layer has 32 neurons and each input is either 0 or 1 . In the second configuration, we categorize the vector bits into opcode, rt, rs, and ALUopcode/offset. Our testing results show that configuration number one, i.e., uncategorized vector bits, produces more accurate results. This implies that the patterns could be too complicated to be manually identified for designers. Therefore, we should let the ANN automatically figure out the patterns and choose configuration 1 as the ANN setting for our experiments.

In terms of the layers and hidden neurons within each layer, we performed several experiments to calculate the optimal configuration. We have chosen a 3-layer ANN including 1 input layer, 1 hidden layer, and 1 output layer with 32, 128, and 128 neurons in each layer, respectively. The number of neurons in the input layer is the same as the number of input bits. We use the ReLU activation function and also cross entropy loss function as the training objective. It is also worth noting that since the input data for our verification problem is not as large as the image or voice data, deep convolutional neural network might not be a proper solution for this problem.

The output layers are the assertion groups. We use the softmax function as the output layer, therefore the output numbers will be fractions, rather than 0 or 1 . To classify the vectors into assertion groups, we can either use the probability (i.e., choose the group with the highest probability) or use a threshold for the output value and in case the output value is higher than the threshold, the vector would be associated with the corresponding assertion group.

### 3.4 Test Phase

After the ANN is well-trained, it will be embedded into the test phase of the verification flow. During the test phase (c.f. figure 3), the randomly generated transactions will first be sent into the ANN for classification. The ANN will filter the redundant transactions that have no relationship with the existing assertion groups and classify the rest of the transactions to the related assertion group. The previous assertion groups proved that the assertions within each assertion group are highly correlated. The ANN extracts the relations between the input transactions and the assertion groups. Therefore, the classified transaction will target more on the assertions within assertion groups and have higher probability to trigger the assertions. Therefore, within the limited time, the ANN-based CDG process can achieve a higher functional coverage.

![0196f2a9-e799-7593-8302-26c64f93c424_4_152_236_1493_429_0.jpg](images/0196f2a9-e799-7593-8302-26c64f93c424_4_152_236_1493_429_0.jpg)

Figure 3: Overview of the test phase. The randomly generated transactions will be first classified by the ANN. The redundant transactions will be filtered whereas the rest of the transactions are valid and will be classified into related assertion groups. After ANN classification, the valid transaction will be sent to the DUV and trigger assertions in order to achieve a higher coverage.

Table 1: Training Time for Different Assertion Groups.

<table><tr><td/><td>AG0</td><td>AG1</td><td>AG2</td><td>AG3</td><td>AG4</td><td>AG5</td></tr><tr><td>#Neuron</td><td>30</td><td>30</td><td>30</td><td>10</td><td>30</td><td>20</td></tr><tr><td>Time (s)</td><td>11.3</td><td>9.98</td><td>13.4</td><td>12.1</td><td>10.1</td><td>11.6</td></tr></table>

![0196f2a9-e799-7593-8302-26c64f93c424_4_152_992_737_406_0.jpg](images/0196f2a9-e799-7593-8302-26c64f93c424_4_152_992_737_406_0.jpg)

Figure 4: The distribution of passing and failing vectors.

## 4 EXPERIMENTAL RESULTS

To test the effectiveness of our ANN-based verification framework we have implemented a dual-core RISC processor, which includes two cores with a static pipeline and DMEM unit, as well as a dispatcher unit and a unified memory. We use the assertion grouping algorithm discussed in Section 3.2 to cluster all processor's assertions into 6 separated groups, i.e., AG0 to AG5. These assertions cover processor specifications such as control hazard handling, arithmetic calculation correctness, memory read/write, etc.

### 4.1 Configuration and Distribution of Vectors

We chose the Neural Net Pattern Recognition toolbox of MATLAB as our ANN platform. In this experiment, we tested the different number of neurons in the hidden layer to find the best configuration with the least error rate and training time for assertion groups. In our experiments, a total of 83620 vectors were randomly generated. ${70}\%$ of those vectors were used for training; 15% of those vectors for testing; and the remaining 15% for validation. Note that the randomly generated vectors are the ones which are correctly classified and misclassified due to some structural and behavioral bugs, such as misconnections of control signals or wrong data forwarding. The vectors that result in failure were labeled based on which assertion group they trigger. Six labels are one-bit numbers ${d}_{5},{d}_{4},\ldots ,{d}_{0}$ where ${d}_{i} = 1$ means the vector passes all the assertions of group $\mathrm{i}$ and ${d}_{i} = 0$ means it triggers at least one assertion in group i. These labels are stored and used for the ANN training. Figure 4 illustrates the distribution of passing and failing vectors for each of the 6 assertion groups. Note that the distribution varies from assertion groups, due to the nature of properties related to each assertion group as discussed in Section 3.2. Table 1 presents the training times for the 6 ANNs corresponding to the 6 assertion groups, given the number of neurons in the hidden layer. Note that the number of neurons is selected such that accuracy over training time is maximized. The training times of the 6 ANNs corresponding to the 6 assertion groups should overlap in time, to maximize the parallelism. As reported in Table 1, the training times of the ANNs are almost identical, which helps with the parallel training of these ANNs.

### 4.2 Failing Vector Prediction Rate (FVPR)

There are 4 possibilities for the ANN in terms of how it would treat a given test vector:

Case 1: ANN: Fail, DUV: Fail;

Case 2: ANN: Pass, DUV: Fail;

Case 3: ANN: Fail, DUV: Pass;

Case 4: ANN: Pass, DUV: Pass.

Out of the 4 cases, only vectors in cases 1 and 3 are identified by ANN as vectors to be fed into the DUV. Vectors in cases 2 and 4 are eliminated, as the ANN calculates them as passing all assertions. Also note that among the 4 cases, vectors in case 2 are wrongly eliminated. Therefore the Failing Vector Prediction Rate (FVPR) of the ANN is formulated as $\frac{Case1}{{Case2} + {Case1}}$ .

Table 2 shows the Failing Vector Prediction Rate for each assertion group, calculated during the test phase. The number of test vectors during the test phase is ${15}\%$ of total 83620, i.e.,12543. Note that FVPR is calculated based on the ratio of row 2 in Table 2 divided by its row 3 . Also note that FVPR for each ANN is independent of the FVPRs of the other ANNs. All the 6 ANNs are used in parallel to decide whether a future vector should be applied to DUV or pruned. A vector is pruned only when all the 6 ANNs report it as a passing vector. This implies that the average FVPR among all the 6 ANNs would be a measure of the quality of verification in terms of managing the randomly generated vectors, i.e., dropping them, or applying them to DUV.

Table 2: Success rate and assertion hit results for the 6 ANNs.

<table><tr><td colspan="7">Testing Data</td></tr><tr><td/><td>AG0</td><td>AG1</td><td>AG2</td><td>AG3</td><td>AG4</td><td>AG5</td></tr><tr><td>FVPR</td><td>34.36</td><td>95.55</td><td>86.54</td><td>43.06</td><td>51.11</td><td>71.20</td></tr><tr><td>Case 1</td><td>1660</td><td>2986</td><td>463</td><td>1518</td><td>415</td><td>136</td></tr><tr><td>Case 1 + Case 2</td><td>4831</td><td>3125</td><td>535</td><td>3525</td><td>812</td><td>191</td></tr></table>

Table 3: FVPR comparison between our ANN-based and the traditional CDG.

<table><tr><td/><td>AG0</td><td>AG1</td><td>AG2</td><td>AG3</td><td>AG4</td><td>AG5</td></tr><tr><td>Traditional CDG</td><td>380</td><td>201</td><td>16</td><td>271</td><td>89</td><td>8</td></tr><tr><td>ANN-based CDG</td><td>1660</td><td>2986</td><td>463</td><td>1518</td><td>415</td><td>136</td></tr><tr><td>Improvement</td><td>4.37</td><td>14.86</td><td>28.94</td><td>5.60</td><td>4.66</td><td>17</td></tr></table>

### 4.3 Performance Evaluation

In addition to using FVPR to make comparisons among different ANN-based implementations of our framework, there are two metrics to evaluate the performance of our ANN-based framework when compared to a baseline technique: one is the measure of the quality of verification in terms of vectors triggering assertions in a limited time. This could be regarded as a measure of coverage, because the higher the number of triggered assertions, the higher the number of detected design errors. Another measure of performance is the time it takes to analyze a given set of test vectors. First we compare our ANN-based CDG verification process with the traditional CDG assuming both are given a fixed verification time limit. This comparison helps us measure the impact of ANN training time overhead. As shown in Table 3, given the same time, our ANN-based CDG determines 1660 vectors for AG0, as case 1 vectors (as defined in section 4.2), which means that these vectors are predicted to trigger at least one assertion in AG0, and they indeed trigger at least one assertion, when applied to DUV. However, using the traditional CDG, only 380 vectors are able to trigger an assertion from AG0. This shows a minimum assertion coverage improvement of ${4.37}\mathrm{x}$ considering a limited verification time. Note that the assertion coverage improvement is as high as ${28.94}\mathrm{x}$ . Next we consider the same number of randomly generated vectors (in this case about 12000 vectors) and compare our framework with the traditional CDG in terms of the verification time. The traditional CDG takes 1.84 seconds, whereas our ANN-based CDG takes a maximum of 0.075 seconds, which means that our ANN-based CDG is 24.5 times as fast as the traditional CDG. The reason is that our ANN-based framework can label the vectors and prune the passing vectors without driving them into the DUV, which is much faster than the simulation on DUV.

## 5 CONCLUSIONS

Design complexity of modern processing systems has significantly impacted the reliability of the conventional test generation. Our framework integrates ANN into the CDG (Coverage-Directed test Generation) process to accelerate the verification process and increase the coverage in a limited verification time. The assertions are clustered using an algorithm for assertion grouping with the goal of efficiently managing properties and the relations among them. The ANN is used to label the randomly generated vectors based on the corresponding assertion group. If test vectors trigger assertion groups with high probability, they will be driven into the DUV whereas the irrelevant transactions are pruned. Our experimental results confirm that our ANN framework can eliminate as high as 40.2% of test vectors with much faster runtime, when compared to a standard testbench implemented in SystemVerilog UVM. The resulting speedup reaches ${24.5}\mathrm{x}$ and assertion coverage improvement ranges from ${4.37} \times$ to ${28.94}\mathrm{x}$ .

Table 4: Simulation time comparison between our ANN-based and the traditional CDG.

<table><tr><td/><td>AG0</td><td>AG1</td><td>AG2</td><td>AG3</td><td>AG4</td><td>AG5</td></tr><tr><td>Traditional CDG</td><td>1.84</td><td>1.84</td><td>1.84</td><td>1.84</td><td>1.84</td><td>1.84</td></tr><tr><td>ANN-based CDG</td><td>0.075</td><td>0.0747</td><td>0.073</td><td>0.0534</td><td>0.0672</td><td>0.0586</td></tr><tr><td>Speedup</td><td>24.53</td><td>24.63</td><td>25.20</td><td>34.46</td><td>27.38</td><td>31.40</td></tr></table>

## 6 ACKNOWLEDGEMENT

The authors gratefully acknowledge the support by the the U.S. Army Defense Advanced Research Projects Agency (DARPA) under grant no. W911NF-17-1-0076, DARPA Young Faculty Award under grant no. N66001-17-1-4044, the National Science Foundation under CAREER Award CPS-1453860 and the Okawa Foundation Award.

## REFERENCES

[1] R. E. Bryant. "Graph-based algorithms for boolean function manipulation". In: TC. 1986.

[2] R. E. Bryant. "Symbolic Boolean manipulation with ordered binary-decision diagrams". In: CSUR. 1992.

[3] B. Bünz and M. Lamm. "Graph Neural Networks and Boolean Satisfiability". In: 2017.

[4] D. Chatterjee, A. DeOrio, and V. Bertacco. "Event-driven gate-level simulation with GPGPUs". In: DAC. 2009.

[5] M. Cheng, J. Li, and S. Nazarian. "DRL-cloud: Deep reinforcement learning-based resource provisioning and task scheduling for cloud service providers". In: ASPDAC. 2018.

[6] S. Crago, K. Dunn, and P. Eads. "Heterogeneous cloud computing". In: CLUSTER.

[7] D. Devlin and B. O'Sullivan. "Satisfiability as a classification problem". In: AICS. 2008.

[8] D. Ernst. "Complexity and internationalisation of innovation". In: IJIM. 2005.

[9] S. Fine and A. Ziv. "Coverage directed test generation for functional verification using bayesian networks". In: DAC. 2003.

[10] M. Guevara, B. Lubin, and B. C Lee. "Navigating heterogeneous processors with market mechanisms". In: HPCA. 2013.

[11] O. Guzey and L. Wang. "Coverage-directed test generation through automatic constraint extraction". In: HLVDT. 2007.

[12] C. Ioannides and K. I Eder. "Coverage-directed test generation automated by machine learning-a review". In: (2012).

[13] M. H. Neishaburi and Z. Zilic. "Enabling efficient post-silicon debug by clustering of hardware-assertions". In: DATE. 2010.

[14] M. L. Rebaiaia, J. M. Jaam, and AM Hasnah. "A neural network algorithm for hardware-software verification". In: ICECS. 2003.

[15] R. R. Schaller. "Moore's law: past, present and future". In: IEEE spectrum. 1997.

[16] J. G Tong, M. Bottlé, and Z. Zilic. "Assertion clustering for compacted test sequence generation". In: ISQED. 2012.

[17] Y. Xue, J. Li, S. Nazarian, and P. Bogdan. "Fundamental Challenges Toward Making the IoT a Reachable Reality: A Model-Centric Investigation". In: TODAES. 2017.