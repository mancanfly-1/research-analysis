本周工作进展和下周计划
2018.5.19~2018.5.25
本周计划目标：
1. 通过优化将overhead降低到20%以下
2. 论文完成90%

本周完成工作：

三个优化手段已经实现前两个。
我们目前的工作目的是验证方案的可行性，因此我们的方案中并不一定要求插桩的正确性和安全性。
因此目前的实现中我们暂时没有考虑register spill和range analysis。


1. 优化：当一个指针被check过，那么之后使用这个指针进行读取值都不应该再check   
优化实现：对一个指针的操作可以被分为load和store：  
对于load操作，新建一个PtrLoadList，当一个指针没有被扫描过（不在PtrLoadList)中，则添加对这个指针的check，并且将指针放入PtrLoadList。如果这个指针已经被扫描过，则添加一个llvm metadata（注释信息），指明第一次check的位置，方便之后再进行优化。
对于Store操作类似。
每次扫描过一个函数，则将List清空。

2. 优化：对数组读写进行优化，gep指令，如果index不超过guard zone，则只对数组基址check一次
优化实现：对于顺序数据（包括数组和结构体）的读写，在LLVM IR中需要两步进行，第一步是通过getelementptr指令，计算出需要访问的地址。第二步是使用第一步算出的地址访问。   
getelementptr接受三个参数：访问数据的类型，访问数据的基址，以及访问数据的索引。
维护一个BasePtrList，出现新的BasePtr的时候对BasePtr插入load 和store 的check。对于使用baseptr计算出来的指针，则可以跳过插入，并插入一个llvm.metadata。
每次扫描过一个函数，则将List清空。

3. 对loop 进行检查，如果循环次数可以大概确定，则可将检查上提。
正在实现。

学习的资料：
包括一些llvm 的maillist中，试图学习llvm相关的优化。
http://lists.llvm.org/pipermail/llvm-dev/2016-May/100093.html
还有LLVM的文档，找llvm 优化pass 学习：
http://llvm.org/doxygen/classllvm_1_1ScalarEvolution.html

论文周末发修改版。

下周计划：
周末出数据和论文修改版，方便之后老师review，以及修改。


2018.5.12~2018.5.18
1. 这周阅读相关的论文，了解相关的使用software fault isolation的实现方式和优化手段。
结合我们的工作，给出相应的优化手段。
所有的优化都是针对内存的读写的（即data region 的保护），因为control flow integrity的插桩没有太多优化的余地（每一个call 和ret，jump都必须插桩，无法避免）。

首先我们需要的工作需要分为三个步骤：
第一个步骤是在IR层对所有可能的load store 指令插入对应的check 函数。这个步骤在上周已经完成。
第二个步骤是在IR层进行相应的优化处理，有一些无法确定的优化，则放在第三步进行处理。
第三个步骤是在后端，当寄存器分配完成后，对指令插桩进行检查，如果符合条件，则删除这个插桩。

其中第二个步骤无法确定优化能否apply，主要是因为存在register spill问题。
register spilling 的意思是，在IR层，虚拟寄存器是无限多的，但是实际上硬件上的寄存器是有限的。因此可能存在某个指针因为硬件寄存器不够用而被写回到内存上之后再读出来的情况。这就是register spill。
因为根据CFI的威胁模型，内存是不可信的，被写回内存的指针需要重新检查。因此我们几乎所有的优化都必须到寄存器分配之后才能确定该优化是否可行。

因此我确定的优化手段主要有一下三点（都是基于观察的启发式的优化）：
1. 当一个指针被check过，那么之后使用这个指针进行读取值都不应该再check（如果该指针被写回到内存中后，则需要check）
2. 对数组读写进行优化，gep指令，如果index不超过guard zone，则只对数组基址check 一次（如果数组基址写回内存，则需要重新check）
3. 对loop 进行检查，如果循环次数可以大概确定，则可将检查上提。

以上是三个高层次的检查，为了了解清楚如何使用llvm 实现这些检查，同时还阅读了UIUC course CS526
Iterative Dataflow Framework 123 三节，并对照算法分析llvm 中的 ConstantProp.cpp pass。
http://misailo.web.engr.illinois.edu/courses/526/  

同时我们对之前的代码进行了一定的改正，例如将之前的调用C函数的内联汇编改为调用LLVM的内联汇编。

阅读的论文主要是上周周报中提到的两篇论文。相应的阅读报告也已放在github上。
Combining Control-Flow Integrity and Static Analysis for Efficient and Validated Data Sandboxing
Strato: A Retargetable Framework for Low-Level Inlined-Reference Monitors

洪亮：
负责benchmark 的移植，使用我们的工作对spec 2006中的程序进行插桩，结果大概是51.3%的overhead。

下周计划：
1. 实现初步的优化，将spec 2006中51.3%的overhead 降低到20%
2. 基本完成（90%）论文的内容。

2018.5.5~2018.5.11
这周完成拥有最基础功能的software fault isolation 实现，我们需要对一个应用程序进行两个部分的处理：
1. 数据访问的检查
2. 代码跳转的检查
因此这周完成的llvm pass 主要有以下两个功能：

1. 对所有的load store 指令进行插桩
在IR层，所有对内存的访问都是load 和store 指令
所以我们对load 进行读内存的检查，只能读取可读区域。
对store 进行写内存检查，可以读取该程序所有区域。

2. 对所有的indirect call、indirectbr 和ret指令进行修改

IR层区分direct和indirect call/jump，direct指的是直接对某个函数或labeled的地址进行跳转、调用
indirect是指目标地址来自寄存器或内存，因此我们只需要对indirect进行检查。

因为我们初期只需要模拟性能，在每一个需要检查的跳转指令之前插入一条add指令和一条mpx指令即可。
对于ret 指令，首先使用llvm.returnaddress 获得当前函数的返回值，再插入add 指令和mpx指令以及jmp指令。


下周计划：
1. benchmark spec 2006基准测试现有实现的性能。
2. 研究现有SFI的优化技术，设计我们的SFI优化方案。为此，需要读下面几篇论文：
读论文
Principles and Implementation Techniques of Software-Based Fault Isolation
range/loop analysis
这是一篇关于Software fault isolation的 survey。
Combining Control-Flow Integrity and Static Analysis for Efficient and Validated Data Sandboxing
Strato: A Retargetable Framework for Low-Level Inlined-Reference Monitors
这两篇是Software fault isolation实现优化的文章。

2018.4.29~2018.5.4
1. knowing llvm

What is llvm infrastructure
http://www.aosabook.org/en/llvm.html

llvm getting start （build a llvm from source code)
write a simple Hello world pass
http://releases.llvm.org/5.0.1/docs/WritingAnLLVMPass.html

how to write a useful pass in llvm:
http://llvm.org/docs/ProgrammersManual.html

了解 LLVM 的IR格式：
http://llvm.org/docs/LangRef.html

上述LLVM的文档都是选读略读。

2. knowing compiler background

Reading "Engineering a compiler" chapter 1 and 5.
Knowing what's IR and the consideration when compiler design a IR format.
Knowing what's SSA and why ssa can be use for control flow and data flow analysis.

read below course slides 1 and 2:
http://misailo.web.engr.illinois.edu/courses/526/
了解SSA form 是什么，为什么要用SSA，因为LLVM的IR是SSA 形式的，我们的代码也是对IR修改，未来的优化也是如此。

3. 代码写了一个简单的开头.

阅读memsentry的源代码.
write a pass iterator all store/load instructions.

Plan:
10 May.
write a minimal working implementation that instruments every store/load and call/ret of a sample enclave program，而且链接初始化MPX的库。
需要解决问题：如何在SGX中初始化MPX，sgx程序没有初始化阶段。
target： workable pass

17 May.
Learning control flow and data-flow opt.
compare instrumented programs and uninstrumented programs.
Spec 2006 Benchmark

24 May.
pass with optimization function.能够对指令的插入进行优化。
optimized benchmark data. 优化后的数据。



2018.4.22~2018.4.28
- 本周工作计划1：了解MPX以及 Software fault isolation 内容
- 完成情况：大概了解
- 本周工作计划2：学习llvm
- 完成情况：根据如下课程要求，编译了LLVM，并写了个hello的pass。
http://misailo.web.engr.illinois.edu/courses/526
目前在学习课程的slides，已学习两课。

下周计划：
对llvm有一定的了解，可以给出一个方案做原型系统

2018.4.9~2018.4.22
- 本周工作计划1：学习Rust语言
- 完成情况：大概了解
- 本周工作计划2：看Graphene-sgx 和Scone等Libos的实现方式
- 完成情况：了解
- 本周工作计划2：了解MPX以及 Software fault isolation 内容
- 完成情况：在看论文


2018.4.2~2018.4.8
- 本周工作计划1：开题准备
- 完成情况： 已失败
- 本周工作计划2：休息
- 完成情况：压力太大，休息了几天

下周计划：
想论文能否放宽条件
学习rust sgx sdk。

2018.3.26~2018.4.1
- 本周工作计划1：开题准备
- 完成情况： 已放弃
- 本周工作计划2：论文
- 完成情况：写了个introduction

下周工作计划：
写论文

2018.3.19~2018.3.25
- 本周工作计划1：进行存储工作测试
- 完成情况： 延迟在50 ms左右，仍有优化空间。较其他工作要好。
- 本周工作计划2：写一个文件系统的测试，比较原生文件系统和SGXSDK 中的protected fs 之间性能差距，判断优化空间
- 完成情况：已完成，小block (256B)读写时，性能差距很大，有较大的优化空间

下周工作计划：
专心完成开题报告。
最近事情有点多，可能助教作业的批改进度有点慢。


2018.3.12~2018.3.18
- 本周工作计划1：进行测试
- 完成情况： 延迟在50 ms左右，仍有优化空间。较其他工作要好。
- 本周工作计划2：熟悉switchless现状
- switchless分了几个小点，包括fastcall负责加速系统调用，memory management帮助用户实现自定义内存管理。

下周工作计划：
- 写一些小的bench，测试SGX sdk中文件操作的性能。
- 写开题报告。

2018.3.4~2018.3.11
- 本周工作计划1：完善工作
- 完成情况： coding
直接随机的proxy并不是一个可靠的机制，我重新考虑这个问题，这涉及到网络拓扑结构，我们需要一个网络拓扑结构，满足以下两点：
1. 从任意一个index，通过proxy 都可以到达集群中所有的节点
2. 任意一个proxy都可以到达集群中所有的节点。
总结成一句话，任意一个index，通过任意一个proxy，都可以到达任意节点。
最简单的情况是当集群中机器数量比较小的时候，full connect是可以接受的。
但是当集群中的机器数量很大的时候，full connect 导致traffic 很大，不能scale out，所以我们需要将机器分组，加入集群机器2500台，如下处理：
1. 将2500台机器分成50组，每个组50台。
2. 机器每一次给group中其他机器都发送一个包。保证group内部是full connect的。
3. 机器每一次在每一个group中随机一个机器发送一个包。
这样，每个机器每个round都只需要发送99个包，集群中的流量从full connect的N^2 降低到 2*N^(3/2)
当一个请求过来的时候，index先随机找group A内部一个机器X负责转发，X找到对应目标机器（store）的group，在这个group B中随机选择一个机器Y发送，最后由Y发送给目标机器Store。
因此机器的延迟理论上变成当前的4倍。

在我们的工作中，因为只有10台机器，所以上面的机制不需要实现。代码上，已经将proxy机制改为了full connect。
另外也将Oblivious 的API替换。
并行的工作进行中。
关于匿名通信的文章，从远到近可以看得包括：
sosp 15：vuvuleza
osdi 16：pung
sosp 17： Stadium 和 atom

还有Prifi

相应的论文中都有对之前匿名通信的总结。

2018.2.26~2018.3.4
- 本周工作计划1：完善工作
- 完成情况： coding

准备写oblivious的话，接下来的工作计划可以分为两个部分：
代码：
1. Oblivious中，需要将原来的objectkey，value改成（indexkey，key，value）
2. 并行化处理，对相同的文件的请求需要进行并行化处理，两个put如何并行？
3. 测试，性能优化

论文：
1. 从性能上分析是否需要proxy，还是只需要mixnet。
2. 如何证明安全性，如何证明是oblivious的


2018.1.15~2018.1.19
- 本周工作计划1：code
- 完成情况： coding

现在机器之间的通信已经建立起来了，接下来是写协议

年度总结：
去年想要基于Intel SGX做一些系统相关的研究，为此花了三个月了解SGX，而后6月份开始着手如何隐藏文件的位置保证用户数据的安全。中间反复讨论过很多次，一直到12月有比较成熟的想法。这一个多月一直在实现。因为工作比较复杂，所以实现的很慢，中间还遇见很多坑。
除了具体的事物，去年阅读了很多的论文，学会了如何通过阅读论文快速了解一个领域，相信之后对其他的领域也能快速上手。

明年计划：
想要将手上的工作尽快完结，之后可以协助田洪亮做switchless OS，他在Enclave中关于性能优化有一些新的idea。
之后还没有安排。
约谈：
平时most of time都在122，大部分时间都可以。

2018.1.3~2018.1.12
- 本周工作计划1：code
- 完成情况： coding

- 下周计划工作：
  最近一直在coding

2017.12.25~2018.1.2
- 本周工作计划1：code
- 完成情况： 本来想这周写完的，但是openssl的RSA,PKCS 不太了解，所以写的稍微慢

- 下周计划工作：
  写论文安全分析部分


2017.12.15~2017.12.24  
- 本周工作计划点1：code  
- 完成情况：完成SSL性能测试，和enclave 中执行aes 128 gcm加密速度和普通的运行环境速度类似。对于4k或8K的块加密，速度达到5Gb/s   
- 本周工作计划点2：code 实现论文逻辑的主体  
- 完成情况： 主要分为两个部分，一个部分是非安全部分的负责将数据包通过网络发送出去，或者接受其他机器发送过来的数据。或者读取磁盘和写磁盘。安全部分和非安全部分通过四个lock-free的queue交互。这些代码之前已经写完了。
另一部分是系统的关键流程，在enclave中运行，根据两个DHT和两个 table处理用户的数据。这个部分正在写。  

2017.12.11~2017.12.15
- 本周工作计划点1：code
- 写ssl性能测试。

2017.12.4~2017.12.10
- 本周工作计划点1: write implement
- 完成情况：half，写了整个系统的序列图，定义详细系统的行为。
- 本周工作计划点2：code
- 开始写ssl性能测试。

- 下周计划：
- 想想怎么写implement 中的Oblivious shuffle
- code

2017.11.27~2017.12.3

- 本周工作计划点1: write Design
- 完成情况：finished


- 下周计划：
- 想想怎么写implement 中的Oblivious shuffle
- code

2017.11.19~2017.11.26

- 本周工作计划点1: write problem definiation
- 完成情况：finished
- 本周工作计划点2: Fix two problem: Oblivious response time issue and Crash may not good under virtual node.
- 完成情况：Done.

2017.11.13~2017.11.18

- 本周工作计划点1: write introduction
- 完成情况：finish half
- 本周工作计划点1: reading oblivious paper
- 完成情况：summary will be update later

2017.10.6~2017.11.10

- 本周工作计划点1: Reading oblivious store papers
- 完成情况：figure out some consideration.
- 本周工作计划点1: Finish abstract
- 完成情况：Finished, but not reviewed by kang chen.

- 下周计划：
- Finish design of paper

2017.10.23~2017.11.4

- 本周工作计划点1: Reading paper of sosp 17 and attend it.
- 完成情况：Finished
- 本周工作计划点1: Finished Slides of intel sgx storage
- 完成情况：Finished

- 下周计划：
- Finish project of Intel SGX storage.
- Finish introduction part.

2017.10.16~2017.10.22

- 本周工作计划点1: Understanding Oblivious storage.
- 完成情况：Finished
- 本周工作计划点1: Find the fix of our approach
- 完成情况：Finished

- 下周计划：
- Update the draft of paper.
- Finish project of Intel SGX storage.

2017.10.09~2017.10.13

- 本周工作计划点1: Finish project of Intel SGX storage.
- 完成情况：Finished outside enclave world including rpc server and client, file server. Finished some enclave function. Finished the interface between enclave and host Application.
The problem is I have to understand oblivious store to continue the functionality inside enclave.

- 下周计划：
- Finish project of Intel SGX storage.
- Understanding Oblivious Memory and store.

2017.09.23~2017.09.30

- 本周工作计划点1: Finish project of Intel SGX storage.
- 完成情况：Almost finished outside part. build an RPC server as network connection part.

- 本周工作计划点2:Finish the PHD thesis proposal
- 完成情况：Failed.

- 本周工作计划点3:Finish Compiler PA2 testcase and homework.
- 完成情况：Finished


- 下周计划：
- Finish project of Intel SGX storage.

2017.09.16~2017.09.22

- 本周工作计划点1: Finish project of Intel SGX storage.
- 完成情况：Starting

- 本周工作计划点2:Finish another idea.
- 完成情况：Finished. For more detail, check the proposal repo in github.

- 本周工作计划点3:Finish PA3 of Compiler.
- 完成情况：Finished


- 下周计划：
- Give the final talk of Phd proposal
- Finish project of Intel SGX storage



2017.09.11~2017.09.15

- 本周工作计划点1: give a talk to about proposal of Phd Thesis.
- 完成情况：Finished but one idea missing
Maybe Database is an good idea. We do not need to consider the memory limit, just think about oblivious store.

- 本周工作计划点1: PA3 of Complier course
- 完成情况：Finished two feature of five.

- 下周计划：
- Finish project of Intel SGX storage.
- Finish another idea.
- Finish PA3 of Compiler.


2017.09.05~2017.09.10

- 本周工作计划点1: Give full version of Phd Proposal
- 完成情况：Finsh almostly except I still need one idea.
在目前这个我们想出来的场景“OS不安全的云中心中保护用户的数据”，用户是谁？用户的特点是啥？用户的应用特点是啥？用户最想得到的安全保证是哪些？这些安全保证是否有个优先级？针对哪些用户应用，我们可以解决哪些安全保证？当前是否已经有一些安全保证的相关/类似实现了？我们比它们好在哪里？或者我们是最早做的，没有可比性？如何评价我们的技术方案？
用户是谁：用户是使用对象存储的人，一般是程序员。比如Facebook中的对象存储的用户是Facebook，Facebook将Facebook用户上传至Facebook的图片和视频上传至对象存储系统中。这些数据通常在Facebook的系统中不会有备份。
用户的特点是啥，用户的应用特点是啥：用户的特点就是会一定的程序开发，使用对象存储开发出程序给第三方使用。
用户最想得到的安全保障是哪些？这些安全保证是否有个优先级？：数据不能丢失>数据不能被破坏>数据不能被泄露。前两个保障在以前的系统中已经完成的较好了。我们不能退步。
当前是否已经有一些安全保证的相关/类似实现了？关于前两个保障是其他领域，关于数据泄露有Facebook的对象存储F4，或者很多安全公司的加密存储系统产品。
我们比他们好在哪：我们的所有机器都可以处理加解密操作，同时所提供的保障更强。
如何评价我们的技术方案：暂时可以考虑的如下，
1. 安全性，包括可以容忍多少台机器的崩溃
2. 性能，普通请求处理的性能。机器崩溃后重启的性能。

>> chy 20170912 没看到上周的周报告

另外，我希望，合理安排时间，确定下面事情的重要性，给出时间表，顺利完成：

- 开题
- 做原型系统
- 读论文
- 写论文

对于课题，我希望了解：在目前这个我们想出来的场景“OS不安全的云中心中保护用户的数据”，用户是谁？用户的特点是啥？用户的应用特点是啥？用户最想得到的安全保证是哪些？这些安全保证是否有个优先级？针对哪些用户应用，我们可以解决哪些安全保证？当前是否已经有一些安全保证的相关/类似实现了？我们比它们好在哪里？或者我们是最早做的，没有可比性？如何评价我们的技术方案？做个表，一个一个列清楚，大家可以比较容易理解。

>> chy 20170904 没看到上周五的周报告。8月缺了1/4

2017.08.29~2017.09.04

- 本周工作计划点1: Survey Database for SGX
- 完成情况：Read paper :
Architecture of a Database System.
Queue encrypted data
[dbsec17]HardIDX- Practical and Secure Index with SGX
[NDSI17]Opaque- An Oblivious and Encrypted Distributed Analytics Platform
[SIGMOD17]Cryptanalysis of Comparable Encryption in SIGMOD’16


2017.08.21~2017.08.28

- 本周工作计划点1: Think more detailed about Problem in Phd proposal.
For Enclave migration, I have consider three problem:  
1. Is it worth to do it? Is it needed in Datacenter?   
Of course it is. For one aspect, the load-balance is needed for energy efficiency(Online migration). On the other hand, the machine requires update(Off line).   
2. How can we do it?  (functionality)
It's not possible to do it from host application/OS since the Intel SGX is designed to against them. And one of important principles is that every chips keeps there secret themselves. So only the enclave can do it.   
How to do it is a subtle job. we can implement an module in enclave or a LibOS in enclave to migrate the whole enclave. The other way is we defined some jobs and only transform the pre-defined states in enclave.   
3. How to make it secure?(security)  
How to make sure the migration How to persuade the customer that the migration/transform will not break the security guarantee.   



2017.08.14~2017.08.20

- 本周工作计划点1: give three ideas/problem for PhD Thesis Proposal.  


- 完成情况：
Our topic is the security for enterprises using public clouds. We have found three points:
1. The security of Storage resource.
2. The security of Compute resource
For computer resource in SGX, we consider two potential idea: Enclave migration and malicious code detection.
3. The security of communication
we may leverage the trusted execution environment to develop an decentralized anonymous communication system under fully compromised infrastructure.

- 下周计划：
1. For the code of storage project, finished remote attestation.
2. For the Phd proposal, read two papers about process migration on ASPLOS 2017 and anonymous communication system on OSDI 16.


2017.06.08~2017.06.16

- 本周工作计划点1: Give a full solution of replication of SGX

- 完成情况： finished execpt the network traffic attack remains to be fix

For now, the discussion of idea for Date 2018 is almost finished. Here is my plan:
1. Finish the project and paper of this paper before August.
2. Start discuss Fast paper once I finished prior works (expected to be finished at July).
3. Finished the project at August
4. Start to write the paper at begining of September.



2017.05.26~2017.06.07

- 本周工作计划点1: Give a sketch of solution

- 完成情况： Gived a model of Data layer, called Replication on Intel SGX.


2017.05.18~2017.05.25

- 本周工作计划点1: Reading papers about blob storage

- 完成情况： half

- 本周工作计划点2: Try Minio and read its code

- 完成情况：Begining.

I have tried several open source SGX program (TaLoS, Panoply and Graphene). The Graphene is the most complete project. I have build a simple file server with SSL and encryption and decryption on files on graphene and carefully work around all the bugs in Graphene. Trying to figure out the data path and control should be in our project.   
However, things turns out that most of people still do not know how to write SGX programs: Is the compatibility really important? Is it necessary to run POSIX program inside SGX? Is the library OS is the ultimate way runnning inside the SGX?
What we need is not put everything in SGX, but separate sensitive path of the program.

- 下周计划：
  1. Reading papers about blob storage
	2. give a sketch of solution of current ideas
	3.


2017.05.07~2017.05.18

- 本周工作计划点1: give a sketch of solution of current ideas

- 完成情况： No

- 本周工作计划点2: Know more about SGX

- 完成情况：YES.

- 下周计划：
  1. Reading papers about blob storage
	2. give a sketch of solution of current ideas
	3. Try Minio and read its code( which is a open source blob storage solution)


2017.04.27~2017.05.6

- 本周工作计划点1: Read more papers

- 完成情况： finished.

- 本周工作计划点2: find new idea

- 完成情况： Finished.


- 下周计划：
	1. give a sketch of solution of current ideas

2017.04.20~2017.04.27

- 本周工作计划点1: Know serverless.

- 完成情况： finished.

- 本周工作计划点2: Reading papers and find new ideas/problems.

- 完成情况： Finished.


- 下周计划：
	1. Know serverless and it's tech, and think about what's the benefit with sgx, what's the challenge it might encounter.

>> chyyuu： serverless service+的综述在问题？值得与SGX结合的地方在哪里？目前已有的研究现状如何？ 你对此serverless service+SGX的想法啥？有何创新性？ Tian Hongliang's Ph.D Thesis有关第4章的内容，请分析如何定位问题的，发现了哪些问题，具体解决方法是什么，如何做测试分析评价的，还有哪些问题没有解决，引入了哪些新问题，能否重现其实验结果？希望周四能够看到你的汇报。

如何定位问题
实际在使用SGX 的SDK 开发应用程序时，这些问题很快就会暴露出来。
发现了哪些问题
SGX中的Library OS 需要syscall的支持，而Syscall需要换入换出Enclave
具体解决方法
实现异步的换入换出和可抢占的enclave 内多线程
如何测试分析评价
分别对单线程和多线程的程序进行性能测试。
哪些问题没有解决
SGX 内存过小的问题。

引入了哪些新问题
将非核心功能下放至host 操作系统，SGXkernel 对kernel 的需求更多，attack surface 更大。

重现实验结果
可以基于开源的graphene系统，将他的工作重现，并测试。

2017.04.13~2017.04.20

- 本周工作计划点1: Reading the Intel SGX explained.

- 完成情况： finished.

- 本周工作计划点2: Reading and use the Intel SGX SDK

- 完成情况： Finished.


- 下周计划：
	1. Know serverless and it's tech, and think about what's the benefit with sgx, what's the challenge it might encounter.

2017.04.03~2017.04.13

- 本周工作计划点1: Reading the Intel SGX explained.

- 完成情况： Ongoing.

- 本周工作计划点2: Reading the Kernel driver of SGX

- 完成情况： Finished.


- 下周计划：
	1. Learning to use the SDK in user level.
	2. finished SGX explained



2017.03.06~2017.03.10

- 本周工作计划点1: Finish the paper on OSDI using SGX.

- 完成情况：Finished.




- 下周计划：
	1. Reading the Intel SGX explained
	2. Think about the problem of SGX, and it's mechanism.


- 其他事宜：

Updated paper list.


本周工作进展和下周计划

2017.03.20~2017.04.02 For two weeks

- 本周工作计划点1: Reading the Intel SGX explained.

- 完成情况： Ongoing.

Finished knowing the the Intel architecture, secure attacks, and SGX programming model.


- 下周计划：
	1. Finished the Intel SGX explained
	2. Read the paper recently published about SGX in paper list



2017.03.06~2017.03.10

- 本周工作计划点1: Finish the paper on OSDI using SGX.

- 完成情况：Finished.




- 下周计划：
	1. Reading the Intel SGX explained
	2. Think about the problem of SGX, and it's mechanism.


- 其他事宜：

Updated paper list.

```
chy: 第二周
上周我提出的问题好像没看到解答。 你写的论文阅读报告可进一步清晰和深入，我觉得如可能，需要实验一下，并对照论文再阅读，比如你说性能是问题，具体测试情况如何。
对于你的想法，建议对一个潜在研究方向，建立一个单独的文件进行撰写。重点论文，需要自己尽量理解清楚，了解相关领域研究现状，并能够给大家清晰的讲解（这一点，在我听到的两次你面向大家的汇报中，觉得还有很大提升空间）。

建议看看 MIT的Sanctum : minimal architectural extensions for isolated execution ，并与SGX对比一下。

对于对于论文阅读报告，请写到单独的文档中，一篇论文一个报告。建议写出：

– Summary of major innovations
– What the problems the paper mentioned?
– How about the important related works/papers?
– What are some intriguing aspects of the paper?
– How to test/compare/analyze the results?
– How can the research be improved?
– If you write this paper, then how would you do?
– Did you test the results by yourself? If so, What’s your test Results?
– Give the survey paper list in the same area of the paper your reading.

建议本周能够给大家汇报一次  Virtual Ghost 或 SGX，最好结合实际的实验。

```
