# CrossFire: Fuzzing macOS Cross-XPU Memory on Apple Silicon

Jiaxun Zhu*

sevenswords@zju.edu.cn

Zhejiang University

Hangzhou, China

Minghao Lin*

minghaolin2000@zju.edu.cn

Zhejiang University

Hangzhou, China

Tingting Yin

yintt@mail.zgclab.edu.cn

Zhongguancun Laboratory

Beijing, China

Zechao Cai

zechao.cai@columbia.edu

Columbia University

New York, USA

Yu Wang

yu.wang@cyberserval.cn

Cyberserval Co., Ltd

Hangzhou, China

Rui Chang

crix1021@zju.edu.cn

Zhejiang University

Hangzhou, China

Wenbo Shen†

shenwenbo@zju.edu.cn

Zhejiang University

Hangzhou, China

## Abstract

Modern computing systems increasingly utilize XPUs, such as GPUs and NPUs, for specialized computation tasks. While these XPUs provide critical functionalities, their security protections are generally weaker than those of CPUs, making them attractive attack targets. In particular, Apple silicon optimizes memory usage by adopting a unified memory architecture (UMA), which employs shared memory regions (termed cross-XPU memory) to facilitate communication between CPUs and XPUs. Although the cross-XPU memory enhances performance, it also introduces a new attack surface. Unfortunately, the difficulty in identifying effective shared memory regions and generating valid payloads makes fuzzing cross-XPU memory a challenging problem that cannot be resolved effectively by existing fuzzing techniques.

Therefore, we propose CrossFire, the first fuzzer targeting Apple silicon XPU by fuzzing cross-XPU memory, to evaluate this new attack surface. Initially, we conduct an in-depth cross-XPU memory analysis to investigate the challenges of fuzzing XPU. To address these challenges, CrossFire introduces two novel techniques to pinpoint effective fuzzing regions in cross-XPU memory and trace kernel execution information to extract data constraints. Leveraging these techniques, we develop CrossFire based on the m1n1 hypervisor to monitor cross-XPU memory accesses and perform grey-box hooking-based fuzzing. We further evaluate CrossFire on macOS Ventura, where it has identified 15 new zero-day bugs, 8 of which have been confirmed by Apple.

## CCS CONCEPTS

- Security and privacy $\rightarrow$ Operating systems security.

## KEYWORDS

macOS, XPU Fuzzing, Cross-XPU Memory

## ACM Reference Format:

Jiaxun Zhu, Minghao Lin, Tingting Yin, Zechao Cai, Yu Wang, Rui Chang, and Wenbo Shen. 2024. CrossFire: Fuzzing macOS Cross-XPU Memory on Apple Silicon. In Proceedings of the 2024 ACM SIGSAC Conference on Computer and Communications Security (CCS '24), October 14-18, 2024, Salt Lake City, UT, USA. ACM, New York, NY, USA, 14 pages. https://doi.org/ 10.1145/3658644.3690376

## 1 INTRODUCTION

Modern computing systems widely adopt XPUs (e.g., GPU and NPU) for specialized computation, which improves performance but also imports new attack surfaces. Compared with the traditional CPU memory vulnerabilities, the XPU memory vulnerabilities have higher risks because the firmware running on XPUs generally does not have modern mitigations [11] like ASLR, Pointer Authentication [12], and Memory Protection Key [45], etc. The unified memory architecture introduced by Apple silicon further exacerbates this problem, where macOS employs shared memory regions (cross-XPU memory) to facilitate communication between CPUs and XPUs. On Apple silicon, XPUs and CPU can read and write the same memory region physically, which makes it possible for attackers to hijack the XPU firmware and further compromise the code running on the CPU. More and more real-world attacks $\left\lbrack  {{20},{24},{28}}\right\rbrack$ are showing that XPU vulnerabilities are becoming serious threats to AppleOSs.

Fuzzing is one of the most effective ways to find vulnerabilities. However, existing fuzzing works targeting userspace programs or kernels on the CPU can not effectively test the XPU firmware through cross-XPU memory. Although some kernel/driver fuzzers may change cross-XPU memory status occasionally, they can hardly test the related syscalls sufficiently and mutate the memory content effectively. None of the state-of-the-art macOS kernel fuzzers, such as IMF [22], SyzGen [13], or KextFuzz [44], have reported any XPU firmware bugs according to their experiments. Due to the following challenges, they are not applicable for evaluating this new attack surface.

---

*Equal contribution.

${}^{ \dagger  }$ Corresponding author.

---

First, the fuzzing payloads need to be injected into the effective region inside the cross-XPU memory under a properly established communication context. CrossFire has conducted a deep dive into real-world macOS cross-XPU memory operations to understand cross-XPU memory usage. We find that while CPU and XPU usually allocate large cross-XPU memory, only a limited part of them are effective for fuzzing. This is because even storing a small amount of effective data in cross-XPU memory requires allocating an entire page. Mutating the other parts would lead to a significant waste of computing resources.

Second, only the payloads that pass kernel checks would be further submitted to the XPU. CrossFire assumes the adversary can only execute code in userspace with common user privileges. The userspace programs should send the cross-XPU memory payloads to the kernel extension (kext) first. Generating legal cross-XPU memory data is more challenging than generating general kernel inputs. 1) The size of cross-XPU memory inputs is much larger, and the structure is more complex. Among the applications we have investigated, cross-XPU memory is $0\mathrm{x}5\mathrm{\;b}3\mathrm{f}{60}$ bytes on average. 2) The kernel checks payloads strictly. In a simple case, we have manually reverse-engineered, 0x130 bytes out of 0xa20 bytes (the size of the effective region) are checked. 3) The kernel code containing cross-XPU memory data checks is hard to locate. Although the kernel parses most of the inputs at syscall entries, the cross-XPU memory data is only retained and checked after other preparation works. 4) The macOS kexts are closed-source and developed in C++, which makes most of the existing static and dynamic analysis methods ineffective.

Third, it's hard to comprehensively intercept the macOS execution to analyze real cross-XPU memory operations. As the macOS kernel and firmware are closed-source, source code instrumentation is impossible. Additionally, the hardware trace mechanism, Core-Sight [31], is not available on Apple silicon. KextFuzz [44] proposed a binary rewriting method that instruments kext binaries, which is limited to locations of these unnecessary instructions, resulting in only about ${30}\%$ of basic blocks being instrumented according to their experiment. This method is insufficient for our requirements.

In this paper, we propose CrossFire, the first fuzzer targeting Apple silicon XPU by fuzzing cross-XPU memory to evaluate this new attack surface. To address the aforementioned challenges, CrossFire emulates real cross-XPU memory operations during fuzzing and generates moderately malformed data before processing in kernel. This approach allows it to emulate valid XPU requests, enabling thorough testing of the related code.

To address Challenge-1, CrossFire proposed Hypervisor-based Effective Cross-XPU Memory Pinpoint (MemPin) to pinpoint the effective region by monitoring the cross-XPU memory accesses based on the hypervisor. More specifically, MemPin monitors the access of the physical address of the cross-XPU memory by manipulating the stage-2 translation table entry. Based on this, MemPin can monitor the memory operations to the cross-XPU memory in both user and kernel spaces to obtain the effective fuzzing region. To overcome Challenge-2, CrossFire proposed a lightweight method named XPU-Targeted Data Constraints Extraction (ConEx-tract) to extract cross-XPU memory data constraints based on real kernel execution traces and taint analysis. ConExtract leverages the symbolic execution engine Triton [35] to implement a symbolic taint analysis for cross-XPU memory. The symbolic taint analysis symbolizes the cross-XPU memory, then starts taint analysis from the entry provided by MemPin based on the execution trace. Finally, it can identify bytes of cross-XPU memory that are used to check in the kernel as data constraints, guiding further mutation. To solve Challenge-3, CrossFire proposed a comprehensive instrumentation technique to intercept the execution of macOS based on a customized hypervisor. Initially, CrossFire instruments the macOS kernel with hypervisor calls in every basic block of the targeted kexts. Then with the trace handling provided by ConExtract, CrossFire can intercept macOS kernel execution comprehensively.

CrossFire performs hooking-based fuzzing with the understanding of the effective region and data constraints of cross-XPU memory, in which way most of the variables are legal, and the payload can be submitted to XPU. To enhance the success rate of cross-XPU memory submissions, CrossFire also instruments the kernel code responsible for parsing this memory data to monitor coverage. This instrumentation sends signals back to the fuzzer when the payload successfully passes the kernel's checks.

We implemented a prototype of CrossFire based on the m1n1 project [10] and evaluated it on macOS Ventura 13.5.2 (22G91) with real Apple-Silicon-based Mac devices. MemPin has successfully identified effective cross-XPU memory by monitoring cross-XPU memory access, and it reduces an average of ${87.3}\%$ of the ineffective mutation space. With the lightweight data constraints collection method, ConExtract increases the cross-XPU memory submitting rate by 83.9%. The hypervisor-based instrumentation technique can instrument ${100}\%$ basic blocks in Apple silicon kernel, which outperforms the existing work [44]. Each part of the CrossFire has shown significant improvements in bug finding. We use CrossFire to test GPU and NPU on macOS and find 15 previously unknown bugs. All bugs have been reported to the Apple.

In summary, our contributions are as follows.

- We conduct an in-depth macOS cross-XPU memory analysis for the first time, revealing how it functions and proposing an effective testing strategy.

- We propose two novel techniques, hypervisor-based effective cross- ${XPU}$ memory pinpoint and ${XPU}$ -targeted data constraints extraction, to pinpoint effective fuzzing regions and extract data constraints for XPU fuzzing.

- We implement a prototype of CrossFire designed to monitor ma-cOS execution with fine granularity, enabling access monitoring, achieving comprehensive coverage collection, and facilitating hook-based cross-XPU memory fuzzing. We release the source code of the prototype in https://github.com/ZJU-SEC/CrossFire.

- We evaluate CrossFire on real-world macOS and found 15 zero-day bugs, 8 of which have been confirmed by Apple.

## 2 BACKGROUND

### 2.1 XPU-based Memory Attack

Apple's CPU memory safety mechanisms continue to improve, and attacking the OS on the CPU has become much harder than before. Therefore, XPU-based attacks have become one of the most significant threats to Apple OSes. Particularly, Apple silicon stands out for its powerful memory safety mechanisms, notable for PAC (Pointer Authentication) [12] and PPL (Page Protection Layer) [5]. These unique features shield kernel-sensitive data, even in the face of attackers who have already gained kernel read/write primitives through traditional CPU memory corruption vulnerabilities (such as vulnerabilities found in drivers or kernel extensions).

![0196f770-223e-7413-929e-685007fc2cc7_2_151_241_716_271_0.jpg](images/0196f770-223e-7413-929e-685007fc2cc7_2_151_241_716_271_0.jpg)

Figure 1: Cross-XPU memory structure.

Modern computing systems like Apple silicon widely use XPUs. For instance, Apple introduced a GPU called Apple Graphics (AGX) and an NPU called Apple Neural Engine (ANE) for specialized computation with Apple silicon. Additionally, to optimize the memory usage between CPU and XPUs, Apple introduced the unified memory architecture (UMA), which unifies the memory of the CPU and XPU into a single physical memory space [23].

However, while the UMA improves overall performance, it also creates a new attack surface because the XPU can access the same memory as the CPU but with its own access control mechanism. Furthermore, the security mechanisms on XPU are less potent than those on the CPU [11]. As a result, attackers are increasingly targeting the XPU. By exploiting the XPU and accessing memory via XPU, attackers can bypass all the powerful CPU memory safety mechanisms to exploit the kernel running on the CPU [39].

### 2.2 Interacting with XPUs

The XPU is widely used in many important application domains beyond scientific computing, including machine learning, graph processing, and computer vision in the macOS on Apple silicon [3]. Moreover, Apple silicon uses a unified memory model for CPU and GPU, which means CPU and GPU always use shared memory for data passing [2]. Similarly, other XPUs also adopt this shared memory mechanism. Therefore, these cross-XPU shared memories are ideal targets for identifying weaknesses in the data handling on the XPU side. Compared with the normal programs running on the CPU, the firmware code running on XPUs is usually less tested and more fragile $\left\lbrack  {{20},{24},{28}}\right\rbrack$ . Due to the UMA and XPU’s lack of advanced mitigations (PAC, ASLR, etc.), Lina [28] was able to more readily exploit the GPU to map critical memory from the CPU side and further corrupt it to achieve privilege escalation. Kaspersky [24] found an undocumented hardware feature of the GPU used in a full-chain exploitation. This hardware feature allows the GPU to directly access memory on the CPU side, enabling the exploitation to bypass the PPL [5]. Furthermore, Fröder [20] jailbroke iOS 16 based on Kaspersky's findings. Unfortunately, to the best of our knowledge, no existing works systematically study these cross-XPU shared memories.

A userspace attacker can exploit XPU and further corrupt macOS by corrupting cross-XPU memory. As shown in Figure 1, there are two types of cross-XPU memory, i.e. Type1: memory shared between XPU and CPU's userspace code, and Type2: memory shared between CPU userspace code and CPU kernel code for further XPU processing. The two types of memory are defined as Tagged Memory, as they are assigned shared tags that help CrossFire identify them. The Type 2 memory is fetched in the kernel using reference ID as the target chunk.

![0196f770-223e-7413-929e-685007fc2cc7_2_925_235_723_510_0.jpg](images/0196f770-223e-7413-929e-685007fc2cc7_2_925_235_723_510_0.jpg)

Figure 2: An example of the usage of Type2 cross-XPU memory in the macOS on Apple silicon.

Documents describing the usages of Type1 and Type2 memory are very limited. Through reverse engineering and conducting experiments, we have two key findings. First, from a functionality perspective, Type2 memory is much more important than Type1 memory. Type1 memory only contains computation-related data processed by XPU userspace, including inputs/outputs of computations and XPU userspace virtual addresses pointing to computation resources such as vertices, fragments, etc. In contrast, Type2 memory contains control-related data (commands) processed by the XPU kernel. Second, from the attack surface perspective, Type 2 memory is more valuable than Type1 memory. There are two layers of isolation in XPU: user-kernel and XPU-CPU isolation (XPU side GXF [37]). In particular, the XPU side GXF isolates the XPU from the OS on the CPU side, preventing the XPU from directly accessing OS memory. As a result, the XPU kernel needs to call the GXF to access the OS's memory. Due to the two-layer isolation, Type1 memory needs to cross two layers of isolation to attack the OS on the CPU side, which is a smaller attack surface. However, Type2 memory directly affects the control flow of the XPU kernel, which makes it easier to bypass those isolations, leading to a larger attack surface. CrossFire can identify the Type 1 memory from the memories just as it does with Type2, enabling it also to be compatible with fuzzing Type1 memory. We focus on the Type2 cross-XPU memory in this paper since Type 1 memory is overall not a valuable fuzzing target based on the above findings.

In Type2 cross-XPU memory, the userspace program first writes the payloads into userspace-kernel shared memory, and then, the kernel fills the kernel-XPU shared memory according to the user's request. The cross-XPU memory data processing progress is shown in Figure 2. ① Userspace program allocates userspace-kernel shared memory and gets the memory identifier id and the target chunk umem0. ② The program prepares the valid data. ③ The program raises a request to the kernel by syscall or driver calls. The id will be sent as part of the syscall parameters. ④ Kernel retains the shared memory target chunk kmem0 by id. kmem0 and umem0 share the same physical memory region. ⑤ Kernel checks the memory data. ⑥ Kernel fills kernel-XPU shared memory kmem1 according to the kmem0 contents. ⑦ Kernel submits the memory to XPU. ⑧ XPU processes the shared memory kmem1. Additionally, the APIs shown in Figure 2 are simplified flows when the cross-XPU memory has been prepared by the CPU and is about to notify the XPU to process. In real cases, they are not independent functions/APIs and different XPUs implement them in different ways, which is hard to identify. These APIs for a normal XPU interaction are not enough to fill all the needed cross-XPU memory or meet the other requirements for asynchronous communication.

Each cross-XPU memory chunk usually contains several memory pages, but only part of the memory is effective for fuzzing. One of the reasons is that cross-XPU memory is allocated as page-aligned, even if the content written does not fill an entire page, the system allocates a full page to accommodate it, leading to substantial unused space in cross-XPU memory. We monitored the macOS memory accessing operations and reverse-engineered the implementation of macOS kexts, and userspace programs and kexts only read and write about ${12.7}\%$ of the target chunk regions. We refer to cross-XPU memory regions that are actually used as effective fuzzing regions in this paper. Mutating other parts of the target chunk would bring a huge waste of computing power and significantly reduce the ability of fuzzing.

Kernel reads and checks the contents of effective regions according to the user-requested XPU operations before submitting. These checks protect the system but also make cross-XPU memory more challenging work. We mark the bytes being checked as immutable bytes. The immutable bytes have to meet specific data constraints in different XPU interaction contexts. Mutating immutable bytes during fuzzing will make the testcases aborted by the kernel or even block the fuzzing process.

### 2.3 Virtualization on Apple Silicon

Virtualization technology has become a crucial aspect of computing systems. On Apple M1, Apple has implemented the VHE (Virtu-alization Host Extension) [7] feature for implementing the type-2 hypervisor. Meanwhile, the virtual machine running on EL1 has an independent physical address or IPA (Intermediate Physical Address) space. When accessing memory, the physical address of the Virtual Machine running on EL1 (IPA) must go through a stage-2 address translation managed by the hypervisor running on EL2 to obtain the actual physical address for accessing memory, also called HPA (Host Physical Address) or PA.

Currently, several existing VM tools including Parallels Desktop [15], UTM [38], etc., support initiating virtual machines on macOS devices with Apple silicon chips. However, the functionality of this support is notably limited. This limitation stems from the foundation of these tools, which are implemented based on the Hypervisor Framework [4] provided by Apple. The Hypervisor Framework APIs fall short in providing assistance for analyzing XPU interactions as they do not provide interfaces for developers to access CPU features for hardware virtualization directly. Even though it is possible to load kernel extensions into macOS, the hardware-enforced GXF (Guarded Exception Level) feature [36] restricts access to crucial CPU features, such as hardware virtualiza-tion support and exception handling. Consequently, it's impossible to implement a customized hypervisor such as DigTool [32] on ma-cOS. Moreover, currently, there is no support for the XPU firmware simulation of Apple silicon, so we need to interact with real XPUs to fuzz the XPU firmware.

Therefore, to monitor the XPU interactions, we choose to run a customized hypervisor based on m1n1 [10] on EL2 and run a macOS VM on EL1, which can interact with real XPUs. The running of the $\mathrm{m}1\mathrm{n}1$ hypervisor takes two PCs, the target PC and server PC, to enable the virtualization of macOS on Apple silicon. The target PC is responsible for running the guest macOS, and the server PC manages it. The hypervisor itself is located in EL2 of the target PC and is in charge of handling common exceptions like the hypervisor call and the stage 2 translation fault, which are caused by the invalid stage 2 translation table entry. The manager on the server PC enhances control over the guest, enabling commands to be sent to the target PC and facilitating communication with the guest macOS via the serial port.

## 3 CHALLENGES

In this section, we use a real-world cross-XPU bug identified by CrossFire as an example to illustrate the technical challenges of fuzzing macOS cross-XPU memory.

Figure 3 is a simplified pseudo-code snippet extracted from the kernel extension from Apple AGXG13G (Apple Graphics for GPU G13G) kernel extension. The userspace program allocates a chunk of cross-XPU memory and then sets the payloads in cross-XPU memory. This chunk is indexed by the id in both user and kernel spaces. The function parse_mem_bufs (line 1) is responsible for retaining the virtual address (mem0) of the payloads sent from userspace through the id of cross-XPU memory. The function parse_mem (line 14) checks the cross-XPU memory data and further submits it to the XPU. parse_mem_bufs invokes parse_mem through an indirect call.

The payloads triggering a crash in the GPU should meet at least two requirements. First, the payload should be written to the effective region of the correct cross-XPU memory chunk, i.e. the memory chunk referenced by id in the current execution context (line 4). Second, the payload needs to pass the extensive data checks in parse_mem (lines 16-18). Note that the checks in AGXG13G are much more complex, the code snippet only shows two examples. Such XPU firmware bugs are hard to find by existing fuzzing techniques due to the following three challenges.

Challenge-1: hard to pinpoint the effective region within the target chunk. Writing fuzzing payloads to effective regions within the correct cross-XPU memory (target chunk) can significantly enhance fuzzing efficiency. To achieve this, we need to complete two steps: identifying the target chunk in the current communication context and pinpointing the effective regions within that chunk. However, these steps are challenging to accomplish.

For identifying the target chunk, although we can identify all the cross-XPU memory chunks by viewing virtual memory area

![0196f770-223e-7413-929e-685007fc2cc7_4_150_238_723_915_0.jpg](images/0196f770-223e-7413-929e-685007fc2cc7_4_150_238_723_915_0.jpg)

Figure 3: An example of GPU crash. The functions in this pseudocode are located by CrossFire. Originally, no information about the processing of cross-XPU memory was publicly available. The GPU crashes due to the malformed cross-XPU memory provided by CrossFire.

(VMA) tags, it is still hard to locate the memory chunk used in the current communication context. First, rather than using system standard APIs, each kext implements its own cross-XPU memory allocation interfaces to allocate shared user-kernel shared memory (kmem0) and kernel-XPU shared memory (kmem1), making it hard to locate memory allocation operations with static analysis. Second, VMA tags mark out all the cross-XPU memory chunks, but only the specific target chunk used in the current CPU and XPU interaction context is the focus of fuzzing. Userspace programs and kernel use an int32 variable id to identify the target chunk. Userspace programs send id as part of the syscall arguments to the kernel. This int32 id does not have special characteristics, making it hard to distinguish those from other int32 variables in reverse engineering.

To pinpoint the effective regions within target chunk, as discussed in §2.2, CrossFire must first locate the target chunk. Subsequently, CrossFire proceeds to identify the specific fuzzing regions that are considered effective. These effective regions are found as those written by userspace programs and read by the kernel. Only the payload in these regions will finally be submitted to the XPU. Mutating the large unused regions in cross-XPU memory chunks will lead to significant computing resource waste. Meanwhile, the effective region changes according to different cross-XPU memory requests, even for the same userspace and kext communication process. However, none of the userspace programs and kexts mark the effective fuzzing region out.

Challenge-2: hard to pass kernel cross-XPU memory data checks and locate the parsing code. All the cross-XPU memory payloads must pass the extensive kernel checks before being submitted to the XPU. As shown in Figure 3 as a simplified example, data at specific effective region offsets must undergo validity checks. In one of the cross-XPU memory requests we have reverse-engineered with manual work, there are $0 \times  {130}$ bytes being checked out of the effective region which contains 0xa20 bytes in total. Even if the fuzzing test case fails to pass one of the checks, the payload will be discarded. Most of the checks are strict (e.g. be equal to magic value), mutating these bytes would make most of the test cases invalid. Therefore, we refer to the checked data in the effective fuzzing region as immutable bytes.

In addition to the complexity of the cross-XPU memory data, it is hard to locate the cross-XPU memory parse code in large and highly complex XPU-related kernels due to the unique way the kernel processes cross-XPU memory data. Unlike most of the syscall or driver call arguments, which are usually parsed at the syscall entries, cross-XPU memory is parsed when other preparation works have been done. The related code is usually deep in the control flow. Meanwhile, the macOS kexts are closed-sourced and developed in C++, widely using indirect calls. Besides, the cross-XPU memory retain APIs and cross-XPU memory submitting APIs are customized by the kexts. In the case shown in Figure 3, it took us a man-week to reverse the cross-XPU memory operations in kext and find parse_mem_bufs and parse_mem functions. Therefore, locating where to parse cross-XPU memory in the kernel is challenging, which is indispensable for providing the starting point for further program analysis.

Existing interface-aware fuzzing works can not analyze the cross-XPU memory data constraints effectively. The state-of-the-art ma-cOS kernel fuzzing work SyzGen [13] leverages concolic symbolic execution to analyze the checks for driver arguments in the closed-source macOS kernel. This method cannot be applied to cross-XPU memory for two main reasons.

First, SyzGen employs concolic symbolic execution to explore the code path starting from predefined driver call entries. However, as mentioned above, locating cross-XPU memory parse code is challenging, which leads to the start point of concolic symbolic execution being missing in this scenario. Second, the cross-XPU memory size is much larger than the common driver arguments and the data checks are more complex, making constraint solving extremely hard.

Challenge-3: difficulty in intercepting the execution of macOS comprehensively. To analyze real cross-XPU memory operations, CrossFire needs to comprehensively intercept macOS execution at a low level. For example, CrossFire needs to trace kernel execution information, and instrument the code to collect coverage.

However, macOS kernel and firmware are closed-source, making source code instrumentation impossible. To solve this problem, SyzGen [13] uses the breakpoint provided by the macOS kernel debugger to record coverage, but this feature is unavailable on macOS running on Apple silicon. Moreover, CoreSight [31], the hardware trace mechanism provided by ARM, is also not available on Apple silicon. The m1n1 hypervisor only offers limited support for hardware breakpoints on macOS running on Apple silicon [1]. To overcome these limitations, KextFuzz [44] proposed a binary rewriting method, instrumenting kext binaries by replacing useless instructions in fuzzing, e.g., the pointer authentication and nop instructions. The instrumentation points of KextFuzz are highly limited to useless instruction locations. According to their experiment, only about ${30}\%$ of basic blocks can be instrumented. This strategy can hardly meet our analysis requirements.

![0196f770-223e-7413-929e-685007fc2cc7_5_154_238_717_403_0.jpg](images/0196f770-223e-7413-929e-685007fc2cc7_5_154_238_717_403_0.jpg)

Figure 4: Workflow of CrossFire. CrossFire pinpoints the effective fuzzing region and collects the cross-XPU memory data constraints in preparation and then injects fuzzing payload into real XPU interactions.

## 4 CROSSFIRE DESIGN

To address the above challenges and enable effective fuzzing of XPU, we developed CrossFire -a fuzzer designed to optimize XPU fuzzing with minimal reverse engineering efforts. This design involves dividing the fuzzer into two main phases: preparation and fuzzing. The preparation phase is pivotal for collecting essential information that guides the subsequent fuzzing phase, thereby reducing the need for labor-intensive reverse engineering.

Figure 4 illustrates the architecture of CrossFire. In the preparation phase, CrossFire runs real-world seed applications while leveraging two novel techniques to collect essential information. Specifically, to resolve Challenge-1 and obtain the effective region of the cross-XPU memory, we propose a novel technique named hypervisor-based effective cross-XPU memory pinpoint (MemPin), which can get byte-grained effective cross-XPU memory without reverse-engineer the proprietary XPU libraries (§4.1). Moreover, to overcome Challenge-2 and guide the fuzzer to pass the payload to the XPU, we propose another technique named XPU-targeted data constraint extraction (ConExtract), which extracts data constraints of the cross-XPU memory from recorded traces to increase the success rate of test cases reaching the XPU (§4.2). Besides, based on the traces, we can extract the full block coverage of the executed code to monitor the submission status of the mutated cross-XPU memory, thereby incidentally solving Challenge-3. In the hooking-based grey-box fuzzing phase, CrossFire continues to run these seed applications, and then utilizes a mutator for pinpointing and mutating the effective mutation areas within cross-XPU memory based on previously collected effective region and data constraints. Ultimately, the mutated cross-XPU memory, featuring effective mutations, is sent to the XPU for processing in a submission-aware manner (§4.3).

![0196f770-223e-7413-929e-685007fc2cc7_5_928_235_713_309_0.jpg](images/0196f770-223e-7413-929e-685007fc2cc7_5_928_235_713_309_0.jpg)

Figure 5: The steps to get the target chunk and its effective fuzzing region that is being processed by the kernel.

### 4.1 Hypervisor-based Effective Cross-XPU Memory Pinpoint

To address Challenge-1, this technique MemPin is proposed to identify the target chunk referenced in the closed-source kernel extension and then pinpoint the effective fuzzing region within the target chunk.

MemPin is inspired by the concept where the same physical address connects the entire tagged cross-XPU memory ranges and the target chunk. Specifically, the target chunk is hard to locate due to the semantics gap arising from the indirect dereferencing and closed-source nature of AppleOSs. The whole sets of the tagged cross-XPU memory can be easily obtained by the VMA tags of share mode in user space. Moreover, the cross-XPU memory uses the same physical memory in both user and kernel spaces. Therefore, the memory operations of the target chunk can be intercepted through the same physical address provided by the tagged memory.

For example, as shown in Figure 5, 1) we can find all allocated cross-XPU memory ranges MEM0, MEM1, and MEM2, which are easily obtained by the VMA tags. If we translate the three ranges into physical addresses and set them to be monitored in MemPin. 2) We can get all the user write regions on these three MEMs. 3) Afterward, the kernel will trap to EL2 when accessing the monitored cross-XPU memory referenced by ID, which means the accessed cross-XPU memory is the target chunk. Through the user write region provided by step 2, we can pinpoint the effective region for fuzzing.

To implement this idea of monitoring, which hooks the memory access of the cross-XPU memory, the memory access breakpoint provided by ARM [9] is not suitable, for it is designed to hook the memory access of the virtual address rather than the physical address. Therefore, MemPin is designed to persistently monitor and record the memory on accessing the physical address of cross-XPU memory based on virtualization. By manipulating the stage 2 translation table, and invalidating the entry of the cross-XPU memory, MemPin can monitor the physical memory access of the cross-XPU memory. This monitoring of physical address bypasses the challenge of indirect dereferencing of the cross-XPU memory in both user and kernel spaces. Further, MemPin directly gets the effective parts indicated by the user-write operations.

Figure 6: Workflow of Hypervisor-based Effective Cross-XPU Memory Pinpoint. The memory access instruction is intercepted

![0196f770-223e-7413-929e-685007fc2cc7_6_151_234_1502_481_0.jpg](images/0196f770-223e-7413-929e-685007fc2cc7_6_151_234_1502_481_0.jpg)

table entry. when the guest OS accesses the physical address of monitored cross-XPU memory ranges due to their invalid stage 2 translation

To make the physical memory access of the cross-XPU memory trap to EL2, correctly handle the memory access trap, and then achieve persistent monitoring, the workflow of MemPin is illustrated in Figure 6. It consists of three main steps: monitoring setup, monitoring executing, and monitoring re-activating, which are introduced as follows.

Step-1. Monitoring Setup. MemPin manipulates the stage-2 translation table to hook the entry of the cross-XPU memory on physical memory access. The page size in the latest Apple-Silicon-based ma-cOS kernel is ${16}\mathrm{{KB}}$ . Meanwhile, the cross-XPU memory is allocated in a page-aligned way. Therefore, to conduct page-level monitoring on cross-XPU memory while avoiding introducing unnecessary address translation overhead, we choose ${16}\mathrm{{KB}}$ as the page size for stage-2 translation of cross-XPU memory and use bigger memory blocks (32MB) for stage-2 translation of other memory, which is shown in Figure 6(a).

To hook the memory access, MemPin manipulates the Level 3 Translation of the targeting cross-XPU memory by invalidating its corresponding page table entry, while keeping the non-cross-XPU memory that is also included in the original 32MB valid. On the right side of Figure 6(a), the stage 2 translation failed due to the invalid page table entry. Therefore, an access trap exception at EL2 is triggered, which the hypervisor is designed to handle.

Step-2. Monitoring Executing. MemPin handles the memory access trap by recovering the cross-XPU memory stage-2 translation table entry, enabling the single-step trap. This step is illustrated in Figure 6(b). Although m1n1 attempts to make the memory access instruction execute normally by simulating the access instructions in EL2 for the MMIO hooking, the simulation approach is still limited for two reasons. First, the simulation requires the manual efforts of the hypervisor developers, which is labor-intensive and tedious. Second, the simulation is hard to guarantee correctness, especially for complex instructions like atomic instructions [8] in the macOS on Apple silicon.

Therefore, MemPin recovers the stage-2 translation table entry of the cross-XPU memory to make the memory access instruction execute in EL1 as in normal execution. This method of validating the invalidated page table entry in trap handling has two advantages. First, it is easy to implement, as MemPin only needs to manipulate the page table entry rather than parse the memory access instruction to simulate memory read/write. Second, it will not cause any memory access errors because the access instruction is executed normally by macOS.

![0196f770-223e-7413-929e-685007fc2cc7_6_926_871_720_658_0.jpg](images/0196f770-223e-7413-929e-685007fc2cc7_6_926_871_720_658_0.jpg)

Figure 7: An example of the requirements of the constraints to deliver the cross-XPU memory to XPU (reach line 11) correctly. The wrong mutation will lead to the failure of the submission.

Step-3. Monitoring Re-activating. MemPin re-activates the hooking by invalidating the Level 3 Translation, then disabling the singlestep trap, and thus achieves persistent monitoring, as shown in Figure 6(c). Since the stage-2 translation is recovered, the guest OS re-executes the memory-accessing instructions normally. However, if we don't revert the page table entry to an invalid state, the hooking will be non-persistent, preventing further access from being monitored. Therefore, we enable the single-step trap after recovering the page table entry to reactivate the monitoring for the cross-XPU memory. Afterward, due to the activated single-step trap, CrossFire catches the trap and re-activates the monitoring by invalidating the Level 3 Translation and disabling the single-step trap. As such, the monitoring is persistent.

Overall, MemPin monitors the access of physical addresses of cross-XPU memory by manipulating the stage-2 translation table, which addresses Challenge-1. As a result, MemPin outputs not only the target chunk and its effective fuzzing region but also the location of the corresponding parsing code of cross-XPU memory in the closed-source kext, partially addressing Challenge-2.

### 4.2 XPU-Targeted Data Constraints Extraction

With the identified effective fuzzing regions and the location of the parsing code from $§{4.1}$ , next we need to pass the kernel to deliver the payloads to XPU. Particularly, the cross-XPU memory data usually has complex patterns and static invariants for checking the data validity. For example, as illustrated in Figure 7, in Basic Block 1, the cross-XPU memory is loaded at line 1 , and the Submit_to_XPU is at line 11. To ensure the cross-XPU memory data can be submitted to XPU, the data must satisfy the following data constraints: the first 4 bytes in the memory must be $0 \times  {10000}$ (line 3), and the 4 bytes in offset 0x30 minus the 4 bytes in offset 0x34 must be 0x30 (line 6).

Trace is essential for being aware of whether the payloads are submitted to the XPU, avoiding tedious reverse engineering. As such, the taint analysis based on the trace is simple and efficient. Therefore, to address Challenge-2, we propose XPU-Targeted Data Constraints Extraction (ConExtract) to extract the data constraints for passing the payloads to the XPU. Particularly, ConExtract utilizes lightweight multi-tags dynamic taint analysis to identify these data constraints.

4.2.1 Procedure of XPU-Targeted Data Constraints Extraction. ConEx-tract is divided into three steps. Firstly, by instrumenting the kernel binary, ConExtract records the execution trace and execution context. Secondly, ConExtract sets up the running environment of the emulation engine based on collected feedback. Finally, ConExtract assigns every byte of cross-XPU memory with a unique taint tag and traces the propagation of taint tags during the emulation execution. When ConExtract encounters a validity check, ConExtract will identify which taint tag propagates the condition of the check and locate the corresponding position of cross-XPU memory, which consists of data constraints. The workflow is illustrated in Figure 8. Step-1. Initialization. To facilitate instruction restoration resulting from instrumentation on the macOS kernel, ConExtract duplicates the macOS kernel in memory and performs binary rewriting to instrument it. This step is shown in Figure 8(a).

Step-1.1 Duplicating the macOS Kernel in Memory. To make sure that the instrumented instruction in the macOS kernel can be recovered, ConExtract duplicates the original macOS kernel in memory. Therefore, ConExtract can instrument arbitrary instructions in the macOS kernel and collect corresponding execution context for further analysis.

Step-1.2 Instrumenting the macOS Kernel through Binary Rewrite. To intercept the execution of the macOS kernel, ConExtract instruments hypercalls to the macOS kernel through binary rewriting. Specifically, ConExtract instruments the first instruction of the basic blocks of the targeted XPU-related kernel extensions.

Step-2. Trace Recording upon Exception Handling. To record the trace data for the dynamic taint analysis, ConExtract handles the hypervisor call and records the CPU states of the macOS kernel for taint analysis, as shown in Figure 8(b).

Step-2.1 Handling Hypervisor Call. To correctly handle the exception from the hypervisor call written by the initialization procedure, ConExtract restores the instruction from the macOS kernel backup and enables the single-step trap to execute the original instruction. To achieve persistent instrumentation, ConExtract re-instruments the running macOS kernel and disables the single-step trap.

Step-2.2 Recording Trace Data. After the hypervisor call is handled, ConExtract records the CPU state of the macOS kernel to construct the trace data, which will be sent to the server PC for dynamic taint analysis. As illustrated in Figure 9, since we are running an instrumented macOS, the ① first code hvc in the Basic Block 5 is executed to trap to the hypervisor. The hypervisor ② restores the instruction to the original one and 3 records the CPU state of the macOS. To achieve persistent instrumentation, the hypervisor enables the single-step trap. Consequently, after ④ the normal execution of the original instruction, it traps in the hypervisor again. ⑤ The hypervisor re-instruments the macOS kernel and ⑥ disables the single-step trap.

Step-3. Dynamic Taint Analysis Based on Trace Data. Based on the trace data recorded in previous steps, ConExtract first extracts control flow information of the path that submits cross-XPU memory data to XPU. Then ConExtract constructs a straight control flow offline, followed by a multi-tag dynamic taint analysis based on the control flow and contextual data. This analysis taints cross-XPU memory and traces the specific bytes used for validity checks, marking their locations as immutable. After the taint analysis, ConExtract extract extracts the data constraints of cross-XPU memory to guide mutation in fuzzing, as shown in Figure 8(c).

Step-3.1 Building Control Flow. To make dynamic taint analysis execute the path that submits cross-XPU memory data to XPU as recorded in the previous step, ConExtract extracts all program counters representing the start address of the basic block from feedback captured from the previous step in server PC based on different signs retrieved from the feedback. The program counters extracted by ConExtract are used to construct the recorded path XPU control flow for the dynamic taint analysis.

Step-3.2 Taint Analysis on Control Flow. Based on the built control flow, ConExtract leverages the symbolizing memory provided by the emulation engine Triton [35] to implement a multi-tag taint analysis for cross-XPU memory in the server PC. The source of taint analysis starts from the instruction where Cross-XPU memory is first accessed in MemPin. Meanwhile, ConExtract symbolizes every byte of the whole cross-XPU memory before the analysis starts. Every byte of the whole cross-XPU memory is assigned to a unique taint tag. During emulation execution for dynamic taint analysis, ConExtract saves all taint expressions related to Cross-XPU memory into the expression pool. When ConExtract encounters check for the data constraints, ConExtract looks up the pool to find which taint tags are involved in checking. According to taint tags, ConExtract uses the byte position of cross-XPU memory to construct data constraints.

![0196f770-223e-7413-929e-685007fc2cc7_8_154_237_1499_523_0.jpg](images/0196f770-223e-7413-929e-685007fc2cc7_8_154_237_1499_523_0.jpg)

Figure 8: Workflow of XPU-Targeted Data Constraints Extraction. The steps start from the preparation in (a), and thus, get the recorded feedback in (b). The dynamic taint analysis from the recorded feedback is used to collect the constraints of the cross-XPU memory in (c).

![0196f770-223e-7413-929e-685007fc2cc7_8_151_924_720_446_0.jpg](images/0196f770-223e-7413-929e-685007fc2cc7_8_151_924_720_446_0.jpg)

Figure 9: An example of collecting the trace with CPU states. The trace is used to reconstruct the control flow, and the CPU states are used to provide real execution context for the taint analysis.

The data constraints extracted by taint analysis become immutable bytes that guide the mutation of cross-XPU memory during fuzzing, enhancing the success rate of malformed cross-XPU memory passing through the kernel to reach the XPU.

4.2.2 Coverage Collecting. CrossFire collects the addresses (program counters) of the executed basic block which can be easily extracted from the collected trace data of $§{4.2.1}$ . Following the extracted executed basic blocks, we can get the virtual addresses of the executed basic blocks in binary by adding the KASLR offset. We can then use these addresses to construct block coverage.

Specifically, similar to SyzGen [13], we also introduce a new kernel extension to the macOS, exposing interfaces to provide management of the collecting via sockets. However, unlike SyzGen's use of the enqueuedata method for data transfer, which is limited to short data, we utilize a customizable, statically allocated memory region within the kernel extension. The region is contiguous in both virtual and physical memory, ensuring the sequential storage of recorded data written by the hypervisor via physical addresses. Then we use the server PC to run the instrumented macOS in the target PC, where CrossFire correctly handles the instrumentation and records the program counter of EL1 as block coverage in EL2, as shown in Figure 9.

### 4.3 Hooking-based Grey-box Fuzzing

CrossFire combines two innovative techniques in fuzzing: MemPin and ConExtract. Utilizing these insights, CrossFire employs a submitting-aware and hooking-based grey-box fuzzing methodology. It incorporates a mutator designed to mutate the target chunk within the bounds set by these techniques. Meanwhile, CrossFire leverages the amount of block coverage to determine if the XPU-related kernel extension rejects further submissions. If the kernel rejects further submissions, CrossFire re-executes the seed application.

Choice of Hooking-based Method. CrossFire prefers hooking-based fuzzing over generation-based fuzzing. Particularly, generation-based fuzzing faces three unique challenges when applied to cross-XPU memory because establishing XPU interaction channels is complex and varies between XPUs. First, cross-XPU memory is highly contextual, containing temporary XPU virtual addresses, timestamps, object references, and other undisclosed fields that make it difficult to simply log and replay. Second, cross-XPU memory is large (0x5b3f60 bytes on average in our evaluation) and complex, making it challenging for generation-based fuzzing to explore. Particularly, all cross-XPU memories are interrelated. For example, one cross-XPU memory may contain the effective range of other memories, while another manages the IDs of different cross-XPU memories. Moreover, the varied IDs for different APIs could confuse API dependency inference. Third, establishing interaction channels is different across XPUs, making it difficult to port generation-based fuzzing from one XPU to another. Therefore, we choose the hooking-based fuzzing method that reuses the existing channels to reach deep code more easily because it keeps most memory states correct.

Fuzzing Flow. For fuzzing a target chunk, CrossFire first runs the seed application with the two techniques MemPin and ConEx-tract to provide mutation information, including effective region and data constraints of the target chunk, which can be used further as mutation constraints.

In our experiment, when the target chunk fails to submit to the $\mathrm{{XPU}}$ , the failure will trigger error handling, and then $\mathrm{{XPU}}$ rejects further submissions of the target chunk from the seed application. Additionally, the collected basic block addresses of the rejected submission are far less than normal submission and checking failure of the target chunk. Therefore, CrossFire can be aware of the status of the submission through the block coverage. Besides, CrossFire chooses to instrument kernel extension rather than XPU firmware due to two reasons. The first reason is that XPU firmware is hard to instrument. The XPU firmware is closed-source and complicated, which leads to difficulty in emulating XPU firmware for instruments. There is no public research on emulating Apple XPU firmware. The second reason is that the XPU firmware is protected by Apple [6], making users unable to instrument XPU firmware in runtime. Additionally, Apple does not allow users to load self-modified XPU firmware. CrossFire uses the instrumentation technique described in $§{4.2}$ to instrument the XPU-related kernel extension to collect coverage feedback. Coverage of XPU-related kernel extension can judge whether the target chunk is submitted to XPU or not.

Lastly, CrossFire continues to re-execute the same seed application and divides the effective region of the target chunk into mutable and immutable bytes as mutation information. The immutable bytes are used for validity checks for the target chunk in kernel space. Data dependency that can change the immutable area exists in XPU memory. However, one type of XPU memory is dedicated to one fixed task, so immutable bytes are fixed in one memory but may vary across memories. The mutator keeps the immutable bytes unmodified and only mutates the mutable bytes. After the mutated test case is executed, CrossFire observes if the collected coverage is far less than the coverage of non-mutated execution. If collected coverage is abnormal, CrossFire kills the seed application and re-executes the seed application with the same procedure described above.

## 5 EVALUATION

To evaluate the effectiveness of the two proposed techniques and the CrossFire, we conduct experiments to answer the following research questions.

RQ1: What is the effectiveness of MemPin?

RQ2: What is the effectiveness of ConExtract?

RQ3: What is the effectiveness of the coverage collecting?

RQ4: How much do the two techniques proposed by CrossFire contribute to fuzzing?

RQ5: How many bugs can CrossFire find in XPU-related code?

Evaluation Environment. We use a MacBook Pro (16GB+1TB) with Apple M1 chip running macOS 13.5.2 with KASAN as the target PC and a MacBook Air with Apple M1 chip running macOS 12.4 (16GB+1TB) as the server PC.

Table 1: The target chunk and its effective fuzzing region of the seed application.

<table><tr><td>Seed App</td><td>CXMSize</td><td>Tarsize</td><td>UWSize</td><td>Decreased</td></tr><tr><td>Metal Sample</td><td>0x1b0000</td><td>0x8000</td><td>0x8f0</td><td>93.02%</td></tr><tr><td>Vision Sample</td><td>0x20000</td><td>0x8000</td><td>0x17c</td><td>98.84%</td></tr><tr><td>Weather</td><td>0x4a8000</td><td>0xc000</td><td>0x7d10</td><td>34.86%</td></tr><tr><td>Keynote</td><td>0x170000</td><td>0x18000</td><td>0x7c4</td><td>97.98%</td></tr><tr><td>PhotoBooth</td><td>0x1ec000</td><td>0x10000</td><td>0x8608</td><td>47.64%</td></tr><tr><td>Xpression Camera</td><td>0x394000</td><td>0x1c000</td><td>0x14e4</td><td>95.34%</td></tr><tr><td>Billiards 2048</td><td>0x138000</td><td>0x40000</td><td>0x1b08</td><td>97.36%</td></tr><tr><td>Total</td><td>0x27f0000</td><td>0xa0000</td><td>0x14534</td><td>87.30%</td></tr></table>

Seed Application. To fuzz cross-XPU memory effectively, CrossFire chooses the applications that interact with XPU at high frequency as seed applications. The applications we use in the experiments contain three categories: developer samples, official applications, and third-party applications. The developer samples are from the official Apple Developer Samples ${}^{12}$ . The official applications are Weather, Keynote, and PhotoBooth. The third-party applications are Xpression Camera and Billiards 2048.

### 5.1 Effectiveness of MemPin (RQ1)

To evaluate the effectiveness of MemPin, we run the seed application in the target macOS and monitor the memory access to get the whole tagged cross-XPU memory ranges, which is noted as CXM-Size. We record the target chunk and its corresponding user-write ranges after the seed apps communicate with XPUs. Then, we refer to the size of the target chunk cross-XPU memory as TarSize, and denote the monitored user-write size of the cross-XPU memory as UWSize. Lastly, we calculate the decreased ratio as Decreased, which equals one minus the ratio of UWSize over TarSize, as shown in Table 1.

To answer the RQ1, we tested the seed application under monitoring. The mutation space of the in-use cross-XPU memory decreased by ${87.3}\%$ on average. The existence of the unused but allocated region in the cross-XPU memory is mainly because the cross-XPU memory allocation is page-aligned (16KB-aligned); even if only small effective data are stored in the cross-XPU memory, the whole page will be allocated.

### 5.2 Effectiveness of ConExtract (RQ2)

We evaluate the effectiveness of the constraints from ConExtract three times, by comparing the submissions to the XPU over time when fuzzing the cross-XPU memory with and without the constraints. We ascertain whether the mutated data from the seed application has been submitted to the XPU, through the block coverage discussed in §4.3. Due to the fact that fuzzing frequently results in XPU-related code hang-ups, we perform a hang-up check by comparing the number of submissions per second to avoid unrecoverable hang-ups as much as possible. If an unrecoverable hang-up occurs, we terminate the seed application and wait for a restart. It's worth noting that the termination is also not instant because of the kernel hang-up. If the submission fails, we also terminate the seed application and wait for it to restart to avoid any interference from the error handling for the failure provided by the system. Particularly, in our experiment, if the mutated cross-XPU memory submission fails to reach the XPU only once, all the following submissions will also fail. Hence, once the failure happens, we restart the seed apps to reconstruct the interaction channels for fuzzing. As a result, the submissions to the XPU over time are shown in Figure 10.

---

${}^{1}$ https://developer.apple.com/metal/sample-code/

${}^{2}$ https://developer.apple.com/documentation/vision/

---

![0196f770-223e-7413-929e-685007fc2cc7_10_206_288_552_406_0.jpg](images/0196f770-223e-7413-929e-685007fc2cc7_10_206_288_552_406_0.jpg)

Figure 10: The rate of submissions to XPU over time with versus without mutation constraints.

To answer the RQ2, submissions of fuzzing payloads with constraints to the XPU demonstrated an increase in speed of approximately 83.9%. The reason is that the constraints avoid the immutable region in the target chunk that is checked in the kernel. Therefore, the constraints effectively help the fuzzing payloads to pass through to the XPU.

### 5.3 Effectiveness of Coverage Collecting (RQ3)

As shown in Table 2, regarding the instrumentation rate, CrossFire outperforms the coverage collection method proposed by Kext-Fuzz [44], which substitutes the aut*/pac*/nop instructions in the basic block to obtain coverage. According to KextFuzz's experiments, it can instrument an average of ${34}\%$ of the basic blocks in kernel extensions, whereas CrossFire can instrument 100% of the basic blocks in the selected kernel extensions. CrossFire runs the instrumented macOS steadily, and the XPU-related kexts work normally under the instrumentation. For other kernel extensions, in our experiments, CrossFire can instrument and run almost all kernel extensions except corecrypto and AppleMobileFileIntegrity out of 349 official kernel extensions. These two kernel extensions are related to security, and Apple has forbidden the hypervisor interactions used by CrossFire in the booting process of these two.

To answer the effectiveness of the coverage collecting of RQ3, CrossFire can instrument ${100}\%$ basic blocks of almost all kernel extensions of macOS on Apple silicon, including all XPU-related kernel extensions. Additionally, the hypervisor can collect not only the coverage but also the context of the running macOS, which is the key to the technique ConExtract for extracting the cross-XPU memory data constraints.

Table 2: Evaluation of the coverage collecting (comparing the rate of the instrument basic blocks of CrossFire with the previous SOTA KextFuzz). All kernel extensions below work normally under the instrumentation in CrossFire.

<table><tr><td>Kernel Extension</td><td>#BBs</td><td>KextFuzz*</td><td>CrossFire</td></tr><tr><td>IOGPUFamily</td><td>7743</td><td>2108(27.22%)</td><td>7743(100%)↑</td></tr><tr><td>IOGraphicsFamily</td><td>7426</td><td>1872(25.21%)</td><td>7426(100%)↑</td></tr><tr><td>IOPCIFamily</td><td>6617</td><td>${1495}\left( {{22.59}\% }\right)$</td><td>6617(100%)↑</td></tr><tr><td>AGXG13G</td><td>21439</td><td>4887(22.79%)</td><td>21439(100%)↑</td></tr><tr><td>AGXFirmwareKextRTBuddy64</td><td>6</td><td>0</td><td>6(100%)↑</td></tr><tr><td>AGXFirmwareKextG13GRTBuddy</td><td>311</td><td>113(36.33%)</td><td>311(100%)↑</td></tr><tr><td>IOAVFamily</td><td>14760</td><td>5451(36.93%)</td><td>14760(100%)↑</td></tr><tr><td>IOMobileGraphicsFamily</td><td>8548</td><td>1714(20.05%)</td><td>8548(100%)↑</td></tr><tr><td>AppleFirmwareKit</td><td>5529</td><td>1938(35.05%)</td><td>5529(100%)↑</td></tr><tr><td>AppleH11ANEInterface</td><td>15738</td><td>2637(16.76%)</td><td>15738(100%)↑</td></tr><tr><td>DCPAVFamilyProxy</td><td>2353</td><td>806(34.25%)</td><td>2353(100%)↑</td></tr><tr><td>AppleDCP</td><td>416</td><td>166(39.90%)</td><td>416(100%)↑</td></tr><tr><td>IOMobileGraphicsFamily-DCP</td><td>7653</td><td>1334(17.43%)</td><td>7653(100%)↑</td></tr><tr><td>IOSlaveProcessor</td><td>212</td><td>69(32.55%)</td><td>212 (100%)↑</td></tr></table>

* BBs that have aut*/pac*/nop are accounted as instrumented.

### 5.4 Fuzzing with CrossFire (RQ4)

Because no prior works aiming to fuzz the cross-XPU memory in macOS on Apple silicon, we evaluate CrossFire by breaking down each technique. We use the seed application to run under the evaluation configurations below. We evaluate the effectiveness of each configuration by comparing the bug-finding results in 24 hours seven times.

- Base. This is a basic configuration, which only uses the basic random mutation for syscall parameters (employed by the previous hooking-based works LLDBFuzzer[42] and Kemon[41]). This configuration represents a baseline for fuzzing normal syscall parameters rather than the cross-XPU memory.

- Cross-XPU-Memory Base. This configuration stands for the baseline of fuzzing cross-XPU memory, which is the new attack surface found by us. It randomly mutates the data in all the cross-XPU memory ranges and does not consider the effective mutation area from MemPin (§4.1) and the constraints from ConExtract (§4.2).

- MemPin: use MemPin to get the effective fuzzing regions and mutate them beyond the Cross-XPU-Memory Base.

- MemPin + ConExtract: use the full-loaded CrossFire beyond MemPin to evaluate the fuzzing effectiveness of the entire CrossFire.

Fuzzing Setup. We have configured a task to run the experiment automatically at startup, and by leveraging the hypervisor to handle the instrumentation in macOS panic processing, we have implemented fully automated fuzzing on macOS. Moreover, the hang-up check is configured in all these experiments. As mentioned in §5.2, coverage provided by ConExtract guides the fuzzing process, determining whether submissions reach the XPU. If a submission fails to reach the XPU, subsequent submissions continue normally via the syscall, but the kernel extension prevents their delivery to the XPU. We use the block coverage in configuration Cross-XPU-Memory

![0196f770-223e-7413-929e-685007fc2cc7_11_253_283_482_391_0.jpg](images/0196f770-223e-7413-929e-685007fc2cc7_11_253_283_482_391_0.jpg)

Figure 11: The fuzzing effectiveness of the 4 configurations, Base, Cross-XPU-Memory Base, MemPin, and MemPin + ConExtract.

## Base, MemPin, and MemPin + ConExtract to get the submission status.

Figure 11 shows the fuzzing effectiveness of each configuration. We use the statistical significance with the Mann Whitney U-test p-value. The colorful line represents the average configuration-finding crashes in the seven experiments. The result shows that the full-loaded CrossFire with MemPin and ConExtract gets the highest effectiveness. To answer the RQ4, for the traditional fuzzing targets of the syscall parameters, the Base that only employs random mutation on syscall parameters shows poor capability (black dashed line) in finding bugs. The basic mutation can be easily rejected by the user input sanitization in the interfaces of kernel extensions.

For the cross-XPU memory, Cross-XPU-Memory Base finds one unique crash (purple solid line) in kernel extension within the seven repeats, improving a little in finding XPU-related bugs compared to Base. This is because we have manually changed the mutation from traditional syscall parameters to the cross-XPU memory. The crash is caused by the mutation on a Type2 cross-XPU memory inside the effective fuzzing region defined in $\$ {2.2}$ . The details of cross-XPU memory have not publicly been disclosed before. Thus, it has hardly been tested previously, and even the simplest random mutation can cause a crash.

Compared with Cross-XPU-Memory Base, MemPin has a significant improvement in fuzzing cross-XPU memory. The ability to monitor the cross-XPU memory access in $\$ {4.1}$ can provide a precisely effective fuzzing region to facilitate mutation. Compared to the above two configurations, the configuration MemPin (blue solid line) finds more crashes compared to Cross-XPU-Memory Base. It is because we limit the mutations only to occur on the region given by $\$ {4.1}$ rather than the syscall parameters or the whole tagged cross-XPU memory. However, it continuously causes unrecoverable kernel hang-ups, slowing down the fuzzing process.

ConExtract helps CrossFire to find the immutable bytes that are checked and thus protected by the kernel. The mutation to these immutable bytes will lead to all the following submissions failing to reach the XPU, because of the internal error handling mechanism in the kernel extensions. Moreover, the immutable region is inside the effective fuzzing region, and thus, they can cooperate to facilitate the fuzzing process, as shown in the configuration MemPin + ConExtract (solid green line). As the mutation avoids causing the following submission to fail, the mutation can focus on the effective mutable region rather than mutating the region that is checked by the kernel; the fuzzing effect reaches the highest.

Table 3: Bug finding result.

<table><tr><td>Reason</td><td>Location</td><td>Status</td></tr><tr><td>Invalid Data Access</td><td>Firmware (GPU Kernel)</td><td>Confirmed</td></tr><tr><td>Type Confusion</td><td>Firmware (GPU Kernel)</td><td>Reported</td></tr><tr><td>OOB Read</td><td>Firmware (GPU Kernel)</td><td>Reported</td></tr><tr><td>GPU Resource Exhaustion</td><td>GPU KEXT</td><td>Reported</td></tr><tr><td>IOGPUFamily Null Object</td><td>GPU KEXT</td><td>Reported</td></tr><tr><td>IOMFB Assert Failure</td><td>GPU KEXT</td><td>Reported</td></tr><tr><td>kmem_realloc Size Assert Failure</td><td>GPU KEXT</td><td>Reported</td></tr><tr><td>Null Pointer 1D48AFF98</td><td>NPU Daemon</td><td>Confirmed</td></tr><tr><td>Arbitrary Read</td><td>Firmware KEXT</td><td>Confirmed</td></tr><tr><td>Null Pointer 1D48B065C</td><td>NPU Daemon</td><td>Confirmed</td></tr><tr><td>Null Pointer 1D48B0188</td><td>NPU Daemon</td><td>Confirmed</td></tr><tr><td>Null Pointer 1D48B09B0</td><td>NPU Daemon</td><td>Confirmed</td></tr><tr><td>Null Pointer 1D48B03D0</td><td>NPU Daemon</td><td>Confirmed</td></tr><tr><td>Null Pointer 1D46CC154</td><td>NPU Daemon</td><td>Confirmed</td></tr><tr><td>Integer Overflow</td><td>NPU KEXT</td><td>Reported</td></tr></table>

Crash analysis. We analyze the crashes found by CrossFire through the crash ${\operatorname{logs}}^{3}$ and the collected trace. The 4 configurations find0,1,10, and 19 unique crashes on average respectively. The crashes found by Cross-XPU-Memory Base are included in MemPin. The crashes found by MemPin are included in MemPin - ConExtract, except for one bug that corrupts different objects, crashing in kmem_realloc. In addition, the firmware crash on macOS also causes kernel panic with special marks GFX in logs.

### 5.5 Bug Finding (RQ5)

To answer the RQ5, CrossFire found 15 zero-day bugs, including invalid data access, type confusion, out-of-bound read, arbitrary read, integer overflow, null pointer dereference, etc. Since we are targeting XPU firmware by mutating cross-XPU contents, we also find bugs in other XPU-related code. We follow the responsible disclosure process and report all the bugs to Apple. So far, Apple has confirmed 8 bugs. The details of the bugs are shown in Table 3. In the rest of this subsection, we'll discuss two bugs found by CrossFire.

Listing 1 shows a simplified bug in kext. The render_setup function checks the validity of cross-XPU memory. When the data at offset $0 \times  2\mathrm{c}$ equals $0 \times  7\mathrm{e}8$ and the data at offset $0 \times  {34}$ minus 1 equals 0 , the desc->a2 variable is assigned the value from the unchecked cross-XPU memory offset 0x1f7. Subsequently, the render_setup function calls the vulnerable retain_buffers, which in turn invokes the retain function. This triggers a bug using the desc->a2 variable. The fuzzing process identifies the effective region (from MemPin) and mutation constraints (from ConExtract) of the cross-XPU memory. It identifies the offsets $0 \times  2\mathrm{c}$ and $0 \times  {34}$ as immutable, while offset $0 \times  1 \times  7$ is marked as mutable. Furthermore, the fuzzing strategy employs a hooking-based method to maintain the validity of the cross-XPU memory at offsets $0 \times  2\mathrm{c}$ and $0 \times  {34}$ but mutates offset $0 \times  1\mathrm{f}7$ to zero in order to trigger the vulnerability.

---

${}^{3}$ We use the crash logs caught by the kernel to mark the unique crashes and to diagnose the crashes, where the logs contain the crash reason and location.

---

---

int64 render_setup(int64 *a1, ..., int64 *mem0, ...)\{

	if (*(int32 *)(mem0 + 0x2c) == 0x7e8)\{

		if $\left( {*\left( {\text{int}{32} * }\right) \left( {\text{mem0} + 0 \times  {34}}\right)  - 1}\right)  =  = 0)\{$

				desc->a2 = *(mem0 + 0x1f7):

				retain_buffers(this, desc);

				...

	\}

	...

\}

void retain_buffers(int64 *this, int64* desc)\{

	... retain(desc->a2);

\}

---

Listing 1: An example of the bug in kext.

int64 post_recovery(int64 *a1, ...)\{// code in the firmware

---

if ( first_bigflag_value_swap )

	do

		v31 = clz( rbit64(v30)) :

		object_pointer = *(int64 *)(object_list)[object_index]; //

		$\hookrightarrow$ object_index could exceed the boundary

		v30 &=~(1LL << v31);

	while ( v30 );

\}

---

Listing 2: An example of the firmware bug.

Additionally, Listing 2 shows another bug in the firmware, where object_index could exceed the boundary of object_list due to cross-XPU memory payloads (line 8), resulting in the retrieved object_pointer being invalid. Meanwhile, we discovered an XPU kernel data region bufferManager that can be directly manipulated from a normal XPU user application, which allows us to create crafted fake objects by placing well-defined data. When object_index reaches the user-controlled XPU kernel data region, because the firmware isn't armed with ASLR or PAC, with the crafted object, fetched object_pointer may become fully controlled by the attacker. When the object_pointer is used to invoke functions within the virtual table, arbitrary code execution possibly occurs.

## 6 RELATED WORKS

### 6.1 Fuzzing on macOS

Fuzzing on x86_64 macOS has been extensively studied in recent years, while the fuzzing of macOS on Apple silicon is an emerging field. Lin et al. [27] utilize Syzkaller [29] and LLVM instrumentation to collect the coverage of the macOS kernel to guide the fuzzing on x86_64 macOS XNU. Lin et al.[26] uses binary rewrite to collect the coverage of the KEXTs to guide the fuzzing on x86_64 macOS KEXTs. IMF [22] leverages the constant values in syscall sequences to model the APIs thus facilitating fuzzing the IOKit. SyzGen [13] utilizes the run-time snapshot of the macOS kernel to deduce the dependencies between the IOKit interfaces and generate the interface sequences for fuzzing. KextFuzz [44] targets to fuzz kernel extensions of macOS on Apple silicon using coverage guidance, and it takes advantage of the user libraries to generate the interfaces. However, the interface deducing methods used by KextFuzz and SyzGen are not suitable for fuzzing the cross-XPU memory because the cross-XPU memory is too complex to generate. None of the above experiments demonstrate the ability to find bugs in the XPU firmware, while CrossFire is specifically designed for this purpose.

### 6.2 Hooking for Fuzzing

Hooking for fuzzing is practical and widely deployed in fuzzing closed-source targets. In Windows, NTFuzz [14] hooks the Windows syscall APIs to mutate the syscall parameters in a type-aware manner. In macOS, for facilitating generating test cases, IMF [22], PanicXNU [26, 27] and SyzGen [13] use hooking to collect the API calls to facilitate generating the fuzzing test cases. For mutating the function parameters, Kuzz [21] hooks IOKit APIs and randomly mutates the IOKit API parameters in user space to fuzz the iOS kernel extensions. Chen et al. [43] mutate IOKit API parameters and furthermore statically resolves the parameter checks of IOKit API to actively fuzz the interfaces. Long et al. [30] optimize the fuzzing efficiency by dynamically collecting the symbols and parameters in memory. Li et al. [25, 33] hook the system APIs to collect valid parameters and then mutate the parameters to fuzz the macOS kernel. LLDBFuzzer [42] hooks the specified IOKit API and mutates the parameters of the IOKit API based on the kernel debugger LLDB. Wang et al. [40] hook IOKit APIs using a kernel monitoring framework called Kemon [41], which is based on inserting kernel extension to implement inline hook. This approach randomly mutates the input of IOKit APIs while running a seed application. However, they all focus on the normal parameters of the IOKit in kernel extensions rather than the cross-XPU memory processed by XPU firmware.

### 6.3 XPU-related Fuzzing

Existing XPU-related fuzzing works focus on the shader compiler and XPU drivers. For shader programs fuzzing, GraphicsFuzz [16, 18, 19] developed by Google tries to provide tools for finding bugs in graphics drivers, specifically graphics shader compilers. Graph-icsFuzz utilizes malformed shader programs to test the rendering bugs in GPU, but it only targets the compiler and causes rendering problems, not serious crashes on Apple. GLeeFuzz [34] aims to fuzz WebGL in browsers across various platforms, such as Apple and Intel. Instead of relying on coverage metrics for feedback, it leverages browser logs. These logs offer comprehensive details about the execution state, facilitating more informed mutation guidance. In contrast, Doré et al.[17] focus on the NVIDIA graphic driver. They employ symbolic execution to determine viable constraints for the GPU driver's memory. This approach aims to explore the driver's basic blocks to enlarge the coverage. Wang et al.[40] focus on fuzzing Apple's graphic driver by a monitoring framework called Kemon [41]. Framework Kemon is designed to inline hook the x86_64 macOS kernel function. Wang et al. then leverage Ke-mon to hook the IOKit APIs of the graphic driver and randomly mutate parameters. However, these works' methods are limited in exploring the GPU driver. None of them found any bugs in the XPU firmware.

## 7 CONCLUSION

This paper proposes CrossFire, the first fuzzer designed to target the XPU by fuzzing the cross-XPU memory on macOS running on Apple silicon. To achieve this, we conducted an in-depth analysis of the cross-XPU memory usage, exploring our findings and identifying associated challenges. We propose two novel techniques Hypervisor-based Effective Cross-XPU Memory Pinpoint (MemPin) and XPU-Targeted Data Constraints Extraction (ConExtract) to address the challenges of fuzzing cross-XPU memory. MemPin identifies the effective fuzzing region, while ConExtract provides immutable byte positions as constraints and traces for XPU submission-aware grey-box fuzzing. Armed with this understanding of the effective fuzzing region, mutation constraints, and comprehensive traces, we implement the CrossFire prototype and discover 15 zero-day bugs.

## ACKNOWLEDGMENTS

The authors would like to thank our shepherd and reviewers for their insightful comments. Those comments helped to reshape this paper. Additionally, the authors would like to express their gratitude to Yutian Yang for his assistance in answering our questions and guiding our efforts. We also extend our thanks to Guanxing Wen for his help in analyzing some of the bugs we discovered. This work is partially supported by the National Key R&D Program of China (No. 2022YFE0113200).

## REFERENCES

[1] Akihiko Odaki. 2022. Introduce gdbserver. https://github.com/AsahiLinux/m1n1/ pull/194/commits/cc420003ef9c929ca64b68723a38234b567395b7.

[2] Apple. 2020. Choosing a Resource Storage Mode for Apple GPUs. https://developer.apple.com/documentation/metal/resource_fundamentals/ choosing_a_resource_storage_mode_for_apple_gpus?language=objc.

[3] Apple. 2024. Accelerate graphics and much more with Metal. https:// developer.apple.com/metal/.

[4] Apple. 2024. Build virtualization solutions on top of a lightweight hyper-visor, without third-party kernel extensions. https://developer.apple.com/ documentation/hypervisor.

[5] Apple. 2024. Page Protection Layer. https://support.apple.com/en-hk/guide/ security/sec8b776536b/1/web/1#sec314c3af61.

[6] Apple. 2024. System Coprocessor Integrity Protection. https://support.apple.com/ en-hk/guide/security/sec8b776536b/1/web/1#sec59f75f8cd.

[7] ARM. 2024. Virtualization-host-extensions. https://developer.arm.com/ documentation/102142/0100/Virtualization-host-extensions.

[8] ARM LTD. [n. d.]. Memory access atomicity. https://developer.arm.com/ documentation/den0024/a/The-A64-instruction-set/Memory-access-instructions/Memory-access-atomicity.

[9] ARM LTD. 2020. Arm Architecture Reference Manual for A-profile architecture. https://developer.arm.com/documentation/ddi0487/latest.

[10] AsahiLinux. 2021. m1n1: an experimentation playground for Apple Silicon. https://github.com/AsahiLinux/m1n1.

[11] Ian Beer. 2023. Abusing iPhone Co-Processors for Privilege Escalation. In (Objective By The Sea (OBTS) v5.0).

[12] Zechao Cai, Jiaxun Zhu, Wenbo Shen, Yutian Yang, Rui Chang, Yu Wang, Jinku Li, and Kui Ren. 2023. Demystifying Pointer Authentication on Apple M1. In 32nd USENIX Security Symposium (USENIX Security 23). 2833-2848.

[13] Weiteng Chen, Yu Wang, Zheng Zhang, and Zhiyun Qian. 2021. Syzgen: Automated generation of syscall specification of closed-source macos drivers. In Proceedings of the 2021 ACM SIGSAC Conference on Computer and Communications Security. 749-763.

[14] Jaeseung Choi, Kangsu Kim, Daejin Lee, and Sang Kil Cha. 2021. NTFuzz: Enabling type-aware kernel fuzzing on windows with static binary analysis. In 2021 IEEE Symposium on Security and Privacy (SP). IEEE, 677-693.

[15] Parallels® Desktop. 2024. Parallels® Desktop 19 for Mac. https:// www.parallels.cn/products/desktop/.

[16] Alastair F Donaldson, Hugues Evrard, Andrei Lascu, and Paul Thomson. 2017. Automated testing of graphics shader compilers. Proceedings of the ACM on Programming Languages 1, OOPSLA (2017), 1-29.

[17] Thierry Doré. 2022. A journey of fuzzing Nvidia graphic driver leading to LPE exploitation. In (Hexacon).

[18] Hugues Evrard and Paul Thomson. 2017. GraphicsFuzz. https://github.com/ google/graphicsfuzz.

[19] Hugues Evrard and Paul Thomson. 2017. GraphicsFuzz: Secure and Robust Graphics Rendering. https://www.khronos.org/assets/uploads/developers/ library/2017-gdc-webgl-webvr-gltf-meetup/10-ImperialCollegeLondon-GraphicsFuzz_Mar17.pdf.

[20] Lars Fröder. 2024. How to Jailbreak iOS 16. In (Power of Community).

[21] Haifisch. 2016. kuzz: an iOS IOKit fuzzer. https://github.com/Haifisch/kuzz

[22] HyungSeok Han and Sang Kil Cha. 2017. Imf: Inferred model-based fuzzer. In Proceedings of the 2017 ACM SIGSAC Conference on Computer and Communications Security. 2345-2358.

[23] Apple Inc. 2020. Apple unleashes M1. https://www.apple.com/newsroom/2020/ 11/apple-unleashes-m1.

[24] Kaspersky. 2023. Operation Triangulation: The last (hardware) mystery. https: //securelist.com/operation-triangulation-the-last-hardware-mystery/111669/.

[25] Moony Li and Jack Tang. 2016. Fuzzing and Exploiting OSX Vulnerabilities for Fun and Profit . In (PacSec). https://papers.put.as/papers/macosx/2016/ PSJ2016_MoonyLi_pacsec-1.8.pdf

[26] Juwei Lin and Junzhi Lu. 2019. PanicXNU 3.0. In Hack In The Box Security Conference (HITB).

[27] Juwei Lin, Lilang Wu, and Moony Li. 2018. Drill the Apple Core: Fuzz Apple Core Component in Kernel and User Mode for Fun and Profit. In (Blackhat EUROPE). https://i.blackhat.com/eu-18/Wed-Dec-5/eu-18-Juwei_Lin-Drill-The-Apple-Core.pdf

[28] Asahi Lina. 2023. agx-exploit. https://github.com/asahilina/agx-exploit.

[29] Google LLC. 2022. Syzkaller: An unsupervised coverage-guided kernel fuzzer. https://github.com/google/syzkaller

[30] Lei Long and Peng Xiao. 2015. Optimized Fuzzing IOKit in iOS. In (Blackhat USA).

[31] ARM LTD. 2024. CoreSight Architecture. https://developer.arm.com/ Architectures/CoreSight%20Architecture.

[32] Jianfeng Pan, Guanglu Yan, and Xiaocao Fan. 2017. Digtool: A \{virtualization-based\} framework for detecting kernel vulnerabilities. In 26th USENIX Security Symposium (USENIX Security 17). 149-165.

[33] Zhenpeng Pan. 2022. The Journey To Hybrid Apple Driver Fuzzing. In (Power of Community). https://powerofcommunity.net/poc2022/ZhenpengPan.pdf

[34] Hui Peng, Zhihao Yao, Ardalan Amiri Sani, Dave Jing Tian, and Mathias Payer. 2023. \{GLeeFuzz\}: Fuzzing \{WebGL\} Through Error Message Guided Mutation. In 32nd USENIX Security Symposium (USENIX Security 23). 1883-1899.

[35] Jonathan Salwan. 2015. Triton: a dynamic binary analysis library. https:// github.com/JonathanSalwan/Triton.

[36] Sven Peter. 2021. Apple Silicon Hardware Secrets: SPRR and Guarded Exception Levels (GXF). https://blog.svenpeter.dev/posts/m1_sprr_gxf/.

[37] Sven Peter. 2021. HW: SPRR and GXF. https://github.com/AsahiLinux/docs/ wiki/HW:-SPRR-and-GXF/.

[38] UTM. 2024. UTM is a full featured system emulator and virtual machine host for iOS and macOS. https://github.com/utmapp/UTM.

[39] Xingkai Wang, Wenbo Shen, Yujie Bu, Jinmeng Zhou, and Yajin Zhou. 2024. DMAAUTH: A Lightweight Pointer Integrity-based Secure Architecture to Defeat DMA Attacks. In 33rd USENIX Security Symposium (USENIX Security 24). USENIX Association, Philadelphia, PA, 1081-1098. https://www.usenix.org/conference/ usenixsecurity24/presentation/wang-xingkai

[40] Yu Wang. 2018. Attacking the macOS Kernel Graphics Driver. In Attacking the macOS Kernel Graphics Driver. https://media.defcon.org/DEF%20CON%2026/ DEF%20CON%2026%20presentations/DEFCON-26-Yu-Wang-Attacking-The-MacOS-Kernel-Graphics-Driver-Updated.pdf

[41] Yu Wang. 2018. Kemon: An Open Source Pre and Post Callback-based Framework for macOS Kernel Monitoring. https://github.com/didi/kemon?tab=readme-ov-file.

[42] Lilang Wu and Moony Li. 2019. LLDBFuzzer: Debugging and Fuzzing the Apple Kernel. https://www.trendmicro.com/en_us/research/19/h/lldbfuzzer-debugging-and-fuzzing-the-apple-kernel-with-lldb-script.html

[43] Chen Xiaobo and Xu Hao. 2012. Find Your Own iOS Kernel Bug. In (Power of Community). https://papers.put.as/papers/ios/2012/Xu-Hao-Xiabo-Chen-Find-Your-Own-iOS-Kernel-Bug.pdf

[44] Tingting Yin, Zicong Gao, Zhenghang Xiao, Zheyu Ma, Min Zheng, and Chao Zhang. 2023. KextFuzz: Fuzzing macOS Kernel EXTensions on Apple Silicon via Exploiting Mitigations. In 32nd USENIX Security Symposium (USENIX Security 23). 5039-5054.

[45] Ziqi Yuan, Siyu Hong, Rui Chang, Yajin Zhou, Wenbo Shen, and Kui Ren. 2023. Vdom: Fast and unlimited virtual domains on multiple architectures. In Proceedings of the 28th ACM International Conference on Architectural Support for Programming Languages and Operating Systems, Volume 2. 905-919.