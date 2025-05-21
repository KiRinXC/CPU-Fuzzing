# Trusting the Trust Anchor: Towards Detecting Cross-Layer Vulnerabilities with Hardware Fuzzing

Chen Chen*, Rahul Kande*, Pouya Mahmoody ${}^{ \dagger  }$ , Ahmad-Reza Sadeghi ${}^{ \dagger  }$ , and JV Rajendran ${}^{ * }$ *Texas A&M University, USA, †Technische Universität, Germany

*\{chenc, rahulkande, jv.rajendran\}@tamu.edu, †\{pouya.mahmoody, ahmad.sadeghi \}@trust.tu-darmstadt.de

## Abstract

The rise in the development of complex and application-specific commercial and open-source hardware and the shrinking verification time are causing numerous hardware-security vulnerabilities. Traditional verification techniques are limited in both scalability and completeness. Research in this direction is hindered due to the lack of robust testing benchmarks. In this paper, in collaboration with our industry partners, we built an ecosystem mimicking the hardware-development cycle where we inject bugs inspired by real-world vulnerabilities into RISC-V SoC design and organized an open-to-all bug-hunting competition. We equipped the participating researchers with industry-standard static and dynamic verification tools in a ready-to-use environment. The findings from our competition shed light on the strengths and weaknesses of the existing verification tools and highlight the potential for future research in developing new vulnerability detection techniques.

## KEYWORDS

Hardware Security, Hack@DAC, Hack@EVENT, Hardware competition, CTF, Fuzzing

## ACM Reference Format:

Chen Chen*, Rahul Kande*, Pouya Mahmoody ${}^{ \dagger  }$ , Ahmad-Reza Sadeghi ${}^{ \dagger  }$ , and JV Rajendran*. 2022. Trusting the Trust Anchor: Towards Detecting Cross-Layer Vulnerabilities with Hardware Fuzzing. In Proceedings of the 59th ACM Design Automation Conference (DAC '22), July 10-14, 2022, San Francisco, CA, USA. ACM, New York, NY, USA, 5 pages. https://doi.org/10.1145/nnnnnnn.nnnnnnnn

## 1 INTRODUCTION

Hardware designs are growing more complex to meet the rising market needs. On the one hand, numerous application-specific commercial and open-source hardware designs are being developed [8]. On the other hand, the time allocated for verification in the design cycle is shirking to cut down the time-to-market [16, 24]. The resulting hardware vulnerabilities compromise not only the hardware but also the software running on it $\left\lbrack  {9,{17}}\right\rbrack$ . Open-source System-On-Chips (SoCs) development like RISC-V-based designs is now mature enough to use in real-world applications, which, if not properly tested, can result in a plethora of security vulnerabilities.

This calls for a need to develop new techniques capable of meeting the surging need to verify the hardware. However, research in this direction is hindered due to the lack of robust testing benchmarks, which will aid in the development of new techniques by evaluating their effectiveness and limitations.

We address these challenges through our hardware capture-the-flag competition, Hack@DAC 2021, a part of the world's largest international hardware security competition series, Hack@EVENT [3]. We developed open-source RISC-V-based [23] SoC designs with security features and bugs inserted in them, creating multiple test scenarios for known hardware common weakness enumerations (CWEs) [17]. The insights from our industry partners ensure that the bugs inserted mimic real-world scenarios. Teams from all across the world from both academia and industry competed for several months to discover and exploit hardware vulnerabilities that we implanted into the SoC designs.

The main contributions of this paper are: (i) We highlight the shortcomings and limitations of the existing verification techniques. (ii) We created a RISC-V OpenPiton SoC based platform with real-world mimicking security features and bugs as a hardware security verification benchmark. (iii) We organize the world's largest hardware security competition, Hack@DAC, motivating the industry and academic communities to develop new, automated verification techniques. (iv) We share the insights observed from the competition highlighting the capabilities of existing tools and the need for new security verification techniques.

## 2 HACK@DAC: COMPETITION OVERVIEW

Hack@DAC is one of the capture-the-flag competitions under the Hack@EVENT franchise that aims to promote automated hardware bug detection and exploitation techniques [3]. The Hack@DAC competition first started in 2018 and has received more than 650 participants from across the world. Hack@DAC 2021 is the fourth Hack@DAC competition, 41 teams participated in Phase 1, and six of them were selected for Phase 2 including three teams each from academia and industry. Phase 1 was organized virtually where the buggy SoC and the documentation with the rules of the competition are provided to the participants. Phase 2 was held in-person at the ACM/IEEE Design Automation Conference (DAC) 2021.

In each phase, the participants are provided with a threat model mimicking a typically hardware security verification scenario, a buggy SoC along with an operating system, and a scoring mechanism. Our threat model limits the scope of bug detection to real-world vulnerabilities.

Threat model. For each phase, we provide the participants with: (i) HDL (RTL) source code of the SoC, (ii) instructions for cycle-accurate simulation of the SoC, (iii) natural-language description of the security properties that are expected out of the SoC, and

---

Permission to make digital or hard copies of all or part of this work for personal or classroom use is granted without fee provided that copies are not made or distributed for profit or commercial advantage and that copies bear this notice and the full citation on the first page. Copyrights for components of this work owned by others than ACM must be honored. Abstracting with credit is permitted. To copy otherwise, or republish, to post on servers or to redistribute to lists, requires prior specific permission and/or a fee. Request permissions from permissions@acm.org.DAC '22, July 10-14, 2022, San Francisco, CA, USA © 2022 Association for Computing Machinery. ACM ISBN 978-1-4503-9142-9/22/07...\$15.00 https://doi.org/10.1145/3489517.3530638

---

![0196f2bf-3d2f-78e4-904b-586b392bfbd6_1_179_223_656_574_0.jpg](images/0196f2bf-3d2f-78e4-904b-586b392bfbd6_1_179_223_656_574_0.jpg)

Figure 1: The OpenPiton SoC. The modules in blue, green, and yellow are the original ones, ones added in previous years, and the ones added for the Hack@DAC'21, respectively.

(iv) test programs written in $\mathrm{C}$ to do a full-system simulation. Participants are at liberty to use any existing or self-developed tools and techniques to detect both inserted and native bugs in the SoC. Teams score points by providing description of the bugs and exploit codes that violate the desired security properties but the codes need to be executed only by an unprivileged user. The participants are not allowed to modify the privileged kernel code when creating the exploits. Table 1 summarizes the access permissions at various levels used for the competition.

SoC Design. We used the OpenPiton SoC [6] for this year's competition which uses a RISC-V ISA-based CVA6 core [10]. OpenPiton is "the world's first pen-source, general-purpose, multithreaded, manycore SoC platform" [6]. We use a 2-core configuration for the competition where core 0 is the primary core that runs the executa-bles whereas core 1 is used as the Direct Memory Access (DMA) engine. We add multiple security peripherals and features like AES, SHA256, RSA, and HMAC crypto-engines, pseudo-random number generator (PRNG), FUSE memory, reset controller (RST), access-control (ACCT), register lock (REGLK), and crypto-checks at bootup to the SoC as shown in Figure 1.

Operating system. We use proxy kernel (PK) [1], a lightweight execution environment for our SoC that handles I/O-related system calls by proxying them to the host computer. It enables virtual memory addresses and memory isolation. PK includes machine mode firmware code to load the data in the FUSE memory to the corresponding peripherals. The executable of the user program is embedded into the PK elf to generate a single executable that begins with running firmware code, switches to the user privilege level, and runs the user application. The participants are not allowed to modify the PK firmware code to trigger any bugs but can verify it to detect firmware-hardware bugs in it.

Scoring Mechanism. Points are allotted to bug description, triggering codes, and proposed mitigation details of each bug. This scoring mechanism, inspired by Common Vulnerability Scoring System (CVSS) [19] is used to encourage attacker-like thinking [20].

![0196f2bf-3d2f-78e4-904b-586b392bfbd6_1_1030_227_524_204_0.jpg](images/0196f2bf-3d2f-78e4-904b-586b392bfbd6_1_1030_227_524_204_0.jpg)

Figure 2: The register lock inserted in the Hack@DAC design.

### 2.1 Competition Resources

To reduce the load on participants, we provide support for fully set up and ready to use simulation and emulation options. For Phase 1, we provided the participants with a virtual machine (VM) image with the RISC-V ISA compiler and the open-source hardware simulation tool Verilator pre-installed to compile the executables and simulate the SoC. For Phase 2, we provided each finalist team with AWS cloud computing instances [2] that are equipped with a full fleet of Synopsys hardware verification tools like VCS, VC Formal, Euclid, VC Spyglass, Z01X, and Verdi with help from our industry partner, Synopsys [5].

## 3 CONSTRUCTING REAL-WORLD BUGGY BENCHMARKS

Open-source SoC designs natively consist of SoC system-bus, processor core, and basic peripherals like UART, interrupt controller, and bootrom. To facilitate insertion of security bugs that span across all the hardware CWEs, we insert security components and features like register locks, fabric access control, encryption engines, and JTAG password protection into the SoC. Our collaboration with semi-conductor partner, Intel, allows us to insert bugs mimickly the known CVEs and commonly encountered hardware vulnerabilities in industry scale designs. The bugs we insert can be broadly classified into four types: (i) denial-of-service, (ii) privilege escalation, (iii) sensitive information leakage, and (iv) code injection. These bugs are inserted in various locations in the SoC like bootrom, crypto-engines, DMA, debug and JTAG interface, processor core, and SoC bus interconnect.

Most of the bugs we insert in hardware are exploitable from software to mimic the real-world scenario where firmware and operating system runs on top of these hardware designs. We inserted a total of 78 bugs for Phase 1 of the competition. Phase 2 SoC consists of variants of the bugs from Phase 1 along with 10 newly inserted bugs. Apart from the bugs we inserted, these SoCs may also contain bugs that were part of the original design; we call such bugs native bugs. It is commonplace for participants to uncover many native bugs in our competitions.

### 3.1 Case Studies

We now briefly present certain bugs inserted for the competition. JTAG causes secret leakage through register lock. The Joint Test Action Group (JTAG) is an industry standard for testing designs post fabrication [4]. Its dedicated debug port enables low-overhead access of test registers that can reflect the functionalities of the design without requiring external interfaces. Hence, JTAG is widely used for debugging embedded systems such as SoCs; verification engineers use JTAG to access the data of SoC's components through the debug module to verify their correctness. Register lock is a SoC security feature that protects the sensitive data in the peripherals such as AES, SHA-256, and PRNG. The registers of peripherals that can potentially leak sensitive data are locked by the register lock, restricting access on them outside the peripherals.

<table><tr><td>ID</td><td>Type</td><td>Description</td></tr><tr><td>1</td><td>Unprivileged software at user-level mode</td><td>Executes on the core with user-level privileges but may exploit bugs to mount privilege escalation attacks</td></tr><tr><td>2</td><td>Physical attacker</td><td>Has physical possession of the device</td></tr><tr><td>3</td><td>Privileged software at supervisor mode</td><td>Executes on the core with Supervisor mode privilege but may target other higher privilege levels</td></tr><tr><td>4</td><td>Authorized debug access</td><td>Has the ability to unlock and debug production device</td></tr></table>

Table 1: Threat model used for the competition.

Listing 1: Secret leakage between JTAG and register lock.

---

always @(posedge clk_i)begin

	if(~rst_ni ||jtag_unlock) begin

		for $\left( {j = 0;j < 6;j = j + 1}\right)$

			reglk[j] <= h0;

	end else if(en &&we)

		...

end

---

To be compatible with JTAG, a SoC needs to disable register lock so that JTAG can access registers of peripherals in the debug mode. One way to achieve this mechanism is to clear the setup of register lock as shown in Listing 1; the JTAG directly resets all signals of the register lock to access sensitive registers of other peripherals.

However, similar to CWE-1234[17], such implementation lacks logic to set the register lock to the general state after the debugging process, which makes the sensitive registers of all peripherals exposed to unprivileged users. An attacker can let SoC enter and then terminate the debug mode to access sensitive data.

In order to detect this bug using conventional verification, one can formally model the security behaviors of the SoC in debug mode and in regular mode, verifying if any data leakage can happen excepted in debug mode. However, the large search space caused by the design complexity of the SoC may exceed the capacity of formal tools. Moreover, modeling a design requires a thorough understanding of its behaviors. As SoC vendors may buy IPs from other vendors for some SoC features, verification engineers can only model them as a black-box and will not know if there is any vulnerability inside. Therefore, lots of stealthy vulnerabilities may happen between the communication of peripherals.

SHA is not checked during bootrom test. Secure Hash Algorithm (SHA) is a set of cryptographic hash functions that has been widely used in security engines and protocols. The Hack@DAC's SoC implements a SHA-256 engine, producing a 256-bit hash value given a 512-bit input. bootrom is a small piece of write-protected flash embedded in the SoC, which contains the code that is executed during a power-on or reset. The bootrom of Hack@DAC's SoC contains a function to verify the correctness or validity of its peripherals. The bootrom will give a fixed input for each peripheral to verify if the feedback from it is the same as the expected result. If not, the bootrom will let SoC execute a set of instructions to halt all processes. Inspired by CWE-1240 [17], we have inserted a bug that disables the verification function of the SHA-256 engine; the system can not discern if the SHA-256 engine works properly before executing a user's program, thereby causing data leakage or denial of service (DoS).

PRNG's entropy source' polynomial is zero. Hack@DAC's SoC contains a pseudo-random number generator (PRNG) with four

Linear-feedback shift register (LFSR) entropy sources.The PRNG sends hard-coded seeds and polynomials to four sources and concatenates the outputs from them as the final 64-bit random number. Since the output of LFSR is deterministic, knowing either the seed or polynomial can derive the another one and determine the logic of entire PRNG further. One LFSR source has zero as its polynomial. Because LFSR is based on the exclusive-or (XOR) logic, this bug makes the output the same as the seed, leaking the seed information.

![0196f2bf-3d2f-78e4-904b-586b392bfbd6_2_922_737_728_740_0.jpg](images/0196f2bf-3d2f-78e4-904b-586b392bfbd6_2_922_737_728_740_0.jpg)

Figure 4: Classification of bugs based on their vulnerability.

## 4 RESULTS AND OBSERVATIONS

We briefly summarize the observations made from the bug submissions in both phases in this section.

Bug classification. We received a total of 33 and 97 bug submissions during Phase 1 and Phase 2, respectively; Figure 3 and Figure 4 show the distribution of bugs based on their locations and vulnerability class respectively. Bugs in SoC indicate the number of inserted and native bugs, bug submissions indicate the number of bugs submitted by the participants, and the bugs detected indicate the number of unique bugs detected by the participants.

Phase 1. The participants mainly used manual inspection to detect bugs in Phase 1. They used approaches like prioritizing the high-risk areas like JTAG password logic, access-control mechanism, and register locks. However, this approach did not help them detect cross-layer bugs since manual analysis is not a scalable approach and requires a thorough understanding of the design. Dynamic verification, where participants wrote test cases to simulate the design is another approach used. While some participants used assertion-based simulation techniques, others used $\mathrm{C}$ programs in the software abstraction level to test the designs.

Bugs were not detected using formal methods in Phase 1 because: (i) the participants prioritize their time on familiarizing themselves with the SoC during, which they tend to manually inspect the SoC; and (ii) the participants were provided with a VM image that only consists of a simulation tool, Verilator, and no formal tools pre-installed. Thus, usage of the formal tools in Phase 1 is hindered due to the unavailability of licensed formal tools and the expertise and time required to correctly configure them with the SoC.

Phase 2. Unlike Phase 1, we observed a huge influx in the usage of formal and automated tools in Phase 2, thanks to all the licensed Synopsys tools provided with the AWS compute instance. Highlights from the Phase 2 submissions are:

Team Gator Cat Community found FSM state transition-related bugs using a custom static analysis tool. Team GOREHOWL demonstrated a confused deputy attack from HMAC crypto-engine to FUSE memory address look-up table peripheral using the Formal Property Verification (FPV) application of Synopsys VC Formal tool. Team GOREHOWL wrote a custom tool to trace register lock propagation logic to detect bugs. Team fsociety_alpha created a custom tool to parse the configuration of the physical memory protection (PMP) registers to detect the bugs. Team fsociety_alpha wrote an exploit to leak the secret key of one of the AES encryption engines in the SoC. Team CICA used the Formal Property Verification (FPV) application of Synopsys VC Formal tool to find reset-related bugs. Team SAT2 wrote exploit code to extract RSA's encrypted output from hardware peripheral registers after reset.

## 5 NEED FOR NEW TECHNIQUES: HARDWARE FUZZING

Fuzzing is a dynamic verification technique that relies on mutations and feedback to automatically verify designs. Mutations modify the existing input data to generate new inputs, while the feedback drives the fuzzer towards rapid detection of new bugs in the designs. Software security uses successful fuzzing techniques like AFL fuzzer [14], libfuzzer [15], and Google OSS-Fuzz [21].

As the existing hardware verification tools are unable to keep up with increasingly complex modern hardware designs, fuzzing has emerged as a promising solution for hardware security verification. Recent research has led to multiple fuzzing techniques targeting hardware $\left\lbrack  {7,{11} - {13},{18},{22}}\right\rbrack$ . [13] facilitated fuzzing of hardware on FPGAs to speedup the bug detection while [11] improved its coverage feedback mechanism. [7] modified [13] to improve performance of targeted fuzzing of specific modules in the hardware. [12] uses multiple coverage feedback mechanisms to capture the different activity in hardware components and also ensures compatibility with traditional hardware verification flow. [22] takes a different approach of fuzzing the software model of the hardware. While some fuzzers like $\left\lbrack  {{11},{12}}\right\rbrack$ specifically target processor cores,[18] builds and fuzzes security properties to verify the SoC peripherals.

The limitations of existing hardware fuzzing techniques are: (1) Not all existing hardware fuzzers are capable of detecting the bugs. Of the existing techniques, only $\left\lbrack  {{11},{12}}\right\rbrack$ demonstrated their ability to detect bugs. (2) Hardware fuzzers either need a golden reference model (GRM) $\left\lbrack  {{11},{12}}\right\rbrack$ or manually written security assertions [18] to detect the bugs; they may not be available for some hardware designs. (3) Hardware simulations are very slow and require time proportional to the size of the design under test (DUT) to detect the bugs; only $\left\lbrack  {{11},{13}}\right\rbrack$ support fuzzing through emulation to speedup bug detection. (4) Some hardware fuzzers are designed for targeted fuzzing (efficient only when fuzzing specific modules of the DUT) [7] while some others do not account for covering all the hardware behaviours (like latches, floating wires, combinational logic) $\left\lbrack  {{11},{13},{22}}\right\rbrack$ because of type of coverage feedback mechanisms used. Thus, we need a automated, scalable, fast, and efficient fuzzer that is not limited by its applicability in terms of hardware simulators or HDLs and has built in mechanisms to detect security vulnerabilities in hardware.

## 6 CONCLUSION

This paper summarizes the details and observations from Hack@DAC 2021 competition organized in collaboration with our industry partners Intel and Synopsys. Our analysis highlights the needs for new techniques, such as hardware fuzzing. Hardware fuzzing is a long way from becoming the most prominently used verification technique in hardware. We envision a change in this trend where fuzzing will become the primary means of detecting bugs and thus scoring points in our competition with development of new hardware fuzzing techniques and works towards this goal through the benchmarks and observations released through our competitions.

## ACKNOWLEDGMENTS

Our research work was partially funded by the US Office of Naval Research (ONR Award N00014-18-1-2058). We thank our industry partners from Intel and Synopsys for their support.

## REFERENCES

[1] 2021. Proxy Kernel source code. https://github.com/riscv/riscv-pk.Accessed: 2021-04-28.

[2] 2022. AWS. https://aws.amazon.com/?nc2=h_lg.Accessed: 2022-04-18.

[3] 2022. Hack@EVENT.https://hackatevent.org/. Accessed: 04/08/2022.

[4] 2022. IEEE Standard Test Access Port and Boundary-Scan Architecture. https: //standards.ieee.org/ieee/1149.1/1727/. Accessed: 04/19/2022.

[5] 2022. Synopsys. https://www.synopsys.com/.Accessed: 2022-04-18.

[6] Jonathan Balkind, et al. 2016. OpenPiton: An Open Source Manycore Research Framework. (2016), 217-232. https://doi.org/10.1145/2872362.2872414

[7] Sadullah Canakci, et al. 2021. Directfuzz: Automated Test Generation for RTL Designs using Directed Graybox Fuzzing. 58th ACM/IEEE Design Automation Conference (2021), 529-534.

[8] Wen Chen, et al. 2017. Challenges and Trends in Modern SoC Design Verification. IEEE D&T 34, 5 (2017), 7-22.

[9] G. Dessouky, et al. 2019. HardFails: Insights into Software-Exploitable Hardware Bugs. USENIX Security (2019), 213-230.

[10] OpenHW Group. 2021. Ariane SoC source code. https://github.com/ openhwgroup/cva6. Accessed: 2021-04-28.

[11] Jaewon Hur, et al. 2021. DifuzzRTL: Differential Fuzz Testing to Find CPU Bugs. IEEE S&P (2021), 1286-1303.

[12] R. Kande, et al. 2022. TheHuzz: Instruction Fuzzing of Processors Using Golden-Reference Models for Finding Software-Exploitable Vulnerabilities. USENIX Security (2022).

[13] Kevin Laeufer, et al. 2018. RFUZZ: Coverage-directed Fuzz Testing of RTL on FPGAs. IEEE/ACM ICCAD (2018), 1-8.

[14] Lcamtuf. [n. d.]. American Fuzzy Lop (AFL) Fuzzer. http://lcamtuf.coredump.cx/ afl/technical_details.txt. Accessed: 04/08/2021.

[15] LLVM. 2022. LibFuzzer - a library for coverage-guided fuzz testing. https: //llvm.org/docs/LibFuzzer.html. Accessed: 04/08/2022.

[16] Ben Marshall. 2019. Hardware Verification in an Open Source Context. ODSA (2019).

[17] MITRE. 2021. Hardware CWEs. https://cwe.mitre.org/data/definitions/1194.html.Accessed: 2021-04-28.

[18] Sujit Kumar Muduli, et al. 2020. Hyperfuzzing for SoC Security Validation. IEEE/ACM ICCAD (2020), 1-9.

[19] NIST. 2022. CVSS. https://nvd.nist.gov/vuln-metrics/cvss.Accessed: 2022-04-18.

[20] Ahmad-Reza Sadeghi, et al. [n. d.]. Organizing The World's Largest Hardware Security Competition: Challenges, Opportunities, and Lessons Learned. ([n. d.]), 95-100.

[21] Kostya Serebryany. 2017. OSS-Fuzz - Google's continuous fuzzing service for open source software. USENIX Security (2017).

[22] Timothy Trippel, et al. 2021. Fuzzing Hardware Like Software. arXiv preprint arXiv:2102.02308 (2021).

[23] Andrew Waterman, et al. 2014. The RISC-V Instruction Set Manual. Volume 1: User-Level ISA, Version 2.0. https://content.riscv.org/wp-content/uploads/2017/ 05/riscv-spec-v2.2.pdf.

[24] Wooseung Yang, et al. 2003. Current Status and Challenges of SoC Verification for Embedded Systems Market. IEEE SOCC (2003), 213-216.