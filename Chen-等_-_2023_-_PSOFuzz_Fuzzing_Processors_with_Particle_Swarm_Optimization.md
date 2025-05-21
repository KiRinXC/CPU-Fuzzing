# PSOFuzz: Fuzzing Processors with Particle Swarm Optimization

Chen Chen ${}^{*, \dagger  }$ , Vasudev Gohil ${}^{*, \dagger  }$ , Rahul Kande ${}^{ \dagger  }$ , Ahmad-Reza Sadeghi ${}^{ \ddagger  }$ , and Jeyavijayan (JV) Rajendran ${}^{ \dagger  }$ ${}^{ \dagger  }$ Texas A&M University, USA, ${}^{ \ddagger  }$ Technische Universität Darmstadt, Germany ${}^{ \dagger  }$ \{chenc, gohil.vasudev, rahulkande, jv.rajendran\}@tamu.edu, *\{ahmad.sadeghi\}@trust.tu-darmstadt.de

Abstract-Hardware security vulnerabilities in computing systems compromise the security defenses of not only the hardware but also the software running on it. Recent research has shown that hardware fuzzing is a promising technique to efficiently detect such vulnerabilities in large-scale designs such as modern processors. However, the current fuzzing techniques do not adjust their strategies dynamically toward faster and higher design space exploration, resulting in slow vulnerability detection, evident through their low design coverage.

To address this problem, we propose PSOFuzz, which uses particle swarm optimization (PSO) to schedule the mutation operators and to generate initial input programs dynamically with the objective of detecting vulnerabilities quickly. Unlike traditional PSO, which finds a single optimal solution, we use a modified PSO that dynamically computes the optimal solution for selecting mutation operators required to explore new design regions in hardware. We also address the challenge of inefficient initial seed generation by employing PSO-based seed generation. Including these optimizations, our final formulation outperforms fuzzers without PSO. Experiments show that PSOFuzz achieves up to ${15.25} \times$ speedup for vulnerability detection and up to ${2.22} \times$ speedup for coverage compared to the state-of-the-art simulation-based hardware fuzzer.

Index Terms-Hardware Security, Security Vulnerabilities, Fuzzing, Particle Swarm Optimization

## I. INTRODUCTION

Hardware designs are becoming increasingly complex to meet the rising need for custom hardware and increased performance. However, as the design complexity grows, verification dominates the development lifecycle-over ${70}\%$ of resources are spent to verify the security of a hardware design [1]. Therefore, a verification technique that can expose design flaws/vulnerabilities as early as possible is critical [2]-[5].

Recent research has shown that coverage feedback-based fuzzers can effectively detect security vulnerabilities in modern processors and achieve coverage faster than traditional hardware verification techniques such as random regression [6]-[13]. Fuzzers start with seed inputs and apply predefined mutation operators on current inputs to generate new inputs. Fuzzers utilize one or several coverage metrics to track the activity caused by the inputs in the hardware and guide themselves to generate inputs that explore new hardware regions, thereby accelerating vulnerability detection. These fuzzers successfully found vulnerabilities that execute undefined behaviors, compromise memory isolation, lead to privilege escalations, etc., in hardware designs. Due to their promising potential, semiconductor giants such as Intel and Google are actively developing new and efficient hardware fuzzers for vulnerability detection [10], [14].

Though hardware fuzzing is promising, its coverage increment usually stagnates quickly, leaving design spaces unexplored and vulnerabilities undetected. One of the main reasons is that all the existing hardware fuzzers schedule the mutation operators and generate seeds using static schemes, and lack feedback-guided dynamic updates via interaction with the coverage achieved and the complexity of the hardware (more details in Sec. V) [6]-[13]. For example, [9] generates input instructions with the same operands to verify the renaming strategies in out-of-order processors. However, this strategy is not optimal for processors without renaming strategies. While [11] uses profiling to analyze the correlation between the mutation operators and the processor instructions, the scheduling is static throughout fuzzing and does not update with coverage achieved. Therefore, to dynamically adjust the schedule of mutation operators and seed generation, we develop an approach to equip any hardware fuzzer with particle swarm optimization (PSO) targeting faster vulnerability detection and coverage achievement.

PSO is a bio-inspired algorithm that uses a swarm consisting of multiple particles to find the optimal solution in a high-dimensional space [15]. Since fuzzers usually run multiple threads in parallel to accelerate the design space exploration, we model the scheduler of each thread's mutation operations as a particle. The probability distribution of each mutation operator being selected composes the position of a particle. Each particle updates its position based on the highest coverage achieved by itself and the highest coverage achieved by the swarm in the current iteration.

Researchers have used PSO in software fuzzing to find the optimal probability distribution of mutation operators [16]. Results show a significant improvement over the fuzzers without PSO. However, one cannot simply use their technique for hardware fuzzing for the following reasons: (i) They assume a static optimal probability distribution of the mutation operators, which is not true for hardware fuzzing. Our experiments show that if a static distribution is used, two critical vulnerabilities are not detected even after using PSO (more details in Sec. III-B and IV). (ii) They use randomly generated programs as seeds, which is not ideal for hardware fuzzing (more details in Sec. III-C). To successfully apply PSO to hardware fuzzing, we must address two critical challenges:

---

*These authors contributed equally to this work.

---

- Challenge 1: Saturation of Particles' Performance. Naively applying PSO to hardware fuzzing results in the particles converging to a fixed position leading to a lack of coverage improvement and vulnerabilities undetected.

- Challenge 2: Ineffective Seed Generation. The seeds from which the fuzzing loop starts strongly affect the coverage achieved. Traditional fuzzers use either statically or randomly generated seeds, which results in slower coverage increments. Moreover, the performance of applying PSO to hardware fuzzing is also subject to the seed: poor quality seeds result in poor coverage.

We address these challenges by designing specific solutions (in Sec. III-B and III-C) that result in faster vulnerability detection and coverage achievement. In particular, we augment PSO with a reset strategy to schedule mutation operators dynamically. Moreover, we employ PSO to select high-quality seed inputs resulting in faster coverage. Overall, the main contributions of this work are as follows:

1) To the best of our knowledge, we develop the first technique that uses the PSO algorithm to schedule the mutation operators and generate seed inputs in hardware fuzzing.

2) We overcome challenges in adapting PSO to hardware fuzzers. In particular, we develop optimizations to reset the particles for selecting mutation operators and a novel PSO-based seed generation algorithm for hardware fuzzers.

3) We evaluate PSOFuzz on three widely-used open-source RISC-V processors and achieve up to ${15.25} \times$ speedup in detecting vulnerabilities and up to ${2.22} \times$ speedup in achieving coverage compared to the state-of-the-art simulation-based fuzzer without PSO.

## II. BACKGROUND

## A. Hardware Processor Fuzzers

Hardware processor fuzzers iteratively generate testing inputs (i.e., inputs or tests) such as binary executables to detect vulnerabilities in target hardware (i.e., design-undertest (DUT)). These fuzzers mainly consist of a seed generator, mutation engine, feedback engine, and vulnerability detector [9], [11]-[13]. The seed generator generates an initial set of tests called seeds. Fuzzer simulates the hardware with these seeds to generate feedback and output. The feedback engine captures the activity in hardware, such as toggling of the value of flip-flops [11], as coverage data during simulation. The mutation engine modifies the tests that achieve new coverage (i.e., interesting tests) to create new tests. For example, the Bitflip mutation operator flips one random bit in the test [17]. The vulnerability detector detects vulnerabilities in hardware by comparing its output with the output of a golden-reference model (GRM) [9], [11], [13].

Existing hardware fuzzers use static mutation operator scheduling and seed generation schemes which cannot dynamically adjust based on the complexity of hardware and the coverage achieved, slowing down vulnerability detection and design space exploration (more details in Sec. III and V).

Listing 1: Code snippet from CVA6 [20] processor (simplified to improve readability)

---

unique case (instr.opcode)

		...

			CSRRS: begin

				if (instr.rs1 == 5'b0)

					instruction_o.op = CSR_READ;// c1

				else

					instruction_o.op = CSR_SET; // c2

---

### B.PSO Algorithm

PSO is a bio-inspired algorithm that allocates multiple particles to iteratively search for an optimal solution [15]. The position of a particle is a candidate solution and is updated based on its velocity. The velocity is determined by the best position the particle ever achieved and the best position the entire swarm ever achieved. A particle’s velocity, ${v}_{i}$ , and position, ${p}_{i}$ , are updated as

$$
{v}_{i}\left( {t + 1}\right)  = k \times  {v}_{i}\left( t\right)  + {r}_{1} \times  \left( {{l}_{i}^{\text{best }}\left( t\right)  - {p}_{i}\left( t\right) }\right)  \tag{1}
$$

$$
+ {r}_{2} \times  \left( {{g}^{\text{best }}\left( t\right)  - {p}_{i}\left( t\right) }\right)
$$

$$
{p}_{i}\left( {t + 1}\right)  = {p}_{i}\left( t\right)  + {v}_{i}\left( {t + 1}\right)  \tag{2}
$$

Here, ${l}_{i}^{\text{best }}\left( t\right)$ represents the best position the particle $i$ ever achieved since the beginning (i.e., $t = 0$ ), and ${g}^{\text{best }}\left( t\right)$ represents the best position ever achieved by any particle in the swarm. $k$ is a pre-defined constant, and ${r}_{1}$ and ${r}_{2}$ are two random displacement weights that decide how fast the particle $i$ will move toward ${l}_{i}^{\text{best }}\left( t\right)$ and ${g}^{\text{best }}\left( t\right)$ , respectively.

PSO has been widely applied to optimize the performance of applications due to its searching efficiency and convergence speed [18]. Though the solutions found by PSO are heuristic results, they are usually close to the real global optimum [19]. The software community has also leveraged PSO (e.g., MOPT [16]) to detect vulnerabilities (e.g., memory crashes) in programs. However, for hardware fuzzing, the optimal probability distribution of the mutation operators changes with coverage achieved so far, and the quality of seeds greatly affects the performance of the fuzzer. Hence, MOPT cannot be applied for hardware fuzzing directly.

## III. PSOFuzz: FUZZING PROCESSORS WITH PSO

In this section, we first formulate the problem of selecting optimal mutation operators as a PSO problem. However, the formulation has critical limitations, such as saturation of particles' performance and inefficient seed generation. We analyze these limitations and devise appropriate solutions, resulting in a final formulation that outperforms fuzzers without PSO.

## A. Selecting Mutation Operators Using PSO

Most existing hardware fuzzers select mutation operators uniformly at random, i.e., they have an equal probability of selecting from all possible mutation operators [6]-[13]. This is not ideal as mutation operators have varying impacts on covering new points in hardware. The following example demonstrates this.

Motivational Example. Consider a component from the decoder module in the CVA6 [20] processor as shown in Listing 1. When decoding the CSRRS rd, csrReg, rs1 instruction, the processor performs CSR_READ (line 5) or CSR_SET (line 7) operation on the control and status register (CSR), csrReg, based on the register operand rs1 (line 4). Assume the coverage points ${c}_{1}$ and ${c}_{2}$ in the if-else block as shown in line 5 and line 7. Suppose the instruction in the current input is CSRRS x16, stvec, x0 . The value of rs1 for this instruction is 0 . Hence the if condition evaluates to True, and the CSR_READ operation is performed on the CSR, stvec, covering the point ${c}_{1}$ . Now, our objective, when mutating this input, is to cover ${c}_{2}$ . Also, suppose we have only two mutation operators in this toy example: Bitflip, which flips one bit of the operand, rs1, and OpcodeMut, which mutates the opcode to another opcode. If both mutation operators are equally likely to be selected, the probability of covering ${c}_{2}$ after mutation is $\left( {{0.5} \times  0}\right)  + \left( {{0.5} \times  1}\right)  = {0.5}$ . On the other hand, if we assign a higher weight to Bitflip, say 0.9, the probability of covering ${c}_{2}$ is $\left( {{0.1} \times  0}\right)  + \left( {{0.9} \times  1}\right)  = {0.9}$ . So, selecting mutation operators with equal probability is not ideal. To address this problem, we formulate the problem of finding the optimal weights for the mutation operators as a PSO problem, as explained next.

Notation. Let $M$ be an ordered list of all mutation operators (i.e., $M\left\lbrack  j\right\rbrack$ is the ${j}^{\text{th }}$ mutation operator). Let ${W}^{M} =$ $\left\lbrack  {{w}_{1}^{M},{w}_{2}^{M},\ldots ,{w}_{\left| M\right| }^{M}}\right\rbrack$ be a weight vector for the $\left| M\right|$ operators. So, ${w}_{j}^{M}$ is the weight of the ${j}^{\text{th }}$ mutation operator $M\left\lbrack  j\right\rbrack$ . Note that since we interpret the weights as the probabilities of selecting the mutation operators, we normalize the weights so that $\mathop{\sum }\limits_{{j = 1}}^{\left| M\right| }{w}_{j}^{M} = 1$ and ${w}^{M} \geq  0,\forall {w}^{M} \in  {W}^{M}$ .

Particles. We associate a particle with one thread of the fuzzer. Since fuzzers run multiple, say $\left| N\right|$ , threads simultaneously, we have a swarm of $\left| N\right|$ particles associated with the fuzzer. Each of these $\left| N\right|$ threads has a different weight vector ${W}_{i}^{M}$ for mutating the inputs in its thread. Each such ${W}_{i}^{M}$ is assigned as the position ${p}_{i}^{M}$ of the corresponding particle ${i}^{M}$ ( $M$ indicates that the particle refers to mutation operators). Mathematically, ${W}_{i}^{M} = {p}_{i}^{M}$ .

Local Best Position. Following PSO, we assign the local best position, ${l}_{i}^{\text{best,}M}$ , of a particle as the best position of that particle so far. In our case, we define a position at iteration ${t}_{2},{p}_{i}^{M}\left( {t}_{2}\right)$ , as better than the position at iteration ${t}_{1},{p}_{i}^{M}\left( {t}_{1}\right)$ , if and only if ${p}_{i}^{M}\left( {t}_{2}\right)$ results in higher coverage than ${p}_{i}^{M}\left( {t}_{1}\right)$ .

Global Best Position. Likewise, following PSO, we assign a single global best position, ${g}^{\text{best,}M}$ , as the best position of all particles in the swarm.

Fig. 1 illustrates the high-level flow of this formulation. Like other hardware fuzzers, we generate a set of seeds, which are simulated to obtain their coverage information. Then, we use this coverage information to update the ${L}^{\text{best }, M} =$ $\left\lbrack  {{l}_{1}^{\text{best }, M},{l}_{2}^{\text{best }, M},\ldots ,{l}_{\left| N\right| }^{\text{best }, M}}\right\rbrack$ and the ${g}^{\text{best }, M}$ . Then, we update the velocities $\left( {v}_{i}^{M}\right)$ and positions $\left( {p}_{i}^{M}\right)$ of all particles according to Eqs. (1) and (2). We sample the mutation operators according to the updated positions $\left( {p}_{i}^{M}\right)$ , i.e., mutation operator weights $\left( {W}_{i}^{M}\right)$ , and perform the mutations to generate new tests. These new tests are simulated to obtain coverage, and the cycle continues. After several iterations, the particles are expected to converge to the optimal solution.

![0196f2bf-9a53-7333-89ae-c9779e21bd0b_2_963_136_645_251_0.jpg](images/0196f2bf-9a53-7333-89ae-c9779e21bd0b_2_963_136_645_251_0.jpg)

Fig. 1: Flow for integrating PSO in TheHuzz.

This preliminary solution provides a way to find a weight distribution of the mutation operators. However, it has two critical limitations, as explained below.

## B. Reset Particles and Seeds

Challenge 1: Saturation of Particles' Performance. Traditional PSO is designed so that the particles converge to an optimum (ideally the global optimum) in the solution space. However, for fuzzing hardware, one must find new coverage points in each iteration that have not been covered. In other words, the optimal solution (i.e., the weights of the mutation operators) changes based on the coverage achieved so far, as explained in the following example.

Motivational Example. Refer to Listing. 1 with two coverage points, ${c}_{1}$ and ${c}_{2}$ . Without loss of generality, assume two mutation operators ${m}_{1}$ and ${m}_{2}$ with the following property: $\mathcal{P}\left( {{c}_{1} \mid  {m}_{1}}\right)  = {0.9}$ and $\mathcal{P}\left( {{c}_{1} \mid  {m}_{2}}\right)  = {0.2}$ , where $\mathcal{P}\left( {{c}_{i} \mid  {m}_{j}}\right)$ denotes the probability of covering ${c}_{i}$ conditioned on the event that mutation operator ${m}_{j}$ is used ${}^{1}$ Suppose during the first iteration $\left( {t = 1}\right)$ , the weights for the two mutation operators are equal, i.e., 0.5 . Then, the probability of covering ${c}_{1},\mathcal{P}{\left( {c}_{1}\right) }^{t = 1} = \left( {{0.5} \times  {0.9}}\right)  + \left( {{0.5} \times  {0.2}}\right)  = {0.55}$ , and $\mathcal{P}{\left( {c}_{2}\right) }^{t = 1} = \left( {{0.5} \times  {0.1}}\right)  + \left( {{0.5} \times  {0.8}}\right)  = {0.45}$ . Now, if ${c}_{1}$ is covered in the first iteration, then, for the second iteration $\left( {t = 2}\right)$ , having equal weights for both operators results in $\mathcal{P}{\left( {c}_{2}\right) }^{t = 2} = \mathcal{P}{\left( {c}_{2}\right) }^{t = 1} = {0.45}$ . On the other hand, if we have a larger weight for ${m}_{2}$ , say 0.9, then $\mathcal{P}{\left( {c}_{2}\right) }^{t = 2} =$ $\left( {{0.1} \times  {0.1}}\right)  + \left( {{0.9} \times  {0.8}}\right)  = {0.73}$ . In fact, to maximize the likelihood of covering ${c}_{2}$ in the second iteration, the weight for ${m}_{2}$ should be 1 .

This simple example shows that to maximize the likelihood of covering the maximal number of points with the minimal number of iterations (i.e., input tests), the weights of the mutation operators should be decided dynamically based on the coverage achieved so far. However, during our experiments with the preliminary PSO formulation, we observe that the positions of the particles $\left( {p}_{i}^{M}\right)$ saturate after a few iterations because their velocities $\left( {v}_{i}^{M}\right)$ become zero according to Eq. (1). In other words, the weights of selecting the mutation operators $\left( {W}_{i}^{M}\right)$ stagnate very quickly. Fig. 2 (a) demonstrates this phenomenon for the CVA6 processor. The particles saturate within 50 iterations. Due to this, the preliminary formulation fails to detect critical vulnerabilities (more details in Sec. IV).

---

${}^{1}$ Note that since the coverage points belong to two different mutually exclusive and exhaustive branches, $\mathcal{P}\left( {{c}_{2} \mid  {m}_{1}}\right)  = {0.1}$ and $\mathcal{P}\left( {{c}_{2} \mid  {m}_{2}}\right)  = {0.8}$ .

---

![0196f2bf-9a53-7333-89ae-c9779e21bd0b_3_142_133_1517_543_0.jpg](images/0196f2bf-9a53-7333-89ae-c9779e21bd0b_3_142_133_1517_543_0.jpg)

Fig. 2: Selection probability trend for all mutation operators for CVA6 [20] (a) without and (b) with reset

Solution 1. To address challenge 1, we reset the position $\left( {p}_{i}^{M}\right)$ and velocity $\left( {v}_{i}^{M}\right)$ of the particles that saturate, i.e., that do not yield new coverage. Resetting removes saturating particles and replaces them with new particles, leading to new weights for the mutation operators $\left( {W}_{i}^{M}\right)$ . Along with resetting the position and velocity of the saturated particle, we also reset the associated seed because the position and velocity of a particle are a function of the seed it started from (as seen in Fig. 1). Using the same seed when resetting the particle will likely lead to the exploration of design space that has already been explored by that particle. This is suboptimal since our objective is to cover new regions in the design.

The condition for resetting the particles and their seeds is decided based on the coverage trend in recent iterations. We reset when there is no improvement in coverage for ${\beta }^{M}$ consecutive iterations. Here, ${\beta }^{M}$ controls the trade-off between runtime and exploitation of learned knowledge by the particles. Larger values of ${\beta }^{M}$ can lead to a deeper exploration of design but will incur runtime overhead since the particles are not reset quickly after they saturate. On the other hand, smaller values of ${\beta }^{M}$ can reduce runtime but will make the algorithm similar to random exploration, which will not cover deeper regions of the design.

Fig. 2 (b) shows the positions of the particles for different mutation operators after implementing this solution for CVA6. When the particles saturate, they are reset (along with the associated seeds), resulting in higher coverage speedup (as seen in Table II). Algorithm 1 details how this solution is incorporated using a reset monitor, RstMon. It takes as input a set of particles, their ${L}^{\text{best }}$ , resetting threshold $\left( \beta \right)$ , a variable to count the number of consecutive iterations with no improvement(Ct), and a function $\mathcal{F}$ that returns the fitness (i.e., coverage) of a particle. The algorithm iterates over all particles and compares the coverage with the particle's local best coverage (lines 2 and 3). If a particle's coverage is more than its local best coverage, the local best is updated, and the counter(Ct)is reset (lines 4 and 5). Otherwise, the counter is incremented, indicating no improvement in coverage for one more consecutive iteration for that particle (line 7). If the counter exceeds the threshold $\beta$ for a particle, that particle is added to the set of particles to be reset, rst_P (lines 8 and 9). Finally, the algorithm updates the global best $\left( {g}^{\text{best }}\right)$ using all particles' local bests (line 10).

Algorithm 1: RstMon: Reset monitor

---

Input: $P,{L}^{\text{best }},\beta ,{Ct},\mathcal{F}$

Output: ${L}^{\text{best }},{g}^{\text{best }},{Ct},{rst}\_ P$

rst_P $\leftarrow  \phi$

for $i \in  P$ do

	if $\mathcal{F}\left( {p}_{i}\right)  > \mathcal{F}\left( {{L}^{\text{best }}\left\lbrack  i\right\rbrack  }\right)$ then

		${L}^{\text{best }}\left\lbrack  i\right\rbrack   \leftarrow  {p}_{i}$

		${Ct}\left\lbrack  i\right\rbrack   \leftarrow  0$

	else

		${Ct}\left\lbrack  i\right\rbrack   \leftarrow  {Ct}\left\lbrack  i\right\rbrack   + 1$

	// add particle to the reset set

	if ${Ct}\left\lbrack  i\right\rbrack   > \beta$ then

		rst_ $P \leftarrow$ rst_ $P \cup  \{ i\}$

${g}^{\text{best }} \leftarrow$ particle with highest ${L}^{\text{best }}$

return ${L}^{\text{best }},{g}^{\text{best }},$ rst_P, Ct

---

## C. Seed Generation Using PSO

Challenge 2: Ineffective Seed Generation. A drawback of existing hardware fuzzers (as well as our preliminary formulation) is that they use a static probability distribution of instructions to generate the seeds [6]-[8], [11]-[13]. However, the optimal set of instructions required by the seeds to trigger new coverage points changes with time during fuzzing ${}^{2}$ Moreover, the performance of each particle for mutation operators $\left( {i}^{M}\right)$ , i.e., the number of iterations it survives, is highly related to the quality of its seed.

---

${}^{2}$ An example similar to the one described in Sec. III-B can demonstrate this, but we omit it in the interest of space.

---

![0196f2bf-9a53-7333-89ae-c9779e21bd0b_4_204_136_1390_472_0.jpg](images/0196f2bf-9a53-7333-89ae-c9779e21bd0b_4_204_136_1390_472_0.jpg)

Fig. 3: PSOFuzz framework.

Solution 2. To address this problem of static distribution of instructions for seed generation, we devise a dynamic seed generation algorithm using PSO. The idea is to map the problem of identifying optimal probabilities of the instructions for seeds as a PSO problem. Next, we discuss this formulation.

Notation. Let $T$ be an ordered list of instruction types, such as R-type and I-type [21]. Let ${W}^{T} = \left\lbrack  {{w}_{1}^{T},{w}_{2}^{T},\ldots ,{w}_{\left| T\right| }^{T}}\right\rbrack$ be the weight vector for the $\left| T\right|$ instruction types. So, ${w}_{j}^{T}$ is the weight of the ${j}^{\text{th }}$ instruction type $T\left\lbrack  j\right\rbrack$ , and $\mathop{\sum }\limits_{{j = 1}}^{\left| T\right| }{w}_{j}^{T} = 1$ and ${w}^{T} \geq  0,\forall {w}^{T} \in  {W}^{T}$ .

Particles. Similar to the PSO formulation for selecting mutation operators (Sec. III-A), we associate a particle with one thread of the fuzzer. So, there are a total of $\left| N\right|$ particles in the swarm, where $\left| N\right|$ is the number of fuzzer threads. Suppose that each seed consists of a sequence of $\left| O\right|$ instructions. Then, for each of the $\left| N\right|$ threads being run simultaneously, we have a different weight vector ${W}_{i}^{T}$ (which contains $\left| O\right|  \times  \left| T\right|$ entries, one for each of the $\left| T\right|$ instruction types for each of the $\left| O\right|$ instructions in the seed).

Each weight vector ${W}_{i}^{T}$ is assigned as the position ${p}_{i}^{T}$ of the corresponding particle ${i}^{T}$ . Here, the $T$ in the superscript indicates that the particle is for the instruction types, as opposed to the $M$ in the particles for selecting mutation operators in Sec. III-A. Mathematically, ${W}_{i}^{T} = {p}_{i}^{T}$ .

Local Best Position. Following PSO, we assign the local best position, ${l}_{i}^{\text{best,}T}$ , of a particle as the best position (i.e., the probabilities of the instruction types) of that particle so far. Since this PSO is for seed generation, the objective it aims to maximize is the quality of the generated seed. We measure the quality of a particle for selecting instruction types $\left( {i}^{T}\right)$ as the number of iterations the associated particles for selecting mutation operators $\left( {i}^{M}\right)$ survive. For instance, for a given particle, ${i}^{T}$ , if at iteration ${t}_{1}$ and ${t}_{2}$ , its associated particle for selecting mutation operators $\left( {i}^{M}\right)$ survived for 150 and 200 iterations, respectively (as explained in Sec. III-B), then ${p}_{i}^{T}\left( {t}_{2}\right)$ is better than ${p}_{i}^{T}\left( {t}_{1}\right)$ .

Global Best Position. Likewise, following PSO, we assign a single global best position, ${g}^{\text{best,}T}$ , as the best position of all particles in the swarm.

The high-level flow of this PSO for seed generation is very similar to that of the PSO for selecting mutation operators. We start with randomly initialized particles and update their velocities and positions according to Eqs. (1) and (2), respectively. Moreover, learning from the prior pitfall of saturation of particles' performance (Sec. III-B), we also reset the particles for this PSO-based seed generation when they saturate. Since the quality of a particle for selecting instruction types $\left( {i}^{T}\right)$ is measured by the number of iterations its corresponding particle for selecting mutation operators $\left( {i}^{M}\right)$ survives, we reset ${i}_{T}$ when its corresponding ${i}^{M}$ is reset ${\beta }^{T}$ consecutive times. Similar to ${\beta }^{M},{\beta }^{T}$ controls the trade-off between runtime and exploitation of learned knowledge by ${i}^{T}$ s.

## D. Putting it All Together

Fig. 3 shows the PSOFuzz framework, which includes PSO formulation along with the resetting of seeds and particles for selecting mutation operators as well as the PSO formulation for seed generation. The PSO formulation of seed generation guides the seed generator by selecting the instruction type from the instruction set architecture (ISA) pool. Once the instruction type is sampled, the seed generator randomly selects an opcode of that instruction type and then the operands to create the instruction. These instructions are then used to create the seeds, which are added to the input test database. The DUT is simulated with the tests to obtain coverage. This coverage data is used to update the particles' velocities and positions through their local and global bests and the reset monitor. The updated positions are used to generate seeds and mutate tests in the subsequent iterations.

Algorithm 2 details the steps of PSOFuzz. It takes the DUT, a user-defined time limit, ${t}_{\text{limit }}$ , a user-defined target coverage, ${tc}$ , the thresholds for resetting the PSOs for mutation operators and seed generation, ${\beta }^{M}$ and ${\beta }^{T}$ , and the constant, $k$ , used to update the velocities of the particles (Eq. 1)). We fuzz the DUT and return the coverage achieved, cov. First, we initialize all required variables for PSO $\left( {{P}^{M},{V}^{M},{P}^{T},{V}^{T},{L}^{\text{best }, M},{g}^{\text{best }, M},{L}^{\text{best }, T},{g}^{\text{best }, T}}\right)$ . We also initialize the saturation counters $\left( {C{t}^{M}, C{t}^{T}}\right)$ along with the sets indicating particles that need to be reset because of saturation (rst_ ${P}^{M},\operatorname{rst}\_ {P}^{T}$ ). Next, on line 7, we use the positions ${P}^{T}$ to generate the initial set of seed tests. Then, the main fuzzing loop starts on line 8 , which iterates until either the target coverage(tc)is achieved or the runtime exceeds the time limit, ${t}_{\text{limit }}$ . In each iteration of the fuzzing loop, we first simulate the DUT with the generated tests to obtain the function ${\mathcal{F}}^{M}$ (which calculates the fitness of the particles) and the coverage achieved so far, cov (line 9). Next, on line 10, this ${\mathcal{F}}^{M}$ is used to calculate the local and global bests for the particles for mutation operators and to decide which particles need to be reset according to ${\beta }^{M}$ following Algorithm 1. Then, skipping ahead on line 15, the velocities and the positions of the particles for mutation operators are updated, and on lines 16 and 17, the new tests are generated for both particles that are reset and not reset, respectively.

Algorithm 2: PSOFuzz

---

Input: ${DUT},{t}_{\text{limit }},{tc},{\beta }^{M},{\beta }^{T}, k$

	Output: ${cov}$ : total coverage achieved

	// initialization

	$t \leftarrow  0,\operatorname{cov} \leftarrow  0$ , tests $\leftarrow  \phi$

Initialize ${P}^{M},{V}^{M},{P}^{T},{V}^{T}$

. Initialize ${L}^{\text{best }, M}$ to ${P}^{M}$ and ${g}^{\text{best }, M}$ to ${p}_{0}^{M}$

4 Initialize ${L}^{\text{best }, T}$ to ${P}^{T}$ and ${g}^{\text{best }, T}$ to ${p}_{0}^{T}$

$C{t}^{M} \leftarrow  0, C{t}^{T} \leftarrow  0$

rst_ ${P}^{M} \leftarrow  \phi ,$ rst_ ${P}^{T} \leftarrow  \phi$

tests $\leftarrow$ GenSeed $\left( {{P}^{T}\text{, tests, rst_}{P}^{M}}\right)$

while $\left( {{cov} < {tc}}\right)$ and $\left( {t < {t}_{\text{limit }}}\right)$ do // main loop

		${\mathcal{F}}^{M},\operatorname{cov} \leftarrow$ Simulate(DUT, tests)

		${L}^{\text{best,}},{g}^{\text{best, M, rst_}}{P}^{M}, C{t}^{M} \leftarrow$

		$\operatorname{RstMon}\left( {{P}^{M},{L}^{\text{best }, M},{\beta }^{M}, C{t}^{M},{\mathcal{F}}^{M}}\right)$

		if rst_ ${P}^{M} \neq  \phi$ then // update PSO seed

			${\mathcal{F}}^{T} \leftarrow$ CalcFitness $\left( {{rst}\_ {P}^{M},{P}^{T}}\right)$

			${L}^{\text{best }, T},{g}^{\text{best }, T},\operatorname{rst}\_ {P}^{T}, C{t}^{T} \leftarrow$

				$\operatorname{RstMon}\left( {{P}^{T},{L}^{\text{best }, T},{\beta }^{T}, C{t}^{T},{\mathcal{F}}^{T}}\right)$

			${P}^{T},{V}^{T} \leftarrow$

				${UpdatePV}\left( {{P}^{T},{V}^{T},{L}^{\text{best }, T},{g}^{\text{best }, T},\operatorname{rst}\_ {P}^{T}, k}\right)$

		${P}^{M},{V}^{M} \leftarrow$

		${UpdatePV}\left( {{P}^{M},{V}^{M},{L}^{\text{best }, M},{g}^{\text{best }, M},\operatorname{rst}\_ {P}^{M}, k}\right)$

		// gen. seeds for reset particles

		tests $\leftarrow$ GenSeed $\left( {{P}^{T}\text{, tests, rst_}{P}^{M}}\right)$

		// mutate tests for other particles

		tests $\leftarrow  \operatorname{Mut}\left( {{P}^{M},\text{ tests },\left\{  {{i}^{M} \mid  {i}^{M} \notin  \text{ rst }\_ {P}^{M}}\right\}  }\right)$

return ${cov}$

---

Additionally, on line 11, we check if any of the particles for the mutation operators needs to be reset, i.e., ${rst}\_ {P}^{M} \neq  \phi$ , and calculate the number of iterations those particles survived (i.e., fitness) and return ${\mathcal{F}}^{T}$ on line 12 . This ${\mathcal{F}}^{T}$ is used to calculate the local and global bests for the particles for seed generation and to decide which ones to reset according to ${\beta }^{T}$ on line 13. Finally, on line 14, the velocities and the positions of the particles for seed generation are updated.

## IV. EVALUATION

In this section, we evaluate the speed of PSOFuzz in detecting vulnerabilities and achieving coverage on three popular open-sourced RISC-V [21] processors.

## A. Evaluation Setup

We use the state-of-the-art simulation-based processor fuzzer, TheHuzz [11] as a baseline for all evaluations 3 To ensure that the enhancements in results exclusively stem from PSO adaptation, we integrate PSO schemes into TheHuzz's mutation engine and seed generator. In addition to reporting the results for our final formulation, PSOFuzz, which is also denoted as PSO+Reset+Seed, we also report the results for (i) the preliminary formulation, PSO, which includes the PSO-based mutation operator selection, but neither the resetting solution (Solution 1), nor the PSO-based seed generation technique (Solution 2), and (ii) the intermediate formulation, PSO+Reset, which includes the PSO-based mutation operator selection with reset, but not PSO-based seed generation.

Following [11], we set the number of simultaneous threads and hence the number of particles, $\left| N\right|$ , as 10 and the number of instructions in each program, $\left| O\right|$ , as 20 . To have a balanced impact of exploration (due to velocity) and exploitation (due to local and global bests), we set $k$ (Eq. (1)) as 0.5 . Based on empirical observations, we set ${\beta }^{M} = {\beta }^{T} = 3$ , as a larger $\beta$ will cause runtime overhead, while a smaller $\beta$ will make PSOFuzz generate test cases similar to random exploration, as discussed in Sec. III-B

Since most commercial processors are close-sourced, we evaluate PSOFuzz using three widely-used open-sourced processors from RISC-V [21] ISA: CVA6 [20], BOOM [23], and Rocket Core [22]. Existing hardware processor fuzzers use similar open-sourced processors for their evaluation [9], [11], [13]. CVA6 supports out-of-order execution (OoO) and a custom single instruction-multiple data (SIMD) floating point unit. Rocket Core is an in-order processor. BOOM is a superscalar processor built using the components of Rocket Core. Moreover, all of these processors are capable of booting Linux operating system while CVA6 is commercially used to build processors for high-security applications [24]. Therefore, these processors with different complexity form a diverse benchmark set to evaluate PSOFuzz.

We use Synopsys ${VCS}$ [25] as the simulation tool and Chipyard [26] as the system-on-chip (SoC) simulation environment. To ensure a fair comparison, we ran experiments on each benchmark for 24 hours and repeated them thrice. We demonstrate our results using the branch coverage metric, which is highly related to vulnerability detection [27]. We use a Linux-based CPU running at ${2.6}\mathrm{{GHz}}$ , with 64 threads and 512GB of RAM for our experiments.

---

${}^{3}$ Following [11], we implemented TheHuzz in Python.

---

TABLE I: Vulnerability detection speedup compared to TheHuzz [11]. N.D. denotes "not detected."

<table><tr><td rowspan="2">Processor</td><td rowspan="2">Vulnerability</td><td rowspan="2">CWE</td><td>TheHuzz</td><td colspan="2">PSO</td><td colspan="2">PSO+Reset</td><td colspan="2">PSO+Reset+Seed (PSOFuzz)</td></tr><tr><td>#Tests</td><td>#Tests</td><td>Speedup</td><td>#Tests</td><td>Speedup</td><td>#Tests</td><td>Speedup</td></tr><tr><td rowspan="7">CVA6 [20]</td><td>V1: Access invalid addresses without throwing exceptions.</td><td>CWE-1252</td><td>${3.52} \times  {10}^{2}$</td><td>${1.38} \times  {10}^{2}$</td><td>${2.55} \times$</td><td>${5.71} \times  {10}^{2}$</td><td>${0.62} \times$</td><td>${2.32} \times  {10}^{2}$</td><td>${1.52} \times$</td></tr><tr><td>V2: Decode multiplication instructions incorrectly.</td><td>CWE-440</td><td>${2.71} \times  {10}^{3}$</td><td>${5.09} \times  {10}^{3}$</td><td>${0.53} \times$</td><td>${1.80} \times  {10}^{3}$</td><td>${1.50} \times$</td><td>${4.51} \times  {10}^{2}$</td><td>${6.01} \times$</td></tr><tr><td>V3: Access unimplemented CSRs and return X-values.</td><td>CWE-1281</td><td>${4.75} \times  {10}^{3}$</td><td>N.D.</td><td>N.D.</td><td>${8.56} \times  {10}^{2}$</td><td>${5.54} \times$</td><td>${5.70} \times  {10}^{3}$</td><td>${0.83} \times$</td></tr><tr><td>V4: Cache coherency violation undetected.</td><td>CWE-1202</td><td>${1.83} \times  {10}^{2}$</td><td>${1.23} \times  {10}^{4}$</td><td>${0.01} \times$</td><td>${1.70} \times  {10}^{1}$</td><td>${10.76} \times$</td><td>${1.20} \times  {10}^{1}$</td><td>${15.25} \times$</td></tr><tr><td>V5: Decode the FENCE. I instruction incorrectly.</td><td>CWE-440</td><td>${6.80} \times  {10}^{1}$</td><td>${4.50} \times  {10}^{2}$</td><td>${0.15} \times$</td><td>${1.80} \times  {10}^{1}$</td><td>${3.78} \times$</td><td>${1.35} \times  {10}^{2}$</td><td>${0.50} \times$</td></tr><tr><td>V6: Incorrect exception type in instruction queue.</td><td>CWE-1202</td><td>${5.55} \times  {10}^{3}$</td><td>${1.94} \times  {10}^{4}$</td><td>${0.29} \times$</td><td>${5.39} \times  {10}^{2}$</td><td>${10.30} \times$</td><td>${1.26} \times  {10}^{3}$</td><td>${4.40} \times$</td></tr><tr><td>V7: Some illegal instructions without throwing exceptions.</td><td>CWE-1242</td><td>${9.30} \times  {10}^{1}$</td><td>N.D.</td><td>N.D.</td><td>${2.54} \times  {10}^{2}$</td><td>${0.37} \times$</td><td>${7.40} \times  {10}^{1}$</td><td>${1.26} \times$</td></tr><tr><td>Rocket Core221</td><td>V8: EBREAK does not increase instruction count.</td><td>CWE-1201</td><td>${1.67} \times  {10}^{3}$</td><td>${6.67} \times  {10}^{3}$</td><td>${0.25} \times$</td><td>${1.07} \times  {10}^{3}$</td><td>${1.56} \times$</td><td>${4.10} \times  {10}^{2}$</td><td>${4.07} \times$</td></tr></table>

## B. Vulnerability Detection

The vulnerability detection strategy used by our techniques (PSO, PSO+Reset, PSO+Reset+Seed) compares the processor's outputs with that of a GRM. Many existing fuzzers use a similar detection strategy [9], [11]-[13]. The strategy compares the architectural states of the processor and the GRM for each executed instruction. The architectural states include the instructions committed; any exceptions triggered; the general purpose registers (GPRs) modified; the CSRs modified; the privilege level of the processor; and the memory address and data accesses. A mismatch in architectural states indicates a potential vulnerability in the processor. Similar to [9], [11]-[13], we use Spike [28], the RISC-V ISA emulator, as the GRM.

Comparison with TheHuzz. Table 1 shows the number of tests and the speedup of our techniques compared to TheHuzz. PSOFuzz detects all the vulnerabilities detected by TheHuzz up to ${15.25} \times$ faster. The results also show the necessity of using the reset strategy for PSO. Naïve PSO is slower than TheHuzz in detecting vulnerabilities because its particles are likely exploring the design spaces that have already been explored, as mentioned in Sec. III-B. PSO+Reset performs better than PSO due to resetting the particles, allowing it to dynamically select the optimal mutation operators. It also outperforms PSO+Reset+Seed (PSOFuzz) in detecting some vulnerabilities, such as V3, because PSOFuzz initially spends time exploring the optimal probabilities for seed generation. However, PSOFuzz achieves the highest speedup compared to TheHuzz and other PSO techniques for most vulnerabilities because it uses the optimal probabilities for both seed generation and instruction mutation, efficiently exploring different regions of the design space.

## C. Coverage Achievement

We now compare the capability of our techniques in achieving coverage against TheHuzz, as coverage achieved is a proxy metric to the extent of design verified and determines its readiness to tape out [29]. Across the three processors, PSO optimizations achieve up to ${6.39} \times$ speedup compared to TheHuzz while obtaining up to 2% more coverage after fuzzing for 24 hours with more than ${20}\mathrm{\;K}$ tests, as shown in Table II. Fig. 4 shows the mean and standard deviation of coverage achieved with respect to the number of input tests.

PSOFuzz outperforms PSO+Reset on Rocket Core and BOOM because the PSO-based seed generation dynamically adjusts the optimal set of instructions required to cover new points. While PSOFuzz is slower than PSO+Reset on CVA6, it still outperforms TheHuzz and PSO techniques.

Among the three processors, PSOFuzz achieves the fastest speedup $\left( {{2.22} \times  }\right)$ and highest coverage increment $\left( {{1.00}\% }\right)$ on CVA6, which is the complex processor with OoO and SMID features. This is because the PSO optimizations aid in covering the hard-to-reach coverage points in the custom floating point unit of CVA6, which are difficult to cover with static mutation operator scheduling schemes. Even on the remaining processors, BOOM and Rocket Core, PSOFuzz achieves faster coverage (up to ${1.72} \times$ ) compared to TheHuzz and covers more points than TheHuzz. The speedup on these processors is less than that of CVA6 because Rocket Core is a simple processor compared to CVA6 while BOOM is built by duplicating most of the components of Rocket Core. Hence, TheHuzz covers more than 87% of their points, resulting in less scope for improvement for PSOFuzz.

In summary, PSOFuzz detects vulnerabilities and achieves coverage faster than TheHuzz on all three processors, demonstrating the necessity of dynamic scheduling of mutation operators and seed generation.

## V. RELATED WORK

This section describes the existing hardware fuzzers, their limitations, and how PSOFuzz addresses those limitations.

RFUZZ [6] uses mux-toggle coverage to capture the activity in the select signals of MUXes. It schedules mutation operators statically, similar to AFL fuzzer [17]. Also, mux-toggle coverage is not scalable to large designs such as BOOM [9]. HyperFuzzing [7] is an SoC fuzzer that defines security properties and uses AFL fuzzer's static mutation operator scheduling and random seed generation to detect vulnerabilities. DirectFuzz [8] modifies RFUZZ to perform directed fuzzing on specific modules of the target hardware. However, it uses similar static schemes as ${RFUZZ}$ .

TABLE II: Coverage improvement compared to TheHuzz [11].

<table><tr><td rowspan="2">Core</td><td>TheHuzz</td><td colspan="3">PSO</td><td colspan="3">PSO+Reset</td><td colspan="3">PSO+Reset+Seed (PSOFuzz)</td></tr><tr><td>Total</td><td>Total</td><td>Increment</td><td>Speedup</td><td>Total</td><td>Increment</td><td>Speedup</td><td>Total</td><td>Increment</td><td>Speedup</td></tr><tr><td>CVA6 [20]</td><td>6214</td><td>6074</td><td>-2.25%</td><td>${0.25} \times$</td><td>6334</td><td>2.00%</td><td>${6.39} \times$</td><td>6277</td><td>1.00%</td><td>${2.22} \times$</td></tr><tr><td>BOOM [23</td><td>23086</td><td>23052</td><td>-0.15%</td><td>${0.73} \times$</td><td>23088</td><td>0.01%</td><td>${1.11} \times$</td><td>23092</td><td>0.01%</td><td>${1.25} \times$</td></tr><tr><td>Rocket Core22</td><td>11947</td><td>11846</td><td>-0.85%</td><td>${0.10} \times$</td><td>11864</td><td>-0.06%</td><td>${0.23} \times$</td><td>11958</td><td>0.09%</td><td>${1.72} \times$</td></tr></table>

![0196f2bf-9a53-7333-89ae-c9779e21bd0b_7_134_371_1526_299_0.jpg](images/0196f2bf-9a53-7333-89ae-c9779e21bd0b_7_134_371_1526_299_0.jpg)

Fig. 4: Coverage achieved compared to TheHuzz [11] for (a) CVA6 [20], (b) BOOM [23], and (c) Rocket Core [22].

Processor fuzzer DIFUZZRTL [9] uses activity in the control registers as coverage to address the scalability issues of ${RFUZZ}$ . It uses grammar-based mutation operators inspired by AFL. But, it generates seeds randomly and uses static mutation operator scheduling schemes. BugsBunny [12] fuzzes specific target signals in the hardware using DIFUZZRTL as the base fuzzer. Hence, its mutation operator scheduling and seed generation schemes are static. Trippel et al. [10] convert hardware to a software model and fuzz using AFL fuzzer with static mutation scheduling. TheHuzz [11] is another processor fuzzer that uses different types of code coverage metrics to capture activity in the combinational and sequential logic of the hardware. It also uses a profiler to optimize the mutation scheduling. However, this optimization is static for a given processor and does not change with the coverage achieved.

A recent hardware fuzzer HyPFuzz [13] attempts to address the slower coverage speeds of hardware fuzzers by combing formal tools with fuzzing. This optimization to fuzzing is orthogonal to PSOFuzz as HyPFuzz also uses static mutation operator scheduling and seed generation schemes which can be optimized through our PSO techniques.

In summary, all of the existing hardware fuzzers use static mutation operator scheduling and seed generation schemes and cannot dynamically adjust their strategies for the target hardware and coverage achieved. In contrast, we propose PSOFuzz, a PSO-guided fuzzer, to demonstrate how hardware fuzzers can be equipped with PSO optimizations to improve their vulnerability detection and coverage achievement speeds.

## VI. DISCUSSION

In this section, we discuss possible extensions to improve PSOFuzz and hardware fuzzing in general.

Dynamically Identifying Optimal Solutions. Though the reset strategy works well in practice, it is a heuristic that modifies the PSO algorithm. In the future, we plan to model and analyze the impact of resetting threshold $\left( \beta \right)$ on the capability of vulnerability detection and design space exploration of PSOFuzz. Moreover, we will also evaluate the performance of other meta-heuristic algorithms, such as simulated annealing [30] and differential evolution [31], and identify the ideal one for hardware fuzzing.

Leveraging Prior Knowledge for Initialization. One key way to improve its performance is by initializing with a good starting point [32]. PSOFuzz initializes particles' positions randomly. However, we can use the static schemes of existing fuzzers to initialize the particles. For example, the static probabilities from the profiling state of TheHuzz can be used as the initial position of particles to select the mutation operators.

## VII. CONCLUSION

Processor fuzzers, which rely on the mutation of input tests, have shown promise in detecting vulnerabilities. However, they use static scheduling strategies for mutation and seed generation, which is not ideal. To overcome this limitation, we develop a novel PSO-based framework, PSOFuzz, that can be integrated with any fuzzer to find optimal scheduling strategies. To ensure better vulnerability detection performance, we augment naïve PSO with (i) a reset strategy that allows it to find dynamic selection probabilities for mutation operators and (ii) a PSO-based seed generation algorithm. Experimental results with the state-of-the-art simulation-based hardware fuzzer demonstrate that PSOFuzz can speed up vulnerability detection by up to ${15.25} \times$ and coverage achievement by up to ${2.22} \times$ . In conclusion, ${PSOFuzz}$ addresses a critical limitation of existing processor fuzzers and significantly improves the effectiveness and efficiency of processor verification.

## VIII. ACKNOWLEDGEMENT

Our research work was partially funded by the US Office of Naval Research (ONR Award #N00014-22-1-2279), by Intel's Scalable Assurance Program, and by the European Union (ERC, HYDRANOS, 101055025). Any opinions, findings, conclusions, or recommendations expressed herein are those of the authors and do not necessarily reflect those of the US Government, the European Union, or the European Research Council.

## REFERENCES

[1] S. Fine and A. Ziv, "Coverage Directed Test Generation for Functional Verification using Bayesian Networks," Design Automation Conference, 2003.

[2] J. Rajendran, V. Vedula, and R. Karri, "Detecting Malicious Modifications of Data in Third-party Intellectual Property Cores," 2015.

[3] G. Dessouky et al., "HardFails: Insights into Software-Exploitable Hardware Bugs," 28th USENIX Security Symposium, 2019.

[4] A.-R. Sadeghi, J. Rajendran, and R. Kande, "Organizing The World's Largest Hardware Security Competition: Challenges, Opportunities, and Lessons Learned," 2021.

[5] C. Chen et al., "Trusting the Trust Anchor: Towards Detecting Cross-Layer Vulnerabilities with Hardware Fuzzing," 2022.

[6] K. Laeufer et al., "RFUZZ: Coverage-Directed Fuzz Testing of RTL on FPGAs," IEEE/ACM International Conference on Computer-Aided Design, pp. 1-8, 2018.

[7] S. K. Muduli, G. Takhar, and P. Subramanyan, "HyperFuzzing for SoC Security Validation," ACM/IEEE International Conference on Computer-Aided Design, pp. 1-9, 2020.

[8] S. Canakci et al., "Directfuzz: Automated Test Generation for RTL Designs using Directed Graybox Fuzzing," ACM/IEEE Design Automation Conference, pp. 529-534, 2021.

[9] J. Hur et al., "DIFUZZRTL: Differential Fuzz Testing to Find CPU Bugs," IEEE Symposium on Security and Privacy, pp. 1286-1303, 2021.

[10] T. Trippel et al., "Fuzzing Hardware Like Software," USENIX Security Symposium, 2022.

[11] R. Kande et al., "TheHuzz: Instruction Fuzzing of Processors Using Golden-Reference Models for Finding Software-Exploitable Vulnerabilities," USENIX Security Symposium, pp. 3219-3236, 2022.

[12] H. Ragab et al., "BugsBunny: Hopping to RTL Targets with a Directed Hardware-Design Fuzzer," SILM, Jun. 2022. [Online]. Available: https://download.vusec.net/papers/bugsbunny_silm22.pdf

[13] C. Chen et al., "HyPFuzz: Formal-Assisted Processor Fuzzing," arXiv preprint arXiv:2304.02485, 2023.

[14] IntelLabs, "Pre-Silicon Hardware Fuzzing Toolkit," https://github.com/ IntelLabs/PreSiFuzz 2023, last accessed on 05/21/2023.

[15] J. Kennedy and R. Eberhart, "Particle swarm optimization," IEEE international conference on neural networks, 1995.

[16] C. Lyu et al., "MOPT: Optimized Mutation Scheduling for Fuzzers." USENIX Security Symposium, pp. 1949-1966, 2019.

[17] lcamtuf, "American Fuzzy Lop (AFL) Fuzzer." 2014, Last accessed on 05/21/2023. [Online]. Available: http://lcamtuf.coredump.cx/afl/ technical details.txt

[18] Z.-H. Zhan et al., "Adaptive Particle Swarm Optimization," IEEE Transactions on Systems, Man, and Cybernetics, Part B (Cybernetics), 2009.

[19] Y. Shi and R. Eberhart, "A Modified Particle Swarm Optimizer," IEEE international conference on evolutionary computation proceedings., 1998.

[20] F. Zaruba and L. Benini, "The Cost of Application-Class Processing: Energy and Performance Analysis of a Linux-Ready 1.7-GHz 64-Bit RISC-V Core in 22-nm FDSOI Technology," IEEE Transactions on Very Large Scale Integration Systems, vol. 27, no. 11, pp. 2629-2640, Nov 2019.

[21] RISC-V, "RISC-V Webpage," https://riscv.org/ 2021, Last accessed on 05/21/2023.

[22] K. Asanović et al., "The Rocket Chip Generator," no. UCB/EECS- 2016-17, Apr 2016. [Online]. Available: http://www2.eecs.berkeley.edu/ Pubs/TechRpts/2016/EECS-2016-17.html

[23] J. Zhao et al., "SonicBOOM: The 3rd Generation Berkeley Out-of-Order Machine," Fourth Workshop on Computer Architecture Research with RISC-V, May 2020.

[24] H. C. GmbH, "MiG-V - Made in Germany RISC-V," 2022, Last accessed on 05/21/2023. [Online]. Available: https://hensoldt-cyber.com/mig-v/

[25] "VCS," https://www.synopsys.com/verification/simulation/vcs.html 2023, Last accessed on 05/21/2023.

[26] A. Amid et al., "Chipyard: Integrated Design, Simulation, and Implementation Framework for Custom SoCs," IEEE Micro, vol. 40, no. 4, pp. 10-21, 2020.

[27] A. Mockus, N. Nagappan, and T. T. Dinh-Trong, "Test Coverage and Post-Verification Defects: A Multiple Case Study," pp. 291-301, 2009.

[28] RISC-V, "SPIKE Source Code," https://github.com/riscv/riscv-isa-sim 2023, last accessed on 05/21/2023.

[29] Synopsys, "Accelerating Verification Shift Left with Intelligent Coverage Optimization," https://www.synopsys.com/cgi-bin/verification/dsdla/ pdfr1.cgi?file=ico-wp.pdf 2022, Last accessed on 05/21/2023.

[30] D. Bertsimas and J. Tsitsiklis, "Simulated Annealing," Statistical science, 1993.

[31] K. V. Price, "Differential Evolution," Handbook of Optimization: From Classical to Modern Approach, pp. 187-214, 2013.

[32] D. Simpson et al., "Penalising model component complexity: A principled, practical approach to constructing priors," 2017.