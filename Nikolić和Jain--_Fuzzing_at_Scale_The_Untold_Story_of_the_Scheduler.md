# Fuzzing at Scale: The Untold Story of the Scheduler

IVICA NIKOLIC, National University of Singapore, Singapore

RACCHIT JAIN*, National University of Singapore, Singapore

How to search for bugs in 1,000 programs using a pre-existing fuzzer and a standard PC? We consider this problem and show that a well-designed strategy that determines which programs to fuzz and for how long can greatly impact the number of bugs found across the programs. In fact, the impact of employing an effective strategy is comparable to that of utilizing a state-of-the-art fuzzer. The considered problem is referred to as fuzzing at scale, and the strategy as scheduler. We show that besides a naive scheduler, that allocates equal fuzz time to all programs, we can consider dynamic schedulers that adjust time allocation based on the ongoing fuzzing progress of individual programs. Such schedulers are superior because they lead both to higher number of total found bugs and to higher number of found bugs for most programs. The performance gap between naive and dynamic schedulers can be as wide (or even wider) as the gap between two fuzzers. Our findings thus suggest that the problem of advancing schedulers is fundamental for fuzzing at scale. We develop several schedulers and leverage the most sophisticated one to fuzz simultaneously our newly compiled benchmark of around 5,000 Ubuntu programs, and detect 4,908 bugs.

## 1 INTRODUCTION

Fuzzing is an effective software testing technique used for uncovering vulnerabilities. As software systems become increasingly complex and pervasive, ensuring their security and reliability is important. Despite the use of various testing methodologies, software bugs and vulnerabilities continue to emerge, posing significant risks to both organizations and individuals relying on the software. Consequently, the demand for more efficient and effective testing techniques has grown, with fuzzing emerging as a leading approach to address these concerns. Fuzzing can identify potential security flaws and weaknesses in software applications by providing unexpected, invalid, or random inputs (called testcases) to the target software, potentially revealing issues that might have remained hidden under normal inputs.

The primary focus of research in fuzzing has been on the development and enhancement of fuzzers, as evidenced by numerous designs and studies [Aschermann et al. 2019; Blazytko et al. 2019; Chen et al. 2020b; Chen and Chen 2018; Chen et al. 2019b, 2020a, 2019a; Cho et al. 2019; Fioraldi et al. 2020; Gan et al. 2020, 2018; Geretto et al. 2022; Koike et al. 2022; Lee et al. 2021; Lemieux and Sen 2018; Liang et al. 2019, 2022; Lyu et al. 2019, 2022; Nagy et al. 2021a, b; Nguyen et al. 2020; Nikolić et al. 2021; Österlund et al. 2020; Rawat et al. 2017; Shah et al. 2022; She et al. 2022; Swiecki et al. 2021; Wang et al. [n. d.]; Yu et al. 2022; Yue et al. 2020; Yun et al. 2018; Zalewski 2017; Zhou et al. 2022; Zhu and Böhme 2021; Zong et al. 2020]. This emphasis is well-aligned with the principal use case of fuzzing, which typically involves testing a single program at a time. For example, a software developer writes a program and then employs a fuzzer to identify potential bugs in its code. The efficiency of a fuzzer, often assessed using metrics such as code coverage (which measures the proportion of program code executed during fuzzing), directly impacts the likelihood of detecting potential bugs in the target program. Consequently, there has been a sustained effort to design increasingly efficient fuzzers. Each new fuzzer or fuzzing technique demonstrates certain advantages over its predecessors. However, despite the constant progress, the disparity between the most effective fuzzers and their less efficient counterparts remains relatively small. For instance, FuzzBench, a platform developed by Google for comparing fuzzers [Metzman et al. 2021], reveals an average performance gap of only ${20}\%  - {30}\%$ between the best and worst-performing fuzzers across multiple programs.

---

*Work done while interning at National University of Singapore.

---

Our research objective deviates from the conventional emphasis on fuzzers. Instead, we pursue an orthogonal approach, focusing on developing techniques to utilize existing fuzzers for simultaneously testing multiple programs. We term this framework fuzzing at scale. More specifically, given a large set of target programs and a fuzzer, our aim is to develop a scheduler that will control the simultaneous fuzzing of all these programs across multiple CPU cores. The goal of the scheduler is to facilitate efficient fuzzing in an aggregated manner, that is, collectively across all the considered programs. For example, one objective could be to maximize the cumulative code coverage, which refers to the total sum of coverage achieved by the fuzzer for each program. To the best of our knowledge, this fuzzing at scale paradigm with such an explicit objective has not been previously considered. In contrast, platforms for distributed fuzzing of multiple programs, such as Google's OSS-Fuzz [Serebryany 2017], allocate computational resources (e.g., a designated fuzz time budget) equally among the fuzzed programs. Furthermore, this trivial approach, known as the baseline scheduler, has been applied to systems where the computational resources per fuzzed program are fairly large, for instance, when one or several CPUs can be allocated to fuzz test a single program. However, the inverse scenario, in which the number of tested programs surpasses the number of available CPU cores, has not been investigated. Nonetheless, use cases of this type are not unusual. For instance, an organization like a defense company or government security agency might prefer first to fuzz all required Linux applications (which could number in the hundreds or even thousands) to address apparent bugs before utilizing this open-source software. A security researcher compiling a database of bugs across multitude of programs would benefit from fuzzing at scale by testing large number of programs at once. In fact, any fuzzing scenario where the number of target programs exceeds that of available CPUs is indeed fuzzing at scale - for instance, fuzz testing 10 programs on 2 CPUs.

In this paper, we consider the fuzzing at scale framework and demonstrate the significance of scheduler selection. Our primary contribution reveals that allocating equal fuzz time to all programs, as the baseline scheduler does, is suboptimal. More advanced schedulers provides up to a 30% advantage over the baseline, emphasizing that in fuzzing at scale, the choice of a scheduler is as critical as the choice of a fuzzer. Furthermore, the combination of an outstanding scheduler with an average fuzzer may match or even surpass the performance of a baseline scheduler and an outstanding fuzzer. This observation highlights the need for research into effective schedulers. As an initial step in this direction, we develop several new schedulers, each grounded in the fundamental and widely-used fuzzing hypothesis that a program's past code coverage correlates to its future coverage. Consequently, these schedulers are designed to prioritize fuzzing programs that have previously resulted in high coverage, scheduling them for fuzzing more frequently. Our final design, referred to as BoIAN, outperforms all other schedulers according to metrics such as total code coverage achieved across all programs and the percentage of programs for which the highest coverage has been attained.

Our research presents an initial exploration into the domain of fuzzing at scale. In order to evaluate and compare the efficiency of algorithms in this field, a comprehensive benchmark set is necessary. We assemble an extensive fuzzing benchmark, referred to as UBUNTUBENCH, which consists of 5,467 programs derived from a collection of approximately 75,799 Ubuntu packages. This benchmark not only serves as the basis for our investigation, but also offers potential benefits for future research efforts in the area of fuzzing at scale. We employ this as well as the smaller OSS-Fuzz [Google 2023] benchmark to assess schedulers and to demonstrate the applicability of our fuzzing at scale framework. More specifically, we utilize the newly developed BOIAN to schedule simultaneous fuzzing of all programs in UbuNTUBENCH using state-of-the-art fuzzers AFL++ [Fioraldi et al. 2020] and Honggfuzz[Swiecki et al. 2021]. As a result, we identify 4,908 bugs among 675 programs within a three-day fuzzing period with each fuzzer on a 30-core machine.

## 2 PROBLEM

Fuzzing at scale consists of distributing fuzzing effort across multiple cores and machines. It can be categorized into two distinct types: the first aims to uncover deep bugs, while the second aims to fuzz a wide range of programs. For simplicity, we can refer to first type as deep, while to the second as wide. While prior fuzzing at scale work has been focused on the deep type, our objective in this study is to concentrate on the wide type.

Deep fuzzing at scale uses potentially large computational resources to fuzz test one program on bugs. The aim of such type of thorough fuzzing is to discover potential vulnerabilities hidden deeply in the program code, that usually require substantial time for detection. The primary use case of this rigorous fuzzing approach is to ensure the security of a small set of critical software components. These components often include the most popular binaries and libraries. The goal is to keep them free of vulnerabilities that are easy or moderately challenging to discover.

Wide fuzzing at scale uses the available resources to fuzz simultaneously a larger number of programs. In contrast to deep, the aim of wide fuzzing is to cover a wide array of target programs, ensuring that none possess easily discoverable vulnerabilities. (But given resources, it can also uncover deeply hidden vulnerabilities, because it uses the same fuzzer as deep fuzzing.) When faced with a large set of programs, such as a Linux distribution containing numerous binaries and libraries of varying importance, it is challenging to decide which of the following two strategies is more vital: deep fuzzing of a small set of programs or wide fuzzing of all programs. The former strategy effectively hardens a small yet crucial set of programs but leaves even trivial bugs in the remaining programs. In contrast, the latter strategy may identify trivial bugs across all programs but may leave deeper bugs undiscovered (assuming time constrained fuzzing, but longer wide fuzzing campaigns will detect them). Given these considerations, it is reasonable to assume that both types of fuzzing at scale are relevant. However, in general the unit cost of bug detection through wide fuzzing at scale is significantly lower than that of deep fuzzing because it is easier to discover some bug in large set of programs than a deep bug in small set. This is attributed to the diminishing returns of fuzzing; as a program undergoes more fuzz testing, fewer bugs are detected, and the cost of discovering each bug increases substantially. So, due to the larger number of targets, the wide approach will produce bugs faster than the deep. Hence, the wide type is the more potent source of vulnerabilities among the two fuzzing at scale types. Interestingly, only deep fuzzing at scale has been explored to date. Our objective is to shift the focus to the second and unexplored type, i.e., to wide fuzzing at scale.

The effectiveness of deep fuzzing at scale relies on the quality of the utilized fuzzer because this approach fuzz tests individual target programs one after another for an extended duration on a multicore system. Since deep fuzzing operates on a single target at a time, improving its efficiency-in terms of a metric like code coverage-requires employing a superior fuzzer. Conversely, wide fuzzing operates on multiple targets concurrently, meaning that the choice of fuzzer may not be the sole determinant of its efficiency. An additional factor to consider is the distribution of computational resources, particularly CPU time, among the fuzzed programs. A basic method involves allocating equal amounts of time to each program in a round-robin manner. For instance, fuzzing 40 programs on a 4-core machine for 10 hours would allocate one hour to each program. However, equal allocation of time may not be the most efficient strategy, as it does not account for the inherent differences between programs that can result in uneven progress in code coverage across the target set. The diversity of programs can directly impact the rate at which a fuzzer explores and discovers new coverage. Thus, allocating an equal amount of time to all fuzzed programs, may lead to inefficient fuzzing, e.g., wasting resources on programs that have already been well examined, instead of on those that have shown consistent increase of code coverage. Inefficiency in fuzzing poses a particularly severe problem due to its inherent time-consuming nature, i.e. often fuzzing runs for days or even weeks, thereby amplifying the significance of resource wastage.

To address this issue, a more adaptive and dynamic approach to time allocation should be considered, wherein fuzzing time is apportioned based on some feedback obtained from the previous fuzzing runs of target programs. More precisely, by monitoring the progress of code coverage and adjusting the fuzzing time accordingly, one can ensure a more efficient and comprehensive wide fuzzing at scale.

Concrete applications. To further motivate our focus on wide fuzzing at scale, let us consider several concrete applications:

- Linux distribution hardening: Consider a new Linux distribution release, encompassing thousands of software packages. A crucial task for the maintainers is to quickly identify and fix vulnerabilities across all these packages. Wide fuzzing at scale allows security researchers to perform a preliminary security assessment by running a short fuzzing campaign. This can help identify easily discoverable vulnerabilities early on, allowing for timely patching before the official release. In contrast, deep fuzzing at scale on a selected subset of critical packages might miss vulnerabilities in less critical ones, leaving the overall system exposed.

- Fuzzer comparison: When evaluating the performance of different fuzzers, wide fuzzing at scale may provide a more robust and diverse test environment than deep fuzzing on a limited set of programs. By fuzzing a large number of varied programs, researchers can obtain a more accurate assessment of a fuzzer's strengths and weaknesses across different program types and complexities. This approach allows for better differentiation between fuzzers and more reliable performance metrics, as it accounts for the fuzzer's adaptability to diverse codebases. Testing a wider range of programs helps prevent potential overfitting, where a fuzzer might perform exceptionally well on a small set of specific programs but fail to generalize to other codebases. By evaluating fuzzers across many more programs, researchers can better measure their true performance and adaptability, reducing the risk of drawing conclusions based on limited sample set.

- Vulnerability database compilation: Using the best available fuzzers, and applied to a large number of programs, wide fuzzing can efficiently discover bugs across diverse software. This approach is particularly effective for building vulnerability databases because its primary objective is to maximize the number of bugs found. As a result, wide fuzzing can quickly build a broad, representative collection of vulnerabilities, creating a valuable resource for security research and software development. The resulting database can offer a comprehensive view of vulnerabilities across different software.

- Fuzzing different program arguments: Wide fuzzing can be employed to fuzz test a program with different command line arguments. In this scenario, each fuzzer within the framework will be assigned to a specific set of fixed command line arguments. Wide fuzzing will utilize coverage feedback to guide the resource allocation across the different fuzzers - it will favour and fuzz more the arguments that show progress in code coverage. In contrast, a simple naive approach to fuzzing different command line arguments lacks this intelligent resource allocation mechanism. Without coverage feedback, it is not clear how to optimally distribute fuzzing efforts among various sets of fixed arguments, potentially leading to less efficient exploration of the program's behavior under different input arguments.

- Security audit of large software repositories: Consider an organization like a bank that utilizes hundreds of open-source libraries and tools within their software ecosystem. Before implementation, it is crucial to conduct a security audit to identify and mitigate potential vulnerabilities in the libraries. Wide fuzzing at scale allows the organization to efficiently uncover superficial bugs across a wide range of programs within a shorter time frame. This approach ensures that all programs meet a minimum security baseline before deployment.

- Building fuzzing benchmarks and datasets: Creating comprehensive fuzzing benchmarks requires collecting diverse programs and datasets, often involving hundreds or thousands of applications. Wide fuzzing at scale can help to automatically assess the difficulty of finding bugs in these programs. For instance, when building a new fuzzing benchmark, researchers can use wide fuzzing to quickly test a large pool of potential programs, analyzing the rate of bugs found per unit of time. This analysis would help in categorizing programs based on their resilience to fuzzing, enabling the creation of a benchmark with varying difficulty levels.

## 3 FUZZING AT SCALE FRAMEWORK

In the remaining of the paper, we use fuzzing at scale to refer exclusively to the wide type of fuzzing at scale. Further, first we provide a definition of the fuzzing at scale problem under consideration. Subsequently, we introduce a variety of schedulers designed to address this problem. Finally, we provide additional details about particular components of the framework.

### 3.1 Problem Setup and Objective

Fuzzing at scale consists of utilizing available computational resources to fuzz a pool of programs. We are interested in improving the efficiency of this task. More precisely, given $p$ programs, a fuzzer, and $c$ available CPU cores, the objective is to design a scheduler that continuously and dynamically allocates and deallocates the fuzzing of the programs among the cores, so as to optimize the code coverage across all programs. This task requires making informed scheduling decisions while accounting for the unpredictable nature of fuzzing, the diversity of software, and the inherent constraints of limited computational resources, i.e. limited amount of available CPUs. It is important to emphasize that the programs are fuzzed independently, utilizing the same off-the-shelf fuzzer that lacks any specialized functionality for conducting fuzz testing on multiple programs simultaneously. We use the term fuzzers, to refer to the instances of the same fuzzer running on different programs. Furthermore, sometimes we use the term program or fuzzed program to refer to the concept that the fuzzer is fuzz testing the program.

Central to fuzzing at scale is the scheduler. However, prior to delving into it, it is important that we quickly touch on two other aspects.

Condition $p > c$ . The fuzzing at scale problem is truly challenging, when the number of programs $p$ exceeds the number of available CPU cores $c$ . When this condition is met, the scheduler must make intelligent decisions regarding which programs to run and when to allocate the limited CPU resources. If the number of programs is comparable or less than the number of cores, all programs can be fuzzed simultaneously without the need for a scheduler to make any decisions, thus eliminating the challenge of resource allocation and making the problem trivial.

Code coverage objective. It is essential to assume that the fuzzer's objective aligns with the fuzzing at scale objective, i.e., both should be focused on maximizing code coverage (or any other shared objective, such as the number of found bugs). If there is a misalignment in the objectives, then the scheduler will be either inefficient or entirely meaningless. Fortunately, for many existing fuzzers, this alignment is already in place, as their primary aim is to uncover as many unique program code regions as possible, i.e. to maximize code coverage. The objective of the fuzzing at scale thus becomes to schedule fuzzing of some programs ${P}_{1},\ldots ,{P}_{p}$ so as to maximize the accumulative coverage over all these programs, i.e.

$$
\max \mathop{\sum }\limits_{i}\operatorname{Cov}\left( {P}_{i}\right)  \tag{1}
$$

where $\operatorname{Cov}\left( {P}_{i}\right)$ is the total code coverage obtained by fuzzing the program ${P}_{i}$ , while max is taken over all schedulers.

### 3.2 Schedulers

A scheduler plays a pivotal role in fuzzing at scale tasks, serving as the primary mechanism for simultaneously fuzzing programs across available CPU cores. Similarly to the objectives of operating system schedulers [Peterson and Silberschatz 1985], the goal of a fuzzing scheduler is to allocate fuzzing programs on the available CPU cores, continuously rotating which programs are actively fuzzed based on a predefined criterion. By managing the execution of fuzzing tasks, the ultimate goal of the scheduler is to ensure that the available CPU cores are utilized optimally to maximize code coverage across the entire pool of programs while adhering to the limited CPU cores constraint.

We tackle the scheduling problem with multitasking. In other words, all programs are fuzzed simultaneously, but due to the limited number of CPUs, the fuzzers take turns to run on the available CPUs. More precisely, we divide the available CPU time into short, equal periods called time slices. At each time slice only $c$ among all $p$ programs are actually being fuzzed on the available $c$ cores. The remaining fuzzers are paused - in operating system terminology they are in so-called stopped state, so we will use the term stopped to refer to fuzzers that have been paused. At the beginning of a time slice, the scheduler decides which stopped fuzzer to start (resume) running. To further simplify the decision, we always stop the longest continuously running fuzzer. It means that if the fuzzing of a program starts at time slice i, it will be stopped at i + c. As a result, the only task of a scheduler is at each time slice to choose which stopped fuzzer to start. During the entire fuzzing at scale process, each program fuzzer is started and stopped multiple times.

Let us now consider potential schedulers, beginning with the basic (or baseline) and moving to more advanced. As a baseline we use a simple scheduler built upon the round-robin scheduling algorithm [Liu and Layland 1973] (in operating systems this scheduler is also considered basic). At each time slice, it chooses to start the fuzzer that has not been ran for the longest of time. This is equivalent to cycling through the pool of programs. By doing so, the round-robin scheduler aims to distribute the fuzzing efforts evenly across all programs. Clearly, this baseline scheduler is not optimized for maximizing code coverage, i.e. it is not necessarily efficient, because it does not have any mechanism to give preference to certain fuzzers.

The next logical step in developing a more efficient scheduler is to incorporate feedback from the fuzzed programs, to make informed scheduling decisions, rather than to rely solely on their order in the pool. This feedback-driven approach allows the scheduler to prioritize programs with promising fuzzing results. It leads to a more efficient and effective allocation of CPU cores. As a feedback, we use the code coverage obtained by the fuzzer per unit of fuzzed time. Furthermore, we assume that this past code coverage is a predictor for future code coverage. In other words, if in the past, fuzzing program A resulted in more code coverage then fuzzing program B, then most likely this will be the case in the future as well. All of our schedulers will rely on this hypothesis and thus will discriminate in favor of fuzzing programs that have resulted in higher past coverage.

Algorithm 1: Simple MAB Scheduler

Pseudo-code for the Simple MAB Scheduler. Each fuzzed program ${P}_{i}$ as assigned bandit, defined with

achieved coverage ${\operatorname{Cov}}_{i}$ , fuzz time ${\operatorname{Time}}_{i}$ , and reward $\frac{{\operatorname{Cov}}_{i}}{{\operatorname{Time}}_{i}}$ . Lists ${L}_{\operatorname{run}},{L}_{\text{stop }}$ hold indices of programs that are currently undergoing fuzzing and paused, respectively. $\operatorname{getCov}\left( {{P}_{i}, t - 1}\right)$ is a function that returns the increase of code coverage of fuzzing program ${P}_{i}$ during the time slice $t - 1$ . RandFloat(0,1) returns random number in the interval(0,1). RandElemFrom $\left( {L}_{\text{stop }}\right)$ returns random element from the list ${L}_{\text{stop }}$ .

---

Input: ${P}_{1},\ldots ,{P}_{p}$ programs, fuzzer $F, c$ CPU cores, $\epsilon$

for $i = 1,\ldots p$ do

			${\text{Cov}}_{i} = {\text{Time}}_{i} = 0$ - init MABs

end

${L}_{\text{run }} = \{ 1,\ldots , c\}$ $\vartriangleright$ init list running

${L}_{\text{stop }} = \{ c + 1,\ldots , p\}$ $\vartriangleright$ init list stopped

for $i$ in ${L}_{\text{run }}$ do

			START $F\left( {P}_{i}\right)$ $\vartriangleright$ start fuzzing programs from ${L}_{\text{run }}$

end

- At beginning of each time slice

for $t = 1,\ldots$ do

			- Update MABs

			for $i$ in ${L}_{\text{run }}$ do

					${\text{Time}}_{i} +  = 1$ $\vartriangleright$ add time

				${\operatorname{Cov}}_{i} +  = \operatorname{get}\operatorname{Cov}\left( {{P}_{i}, t - 1}\right) \; \vartriangleright$ add new coverage produced in the prev time slice

			end

			last $= {L}_{\text{run }}\left\lbrack  c\right\rbrack$ - get last running

			STOP $F\left( {P}_{\text{last }}\right)$ - stop fuzzing it

			${L}_{\text{stop }} = {L}_{\text{stop }} \cup  \{$ last $\}$

			${L}_{\text{run }} = {L}_{\text{run }} \smallsetminus  \{$ last $\}$

			- Use MAB to decide which to fuzz next

			if $\epsilon  < \operatorname{RandFloat}\left( {0,1}\right)$ then

					${new} = \arg \mathop{\max }\limits_{{i \in  {L}_{\text{stop }}}}\frac{{\operatorname{Cov}}_{i}}{{\operatorname{Time}}_{i}}$ - get best program

			else

					new $=$ RandElemFrom $\left( {L}_{\text{stop }}\right)$ - get random

			end

			START $F\left( {P}_{\text{new }}\right)$ $\vartriangleright$ start/resume fuzzing it

			${L}_{\text{stop }} = {L}_{\text{stop }} \smallsetminus  \{$ new $\}$

			${L}_{\text{run }} = \{$ new $\}  \cup  {L}_{\text{run }}$

end

---

To incorporate the code coverage feedback from the fuzzed programs, we employ the multiarmed bandits (MAB) algorithm [Slivkins et al. 2019], a well-established decision-making approach designed to optimize long-term rewards, which in our fuzzing at scale context corresponds to maximizing code coverage (1). We assume that at the beginning of each time slice, the scheduler must decide which stopped (paused) program should resume fuzzing. The MAB algorithm guides this decision-making process by considering the previous performance of each program, i.e. by factoring the amount of code coverage achieved per unit of time during a program fuzzing. It does not always choose the best-performing fuzzer, as that is not an optimal strategy in the long run. Rather, in accordance to the MAB approach, the scheduler dynamically balances exploration (choose random fuzzer) and exploitation (choose best performing fuzzer), ensuring that it maintains a diverse pool of fuzzing targets while also focusing on programs with the highest potential for code coverage. The balance (or the tradeoff) between exploration and exploitation is controlled by a parameter $\epsilon$ where $0 < \epsilon  < 1$ and is called $\epsilon$ -greedy [Sutton and Barto 2018]. For example, when $\epsilon  = {0.1}$ , then in ${10}\%$ of the time slices MAB will schedule for fuzzing a randomly chosen program, and in 90% the best performing program (i.e. the one that achieved highest coverage per time unit so far). We call this scheduler Simple MAB scheduler and give its pseudo code in Algorithm 1.

Multi-armed bandits are effective under the so-called stationary assumption, which posits that the rewards remains constant. In our case, this translates to the assumption that each program gets a constant increase in coverage during fuzzing. However, in practice this is not true. Observations indicate that for the majority of programs, fuzzers provide more code coverage at the beginning of fuzzing, followed by a decline - our own experiments confirm this finding as well, refer to Figure 1.

![0196f76b-61e7-7503-9e9d-08ce31d54b6f_7_523_852_514_362_0.jpg](images/0196f76b-61e7-7503-9e9d-08ce31d54b6f_7_523_852_514_362_0.jpg)

Fig. 1. The total amount of code coverage over time obtained by fuzzing 1,000 programs with AFL++ fuzzer.

This outcome is not unexpected, as not all so-called program branches (i.e. conditional statements if, switch) can be easily flipped (e.g. force statement in if to change value from true to false and vice-versa). Instead, the simpler branches are flipped early on with minimal effort, while substantial time is devoted to tackling the more complex ones. We thus next investigate MABs featuring non-stationary rewards, which are characterized by rewards that vary over time, or in our context, a code coverage rate that declines with time. To handle this case, we apply discounting [Sutton and Barto 2018]. Specifically, instead of presuming that all code coverage contributes equally to reward calculations, we exponentially discount earlier coverage results using a factor $\gamma$ , where $0 < \gamma  < 1$ . More precisely, we update the reward estimate for a program from the old $\frac{\mathop{\sum }\limits_{{i = 1}}^{n}{c}_{i}}{\mathop{\sum }\limits_{{i = 1}}^{n}{t}_{i}}$ , where ${c}_{i}$ and ${t}_{i}$ represent the corresponding code coverages and time intervals it ran, to the new reward $\frac{\mathop{\sum }\limits_{{i = 1}}^{n}{c}_{i}{\gamma }^{n - i}}{\mathop{\sum }\limits_{{i = 1}}^{n}{t}_{i}{\gamma }^{n - i}}$ . This signifies that we assign exponentially greater importance to recent fuzzing results compared to older outcomes. We refer to this scheduler as the Discounted MAB scheduler.

In the present MAB framework, we must determine the values for two parameters: the exploration-exploitation balance $\epsilon$ and the discount rate $\gamma$ . Identifying the optimal values for these parameters can be challenging or even unattainable. As a result, we employ a more cautious approach, allowing these values to fluctuate rather than remain fixed. Specifically, we permit $\gamma$ to continually cycle through a range of values. This implies that, at times, we assume older code coverage results are more significant, while at other times, they are deemed less. Conversely, we gradually increase the value of $\epsilon$ over time. This strategy initially compels the scheduler to allocate fuzzing resources to programs that demonstrate the largest increase in code coverage. However, as time progresses, the

Algorithm 2: BOIAN Scheduler scheduler explores more other programs as well. The rationale behind this approach is that, over time, previous code coverage becomes a less reliable predictor of future coverage. This is because only challenging, unflipped program branches are likely to remain, and they are equally hard to flip for all programs, regardless of the previous code coverage progress. Thus, by gradually increasing $\epsilon$ , we assure the scheduler is giving more balanced fuzz time to all programs. However, we never increase the value of $\epsilon$ to 1, so the scheduler still favors, albeit less, programs that have achieved higher code coverage. We refer to our final scheduler as BOIAN, and present its pseudo-code in Algorithm 2.

Pseudo-code for BoIAN scheduler. timeToIncreaseEpsilon(t), timeToChangeGamma(t) are predicates.

For other notations refer to Algorithm 1.

---

Input: ${P}_{1},\ldots ,{P}_{p}$ programs, fuzzer $F, c$ CPU cores, $\epsilon$

for $i = 1,\ldots p$ do

	${LTim}{e}_{i} = {LCo}{v}_{i} = \varnothing$ - init MABs lists

end

${L}_{\text{run }} = \{ 1,\ldots , c\}$ $\vartriangleright$ init list running

${L}_{\text{stop }} = \{ c + 1,\ldots , p\}$ $\vartriangleright$ init list stopped

for $i$ in ${L}_{\text{run }}$ do

	START $F\left( {P}_{i}\right)$ $\vartriangleright$ start fuzzing programs from ${L}_{\text{run }}$

end

$\epsilon  = {\epsilon }_{min}$ $\vartriangleright$ starting $\epsilon$ value

$\Gamma  = \left\{  {{\gamma }_{1},\ldots ,{\gamma }_{g}}\right\}$ - all $\gamma$ values

$\gamma  = {\gamma }_{1}$ - starting $\gamma$ value

- At beginning of each time slice

for $t = 1,\ldots$ do

	- Update MABs

	for $i$ in ${L}_{\text{run }}$ do

		Append 1 to ${LTim}{e}_{i}$ - append new time

		Append get $\operatorname{Cov}\left( {{P}_{i}, t - 1}\right)$ to $L{\operatorname{Cov}}_{i}$ $\vartriangleright$ append new coverage

	end

	last $= {L}_{\text{run }}\left\lbrack  c\right\rbrack$ - get last running

	STOP $F\left( {P}_{\text{last }}\right)$ - stop fuzzing it

	${L}_{\text{stop }} = {L}_{\text{stop }} \cup  \{$ last $\}$

	${L}_{\text{run }} = {L}_{\text{run }} \smallsetminus  \{$ last $\}$

	$\vartriangleright$ Update $\epsilon ,\gamma$ if right time step

	if timeToIncreaseEpsilon(t) then

		$\epsilon  =$ increase $\left( \epsilon \right)$

	end

	if timeToChangeGamma(t)then

		$\gamma  = \operatorname{next}\left( {\Gamma ,\gamma }\right)$

	end

	- Use MAB to decide which to fuzz next

	if $\epsilon  < \operatorname{RandFloat}\left( {0,1}\right)$ then

		${new} = \arg \mathop{\max }\limits_{{i \in  {L}_{stop}}}\frac{\mathop{\sum }\limits_{j}{LCo}{v}_{i}\left\lbrack  j\right\rbrack   \cdot  {\gamma }^{n - j}}{\mathop{\sum }\limits_{j}{LTim}{e}_{i}\left\lbrack  j\right\rbrack   \cdot  {\gamma }^{n - j}}$ - get best prog

	else

		new $=$ RandElemFrom $\left( {L}_{\text{stop }}\right)$ - get random

	end

	START $F\left( {P}_{\text{new }}\right)$ $\vartriangleright$ start/resume fuzzing it

	${L}_{\text{stop }} = {L}_{\text{stop }} \smallsetminus  \{$ new $\}$

	${L}_{\text{run }} = \{$ new $\}  \cup  {L}_{\text{run }}$

end

---

In Table 1, we provide a brief overview of the examined schedulers, outlining the problems they aim to address and the solutions they propose.

Measure of efficiency. Assessing the efficiency of a scheduler can be ambiguous. In this context, we explore two distinct metrics of efficiency. The first metric focuses solely on the accumulative coverage across all fuzzed programs. This equates to the total sum of all coverages of individual programs. It is aligned with the objective of the scheduler. This metric referred to as accumulative is relevant when the aim is to measure total coverage. However, it neglects the possibility that a smaller subset of programs may contribute a disproportionately large portion to the overall coverage. In essence, it measures the mean code coverage across all programs but ignores the variance.

The second metric, referred to as voting, attempts to correct this potential oversight by considering the variance. It compares fuzzing outcomes based on the fraction of programs that have been fuzzed better. For instance, in a scenario where 100 programs are fuzzed, if approach A yields better coverage than approach B for 35 programs, worse coverage for 25 , and same coverage for the remaining 40 , then according to the second metric (voting), approach A would be deemed superior because 35 > 25 . (Furthermore, we say that A is ${10}\%$ better than B because $\frac{{35} - {25}}{100} = {0.1}$ .) Conversely, the first metric (accumulative) would compute the total coverage produced for all 100 programs and decide on the preferable approach solely based on this sum. Therefore, the accumulative metric prioritizes candidates that yield the highest overall aggregate coverage, whereas the vote metric gives preference to those providing higher coverage for the majority of the programs.

### 3.3 Fuzzing at Scale

We can use each of the schedulers described in the previous section to simultaneously fuzz test multiple programs and thus carry out fuzzing at scale. Recall, a common feature of all these schedulers is the idea of multitasking, i.e. frequently switching between executing fuzzers on the available processing cores. This switching process occurs once at the beginning of each time slice and the choice on which fuzzer to resume executing is determined by the scheduler. Further we provide additional details about the internals of the framework:

- Time slice duration. Shorter time slices are preferable because they result in finer granularity, leading to reduced time spent on underperforming fuzzers. However, if the slices are too short, the impact of recomputing the MAB rewards and starting or stopping fuzzers may demand significant CPU resources. Furthermore, the duration of the time slices should be sufficiently large to facilitate proper support and implementation by the operating system (OS), which also relies on a scheduler for multitasking purposes. This implies that the fuzzing scheduler's time slices must exceed those employed by the OS scheduler, which, in the case of the present Linux kernel, default to ${20}\mathrm{\;{ms}}$ . Otherwise, the OS scheduler cannot perform the context switch on time, i.e. cannot preempt one program and resume another.

- Fuzzing at scale on clusters. If the time slices are sufficiently long, the aforementioned multitasking approach can be used to fuzz at scale, not only on an individual machine, but also across a cluster. The scheduler will run on the server (in a similar fashion as in the case of a single machine), but will communicate with fuzzers that run on client machines. To accommodate this scenario, the slice duration should be set larger than the latency between the nodes of the cluster. If this condition is met, our framework can be easily extended to support multiple machines.

Table 1. Schedulers for fuzzing at scale. Each subsequent Scheduler aims to address a remaining Problem that the previous scheduler left unsolved, by utilizing the Solution.

<table><tr><td>Scheduler</td><td>Problem</td><td>Solution</td></tr><tr><td>Baseline (round-robin)</td><td>basic scheduling of multiple fuzzers</td><td>equal time to all fuzzers; in each time slice choose the longest paused fuzzer</td></tr><tr><td>Simple MAB</td><td>prefer high code coverage fuzzers</td><td>MAB; compute rewards as cover- age/time and either choose fuzzer with best reward (in $1 - \epsilon$ of time slices) or random fuzzer (in $\epsilon$ of time slices)</td></tr><tr><td>Discounted MAB</td><td>prefer more recent high code coverage fuzzers</td><td>discounted MAB; when comput- ing rewards give exponentially lower importance to older cover- age results (by multiplying each $T$ - periods old with ${\gamma }^{T}$ )</td></tr><tr><td>BOIAN</td><td>choose good discounting $\gamma$ ; with time, past coverage be- comes less reliable predictor of future coverage</td><td>discounted MAB but cycle $\gamma$ and slowly keep increasing $\epsilon$</td></tr></table>

- Limit on the number of programs. The OS may impose limit on the number of programs that can be fuzzed concurrently. This restriction aligns with the standard limitations of multitasking OSs, which involve managing multiple processes that consume varying amounts of system resources, including RAM, disk usage, open files, etc. Based on our experimental results presented in Section 6, the imposed limit appears to be relatively generous, as we were able to fuzz up to 5,000 programs simultaneously on a single Linux machine.

## 4 BENCHMARKS

We now shift our focus to the second major component of a fuzzing at scale framework: a substantial collection of target programs for fuzzing. Most of the currently used benchmarks for fuzzers are rather small. We inspected all such collections presented in [Aschermann et al. 2019; Blazytko et al. 2019; Chen et al. 2020b; Chen and Chen 2018; Chen et al. 2019b, 2020a, 2019a; Cho et al. 2019; Gan et al. 2020, 2018; Geretto et al. 2022; Koike et al. 2022; Lee et al. 2021; Lemieux and Sen 2018; Liang et al. 2019, 2022; Lyu et al. 2019, 2022; Nagy et al. 2021a, b; Nguyen et al. 2020; Nikolić et al. 2021; Österlund et al. 2020; Rawat et al. 2017; Shah et al. 2022; She et al. 2022; Wang et al. [n. d.]; Yu et al. 2022; Yue et al. 2020; Yun et al. 2018; Zhou et al. 2022; Zhu and Böhme 2021; Zong et al. 2020] and found that they consist of 10-30 programs, and in total, across all of these 33 papers, there are only 184 distinct programs. As a general rule, these programs are sampled from the OSS-Fuzz [Serebryany 2017] package - a curated collection of open-source programs and libraries. Currently, OSS-Fuzz contains around 1,000 target programs, but only roughly half of them have source code in $\mathrm{C}/\mathrm{C} +  +$ . Furthermore, due to widespread adoption as a predominant fuzzing target, there is a risk of overfitting. That is, recent fuzzers might inadvertently be optimized to excel on programs from OSS-Fuzz, while only performing adequately on other targets. Therefore, using OSS-Fuzz as a sole benchmark may not provide objective evaluation results.

This calls for the development of a new fuzzing benchmark that will include a large number of programs. Such a set is significant for both the success of our fuzzing at scale task and for the advancement of future fuzzing research. A comprehensive and diverse benchmark offers numerous benefits for fuzzing projects. It provides a rich and representative sample of programs, capturing the complexity and variety of real-world software. This enables the evaluation of fuzzing techniques in a more realistic context. The task of creating benchmark includes identifying potential candidate programs, retrieval of their corresponding compilation rules, determining the appropriate invocation methods for input execution, and providing testcases.

The currently limited number of candidate programs might be attributed to the fact that benchmarks are created manually. That is a tedious and time-consuming process. In addition, maintaining and updating a manually produced benchmark becomes an ongoing task, as otherwise, the reuse of the same benchmarks for prolonged period does not take into account that software is continuously changing. Producing the benchmark automatically is a valuable alternative because it is the most efficient and scalable way to generate a large and diverse set of target programs. Moreover, automating the process ensures easy rotation of programs in the benchmark, as well as staying up-to-date with rapidly evolving software. In comparison to manual production, there are a few drawbacks such as the potential exclusion of certain important programs or the inclusion of irrelevant. However, we believe the tradeoff is well justified because the advantage of having a large, diverse, and up-to-date benchmark outweights the potential drawbacks of not including certain programs. Thus we focus on generating the benchmark automatically.

To obtain programs for inclusion in the benchmark, we opt to utilize the set of Ubuntu packages as our primary source. This offers multiple benefits. First, it provides a single, unified source consisting of a vast pool of programs, simplifying the process of collecting, maintaining, and even updating the benchmark. Second, the programs within Ubuntu packages are highly diverse, spanning various application domains, thus ensuring that the benchmark is representative and well-suited for evaluating fuzzing techniques across a wide range of software. Third, a considerable number of these programs adhere to an installation template, which simplifies the process of setting up and incorporating programs into the benchmark.

We begin by considering the entire set of available Ubuntu packages and apply several fully automatic steps to obtain the final benchmark:

(1) Eliminate unsuitable candidates. We remove obviously unsuitable candidates based solely on the names of the packages. For instance, packages with names beginning with lib are excluded, as libraries typically require manual inclusion in larger programs, rendering them unsuitable for automatic benchmark generation.

(2) Compile remaining programs. We attempt to compile all remaining programs, either by utilizing the provided makefiles or by resorting to general compilation routines when necessary.

(3) Identify executable binaries and inputs arguments. We examine all compiled packages and identify executable binaries. For each such program, we apply known heuristics (as indicated in Section 5) and try to infer input arguments that would prompt reading of files, so the fuzzer can supply testcases during fuzz testing the program.

All such candidate programs are collected to form the fuzzing benchmark UBUNTUBENCH.

## 5 IMPLEMENTATION

We implement the scheduler BOIAN in C++ for efficiency purposes, comprising 4,320 lines of code. The same implementation can be utilized to run the other aforementioned schedulers, by disabling some of the functionalities of BOIAN, as all of these schedulers are subsets of BOIAN. Internally, the scheduler maintains two lists of fuzzed programs: those that are currently running and those that are stopped. Each fuzzed program is executed as a separate process, running the fuzzer on the target program. At the beginning of each time slice, the scheduler makes a decision on which stopped fuzzed program to start, as previously mentioned, based on the multi-armed bandits algorithm. The actual stopping and starting of fuzzers are accomplished by sending the signals SIGSTOP and SIGCONT to the corresponding processes running the fuzzer on programs. BOIAN enables the selection of the number of utilized CPU cores by setting hard affinity [Peterson and Silberschatz 1985] for all processes initiated by the scheduler. This is achieved through a system call implemented with the C function sched_setaffinity, which allows to select precisely the set of CPU cores allowed to run the fuzzing processes. BOIAN also incorporates a range of mechanisms that mitigate the excessive utilization of CPU and RAM resources by the targeted programs by continuously monitoring and potentially terminating processes with the use of the library libprocps. This feature is particularly useful, given that the UBUNTUBENCH is automatically compiled and may potentially include resource-intensive programs.

We implement the generation of UBUNTUBENCH as a series of Python scripts, one for each step of this process, and in total with 1965 lines of code. All of these scripts support parallelization by utilizing the Python module multiprocessing. We first download all Ubuntu package that pass the filtering criterion based on their name. Then, before compiling the packages we install the necessary dependencies according to the information provided in the file debian/control that is present in each Ubuntu package. We try to compile the Ubuntu packages with dedicated $\mathrm{C}/\mathrm{C} +  +$ compilers provided by the fuzzers, so that the produced binaries are well instrumented to provide code coverage information. For this purpose, we initially employ the prevalent method of executing debian/rules build that is also present in all packages. When this approach does not generate an instrumented binary, we resort to an ad-hoc compilation process, based on installation files such as autogen. sh, Makefile, CMakeLists.txt, which indicate the appropriate build system to use. By leveraging these files, we can specify to use fuzzer provided compilers and obtain instrumented binaries. To detect the required fuzzer instrumentation in compiled binaries, we search for an appropriate fuzzer fingerprint. This is done by parsing the output produced with the command line tool objdump. We also check for dependencies of the binaries on dynamic libraries and on particular requirement for currently working directory, which is important to binaries that encode relative path. In order to deduce the input arguments that prompt the binary to read from an input file, we employ a heuristic approach and parse the outputs of ./binary -help and ./binary - -help, and subsequently examine all plausible candidates. To determine whether the binary indeed reads from a file when a certain input argument is provided, we utilize the command line tool strace, which detects file-opening operations for reading. This method enables us to effectively identify specific input arguments that cause the binary to perform file-reading actions.

All of the above implementations are publicly released ${}^{1}$ .

## 6 EVALUATION

Further we present experimental results highlighting the importance of schedulers and the relevance of the whole fuzzing at scale framework.

---

${}^{1}$ https://github.com/ivicanikolicsg/fuzzing_at_scale.

---

Experimental setup. For all experiments we use the same box with Ubuntu Desktop 20.04, two Intel Xeon E5-2680v2 CPUs @2.80GHz with total of 40 cores, and 64GB DDR3 RAM. All subsequent fuzzing at scale tasks are executed on 30 cores and all the schedulers are set to use 0.1 s times slices. In these tasks, the number of fuzzed programs and the duration of fuzzing are determined by two basic goals: provide meaningful results and finish all experiments within two weeks. For these reasons, the number of fuzzed programs ranges in 1,000 to 5,000 for the larger benchmark (UвunтцВепсн) and around 200 to 300 for the smaller (OSS-Fuzz), and the fuzzing duration in 15 minutes to 1 hour on average per program. For the actual fuzzing, we make use of three well-established fuzzers, AFL [Zalewski 2017], AFL++ [Fioraldi et al. 2020] and Honggfuzz [Swiecki et al. 2021]. This selection of fuzzers is based on the fact that AFL is the most popular baseline grey-box fuzzer, whereas AFL++ and Honggfuzz are the top performing fuzzers on Google fuzzer comparison platform FuzzBench [Metzman et al. 2021]. As a measure of code coverage we use edge coverage provided by AFL tool afl-showmap.

Instead of running each experiment multiple times and reporting the average of those runs, we run each only once and report the outcome. We do this for two reasons. First, the number of experiments in Sections 6.1, 6.2 alone is 14 , each runs for around half a day, thus we need close to a week just to run once all experiments. Multiple runs would thus require a month or so of running. Second, the large number of programs in each experiment significantly reduces the variance between different runs. Therefore, repeating experiments and averaging their outcomes may not significantly improve the accuracy of the reported results.

The benchmarks UBUNTUBENCH and OSS-Fuzz. As described in Section 4, we use packages from Ubuntu to obtain the benchmark UBUNTUBENCH- refer to Table 2 for the steps. More precisely, from the pool of all 75,799 available packages from Ubuntu 20.04, with filtering and compilation we produce 5,467 target programs that compose UBUNTUBENCH. All of these programs can be compiled at least with the standard gcc and g++ compilers. During fuzzing, as initial testcases (seeds) we provide randomly chosen files found in the corresponding Ubuntu packages. In the OSS-Fuzz suite [Serebryany 2017], out of the total 1,193 packages, a mere 326 possess C/C++ source code and are recognized as being compatible with the AFL+ compiler, as indicated in their respective configuration files. These yield a total of 345 distinct binaries that constitute our OSS-Fuzz benchmark set. Out of these binaries, 216 can also be compiled with AFL compiler.

Table 2. The number of packages/programs handled at different stages of producing UBUNTUBENCH.

<table><tr><td/><td>UbuntuBench</td><td>OSS-Fuzz</td></tr><tr><td>Targets packages</td><td>75,799</td><td>1,193</td></tr><tr><td>Filtered packages</td><td>22,368</td><td>326</td></tr><tr><td>Compiled packages</td><td>15,551</td><td>326</td></tr><tr><td>Final binaries</td><td>5,467</td><td>345</td></tr></table>

### 6.1 The Importance of a Scheduler

Our first experiment is to evaluate the significance of schedulers in fuzzing at scale tasks. More precisely, we want to test empirically if the choice of scheduler can impact the outcome (produced coverage) of this task. For this purpose we compare all schedulers discussed in Section 3.2. We consider 4 different schedulers: baseline round-robin, simple MAB, discounted MAB with fixed $\gamma  = {0.9}^{2}$ , and BoIAN with cycling $\gamma  \in  \{ {0.9},{0.99},{0.999}\}$ and $\epsilon$ increasing in $\left\lbrack  {{0.01},{0.75}}\right\rbrack$ . For each scheduler, we conduct experiments independently but use the same set of selected programs. The programs are fuzzed simultaneously with AFL++ on 30 CPUs. We run two fuzzing at scale tasks: one for UBUNTUBENCH with 1,000 programs sampled at random and with average fuzz time of 15 minutes, and one for OSS-Fuzz where we take all available 345 programs and fuzz for 1 hour on average per program ${}^{3}$ . We evaluate the performance of the four schedulers on both metrics, accumulative and voting. Recall, accumulative metric favors schedulers that provide better total coverage across all fuzzed programs, whereas voting metric favors those that provide higher coverage for more programs. We present the results as pairwise comparisons of the different schedulers and in terms of percentage increase or decrease between the first and the second elements of the pair.

Table 3. Comparison of 4 different schedulers with accumulative metric. The numbers in the cells are the percentage increase/decrease in total coverage provided by the row scheduler in comparison to the column scheduler.

<table><tr><td colspan="9">UbuntußenchOSS-Fuzz</td></tr><tr><td/><td>Baseline</td><td>Simple</td><td>Disc</td><td>Boian</td><td>Baseline</td><td>Simple</td><td>Discounted</td><td>Boian</td></tr><tr><td>Baseline</td><td>0</td><td>-13</td><td>-16</td><td>-22</td><td>0</td><td>-10</td><td>-8</td><td>-9</td></tr><tr><td>Simple MAB</td><td>15</td><td>0</td><td>-3</td><td>-10</td><td>11</td><td>0</td><td>1</td><td>0</td></tr><tr><td>Discounted</td><td>19</td><td>3</td><td>0</td><td>-7</td><td>9</td><td>- 1</td><td>0</td><td>0</td></tr><tr><td>Boian</td><td>29</td><td>11</td><td>8</td><td>0</td><td>10</td><td>0</td><td>0</td><td>0</td></tr></table>

Let us first take a look at the results in accumulative metric given in Table 3. First, it is clear that the baseline is the worst performing scheduler. This indicates that, when fuzzing simultaneously multiple programs (fuzzing at scale), allocating equal fuzz time to each program (a strategy implemented by the baseline scheduler via simple round-robin allocation) provides inferior accumulative code coverage over all programs. All other considered schedulers provide up to 10%-29% higher code coverage in comparison to the baseline. In particular, according to the results of Table 3, the top scheduler on UBUNTUBENCH is BOIAN which provides clear advantage over the remaining schedulers. Similar results are obtained on OSS-Fuzz benchmark as well - BOIAN is the top performing scheduler. From the fuzzing logs provided by our implementation we could also get information about the exact fuzzing time allocated to each program. Accordingly, the baseline scheduler assigned approximately 15 minutes (900 seconds) to fuzzing of each program. In contrast, the time assigned by other schedulers varies. For instance, BoIAN assigned from 119 seconds to 8,878 seconds for fuzzing of different UBUNTUBENCH programs, and 390 to 8,026 seconds for OSS-Fuzz programs. Thus, by monitoring and adjusting the allocated fuzzing time, it provided 10-29% higher total coverage in comparison to the baseline.

We now turn our attention to the results of the voting metric, presented in Table 4. Several key differences from the previous metric are worth noting, the most striking one being that the baseline scheduler demonstrates good performance in this metric as it surpasses the performance of some of the other schedulers. It means allocating equal fuzz time to each program is a reasonable

---

${}^{2}$ The value of 0.9 was taken as the best performing among our tested set $\{ {0.9},{0.99},{0.999}\}$ .

${}^{3}$ It means that each experiment for both UBUNTUBENCH and OSS-Fuzz runs roughly for half a day.

---

Table 4. Comparison of 5 different schedulers with the voting metric. The numbers in the cells are the percentage increase/decrease of fuzzed programs for which the row scheduler provides better coverage than the column scheduler. approach when aiming to generate good coverage for most of fuzzed programs. However, even under this metric, there is a scheduler that strictly outperforms the baseline, and this is BOIAN. This new scheduler provides better coverage in comparison to the baseline, as well as the other two schedulers.

<table><tr><td colspan="9">UbuntußenchOSS-Fuzz</td></tr><tr><td/><td>Baseline</td><td>Simple</td><td>Disc</td><td>Boian</td><td>Baseline</td><td>Simple</td><td>Discounted</td><td>BOIAN</td></tr><tr><td>Baseline</td><td>0</td><td>47</td><td>19</td><td>-8</td><td>0</td><td>17</td><td>3</td><td>-9</td></tr><tr><td>Simple MAB</td><td>-47</td><td>0</td><td>-45</td><td>-56</td><td>-17</td><td>0</td><td>-11</td><td>-21</td></tr><tr><td>Discounted</td><td>-19</td><td>45</td><td>0</td><td>-24</td><td>-3</td><td>11</td><td>0</td><td>-9</td></tr><tr><td>Boian</td><td>8</td><td>56</td><td>24</td><td>0</td><td>9</td><td>21</td><td>9</td><td>0</td></tr></table>

From the analysis presented above, we can conclude that the choice of a scheduler significantly influences the outcome of concurrently fuzzing multiple programs from the two distinct benchmarks, as demonstrated by the two different code coverage metrics. Specifically, employing a more sophisticated scheduler like BOIAN can yield up to around ${30}\%$ greater fuzzing efficiency (based on the metrics) as opposed to simply allocating equal fuzzing time to each program. Interestingly, this suggests that the efficiency boost provided by a scheduler in large-scale fuzzing tasks is comparable to the improvement offered by a fuzzer in a single program scenario. This is based on the fact that, for example, Google FuzzBench [Metzman et al. 2021] reports an average increase of 20%- ${30}\%$ in coverage across multiple programs when using the best fuzzers compared to less effective ones [Google 2022].

Result 1: Advanced schedulers such as BOIAN lead to more effective fuzzing both in terms of overall code coverage across all programs and in terms of better coverage for majority of programs.

### 6.2 Scheduler vs Fuzzer

In our second experiment, we aim to evaluate the significance of schedulers relative to fuzzers in the context of fuzzing at scale tasks. Specifically, we want to determine whether improving schedulers can yield a comparable impact to improving fuzzers. If this is the case, it would imply that schedulers deserve equal focus as fuzzers when developing more efficient fuzzing at scale frameworks.

We carry out three different fuzzing at scale tasks, independently on UBUNTUBENCH and on OSS-Fuzz. In the first task, we use the baseline scheduler (round-robin) and the baseline fuzzer AFL. In the second, we switch the fuzzer from AFL to AFL++, but keep the same baseline scheduler. In the third, we keep AFL as a fuzzer, but replace the baseline scheduler with BOIAN. We then compare the outcomes of the three tasks on accumulative and voting metrics.

The comparison results are presented in Table 5. It is evident that using a superior fuzzer, such as replacing the standard AFL with the top-performer AFL++, while maintaining the baseline scheduler, leads to a 14% increase in overall code coverage on UBUNTUBENCH and 11% on OSS-Fuzz. This means that AFL++ achieves, on average, 11-14% more coverage than AFL. Furthermore, AFL++ offers better coverage for ${10} - {11}\%$ more programs than AFL. Thus, using a better fuzzer undoubtedly results in more efficient fuzzing at scale. However, comparable and even superior results can be achieved simply by improving the scheduler, rather than the fuzzer. More precisely, by keeping the same fuzzer AFL, but utilizing BOIAN instead of the baseline scheduler, we observe 10-41% increase in accumulative metric and 3-12% increases in voting metric. In fact, the direct comparison between using an advanced fuzzer and baseline scheduler (AFL++ combined with round-robin), and baseline fuzzer and advanced scheduler (AFL combined with BOIAN) demonstrates that the latter provides higher overall coverage, whereas the former provides better coverage for more programs. We can thus conclude that, for effective fuzzing at scale, the scheduler is as crucial as the used fuzzer. Therefore, schedulers should receive a similar level of attention as fuzzers. However, they have been overlooked thus far.

Table 5. Comparison of importance of schedulers and fuzzers. The accumulative and vote columns represent the percentage increase in these two metrics between tasks in the first column and second columns. 'RR' stands for round-robin (baseline) scheduler.

<table><tr><td colspan="8">UbuntuBenchOSS-Fuzz</td></tr><tr><td colspan="2">Task 1</td><td colspan="2">Task 2</td><td>Accum</td><td>Vote</td><td>Accum</td><td>Vote</td></tr><tr><td>fuzz</td><td>sch</td><td>fuzz</td><td>sch</td><td>incr %</td><td>incr %</td><td>incr %</td><td>incr %</td></tr><tr><td>AFL</td><td>RR</td><td>AFL++</td><td>RR</td><td>14</td><td>11</td><td>11</td><td>10</td></tr><tr><td>AFL</td><td>RR</td><td>AFL</td><td>BOIAN</td><td>41</td><td>12</td><td>10</td><td>3</td></tr><tr><td>AFL++</td><td>RR</td><td>AFL</td><td>Boian</td><td>23</td><td>1</td><td>0</td><td>-11</td></tr></table>

Result 2: Improving schedulers is as important as improving fuzzers.

### 6.3 Bugs in Ubuntu

To underscore the importance of the fuzzing at scale framework, we further consider one practical task - identifying bugs in Ubuntu packages. For this purpose, we employ both AFL++ and Honggfuzz to fuzz UBUNTUBENCH and discover bugs. We use BOIAN to schedule fuzzing at scale of the benchmark programs, independently with each fuzzer for 3 days on 30 CPUs, and collect all the bugs found and reported by the fuzzers. We intentionally choose not to separately report bugs found by AFL++ or Honggfuzz. This helps avoid unfair comparisons between the two fuzzers on bug-finding abilities which might have occurred due to our setup misconfigurations or the use of different sanitizers. We merge the bugs identified by both fuzzers and deduplicate the resulting set. Specifically, we gather all inputs reported by the fuzzers that cause program crashes. Next, we verify that these inputs lead to crashes by executing the programs with the crashing inputs, ensuring that some signal is raised. Programs and inputs that pass this filter are then run with the dynamic binary analysis tool Valgrind [Nethercote and Seward 2007] to generate crash trace for each crash. We filter out duplicate traces and report all unique traces as individual bugs. As a result, we find 4,908 bugs in 675 distinct programs among all the evaluated programs. Table 6 provides information on the classification of the discovered bugs.

The quantity of bugs identified during the three-day fuzzing session is fairly high. However, this number alone does not allow to speculate whether extended fuzzing campaigns would yield a

Table 6. Classification of the bugs found by fuzzing at scale UBUNTUBENCH with BOIAN by using AFL++ and Honggfuzz for three days each on 30 CPUs. OTHER refers to all signals with signal number 32 and higher.

<table><tr><td>Signal</td><td>Valgrind description</td><td>#bugs</td></tr><tr><td>SIGHUP</td><td><none></td><td>341</td></tr><tr><td>SIGHUP</td><td>Access not within mapped region</td><td>4</td></tr><tr><td>SIGINT</td><td><none></td><td>53</td></tr><tr><td>SIGQUIT</td><td><none></td><td>16</td></tr><tr><td>SIGILL</td><td><none></td><td>3</td></tr><tr><td>SIGTRAP</td><td><none></td><td>5</td></tr><tr><td>SIGABRT</td><td><none></td><td>1114</td></tr><tr><td>SIGBUS</td><td><none></td><td>5</td></tr><tr><td>SIGBUS</td><td>Non-existent physical address</td><td>1</td></tr><tr><td>SIGFPE</td><td><none></td><td>20</td></tr><tr><td>SIGFPE</td><td>Integer divide by zero</td><td>74</td></tr><tr><td>SIGKILL</td><td><none></td><td>3</td></tr><tr><td>SIGUSR1</td><td><none></td><td>1</td></tr><tr><td>SIGSEGV</td><td><none></td><td>321</td></tr><tr><td>SIGSEGV</td><td>Access not within mapped region</td><td>1556</td></tr><tr><td>SIGSEGV</td><td>Bad permissions for mapped region</td><td>878</td></tr><tr><td>SIGSEGV</td><td>General Protection Fault</td><td>476</td></tr><tr><td>SIGUSR2</td><td><none></td><td>1</td></tr><tr><td>SIGXCPU</td><td><none></td><td>1</td></tr><tr><td>SIGXFSZ</td><td><none></td><td>4</td></tr><tr><td>SIGVTALRM</td><td><none></td><td>1</td></tr><tr><td>OTHER</td><td><none></td><td>30</td></tr></table>

larger number of bugs. To address this, in Figure 2 we illustrate the cumulative count of discovered bugs as the fuzzing progresses ${}^{4}$ . The plot strongly implies that the fuzzing process has not yet reached a plateau. Consequently, we can infer that extending the fuzzing period may result in a substantial increase in the number of found bugs.

Result 3: By utilizing fuzzing at scale for several days, we found 4,908 bugs across 675 Ubuntu programs and longer fuzzing campaigns will likely lead to significantly more bugs.

Given the sheer volume of bugs identified, it is challenging to estimate their potential exploitabil-ity one-by-one. Instead, automated approaches are needed to identify vulnerabilities and report them. To seek assistance, we reached out to Ubuntu and Debian maintainers, but they indicated that no such approaches exist, as previously mentioned by a decade-old similar initiative [Debian 2013; lwn.net 2013] by project Mayhem [Cha et al. 2012]. We were advised to report the bugs directly to the developers and examine potential CVEs case-by-case. Currently, we are working on developing a strategy to automate these steps.

---

${}^{4}$ We approximate the timing of each bug using the Linux timestamp associated with the update of the corresponding crash file.

---

![0196f76b-61e7-7503-9e9d-08ce31d54b6f_18_418_282_721_532_0.jpg](images/0196f76b-61e7-7503-9e9d-08ce31d54b6f_18_418_282_721_532_0.jpg)

Fig. 2. The total number of found bugs in UBUNTUBENCH during three-day fuzzing at scale campaign with AFL++ and Honggfuzz, each running on 30 CPUs.

We did not run a similar fuzzing campaign for OSS-Fuzz programs because they have been fuzzed extensively on the Google infrastructure, so it is highly unlikely that a few days of fuzzing on our server would have produced new, previously unknown bugs.

## 7 RELATED WORK

Platforms for fuzzing multiple programs distributed across multiple cores and machines have been developed by Google. OSS-Fuzz [Serebryany 2017] is a fuzzing platform for open-source programs. It is designed to find vulnerabilities in bulk, by continuously fuzzing programs on the Google infrastructure. It comes with around 1,000 open-source programs written in C, C++, Python, Go, JavaScript, Rust, Swift, it supports AFL++, Honggfuzz, and libFuzzer fuzzers, and has automatic triage and reporting of bugs. As of February 2023, OSS-Fuzz has found around 28,000 bugs accross 850 projects [Google 2023]. On the other hand, ClusterFuzz [Arya et al. 2019] is the distributed fuzzing infrastructure behind OSS-Fuzz. It is open-source and allows for deployment within a personal environment. Note, OSS-Fuzz is a production instance of ClusterFuzz, however, it has some features that are not available in the open-source version. Neither OSS-Fuzz nor ClusterFuzz are wide fuzzing at scale frameworks. Their objective is more in line of distributing the fuzzing of smaller set of programs over vast resources. In contrast, our goal is to optimize small resources for fuzzing of large number of programs.

In the context of fuzzers, the term scheduler refers to the strategy that determines which testcase to choose for fuzzing, also known as seed scheduling. Various strategies exist for this purpose. Some employ a purely random selection [Stephens et al. 2016], while others utilize feedback from previous fuzzing sessions, such as seed size and execution time [Zalewski 2017], favoring less explored code regions [Böhme et al. 2016; Lemieux and Sen 2018], applying supervised machine learning [Chen et al. 2020a], or using other heuristics [Dang et al. 2012; She et al. 2022; Zhao et al. 2020]. In contrast, our usage of the term scheduler aligns more closely with its meaning in multitasking operating systems. In this sense, its primary goal is to manage the execution of different fuzzers on the available CPU resources.

Utilizing advanced techniques to guide selection in fuzzers is not a novel concept. For example, MOpt [Lyu et al. 2019] employs particle swarm optimization to steer the seed selection process. Meanwhile, EcoFuzz [Yue et al. 2020] leverages the multi-armed bandit (MAB) approach for seed selection. Sivo fuzzer [Nikolić et al. 2021] also utilizes MABs, for guiding the selection of not just seeds, but other fuzzer components too. Our schedulers, particularly BOIAN, use MAB to assist in selecting the best fuzzer for execution on CPUs.

## 8 CONCLUSION AND FUTURE WORK

In this study, we explored the problem of fuzzing at scale and demonstrated that non-trivial schedulers can impact its effectiveness. The impact of such schedulers is substantial, comparable to that of the fuzzers themselves. Our scheduler BOIAN is approximately ${30}\%$ more efficient than a basic scheduler. Further advancements in schedulers may lead to an even greater performance gap. There are two prominent research directions that could potentially improve schedulers. The first direction is to concentrate solely on developing and using more advanced strategies for deciding which fuzzer to start based on the available past coverage information for each program. This could involve utilizing better multi-armed bandits or employing entirely new decision-making algorithms. The second approach is to leverage the knowledge gained from fuzzing certain programs and applying it to fuzz other programs. For example, if a method can be devised to transfer the knowledge acquired from fuzzing one program (or a group of programs) to another similar program, the second programs may be fuzzed faster. Advancing any of these two research directions will result in betters schedulers. This in turn will provide significant improvement in effectiveness of fuzzing at scale. We believe that the primary source of improvement will be derived from the progress in schedulers, as opposed to fuzzers, due to the relatively unexplored nature of the former and the well-established maturity of the latter.

## ACKNOWLEDGEMENTS

We thank Prateek Saxena and the anonymous reviewers for their valuable feedback. This research was supported by the Ministry of Education Singapore Tier 2 grant MOE-T2EP20220-0014.

## REFERENCES

Abhishek Arya, Oliver Chang, Max Moroz, Martin Barbella, and Jonathan Metzman. 2019. Open sourcing Clusterfuzz. Google, Inc. Feb (2019).

Cornelius Aschermann, Sergej Schumilo, Tim Blazytko, Robert Gawlik, and Thorsten Holz. 2019. REDQUEEN: Fuzzing with Input-to-State Correspondence.. In NDSS, Vol. 19. 1-15.

Tim Blazytko, Cornelius Aschermann, Moritz Schlögel, Ali Abbasi, Sergej Schumilo, Simon Wörner, and Thorsten Holz. 2019. GRIMOIRE: Synthesizing Structure while Fuzzing.. In USENIX Security Symposium, Vol. 19.

Marcel Böhme, Van-Thuan Pham, and Abhik Roychoudhury. 2016. Coverage-based greybox fuzzing as markov chain. In Proceedings of the 2016 ACM SIGSAC Conference on Computer and Communications Security. 1032-1043.

Sang Kil Cha, Thanassis Avgerinos, Alexandre Rebert, and David Brumley. 2012. Unleashing mayhem on binary code. In 2012 IEEE Symposium on Security and Privacy. IEEE, 380-394.

Hongxu Chen, Shengjian Guo, Yinxing Xue, Yulei Sui, Cen Zhang, Yuekang Li, Haijun Wang, and Yang Liu. 2020b. MUZZ: Thread-aware grey-box fuzzing for effective bug hunting in multithreaded programs. arXiv preprint arXiv:2007.15943 (2020).

Peng Chen and Hao Chen. 2018. Angora: Efficient fuzzing by principled search. In 2018 IEEE Symposium on Security and Privacy (SP). IEEE, 711-725.

Peng Chen, Jianzhong Liu, and Hao Chen. 2019b. Matryoshka: fuzzing deeply nested branches. In Proceedings of the 2019 ACM SIGSAC Conference on Computer and Communications Security. 499-513.

Yaohui Chen, Mansour Ahmadi, Reza Mirzazade Farkhani, Boyu Wang, and Long Lu. 2020a. MEUZZ: Smart Seed Scheduling for Hybrid Fuzzing.. In RAID. 77-92.

Yuanliang Chen, Yu Jiang, Fuchen Ma, Jie Liang, Mingzhe Wang, Chijin Zhou, Xun Jiao, and Zhuo Su. 2019a. EnFuzz: Ensemble Fuzzing with Seed Synchronization among Diverse Fuzzers.. In USENIX Security Symposium. 1967-1983.

Mingi Cho, Seoyoung Kim, and Taekyoung Kwon. 2019. Intriguer: Field-level constraint solving for hybrid fuzzing. In Proceedings of the 2019 ACM SIGSAC Conference on Computer and Communications Security. 515-530.

Yingnong Dang, Rongxin Wu, Hongyu Zhang, Dongmei Zhang, and Peter Nobel. 2012. Rebucket: A method for clustering duplicate crash reports based on call stack similarity. In 2012 34th International Conference on Software Engineering (ICSE). IEEE, 1084-1093.

Debian. 2013. Reporting 1.2K crashes. https://lists.debian.org/debian-devel/2013/06/msg00720.html.Accessed: May 17, 2023.

Andrea Fioraldi, Dominik Maier, Heiko Eißfeldt, and Marc Heuse. 2020. AFL++ combining incremental steps of fuzzing research. In Proceedings of the 14th USENIX Conference on Offensive Technologies. 10-10.

Shuitao Gan, Chao Zhang, Peng Chen, Bodong Zhao, Xiaojun Qin, Dong Wu, and Zuoning Chen. 2020. GREYONE: Data Flow Sensitive Fuzzing.. In USENIX Security Symposium. 2577-2594.

Shuitao Gan, Chao Zhang, Xiaojun Qin, Xuwen Tu, Kang Li, Zhongyu Pei, and Zuoning Chen. 2018. Collafl: Path sensitive fuzzing. In 2018 IEEE Symposium on Security and Privacy (SP). IEEE, 679-696.

Elia Geretto, Cristiano Giuffrida, Herbert Bos, and Erik Van Der Kouwe. 2022. Snappy: Efficient Fuzzing with Adaptive and Mutable Snapshots. In Proceedings of the 38th Annual Computer Security Applications Conference. 375-387.

Google. 2022. FuzzBench: 2022-09-29 report. https://www.fuzzbench.com/reports/2022-09-29/index.html.Accessed: May 17, 2023.

Google. 2023. OSS-Fuzz. https://google.github.io/oss-fuzz/.Accessed: May 17, 2023.

Yuki Koike, Hiroyuki Katsura, Hiromu Yakura, and Yuma Kurogome. 2022. SLOPT: Bandit Optimization Framework for Mutation-Based Fuzzing. In Proceedings of the 38th Annual Computer Security Applications Conference. 519-533.

Gwangmu Lee, Woochul Shim, and Byoungyoung Lee. 2021. Constraint-guided Directed Greybox Fuzzing.. In USENIX Security Symposium. 3559-3576.

Caroline Lemieux and Koushik Sen. 2018. Fairfuzz: A targeted mutation strategy for increasing greybox fuzz testing coverage. In Proceedings of the 33rd ACM/IEEE International Conference on Automated Software Engineering. 475-485.

Jie Liang, Yu Jiang, Mingzhe Wang, Xun Jiao, Yuanliang Chen, Houbing Song, and Kim-Kwang Raymond Choo. 2019. Deepfuzzer: Accelerated deep greybox fuzzing. IEEE Transactions on Dependable and Secure Computing 18, 6 (2019), 2675-2688.

Jie Liang, Mingzhe Wang, Chijin Zhou, Zhiyong Wu, Yu Jiang, Jianzhong Liu, Zhe Liu, and Jiaguang Sun. 2022. PATA: Fuzzing with path aware taint analysis. In 2022 IEEE Symposium on Security and Privacy (SP). IEEE, 1-17.

Chung Laung Liu and James W Layland. 1973. Scheduling algorithms for multiprogramming in a hard-real-time environment. Journal of the ACM (JACM) 20, 1 (1973), 46-61.

lwn.net. 2013. Mayhem finds 1200 bugs. https://lwn.net/Articles/557055/.Accessed: May 17, 2023.

Chenyang Lyu, Shouling Ji, Chao Zhang, Yuwei Li, Wei-Han Lee, Yu Song, and Raheem Beyah. 2019. MOPT: Optimized Mutation Scheduling for Fuzzers.. In USENIX Security Symposium. 1949-1966.

Chenyang Lyu, Shouling Ji, Xuhong Zhang, Hong Liang, Binbin Zhao, Kangjie Lu, and Raheem Beyah. 2022. Ems: History-driven mutation for coverage-based fuzzing. In 29rd Annual Network and Distributed System Security Symposium, NDSS. 24-28.

Jonathan Metzman, László Szekeres, Laurent Simon, Read Sprabery, and Abhishek Arya. 2021. Fuzzbench: an open fuzzer benchmarking platform and service. In Proceedings of the 29th ACM joint meeting on European software engineering conference and symposium on the foundations of software engineering. 1393-1403.

Stefan Nagy, Anh Nguyen-Tuong, Jason D Hiser, Jack W Davidson, and Matthew Hicks. 2021a. Breaking through binaries: Compiler-quality instrumentation for better binary-only fuzzing. In 30th USENIX Security Symposium.

Stefan Nagy, Anh Nguyen-Tuong, Jason D Hiser, Jack W Davidson, and Matthew Hicks. 2021b. Same Coverage, Less Bloat: Accelerating Binary-only Fuzzing with Coverage-preserving Coverage-guided Tracing. In Proceedings of the 2021 ACM SIGSAC Conference on Computer and Communications Security. 351-365.

Nicholas Nethercote and Julian Seward. 2007. Valgrind: a framework for heavyweight dynamic binary instrumentation. ACM Sigplan notices 42, 6 (2007), 89-100.

Manh-Dung Nguyen, Sébastien Bardin, Richard Bonichon, Roland Groz, and Matthieu Lemerre. 2020. Binary-level Directed Fuzzing for Use-After-Free Vulnerabilities.. In RAID. 47-62.

Ivica Nikolić, Radu Mantu, Shiqi Shen, and Prateek Saxena. 2021. Refined grey-box fuzzing with Sivo. In Detection of Intrusions and Malware, and Vulnerability Assessment: 18th International Conference, DIMVA 2021, Virtual Event, July 14-16, 2021, Proceedings 18. Springer, 106-129.

Sebastian Österlund, Kaveh Razavi, Herbert Bos, and Cristiano Giuffrida. 2020. Parmesan: Sanitizer-guided greybox fuzzing. In Proceedings of the 29th USENIX Conference on Security Symposium. 2289-2306.

James L Peterson and Abraham Silberschatz. 1985. Operating system concepts. Addison-Wesley Longman Publishing Co., Inc.

Sanjay Rawat, Vivek Jain, Ashish Kumar, Lucian Cojocar, Cristiano Giuffrida, and Herbert Bos. 2017. Vuzzer: Application-aware evolutionary fuzzing.. In NDSS, Vol. 17. 1-14.

Kostya Serebryany. 2017. OSS-Fuzz-Google's continuous fuzzing service for open source software. In USENIX Security symposium. USENIX Association.

Abhishek Shah, Dongdong She, Samanway Sadhu, Krish Singal, Peter Coffman, and Suman Jana. 2022. MC2: Rigorous and Efficient Directed Greybox Fuzzing. In Proceedings of the 2022 ACM SIGSAC Conference on Computer and Communications Security. 2595-2609.

Dongdong She, Abhishek Shah, and Suman Jana. 2022. Effective seed scheduling for fuzzing with graph centrality analysis. In 2022 IEEE Symposium on Security and Privacy (SP). IEEE, 2194-2211.

Aleksandrs Slivkins et al. 2019. Introduction to multi-armed bandits. Foundations and Trends® in Machine Learning 12, 1-2 (2019), 1-286.

Nick Stephens, John Grosen, Christopher Salls, Andrew Dutcher, Ruoyu Wang, Jacopo Corbetta, Yan Shoshitaishvili, Christopher Kruegel, and Giovanni Vigna. 2016. Driller: Augmenting fuzzing through selective symbolic execution.. In NDSS, Vol. 16. 1-16.

Richard S Sutton and Andrew G Barto. 2018. Reinforcement learning: An introduction. MIT press.

Robert Swiecki et al. 2021. Honggfuzz-Security oriented software fuzzer.

Dawei Wang, Ying Li, Zhiyu Zhang, and Kai Chen. [n. d.]. CarpetFuzz: Automatic Program Option Constraint Extraction from Documentation for Fuzzing. ([n. d.]).

Yuanping Yu, Xiangkun Jia, Yuwei Liu, Yanhao Wang, Qian Sang, Chao Zhang, and Purui Su. 2022. HTFuzz: Heap Operation Sequence Sensitive Fuzzing. In Proceedings of the 37th IEEE/ACM International Conference on Automated Software Engineering. 1-13.

Tai Yue, Pengfei Wang, Yong Tang, Enze Wang, Bo Yu, Kai Lu, and Xu Zhou. 2020. Ecofuzz: Adaptive energy-saving greybox fuzzing as a variant of the adversarial multi-armed bandit. In Proceedings of the 29th USENIX Conference on Security Symposium. 2307-2324.

Insu Yun, Sangho Lee, Meng Xu, Yeongjin Jang, and Taesoo Kim. 2018. \{QSYM\}: A practical concolic execution engine tailored for hybrid fuzzing. In 27th \{USENIX\} Security Symposium (\{USENIX\} Security 18). 745-761.

Michal Zalewski. 2017. American fuzzy lop (AFL) fuzzer.

Lei Zhao, Pengcheng Cao, Yue Duan, Heng Yin, and Jifeng Xuan. 2020. Probabilistic path prioritization for hybrid fuzzing. IEEE Transactions on Dependable and Secure Computing 19, 3 (2020), 1955-1973.

Shunfan Zhou, Zhemin Yang, Dan Qiao, Peng Liu, Min Yang, Zhe Wang, and Chenggang Wu. 2022. Ferry: \{State-Aware\} Symbolic Execution for Exploring \{State-Dependent\} Program Paths. In 31st USENIX Security Symposium (USENIX Security 22). 4365-4382.

Xiaogang Zhu and Marcel Böhme. 2021. Regression greybox fuzzing. In Proceedings of the 2021 ACM SIGSAC Conference on Computer and Communications Security. 2169-2182.

Peiyuan Zong, Tao Lv, Dawei Wang, Zizhuang Deng, Ruigang Liang, and Kai Chen. 2020. Fuzzguard: Filtering out unreachable inputs in directed grey-box fuzzing through deep learning. In Proceedings of the 29th USENIX Conference on Security Symposium. 2255-2269.