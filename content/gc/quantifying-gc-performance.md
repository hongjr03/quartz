---
title: "量化垃圾回收与显式内存管理的性能差距"
description: "Hertz 与 Berger 经典论文的中文翻译：通过谕示内存管理器，对精确垃圾回收与显式内存管理进行系统的性能对比。"
aliases:
  - Quantifying the Performance of GC vs. Explicit Memory Management
  - 谕示内存管理
  - oracular memory management
tags:
  - gc
  - explicit-memory-management
  - performance
  - paging
  - time-space-tradeoff
  - translation
lang: "zh-CN"
enableToc: true
author: "Matthew Hertz, Emery D. Berger"
published: "2005-10-16"
source: "OOPSLA 2005, San Diego, California, USA"
---

> [!info] 来源
> Matthew Hertz, Emery D. Berger. *Quantifying the Performance of Garbage Collection vs. Explicit Memory Management*. OOPSLA'05, October 16–20, 2005, San Diego, California, USA. ACM 1-59593-031-0/05/0010.

> 著作权声明：允许为个人或课堂教学目的免费制作本作品的全部或部分数字或纸质副本，前提是副本不以盈利或商业优势为目的进行制作或分发，且副本需在首页附带本声明和完整的引用信息。以其他方式复制、重新发布、上传到服务器或分发给列表，需事先获得特定许可和/或付费。Copyright 2005 ACM 1-59593-031-0/05/0010 ...\$5.00.

## 摘要

垃圾回收（garbage collection）带来了诸多软件工程上的好处，但其对性能的量化影响一直难以捉摸。我们可以通过链接适当的回收器，在 C/C++ 程序中比较保守式垃圾回收与显式内存管理的性能。然而，这种直接比较对于为垃圾回收设计的语言（如 Java）来说是不可能的，因为这些语言的程序自然不包含 `free` 调用。因此，显式内存管理与精确、复制式垃圾回收在时间和空间性能上的真实差距一直不为人知。

我们引入了一种新颖的实验方法，能够量化精确垃圾回收与显式内存管理之间的性能差异。我们的系统通过依赖**谕示**（oracle）来插入 `free` 调用，使得未经修改的 Java 程序如同使用显式内存管理一样运行。这些谕示来源于早期程序运行中收集的剖析信息。通过在架构细节级模拟器中执行，这种"谕示式"内存管理器（oracular memory manager）在测量 `malloc` 和 `free` 代价的同时，消除了咨询谕示所带来的影响。我们评估了两种不同的谕示：基于**活跃性**的谕示（在对象最后一次使用后立即激进地释放），以及基于**可达性**的谕示（在对象刚变为不可达之后保守地释放）。这两种谕示跨越了显式释放调用可能放置位置的全部范围。

我们通过谕示式内存管理器，在一系列基准测试中比较了显式内存管理与复制式和非复制式垃圾回收器，并给出了真实的（非模拟的）运行结果为我们的结论提供进一步的验证。这些结果量化了垃圾回收的时间-空间权衡：在五倍内存下，配有非复制式成熟空间的 Appel 式分代回收器能达到与基于可达性的显式内存管理相匹配的性能；在三倍内存下，回收器平均比显式内存管理慢 17%；但在两倍内存下，垃圾回收的性能下降近 70%。当物理内存稀缺时，分页（paging）导致垃圾回收比显式内存管理慢一个数量级。

## 1. 引言

垃圾回收，即自动内存管理，相对于显式内存管理提供了显著的软件工程收益。例如，垃圾回收将程序员从内存管理的负担中解放出来，消除了大多数内存泄漏，改善了模块化，同时防止了意外的内存覆写（"悬垂指针"）[^50][^59]。由于这些优势，垃圾回收已被许多主流编程语言采纳为特性。

垃圾回收可以提高程序员生产力 [^48]，但其对性能的影响难以量化。先前的研究人员测量了保守式、非复制式垃圾回收在 C 和 C++ 程序中的运行时性能和空间影响 [^19][^62]。对于这些程序，比较显式内存管理与保守式垃圾回收的性能只需链接一个如 Boehm-Demers-Weiser 回收器 [^14] 的库。但不幸的是，在为垃圾回收设计的语言中测量性能权衡并不那么简单。由于用这些语言编写的程序不会显式释放对象，不能简单地用显式内存管理器替换垃圾回收。从保守式回收器的研究中推断结论也是不可能的，因为精确、可重定位的垃圾回收器（仅适用于垃圾回收语言）在性能上始终优于保守式、非重定位的垃圾回收器 [^10][^12]。

可以测量垃圾回收活动的成本（例如追踪和复制）[^10][^20][^30][^36][^56]，但**无法**扣除垃圾回收对 mutator 性能的影响。垃圾回收通过访问和重新组织内存改变应用程序行为。它还降低局部性，尤其是在物理内存稀缺时 [^61]。扣除垃圾回收的成本也忽略了显式内存管理器通过立即回收刚释放的内存所能提供的改进局部性 [^53][^55][^57][^58]。由于这些原因，精确、复制式垃圾回收与显式内存管理之间的成本从未被量化。

### 贡献

在本文中，我们对 Java 中垃圾回收与显式内存管理进行了实证比较。为实现这一比较，我们开发了一个"谕示式"内存管理器。该内存管理器依赖一个**谕示**（oracle），它指示系统何时应该释放对象（即对其调用 `free`）。在剖析运行期间，系统收集对象生命周期并生成程序堆轨迹（program heap trace），随后处理该轨迹以生成谕示。我们使用两种不同的谕示，它们跨越了可能的显式释放调用的全部范围：

- **基于活跃性的谕示**：最激进——使用对象生命周期指示内存管理器在对象**最后一次使用后**立即释放——这是它们可以被安全释放的最早时刻。
- **基于可达性的谕示**：最保守——在程序能调用 `free` 的最后时刻回收对象（即当它们变得不可达时）。基于可达性的谕示依赖于通过 Merlin 算法 [^33][^34] 处理程序堆轨迹获得的精确对象可达性信息。

我们发现，在线（on-line）版本的基于可达性的谕示会干扰 mutator 局部性，使运行时间增加 2% 至 33%。我们通过在扩展版的 Dynamic SimpleScalar（一个架构细节级模拟器）[^15][^39] 内执行谕示式内存管理器来消除此问题。这种方法使我们能够测量 Java 执行和内存管理操作的代价，同时排除咨询谕示引起的干扰。我们相信，该框架对于研究内存管理策略具有独立的意义。

我们使用该框架测量垃圾回收与显式内存管理对运行时性能、空间消耗和页级局部性的影响。我们在一系列基准测试、垃圾回收器（包括复制式和非复制式回收器）和显式内存管理器上执行这些测量。

我们发现，**GenMS**——一个配有标记-清除成熟空间的 Appel 式分代回收器——在给定五倍内存时，能匹配或超过（优于最多 9%）最佳显式内存管理器的运行时性能。在三倍内存下，垃圾回收使性能平均下降 17%。在更小的堆大小下，垃圾回收性能进一步恶化，最终平均慢 70%。显式内存管理还表现出更好的内存利用率和页级局部性，通常只需一半或更少的页面即可在相同的缺页次数下运行，并在物理内存稀缺时快数量级。

本文的其余部分组织如下：第 2 节详细介绍谕示式内存管理框架，第 3 节讨论其影响和局限。第 4 节和第 5 节介绍实验方法论和结果，将显式内存管理与一系列不同的垃圾回收器进行比较。第 6 节讨论相关工作，第 7 节探讨未来方向，第 8 节给出结论。

## 2. 谕示式内存管理

图 1 展示了谕示式内存管理框架的概览。如图 1(a) 所示，它首先执行 Java 程序以计算对象生命周期并生成程序堆轨迹。系统处理程序堆轨迹，使用 Merlin 算法计算对象可达性时间并生成基于可达性的谕示。基于活跃性的谕示直接来自剖析运行期间计算的生存周期。利用这些谕示，谕示式内存管理器执行程序，如图 1(b) 所示，使用 `malloc` 调用分配对象，并在谕示指示时对对象调用 `free`。由于轨迹生成在模拟器内部进行，而谕示生成离线进行，系统仅测量分配和释放的代价。

<figure>
  <img src="gc/images/hzb-fig1a.jpg" loading="lazy" alt="图 1(a)：第一步——收集并推导死亡记录">
  <figcaption>图 1(a)：第一步——收集并推导死亡记录</figcaption>
</figure>

<figure>
  <img src="gc/images/hzb-fig1b.jpg" loading="lazy" alt="图 1(b)：第二步——以显式内存管理执行程序">
  <figcaption>图 1(b)：第二步——以显式内存管理执行程序</figcaption>
</figure>

下面我们详细描述这些步骤，并讨论在生成谕示、检测内存分配操作和在不扭曲程序执行的前提下插入显式释放调用时所遇到的挑战及其解决方案。我们在第 3 节讨论该方法的影响和局限。

### 2.1 第一步：数据收集与处理

对于 Java 平台，谕示式内存管理器使用扩展版的 Jikes RVM 2.3.2，配置为生成 PowerPC Linux 代码 [^2][^3]。Jikes 是一个广泛使用的研究平台，几乎完全用 Java 编写。Jikes 及其附带的 MMTk（内存管理工具包）的一个关键优势在于它允许我们使用多种垃圾回收算法 [^11]。谕示式内存管理器在 Dynamic SimpleScalar (DSS) [^39] 内执行，后者是 SimpleScalar 超标量架构模拟器 [^15] 的扩展，允许使用动态生成的代码。

**可重复运行**：由于谕示式内存管理器使用分配顺序来识别对象，我们必须确保分配序列在每次运行中完全相同。我们在 Jikes RVM 和模拟器中采取了多项措施来确保可重复运行。我们使用 Jikes RVM 的 "fast" 配置，该配置尽可能多地优化系统并将其编译进预构建的虚拟机。我们采用伪自适应方法（pseudo-adaptive methodology）[^38][^49]，仅优化"热"方法（基于 5 次运行的平均值确定）。我们还采用确定性线程切换，基于已执行的方法数量而非固定时间间隔来切换线程。最后，我们修改 DSS 以确定性地更新模拟操作系统时间和寄存器时钟。

**基于活跃性的谕示所需的追踪**：在剖析运行期间，模拟器计算对象生命周期并为后续使用生成程序堆轨迹。模拟器通过记录每个已分配对象的位置来获取每个对象的生命周期信息。在每次内存访问时，模拟器查找正在使用的对象并更新其最近的生命周期（以分配时间计）。为捕获那些不检查内存位置的用法（如 Java 的相等比较使用的是地址比较），我们还将所有被根引用的对象标记为使用中。

系统还保留那些我们认为程序员无法合理释放的对象。例如，虽然我们的系统可以检测代码和类型信息的最后使用，但这些对象并非开发者在真实程序中能够释放的东西。类似地，我们的系统不会释放用于优化类加载的对象。在剖析运行结束时，系统保留所有这些对象及其引用的对象，将它们生命周期延长到程序结束。

**基于可达性的谕示所需的追踪**：为生成基于可达性的谕示，我们使用 Merlin 算法 [^33][^34] 高效且精确地计算对象可达性信息。我们的离线实现通过分析程序堆轨迹来操作。在轨迹处理期间，Merlin 算法在对象可能变为不可达时（例如，当指向它的指针被覆写时）更新与该对象关联的时间戳。

一个关键困难是在不影响程序执行的情况下捕获 Merlin 算法所需的全部信息。我们用非法操作码（illegal opcodes）替换正常操作码，以此作为非侵入式生成所需堆轨迹方法的核心。新的操作码唯一地标识关键事件，例如新对象何时被分配。当模拟器遇到这样的操作码时，它将一条记录输出到轨迹中，然后像执行其合法变体一样执行该非法操作码。

<figure>
  <img src="gc/images/hzb-fig2.jpg" loading="lazy" alt="图 2：对象生命周期">
  <figcaption>图 2：对象生命周期——活跃性谕示在最后使用后（最早安全点）释放，可达性谕示在变为不可达后（最晚可能时刻）释放</figcaption>
</figure>

为实现这些非法操作码的生成，我们扩展了 Jikes RVM 的编译器中间表示。我们在其中加入了一组节点来表示对 `malloc` 的调用。此扩展允许编译器像对待任何其他函数调用一样处理对象分配，同时发出非法操作码而非通常的分支指令。我们还修改了 Jikes RVM，用非法指令替换堆内引用存储。这些操作码使我们能够检测堆追踪所需的事件，而无需插入会扭曲指令缓存行为的代码。

### 2.2 第二步：模拟显式内存管理

在每次分配之前，模拟器咨询谕示以确定是否有对象应当被释放。当释放一个对象时，它保存函数参数（`malloc` 的大小请求）并跳转到 `free`，但将返回地址设置为使执行返回到 `malloc` 调用而非下一条指令。模拟器重复此循环直到没有对象需要回收，然后分配和程序执行照常继续。`malloc` 和 `free` 均通过方法调用调用。当这些函数在 VM 外部实现时，通过 Jikes 外部函数接口（VM SysCall）调用它们。

### 2.3 验证：在线谕示式内存管理

除了上述基于模拟的框架，我们还实现了一个"在线"（live）版本的谕示式内存管理器，它使用基于可达性的谕示，但实际运行在真实机器上。在线谕示使用对象生命周期信息和一个记录对象分配位置的缓冲区，来填充一个包含待释放对象地址的特殊"谕示缓冲区"。为确定程序的运行时间，我们测量总执行时间，然后减去检查对象是否应被释放所花的时间以及重新填充谕示缓冲区所花的时间。

为测量谕示引入的扭曲，我们比较了正常运行垃圾回收与运行空谕示（null oracle）的成本。空谕示以与真实谕示式内存管理器相同的方式加载缓冲区，但此后执行正常进行（实际上不释放任何对象）。我们发现引入的扭曲大到不可接受且不稳定。例如，使用 GenMS 回收器时，228 jack 基准测试在空谕示下报告运行时间增加 12% 至 33%，而在 213 javac 基准测试下，同一回收器最多仅慢 3%。其他回收器也表现出空谕示的扭曲，但没有任何明显或可预测的模式。我们将这些扭曲归因于谕示缓冲区处理引起的 L1 和 L2 缓存污染。

虽然在线谕示式内存管理器由于噪声太大而无法用于精确测量，但其结果为基于模拟的方法提供了可信度。如图 5 所示，在线版本紧密反映了可达性谕示模拟结果的趋势。

## 3. 讨论

在前面的章节中，我们专注于所采用的方法论，力图消除测量噪声和扭曲。在此我们讨论该方法的一些关键假设并回应可能的疑虑。这些包括：对不可达和死对象调用 `free`，内存操作使用外部函数调用的代价，多线程环境的影响，显式内存管理未被测量的成本，自定义内存分配器的作用，以及内存管理器对程序结构的影响。虽然我们的方法论可能会显得对显式内存管理不利（使垃圾回收看起来更好），但我们论证这些差异是微不足道的。

### 3.1 可达性 vs. 活跃性

谕示式内存管理器使用两种截然不同的谕示来分析显式内存管理性能。基于活跃性的谕示激进地释放对象，在能够安全执行的最早时机调用 `free`。基于可达性的谕示则在程序执行的最后可能时刻释放对象，因为调用 `free` 需要一个可达的指针作为参数。

基于活跃性的谕示保留了一些超出最后使用的对象，同时也释放了一些基于可达性谕示不会释放的对象。涉及的对象数量很少：仅 pseudoJBB（3.8%）和 201 compress（4.4%）多释放了超过 0.8% 的对象。

真实的程序行为很可能落在这两个极端之间。我们预期很少有程序员会在对象最后一次使用后立即回收它们，同样也不会等到对象可达的最后时刻才释放。这两种谕示因此界定了显式内存管理选项的范围。

在第 5.1 节中，我们展示了这两种谕示之间的差距很小。两种谕示提供相似的运行时性能，而活跃性谕示最多将堆占用减少 15%（相对于可达性谕示）。这些结果与之前对 C 和 Java 程序的研究大体一致。Hirzel 等人在包含七个 C 应用程序的基准测试套件上比较了活跃性与可达性 [^37]。他们发现，使用激进的跨过程活跃性分析时，两个基准测试存在显著差距：gzip 的平均对象生存期（以分配时间计）缩短了 11%，yacr2 缩短了 21%；对于其他基准测试，差距保持在 2% 以下。Shaham 等人 [^51] 在包含本文五个基准测试的研究中，测量了在 Java 代码中插入 `null` 赋值的平均影响——模拟接近完美的显式释放调用放置。他们报告相对于对象变得不可达时才释放，空间消耗的平均差异为 15%。

### 3.2 malloc 开销

当使用 C 实现的分配器时，谕示式内存管理器通过 Jikes VM SysCall 外部函数调用接口调用分配和释放函数。虽然并非零代价，但这些调用的开销不如 JNI 调用大。它们的总成本仅为 11 条指令。此成本与在 C 和 C++ 中调用内存操作相似，其中 `malloc` 和 `free` 是在外部库中定义的函数。

我们还考察了在 Jikes RVM 内实现 `malloc` 和 `free` 的分配器。在这种情况下，谕示式内存管理器使用普通的 Jikes RVM 方法调用接口而非 VM SysCall 接口。由于我们仍需要确定分配何时发生以及何时适于插入 `free` 调用，我们仍然不能内联分配快速路径。虽然这可能阻止一些潜在的优化，但我们不知道有任何显式管理的编程语言在没有函数调用开销的情况下实现内存操作。

### 3.3 多线程 vs. 单线程

在本文介绍的实验中，我们假设单处理器环境，并对 Jikes RVM 和 Lea 分配器均禁用原子操作。在多线程环境中，大多数线程安全的内存分配器也需要为每次 `malloc` 和 `free` 调用执行至少一个原子操作：对于基于锁的分配器是 test-and-set 操作，对于无锁分配器则是 compare-and-swap 操作 [^46]。这些原子操作在某些架构上非常昂贵。例如，在 Pentium 4 上，原子 CMPXCHG 操作（compare-and-swap）的代价约为 124 个周期。由于垃圾回收可以通过批量分配和释放来分摊原子操作的成本，Boehm 观察到它可能比显式内存分配快得多 [^13]。然而，多线程与单线程环境的问题与垃圾回收器和显式内存管理器的比较是正交的，因为显式内存分配器也可以为大部分内存操作避免原子操作。特别是，Hoard 的最新版本（3.2）维护线程局部的空闲列表，通常仅在刷新或重新填充它们时才使用原子操作。使用这些线程局部空闲列表成本很低，通常通过一个专用于访问线程局部变量的寄存器来实现。在没有此类支持的架构上，Hoard 将空闲列表放置在每个线程栈的开头（按 1MB 边界对齐），并通过位掩码一个栈变量来访问它们。

### 3.4 智能指针

显式内存管理还可能产生其他性能代价。例如，C++ 程序可能通过使用智能指针来管理对象所有权。这些模板类透明地实现引用计数，这会为每次指针更新增加开销。例如，在 gc-bench 基准测试中，使用在现有类内部嵌入引用计数的 Boost "侵入式指针"（intrusive pointer），其性能比 Boehm-Demers-Weiser 回收器慢最多两倍。然而，智能指针似乎并未被广泛使用。我们搜索了使用标准 `auto_ptr` 类或 Boost 库 `shared_ptr` [^16] 的程序，仅在 sourceforge.net 上找到两个使用它们的大型程序。我们将这种缺乏使用归因于它们的成本——因为 C++ 程序员往往对昂贵的操作特别敏感——以及它们的不灵活性。

实际上，C 和 C++ 程序员通常使用以下约定之一：函数调用者要么分配对象然后传递给被调用者，要么被调用者分配对象然后返回给调用者。这些约定在优化后的代码中几乎不引入性能开销。

尽管如此，某些内存使用模式天生难以用 `malloc` 和 `free` 管理。例如，解析器的分配模式使管理单个对象成为难以接受的负担。在这些情况下，C 和 C++ 程序员通常求助于自定义内存分配器。

### 3.5 自定义分配

许多显式管理的程序使用自定义分配器而非通用分配器，以简化和加速内存管理。特别是，Berger 等人表明区域式（region-style）分配对多种工作负载既有用又可能比通用分配快得多，但通常消耗远超所需的空间 [^8]。探索如区域等自定义分配策略超出了本文的范围。

### 3.6 程序结构

我们在此考察的程序是为垃圾回收环境编写的。如果它们是用具有显式内存管理的语言编写的，可能会以不同的方式编写。不幸的是，我们没有看到量化此效应的任何方法。尝试通过手动重写基准测试应用程序来使用显式释放（虽然繁重）来测量这种效应是可能的，但我们将不得不设法排除个人编程风格的影响。尽管人们可能预期程序结构有明显差异，但我们观察到 Java 程序为不再使用的对象赋予 `null` 的模式是常见的。在这个意义上，在垃圾回收环境中编程至少偶尔类似于显式内存管理。特别地，显式赋予 `null` 类似于 C++ 中 `delete` 的使用，后者随后可以触发一条类特定的对象析构函数链。

## 4. 实验方法论

为量化垃圾回收与显式内存管理的性能，我们比较了八个基准测试在各种垃圾回收器下的性能。

### 基准测试

| 基准测试 | 总分配量 (bytes) | 最大可达 (bytes) | 分配/最大 |
|---|---|---|---|
| _201_compress | 125,334,848 | 13,682,720 | 9.16 |
| _202_jess | 313,221,144 | 8,695,360 | 36.02 |
| _205_raytrace | 151,529,148 | 10,631,656 | 14.25 |
| _209_db | 92,545,592 | 15,889,492 | 5.82 |
| _213_javac | 261,659,784 | 16,085,920 | 16.27 |
| _228_jack | 351,633,288 | 8,873,460 | 39.63 |
| ipsixql | 214,494,468 | 8,996,136 | 23.84 |
| pseudoJBB | 277,407,804 | 32,831,740 | 8.45 |

我们包含了大部分 SPECjvm98 基准测试 [^18]。ipsixql 是一个持久化 XML 数据库系统，pseudoJBB 是 SPECjbb 基准测试的固定工作量变体 [^17]。

### 垃圾回收器与分配器

| 回收器 | 描述 |
|---|---|
| **MarkSweep** | 非重定位、非复制、单代 |
| **GenCopy** | 两代，复制式成熟空间 |
| **SemiSpace** | 双空间单代 |
| **GenMS** | 两代，非复制式成熟空间 |
| **CopyMS** | 新生区 + 全堆回收 |

| 分配器 | 描述 |
|---|---|
| **Lea** | 结合快速列表和近似最佳适配 |
| **MSExplicit** | MMTk 的 MarkSweep 配合显式释放 |

表 2 列出了本文考察的垃圾回收器，均为高吞吐量的 "stop-the-world" 回收器。下面对各回收器的详细描述改编自 Blackburn 等人 [^11]：

- **MarkSweep**：将堆组织为被划分为固定大小块的区块（blocks），通过空闲列表（freelists）管理。MarkSweep 追踪并标记可达对象，在分配时惰性地寻找空闲槽位。
- **SemiSpace**：使用 bump pointer 分配，拥有两个复制空间。它在其中一个空间分配，当该空间填满时，将可达对象复制到另一个空间并交换两者。
- **GenCopy**：使用 bump pointer 分配，是一个经典的 Appel 式分代回收器 [^5]。它在年轻（新生区）复制空间中分配，将幸存者提升到老年代 SemiSpace 中。其写屏障记录从老年代指向新生区对象的指针。当新生区填满时 GenCopy 进行回收，并根据幸存者的大小缩小新生区。当老年代空间填满时，对整个堆进行回收。
- **GenMS**：这个混合式分代回收器与 GenCopy 类似，但使用 MarkSweep 老年代空间。
- **CopyMS**：CopyMS 是一个非分代回收器（即没有写屏障），使用 bump pointer 分配到一个复制空间。当该空间填满时，CopyMS 执行全堆回收，将幸存者复制到 MarkSweep 老年代空间。

对于每个基准测试，我们以从能完成运行的最小值到四倍大的堆大小范围运行每个垃圾回收器。对于模拟运行，我们使用 PowerPC G5 处理器 [^1] 的内存和处理器配置，假设 2 GHz 时钟。使用 4K 页面大小，与 Linux 和 Windows 相同。表 3 给出了详细的架构参数。

| 缓存层级 | 模拟 PowerPC G5 系统 | 实际 PowerPC G4 系统 |
|---|---|---|
| L1, I-缓存 | 64K，直接映射，3 周期延迟 | 32K，8 路组相联，3 周期延迟 |
| L1, D-缓存 | 32K，2 路组相联，3 周期延迟 | 32K，8 路组相联，3 周期延迟 |
| L2（统一） | 512K，8 路组相联，11 周期延迟 | 256K，8 路组相联，8 周期延迟 |
| L3（片外） | 不适用 | 2048K，8 路组相联，15 周期延迟 |
| 缓存行 | 所有缓存均有 128 字节行 | 所有缓存均有 32 字节行 |
| RAM | 270 周期（135ns） | 95 周期（95ns） |

**表 3**：基于模拟和"在线"实验框架的内存时序参数（见第 2.3 节）。模拟器基于 2GHz PowerPC G5 微处理器，实际系统使用 1GHz PowerPC G4 微处理器。

我们通过检查每次运行使用的最大堆页数来比较实际堆占用，而非依赖报告的堆使用量。页面仅在已从内核分配并已被触碰（touched）时才"在使用中"。计算使用中的页面确保了所有内存使用的正确核算，包括元数据空间——该部分偶尔被低报。

对于谕示式内存管理实验，我们使用 Lea 分配器（GNU libc，"DLmalloc"）[^44] 和 MMTk MarkSweep 回收器的一个变体。Lea 分配器是一个近似最佳适配分配器，兼具高速和低内存消耗。它构成了 GNU C 库中内存分配器的基础 [^28]。这里使用的版本（2.7.2）是混合分配器，根据对象大小表现出不同行为，尽管不同大小的对象可能在内存中相邻。小对象（小于 64 字节）使用精确大小的快速列表（quicklists）分配——每个 8 字节倍数对应一条已释放对象的链表。Lea 分配器在多种条件下（例如接收到中等大小对象的请求时）会合并这些列表中的对象（将相邻的空闲对象合并）。中等大小的对象则通过即时合并和拆分快速列表上的内存，以近似最佳适配的方式管理。大对象通过 `mmap` 分配和释放。Lea 分配器是就我们所知的在速度和内存使用综合表现上最优的分配器 [^40]。

虽然 Lea 分配器是一个优秀的比较对象，但它与我们在此考察的垃圾回收器差异显著。为分离显式内存管理的影响，我们向 MMTk 的 MarkSweep 回收器和大对象管理器（"Treadmill"）中添加了个体对象的释放功能。每个内存块维护自己的空闲槽位栈，并重用最近被释放的槽位。该"分配器"在图中标注为 **MSExplicit**。

## 5. 实验结果

在本节中，我们探讨垃圾回收和显式内存管理对总执行时间、内存消耗和页级局部性的影响。

### 5.1 运行时与内存消耗

图 3 展示了以基于可达性谕示下 Lea 分配器为基准，各垃圾回收器性能的几何平均值。图 4 展示了各个基准测试在所有垃圾回收器下的运行时 vs. 空间结果。这些图紧凑地总结了这些结果，展示了使用垃圾回收所涉及的时间-空间权衡。由于我们呈现的是实际触碰的页面数而非请求的堆大小，它们偶尔会出现一种令人惊讶的"锯齿"（zig-zag）效应。随着堆大小增加，堆页数通常也会增加，但堆大小的增加有时反而会减少被访问的堆页数。例如，由于碎片化或对齐限制，较大的堆大小可能导致对象跨越两个页面。这种效应在 MarkSweep 上最为显著，因为它无法通过压缩堆来减少碎片化。

<figure>
  <img src="gc/images/hzb-fig3.jpg" loading="lazy" alt="图 3：垃圾回收器相对 Lea 分配器性能的几何平均值">
  <figcaption>图 3：垃圾回收器相对 Lea 分配器（可达性谕示）性能的几何平均值</figcaption>
</figure>

随着堆大小增长，全堆垃圾回收的次数相应减少。最终，总执行时间渐近趋近于一个固定值。对 GenMS 而言，该值略低于显式内存管理的成本。在最大堆大小时，GenMS 与 Lea 分配器的性能持平。其在每个基准测试上的最佳相对性能范围从 ipsixql 快 10% 到 209 db 慢 26%（该基准测试对局部性效应异常敏感）。

各回收器之间的性能差距在**分配强度**（总分配字节/最大可达字节的比率）较低的基准测试中最小。对于这些基准测试，MarkSweep 往往提供最佳性能，尤其是在较小的堆倍数下。与其它回收器不同，MarkSweep 不需要复制保留空间（copy reserve），因此更有效地利用堆。随着分配强度的增长，分代垃圾回收器通常表现出更好的性能。

<figure>
  <img src="gc/images/hzb-fig4a.jpg" loading="lazy" alt="图 4(a)：209_db">
  <figcaption>图 4(a)：209_db（分配强度 5.82）</figcaption>
</figure>
<figure>
  <img src="gc/images/hzb-fig4b.jpg" loading="lazy" alt="图 4(b)：pseudoJBB">
  <figcaption>图 4(b)：pseudoJBB（分配强度 8.45）</figcaption>
</figure>
<figure>
  <img src="gc/images/hzb-fig4c.jpg" loading="lazy" alt="图 4(c)：213_javac">
  <figcaption>图 4(c)：213_javac（分配强度 16.27）</figcaption>
</figure>
<figure>
  <img src="gc/images/hzb-fig4d.jpg" loading="lazy" alt="图 4(d)：ipsixql">
  <figcaption>图 4(d)：ipsixql（分配强度 23.84）</figcaption>
</figure>
<figure>
  <img src="gc/images/hzb-fig4e.jpg" loading="lazy" alt="图 4(e)：202_jess">
  <figcaption>图 4(e)：202_jess（分配强度 36.02）</figcaption>
</figure>
<figure>
  <img src="gc/images/hzb-fig4f.jpg" loading="lazy" alt="图 4(f)：228_jack">
  <figcaption>图 4(f)：228_jack（分配强度 39.63）</figcaption>
</figure>

垃圾回收曲线的形状证实了预测垃圾回收性能与堆大小成反比的解析模型 [^4];[^41][^35] p.。注意，显式内存管理的成本不依赖于堆大小，而与分配的对象数量成线性关系。虽然此反比关系对 MarkSweep 和 SemiSpace 成立，但我们发现，平均而言，GenMS 的运行时间与堆大小的**平方**成反比。具体而言，函数 `执行时间因子 = a/(b − 堆大小因子²) + c` 刻画了 GenMS 的趋势，其中执行时间因子是相对于 Lea 的性能膨胀，堆大小因子是最小所需堆大小的倍数。使用参数 a = −0.246, b = 0.59, c = 0.942，曲线拟合极佳：均方根误差仅为 0.0024（0 为完美拟合）。GenCopy 也有类似结果，均方根误差仅 0.0067。据我们所知，这种逆二次行为此前未被注意到。我们尚未有解释模型，但推测此行为的出现是因为新生区回收的存活率也与堆大小成反比。

如下表所示，GenMS 相对于 Lea 的内存占用与运行时的几何平均值：

| 堆倍数 | 占用 (可达性) | 运行时 (可达性) | 占用 (活跃性) | 运行时 (活跃性) |
|---|---|---|---|---|
| 1.00 | 210% | 169% | 253% | 167% |
| 1.25 | 252% | 130% | 304% | 128% |
| 1.50 | 288% | 117% | 347% | 115% |
| 1.75 | 347% | 110% | 417% | 109% |
| 2.00 | 361% | 108% | 435% | 106% |
| 2.25 | 406% | 106% | 488% | 104% |
| 2.50 | 419% | 104% | 505% | 102% |
| 2.75 | 461% | 103% | 554% | 102% |
| 3.00 | 476% | 102% | 573% | 100% |
| 3.25 | 498% | 101% | 600% | 100% |
| 3.50 | 509% | 100% | 612% | 99% |
| 3.75 | 537% | 101% | 646% | 100% |
| 4.00 | 555% | 100% | 668% | 99% |

**表 4**：GenMS 相对于 Lea 的内存占用与运行时的几何平均值。堆大小是使用 GenMS 运行所需最小堆大小的倍数。

表 5 比较了 MSExplicit 与 Lea 分配器在使用相同谕示时的占用和运行时。

| 基准测试 | 占用 (可达性) | 运行时 (可达性) | 占用 (活跃性) | 运行时 (活跃性) |
|---|---|---|---|---|
| _201_compress | 162% | 106% | 251% | 101% |
| _202_jess | 154% | 104% | 165% | 103% |
| _205_raytrace | 131% | 102% | 147% | 100% |
| _209_db | 112% | 118% | 118% | 96% |
| _213_javac | 133% | 95% | 124% | 93% |
| _228_jack | 158% | 103% | 168% | 105% |
| ipsixql | 149% | 100% | 163% | 97% |
| pseudoJBB | 112% | 106% | 116% | 87% |
| **几何平均** | **138%** | **104%** | **152%** | **98%** |

**表 5**：MSExplicit 相对于 Lea 的内存占用和运行时——两者使用相同谕示。

最后，表 5 比较了 **MSExplicit**（基于 MMTk MarkSweep 实现的显式内存管理）和 Lea 分配器在使用相同谕示时的占用和运行时。MSExplicit 的内存效率远低于 Lea，需要多 38% 至 52% 的空间。然而，运行时性能结果相似。使用可达性谕示时，MSExplicit 平均比 Lea 慢 4%；使用活跃性谕示时，快 2%。MSExplicit 的最差情况是对局部性敏感的 209 db，其按大小分隔的类导致使用可达性谕示时慢 18%。另一方面，它对 213 javac 使用可达性谕示时比 Lea 快 5%，因为该基准测试对原始分配速度构成压力。图 4(f) 尤其具有揭示意义：在该情况下，MSExplicit 的运行时性能仅比 Lea 高 3%，但 MarkSweep 使用相同的分配基础设施，却慢 300% 到 50% 不等。除 209 db 外，两个分配器的性能大致相当，这既证实了所生成 Java 代码的良好性能特征，也证实了 MMTk 基础架构的良好性能。**这些实验表明，显式内存管理与垃圾回收之间的性能差异源于垃圾回收本身，而非底层分配器基础设施的差异。**

### 比较模拟与在线谕示

我们还比较了各种垃圾回收器与第 2.3 节中描述的在线谕示的运行时性能。对于这些实验，我们使用了一台配备 512MB RAM 的 PowerPC G4，以单用户模式运行 Linux，并报告 5 次运行的平均值。实验机器的架构细节见表 3。在线谕示与模拟实验结果的对比见图 5。这些图比较了除三个基准测试外所有基准测试执行时间的几何平均值。由于堆轨迹生成过程的内存需求以及复制基于时间的操作系统调用的困难，我们目前无法使用在线谕示运行 pseudoJBB、ipsixql 和 205 raytrace。

<figure>
  <img src="gc/images/hzb-fig5a.jpg" loading="lazy" alt="图 5(a)：在线谕示结果">
  <figcaption>图 5(a)：在线谕示结果</figcaption>
</figure>
<figure>
  <img src="gc/images/hzb-fig5b.jpg" loading="lazy" alt="图 5(b)：模拟谕示结果">
  <figcaption>图 5(b)：模拟谕示结果——在线与模拟谕示式内存管理器对比</figcaption>
</figure>

尽管环境不同，在线和模拟谕示式内存管理器取得了惊人相似的结果。两组图之间的差异可以归因于 G4 的 L3 缓存和比我们的模拟器更小的主存延迟。虽然空谕示为我们的数据增加了太多噪声而不适合取代模拟器使用，但结果的相似性为模拟运行的有效性提供了强有力的证据。

### 比较活跃性与可达性谕示

我们在图 6 中比较了使用基于活跃性和基于可达性的谕示的效果。该图展示了使用两种谕示的分配器的平均相对执行时间和空间消耗。如常，所有值均相对于使用基于可达性谕示的 Lea 分配器进行归一化。x 轴显示相对执行时间；注意其压缩的尺度，仅从 0.98 到 1.04。y 轴显示相对堆占用，此处的尺度范围从 0.8 到 1.7。汇总和单个运行时图（图 3 和图 4）也包含了使用活跃性谕示的 Lea 分配器的数据点。

我们发现谕示的选择对执行时间影响很小。我们原本预期基于活跃性的谕示会通过增强缓存局部性来改进性能，因为它尽快回收对象。然而，这种回收对运行时的影响在最好情况下是混合的：使 Lea 分配器性能下降 1%，而使 MSExplicit 性能提升最多 5%。

<figure>
  <img src="gc/images/hzb-fig6.jpg" loading="lazy" alt="图 6：显式内存管理器相对 Lea 分配器（可达性谕示）的几何平均值">
  <figcaption>图 6：显式内存管理器相对 Lea 分配器（可达性谕示）的几何平均值——虽活跃性谕示显著减少堆占用，但对平均执行时间影响很小</figcaption>
</figure>

当基于活跃性的谕示确实改进运行时性能时，它通过减少 L1 数据缓存缺失来实现。例如，ipsixql 在活跃性谕示下比可达性谕示快 18%，这是因为 L1 数据缓存缺失率减半。另一方面，活跃性谕示显著恶化了 209 db 和 pseudoJBB 的缓存局部性，使它们分别慢 23% 和 13%。虽然 209 db 对缓存效应出了名地敏感，但 pseudoJBB 的结果令人惊讶。在这种情况下，使用基于活跃性的谕示导致了较差的对象放置，使 L2 缓存缺失率增加了近 50%。然而，这些基准测试是异常值。图 3 显示，平均而言，使用活跃性谕示的 Lea 分配器仅比使用可达性谕示慢 1%。

基于活跃性的谕示对空间消耗有更显著的影响，将堆占用减少最多 15%。使用活跃性谕示，Lea 的平均堆占用减少 17%，MSExplicit 减少 12%。

### 5.2 页级局部性

对于虚拟内存系统，页级局部性对性能的影响可能比总内存消耗更大。我们以**增强缺失曲线**（augmented miss curves）[^52][^61] 的形式呈现页级局部性实验结果。假设虚拟内存管理器遵循 LRU 淘汰策略，这些图展示了在不同分配给进程的页数（x 轴）下所需的时间（y 轴，对数刻度）。假设固定的 5 毫秒缺页服务时间。

图 7 显示了所有基准测试中 Lea 分配器、MSExplicit 和每个垃圾回收器的总执行时间。对于每个垃圾回收器，选择了性能最快的堆大小。

<figure>
  <img src="gc/images/hzb-fig7a.jpg" loading="lazy" alt="图 7(a)：209_db 缺页性能">
  <figcaption>图 7(a)：209_db</figcaption>
</figure>
<figure>
  <img src="gc/images/hzb-fig7b.jpg" loading="lazy" alt="图 7(b)：pseudoJBB 缺页性能">
  <figcaption>图 7(b)：pseudoJBB</figcaption>
</figure>
<figure>
  <img src="gc/images/hzb-fig7c.jpg" loading="lazy" alt="图 7(c)：213_javac 缺页性能">
  <figcaption>图 7(c)：213_javac</figcaption>
</figure>
<figure>
  <img src="gc/images/hzb-fig7d.jpg" loading="lazy" alt="图 7(d)：ipsixql 缺页性能">
  <figcaption>图 7(d)：ipsixql</figcaption>
</figure>
<figure>
  <img src="gc/images/hzb-fig7e.jpg" loading="lazy" alt="图 7(e)：202_jess 缺页性能">
  <figcaption>图 7(e)：202_jess</figcaption>
</figure>
<figure>
  <img src="gc/images/hzb-fig7f.jpg" loading="lazy" alt="图 7(f)：228_jack 缺页性能">
  <figcaption>图 7(f)：228_jack</figcaption>
</figure>

这些图表明，在合理范围的可用内存下（但不足以容纳整个应用程序），两种显式内存管理器**远优于**所有垃圾回收器。例如，pseudoJBB 在 63MB 可用内存下使用 Lea 分配器在 25 秒内完成。使用相同可用内存和 GenMS，需要超过十倍的时间（255 秒）才能完成。我们在整个基准测试套件中看到类似的趋势。最显著的案例是 213 javac：在 36MB 下使用 Lea 分配器，总执行时间为 14 秒，而使用 GenMS 为 211 秒，超过 15 倍的增加。

这里的罪魁祸首是垃圾回收活动，它访问的页面远超应用程序本身 [^61]。随着分配强度的增加，主回收（major collections）的数量也增加。由于每次垃圾回收很可能会访问已被换出的页面，垃圾回收器与显式内存管理器之间的性能差距随着主回收次数的增加而扩大。

## 6. 相关工作

先前的垃圾回收与显式内存管理比较通常发生在保守式、非重定位垃圾回收器与 C/C++ 的上下文中。Detlefs 在他的博士论文中比较了三个 C++ 程序的垃圾回收与显式内存管理性能 [^19]，发现垃圾回收通常导致更差的性能（2% 到 28% 的开销）。Zorn 在 C 程序的上下文中比较了保守式垃圾回收与显式内存管理 [^62]。他发现 BDW 回收器 [^14] 偶尔比显式内存分配更快，但 BDW 回收器消耗的内存几乎总是高于显式内存管理器。Hicks 等人也发现用 Cyclone 编写并链接 BDW 回收器的程序可能比纯显式内存管理需要多得多的内存 [^35]。虽然这些研究考察了在 C 和 C++ 程序内运行的保守式垃圾回收器，我们则专注于从一开始就为使用垃圾回收而编写的代码的性能。

与该工作最接近的或许是 Blackburn 等人，他们在 Jikes RVM 和 MMTk 中测量了类似范围的垃圾回收器和基准测试 [^10]。他们得出结论：分代垃圾回收实现了局部性收益，使其比空闲列表式分配更快。为近似显式内存管理，他们测量了使用 MMTk 标记-清除垃圾回收器时的 mutator 时间，并表明这超过了使用分代回收器的总执行时间。这种方法既未考虑垃圾回收引起的缓存污染，也未考虑显式内存管理器通过立即回收已分配对象所达到的有益局部性效应。

许多研究试图量化垃圾回收和显式内存管理对应用程序性能的开销 [^7][^9][^40][^43][^62]。Steele 观察到 LISP 中垃圾回收的开销约占应用程序运行时间的 30% [^30]。Ungar 测量了 Berkeley Smalltalk 中分代 scavenging 的代价，发现其仅占 CPU 时间的 2.5% [^56]。然而，这一测量排除了垃圾回收对内存系统的影响。Diwan 等人 [^20] 使用八个 SML/NJ 基准测试的追踪驱动模拟得出结论：分代垃圾回收占应用程序运行时间的 19% 到 46%（以每指令周期数衡量）。

Appel 提出了一项分析，使用统一的内存访问成本模型，表明在给定足够空间的情况下垃圾回收可以比显式内存管理更快 [^4]（Miller 对此结论关于栈分配的方面提出了反驳 [^47]）。他观察到回收的频率与堆大小成反比，而回收的成本本质上是常量（最大可达大小的函数）。因此，增大堆会降低垃圾回收的成本。Wilson 认为这一结论在现代机器上不太可能成立，因为它们具有深层内存层次 [^60]。我们在这样一个系统上的结果支持 Appel 的分析，尽管我们发现 Appel 式回收器的运行时间与堆大小的**平方**成反比。

## 7. 未来工作

本文仅处理个体对象管理，即所有对象用 `malloc` 分配，用 `free` 释放。然而，像区域（regions）这样的自定义分配方案可以显著改进使用显式内存管理的应用程序的性能 [^9][^31]。区域也日益流行作为垃圾回收的替代或补充 [^26][^27][^29][^54]。在未来的工作中，我们计划使用我们的框架来考察区域和混合分配器 reaps [^9] 与垃圾回收相比的影响。

我们使用的 Lea 分配器在每个已分配对象之前放置 8 字节的对象头部。这些头部可能增加空间消耗并损害缓存级局部性 [^24]。我们计划评估使用 BiBoP 式分配从而避免每对象头部的内存分配器，如 PHKmalloc [^42] 和 Vam [^24]。我们打算将显式内存管理的虚拟内存性能与 bookmarking 回收器进行比较，后者是专门为避免分页而设计的 [^32]。

最后，本文仅考察了 stop-the-world、非增量、非并发的垃圾回收器。虽然这些通常提供最高的吞吐量，它们也表现出最大的暂停时间。我们希望探索各种垃圾回收器相对于显式内存管理器的暂停时间效应——后者同样会表现出暂停。例如，Lea 分配器通常在数个周期内分配对象，但偶尔会清空和合并其快速列表。它还对大对象执行线性最佳适配搜索。据我们所知，显式内存管理引起的暂停从未被测量或与垃圾回收暂停进行比较。

## 8. 结论

本文提出了一种基于追踪和模拟的实验方法论，能够使未经修改的 Java 程序如同使用显式内存管理一样执行。我们使用该框架在一系列垃圾回收器与使用 Lea 内存分配器的显式内存管理之间比较了时间-空间性能。

通过在一系列基准测试上比较运行时、空间消耗和虚拟内存占用，我们表明，表现最佳的垃圾回收器在**给定足够内存时**，其运行时性能与显式内存管理具有竞争力。具体而言：

- 当垃圾回收拥有**五倍**于最低需求的内存时，其运行时性能与显式内存管理持平或略优；
- 在**三倍**内存下，平均慢 17%；
- 在**两倍**内存下，平均慢 70%。

垃圾回收在物理内存稀缺时也更容易受到分页的影响。在这种情况下，我们考察的所有垃圾回收器相对于显式内存管理都遭受了数量级的性能损失。

我们相信这些结果对实践者和研究者都有用：

- **实践者**可以利用这些结果来指导选择显式管理语言（如 C/C++）还是垃圾回收语言（如 Java/C#）。如果应用程序将部署在至少拥有三倍所需 RAM 的系统上，垃圾回收应能提供合理的性能。但如果部署的系统 RAM 更少，或应用程序将与其他进程竞争内存，实践者应预期垃圾回收会带来实质性的性能代价。对于性能依赖于内存高效使用的应用程序，如内存数据库和搜索引擎，这种代价将尤为显著。

- **研究者**可以利用这些结果来指导内存管理算法的开发。本研究表明，垃圾回收的关键弱点在于其在紧张堆和物理内存稀缺环境下的糟糕表现。另一方面，在非常大的堆中，垃圾回收已经与显式内存管理具有竞争力或略优。

## 9. 致谢

感谢 Steve Blackburn、Hans Boehm、Sam Guyer、Martin Hirzel、Richard Jones、Scott Kaplan、Doug Lea、Kathryn McKinley、Yannis Smaragdakis、Trevor Strohman 以及匿名审稿人对本文草稿提出的有益意见。

本文基于美国国家科学基金会（NSF）在奖项编号 CNS-0347339 和 CISE 研究基础设施资助 EIA-0303609 下支持的工作。本文所表达的任何观点、发现和结论或建议均为作者的观点，不一定反映国家科学基金会的观点。

## 参考文献

[^1]: Hardware — G5 Performance Programming, Dec. 2003. http://developer.apple.com/hardware/ve/g5.html.

[^2]: B. Alpern, et al. The Jalapeño virtual machine. *IBM Systems Journal*, 39(1), Feb. 2000.

[^3]: B. Alpern, et al. Implementing Jalapeño in Java. In *Proc. OOPSLA'99*, 1999.

[^4]: A. W. Appel. Garbage collection can be faster than stack allocation. *Information Processing Letters*, 25(4):275–279, 1987.

[^5]: A. W. Appel. Allocation without locking. *SP&E*, 19(7), 1989.

[^6]: E. D. Berger. The Hoard memory allocator. http://www.hoard.org.

[^7]: E. D. Berger, B. G. Zorn, and K. S. McKinley. Composing high-performance memory allocators. In *Proc. PLDI'01*, 2001.

[^8]: E. D. Berger, B. G. Zorn, and K. S. McKinley. Reconsidering custom memory allocation. In *Proc. OOPSLA'02*, 2002.

[^9]: E. D. Berger, B. G. Zorn, and K. S. McKinley. Reconsidering custom memory allocation. In *Proc. OOPSLA'02*, 2002.

[^10]: S. M. Blackburn, P. Cheng, and K. S. McKinley. Myths and reality: The performance impact of garbage collection. In *SIGMETRICS'04*, 2004.

[^11]: S. M. Blackburn, P. Cheng, and K. S. McKinley. Oil and Water? High Performance Garbage Collection in Java with MMTk. In *ICSE'04*, 2004.

[^12]: S. M. Blackburn and K. S. McKinley. Ulterior reference counting: Fast garbage collection without a long wait. In *Proc. OOPSLA'03*, 2003.

[^13]: H.-J. Boehm. Reducing garbage collector cache misses. In *ISMM'00*, 2000.

[^14]: H.-J. Boehm and M. Weiser. Garbage collection in an uncooperative environment. *SP&E*, 18(9):807–820, 1988.

[^15]: D. Burger, T. M. Austin, and S. Bennett. Evaluating future microprocessors: The SimpleScalar tool set. Tech. Rep. CS-TR-1996-1308, UW-Madison, 1996.

[^16]: G. Colvin, B. Dawes, and D. Adler. C++ Boost Smart Pointers, 2004.

[^17]: SPEC. Specjbb2000. http://www.spec.org/jbb2000/.

[^18]: SPEC. Specjvm98 documentation, Mar. 1999.

[^19]: D. L. Detlefs. Concurrent garbage collection for C++. In *Topics in Advanced Language Implementation*, MIT Press, 1991.

[^20]: A. Diwan, D. Tarditi, and E. Moss. Memory system performance of programs with intensive heap allocation. *ACM TOCS*, 13(3):244–273, 1995.

[^21]: J. Dolby. Automatic inline allocation of objects. In *Proc. PLDI'97*, 1997.

[^22]: J. Dolby and Chien. An evaluation of object inline allocation techniques. In *Proc. OOPSLA'98*, 1998.

[^23]: J. Dolby and Chien. An automatic object inlining and its evaluation. In *Proc. PLDI'00*, 2000.

[^24]: Y. Feng and E. D. Berger. A locality-improving dynamic memory allocator. In *Proc. MSP'05*, 2005.

[^25]: R. R. Fenichel and J. C. Yochelson. A LISP garbage-collector for virtual-memory computer systems. *CACM*, 12(11):611–612, 1969.

[^26]: D. Gay and A. Aiken. Memory management with explicit regions. In *Proc. PLDI'98*, 1998.

[^27]: D. Gay and A. Aiken. Language support for regions. In *Proc. PLDI'01*, 2001.

[^28]: W. Gloger. Dynamic memory allocator implementations in Linux system libraries.

[^29]: D. Grossman, et al. Region-based memory management in Cyclone. In *Proc. PLDI'02*, 2002.

[^30]: J. Guy L Steele. Multiprocessing compactifying garbage collection. *CACM*, 18(9):495–508, 1975.

[^31]: D. R. Hanson. Fast allocation and deallocation of memory based on object lifetimes. *SP&E*, 20(1):5–12, 1990.

[^32]: M. Hertz and E. D. Berger. Garbage collection without paging. In *Proc. PLDI'05*, 2005.

[^33]: M. Hertz, et al. Error-free garbage collection traces: How to cheat and not get caught. In *Proc. SIGMETRICS'02*, 2002.

[^34]: M. Hertz, N. Immerman, and J. E. B. Moss. Framework for analyzing garbage collection. In *IFIP TCS'02*, 2002.

[^35]: M. Hicks, et al. Experience with safe manual memory-management in Cyclone. In *ISMM'04*, 2004.

[^36]: M. W. Hicks, J. T. Moore, and S. M. Nettles. The measured cost of copying garbage collection mechanisms. In *Proc. ICFP'97*, 1997.

[^37]: M. Hirzel, A. Diwan, and T. Hosking. On the usefulness of liveness for garbage collection and leak detection. In *ECOOP'01*, 2001.

[^38]: X. Huang, et al. The garbage collection advantage: Improving program locality. In *Proc. OOPSLA'04*, 2004.

[^39]: X. Huang, et al. Dynamic SimpleScalar: Simulating Java virtual machines. Tech. Rep. TR-03-03, UT Austin, 2003.

[^40]: M. S. Johnstone and P. R. Wilson. The memory fragmentation problem: Solved? In *ISMM'98*, 1998.

[^41]: R. E. Jones and R. Lins. *Garbage Collection: Algorithms for Automatic Dynamic Memory Management*. Wiley, 1996.

[^42]: P.-H. Kamp. Malloc(3) revisited. http://phk.freebsd.dk/pubs/malloc.pdf.

[^43]: D. G. Korn and K.-P. Vo. In search of a better malloc. In *USENIX'85*, 1985.

[^44]: D. Lea. A memory allocator, 1998. http://g.oswego.edu/dl/html/malloc.html.

[^45]: J. McCarthy. Recursive functions of symbolic expressions and their computation by machine. *CACM*, 3:184–195, 1960.

[^46]: M. Michael. Scalable lock-free dynamic memory allocation. In *Proc. PLDI'04*, 2004.

[^47]: J. S. Miller and G. J. Rozas. Garbage collection is fast, but a stack is faster. Tech. Rep. AIM-1462, MIT AI Lab, 1994.

[^48]: G. Phipps. Comparing observed bug and productivity rates for Java and C++. *SP&E*, 29(4):345–358, 1999.

[^49]: N. Sachindran, J. E. B. Moss, and E. D. Berger. MC2: High-performance garbage collection for memory-constrained environments. In *Proc. OOPSLA'04*, 2004.

[^50]: P. Savola. LBNL traceroute heap corruption vulnerability. http://www.securityfocus.com/bid/1739.

[^51]: R. Shaham, E. Kolodner, and M. Sagiv. Estimating the impact of liveness information on space consumption in Java. In *ISMM'02*, 2002.

[^52]: Y. Smaragdakis, S. F. Kaplan, and P. R. Wilson. The EELRU adaptive replacement algorithm. *Performance Evaluation*, 53(2):93–123, 2003.

[^53]: D. Sugalski. Squawks of the parrot: What the heck is: Garbage collection, 2003.

[^54]: M. Tofte and J.-P. Talpin. Region-based memory management. *Information and Computation*, 132(2):109–176, 1997.

[^55]: L. Torvalds. Re: Faster compilation speed, 2002.

[^56]: D. Ungar. Generation scavenging: A non-disruptive high performance storage reclamation algorithm. In *Proc. ACM SIGSOFT/SIGPLAN'84*, 1984.

[^57]: B. Venners. *Inside the Java Virtual Machine*. McGraw-Hill, 2000.

[^58]: Wikipedia. Comparison of Java to C++, 2004.

[^59]: P. R. Wilson. Uniprocessor garbage collection techniques. In *Proc. IWMM'92*, LNCS 637, 1992.

[^60]: P. R. Wilson, M. S. Lam, and T. G. Moher. Caching considerations for generational garbage collection. In *LFP'92*, 1992.

[^61]: T. Yang, M. Hertz, E. D. Berger, S. F. Kaplan, and J. E. B. Moss. Automatic heap sizing: Taking real memory into account. In *Proc. ISMM'04*, 2004.

[^62]: B. G. Zorn. The measured cost of conservative garbage collection. *SP&E*, 23(7):733–756, 1993.
