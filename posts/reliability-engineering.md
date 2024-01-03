---
title: 可靠性工程概述
date: 2023-10-19 12:43:00
categories: 架构
tags: [架构, 可靠性, 稳定性, SRE]
---

为了应对负载的增长，目前大型网站普遍都采用分布式架构。相对于采用单体架构的系统，分布式系统中有大量的服务器及设备，各模块之间存在错综复杂的依赖关系，存在更多的不确定性。整个系统的故障率会随设备的增加而呈指数级增加，单一节点问题可能会被无限放大，日常运行过程中一定会伴随故障发生。所以，可靠性开始成为大型网站关注的最重要的质量属性之一，并因此发展出了站点可靠性工程（[Site reliability engineering](https://en.wikipedia.org/wiki/Site_reliability_engineering)，SRE）。站点可靠性工程，是从可靠性工程发展而来的，从可靠性工程中借鉴了概念和成果。本文溯本求源，内容主要是总结概括，可靠性工程的历史演进和核心概念，软件可靠性工程的核心概念，以及可靠性设计的方法。

<!--more-->

# 历史演进

可靠性工程起源于第二次世界大战[^1]。“二战”期间，美国 60% 的机载电子设备运到远东后不能使用，50% 的电子设备在存储期间失效。经过分析，发现这些电子设备故障的主要原因是电子管的可靠性太差，为此美国在 1943 年成立电子管研究委员会，在 1952 年美国国防部成立一个由军方、工业部门及学术界组成的小组，名为“电子设备可靠性咨询小组”（AGREE，Advisory Group on the Reliability of Electronic Equipment）。1957 年 6 月，AGREE 小组出版报告《军用电子设备可靠性》（Reliability of Military Electronic Equipment），该报告是公认的可靠性工程的奠基性文件，研究报告提出一整套可靠性设计、试验和管理方法，确立了可靠性工程发展方向，标志着可靠性工程学科诞生。此后，它不断向工业和民用产品领域渗透，20 世纪 60 年代推广到核工业, 70 年代在化学工业普及，并陆续扩散到其他工程领域。

随着可靠性工程学科的发展演进，围绕故障展开，逐渐衍生出对产品的维修性（maintainability）、可用性（availability）、保障性（supportability）、测试性（testability）、安全性（safety）等质量特性的工程学研究。“维修性工程”和“安全性工程”是从“可靠性工程”中分离出来的，而“保障性工程”和“测试性工程”又是从“维修性工程”中独立出来的。新的 XX 性陆续分出，因为这些特性紧密相关，已经分出的 XX 性围绕可靠性又重新集成起来，在 20 世纪 80 年代呈现综合化发展趋势。可靠性的含义不断扩展，从狭义可靠性演变为广义可靠性，从狭义可靠性工程演变为广义可靠性工程。

1980 年代初期，为了避免因为扩展可靠性固有的含义引发的可靠性定性含义和可靠性的定量含义之间的理解混乱，Jean-Claude Laprie 选择“可信性”（dependability）作为术语，根据国际电工委员会标准，可信性的定义是，用以描述可用性及其影响因素（可靠性、维修性和维修保障性）的集合性术语，可信性仅用于非定量术语的一般描述。某些情况下，可信性还包含耐久性（durability）、安全性（safety）、安全保密性（security）等其他特性。**可信性可以理解为广义可靠性**。

当今在国际上有两个比较完整的可靠性标准化体系，一个是美国军用标准（MIL-STD），另一个是国际电工委员会（IEC）标准。美国军用标准多年来一直扮演着研究开发可靠性相关标准文件的带头角色，也是最早制定可靠性标准的。最早的可靠性定义是由美国 AGREE 在 1957 年的报告中提出的，1966 年美国的 MIL-STD-721B 又给出了传统的或经典的可靠性定义，即产品在规定的条件下和规定的时间内完成规定功能的能力。最主要的可靠性国际标准组织是国际电工委员会的 TC56 技术委员会，TC56 的发展时间线[^2]：

- 1965，国际电工委员会 IEC 成立名为“电子元件和设备可靠性”（Reliability of Electronic Components and Equipment）的技术委员会，即 TC56。
- 1973，TC56 更名为“可靠性与维修性”（Reliability and Maintainability）技术委员会。
- 1985，TC56 技术委员会成立了软件可靠性工作组，开始制定软件可靠性和维修性标准。
- 1989，TC56 更名为“可信性”（Dependability）技术委员会，此名称一直沿用至今。
- 1990，TC56 在与国际标准化组织（ISO）协商后，工作范围应不再局限于电工技术领域，而是解决所有学科的通用可靠性问题，从而使 IEC/TC56 成为所谓的横向委员会。

我国与 IEC/TC56 对口的专业技术标准化组织是，TC24 全国电工电子产品可靠性与维修性标准化技术委员会（简称“可标委”），成立于 1982 年，挂靠在工业和信息化部电子第五研究所（又名中国电子产品可靠性与环境试验研究所）。

# 可信性与质量六性

除了上文的“可信性”集合性术语外，有些资料将广义可靠性解释为，“可用性 + 可靠性 + 维修性”，这三个质量特性也被缩写为 RAM。RAM 有时候会再加上安全性（Safety），被缩写为 [RAMS](https://en.wikipedia.org/wiki/RAMS)。使用 RAMS 缩写的典型例子是国际电工委员会的 [IEC 62278:2002](https://webstore.iec.ch/publication/6747) 标准（等同的国家标准 [GB/T 21562-2008](https://std.samr.gov.cn/gb/search/gbDetailed?id=71F772D76DCCD3A7E05397BE0A0AB82A)）。

另外，常见的可靠性相关的缩写是 RMS 和 RMTSS。RMS 代表可靠性、维修性、保障性，或称“三性”。RMTSS 代表可靠性、维修性、测试性、保障性、安全性，或称“五性”。质量“三性”或“五性”，也是我国军用武器装备的军用标准要求的通用质量特性。我国武器装备的军用标准学习和借鉴的是美国军用标准，军用装备的质量特性分为专用质量特性和通用质量特性。专用质量特性，反映的是不同系统或者装备自身的特点和个性特征，主要指的是功能和性能，如某型飞机的最大（最小）飞行速度、巡航速度、飞行高度等指标。通用质量特性，则表征不同装备的共性特征。通用质量特性是逐渐演变的，从一开始的“二性”演变为“六性”[^3][^4]，通用质量特性包含的特性如下：

- 二性：可靠性、维修性（也缩写为 R&M）
- 三性：可靠性、维修性、保障性（也缩写为 RMS）
- 五性：可靠性、维修性、测试性、保障性、安全性（也缩写为 RMTSS）
- 六性：可靠性、维修性、保障性、测试性、安全性、环境适应性

**通用质量特性与可信性的含义大体上相同**。质量特性、可信性、可靠性的关系，如下图所示：

<img width="700" alt="质量特性、可信性、可靠性的关系" title="质量特性、可信性、可靠性的关系" src="https://static.nullwy.me/quality-characteristics-classification.png">


根据国际电工委员会 IEC 60050 (191):1990（等同的国家标准 GB/T 3187-1994 《可靠性、维修性术语》）、IEC 60050-192:2015（等同于 GB/T 2900.99-2016 《电工术语 可信性》）以及国家军用标准 GJB 451A-2005 《可靠性维修性保障性术语》（与美国军用标准 [MIL-STD-721C](http://everyspec.com/MIL-STD/MIL-STD-0700-0799/MIL-STD-721C_1040/) 相似）等标准文档，相关特性的定义和度量指标如下：

- **可信性（[dependability](https://www.electropedia.org/iev/iev.nsf/display?openform&ievref=192-01-22)）**：用以描述可用性及其影响因索（可靠性、维修性和维修保障性）的集合性术语。可信性仅用于非定量术语的一般描述。
- **可靠性（[reliability](https://www.electropedia.org/iev/iev.nsf/display?openform&ievref=192-01-24)）**：在给定的条件，给定的时间区间，能无故障地执行要求的能力。可靠性一般用可靠度（reliability）、平均故障间隔时间（[MTBF](https://www.electropedia.org/iev/iev.nsf/display?openform&ievref=192-05-13)）、使用寿命（useful life）等参数来度量。
- **可用性（[availability](https://www.electropedia.org/iev/iev.nsf/display?openform&ievref=192-01-23)）**：在所要求的外部资源得到提供的情况下，产品在给定的条件下，在给定的时刻或时间区间内处于能完成要求的功能的状态的能力。此能力是产品的可靠性、维修性和维修保障性的综合反映。可用性的度量指标称为可用度（availability），表示为平均可用时间同平均可用时间与平均不可用时间的和之比。
- **维修性（[maintainability](https://www.electropedia.org/iev/iev.nsf/display?openform&ievref=192-01-27)）**：在给定的条件下，使用所述的程序和资源实施维修时，产品在给定的使用条件下保持或恢复能完成要求的功能的状态的能力。维修性反映产品修理的难易程度，主要使用平均修复时间（MTTR）来度量。
- **保障性（[supportability](https://www.electropedia.org/iev/iev.nsf/display?openform&ievref=192-01-31)）**：在规定的运行剖面和给定的后勤与维修资源下，保防能维待要求的可用性的能力。维修保障（maintenance support），是维修产品的资源的供给，资源包括人力资源、保障设备、材料和备件、维修设施、文档和信息以及维修信息系统。保障性一般用平均保障延误时间、资源满足率、资源利用率等参数来度量。
- **测试性（testability）**：是指产品能及时并准确地确定其状态（可工作、不可工作或性能下降），并隔离其内部故障的能力。测试性反映产品是否易于测试、出现故障时是否易于检测和隔离，一般用检测时间、技术准备时间、故障检测率、故障隔离率等参数来度量。
- **安全性（safety）**：是指产品所具有的不导致人员伤亡、系统毁坏、重大财产损失或不危及人员健康和环境的能力。安全性可理解为产品在任何情况下对人员、系统、财产和环境都不构成安全威胁。它可定义为产品在规定的条件下和规定的时间内，以可接受的风险执行规定功能的能力。安全性一般用事故概率、损失率、安全可靠度等参数来度量。
- **环境适应性（environmental worthiness）**：是指产品在其寿命期内预计可能遇到的各种环境作用下能实现其所有预定功能、性能和（或）不被破坏的能力。它反映了产品对各种环境的适应能力，即在其可能遇到的各种环境下均能正常工作的能力，是可靠性的一种特殊情况。环境适应性鉴定试验通过的判定准则是，在所有试验条件下“零故障”。

**通用质量特性的“六性”之间是紧密联系的**[^4][^5]：

- 维修性是对可靠性的补充，如果产品不发生故障就不需要修复性维修。
- 可用性是产品的可靠性、维修性和维修保障性的综合反映。
- 保障性为产品正常使用与维修提供外部资源的支持，提供使用保障和维修保障，可靠性和维修性依赖于保障性。
- 测试性是维修性的基础，维修依赖于测试，产品要修理一定要先发现故障和隔离故障，所以测试性设计得好，维修时间就可大大缩短。
- 安全性本是可靠性的一部分，是避免人员伤亡、健康损害、财产或环境损害风险的可靠性。可靠性是安全性的基础，很多安全性问题都是因为产品不可靠造成的，所以提高产品可靠性也能提高安全性，当然并非所有安全性问题都是不可靠引起的。
- 环境适应性是可靠性的一种特殊情况，是可靠性研究的前提，研究可靠性首先要确定产品是否有足够的环境适应性。

通用质量特性的“六性”工作围绕故障（failure）而展开，也被人称为“故障六性”[^6][^4]。可靠性的目标是减少故障；维修性的目标是修复故障；测试性的目标是检测故障；保障性保证出现故障时可以快速供应维修资源；安全性旨在出现故障以后降低风险；环境适应性鉴定试验通过的判定准则是，在所有试验条件下“零故障”[^5]。

<img width="450" alt="故障六性" title="故障六性" src="https://static.nullwy.me/quality-6-characteristics.png">

可靠性是产品质量特性之一，是一种面向时间的质量特性（time oriented quality characteristic）[^7]。Lloyd Condra 在[书中](https://www.amazon.com/dp/B00SC8DKDG)对可靠性和质量的关系的解释是，“可靠性是质量随着时间的变化（reliability is quality over time）”，“为了衡量产品的质量水平，我们对现在的产品进行评判，而为了度量产品的可靠性水平，则要对产品未来会是什么样子进行评判”。这里的讨论的质量其实指的是符合性质量，质量管理一开始是从符合性质量开始的，质量管理的主要工作是质量检验（Quality Inspection），检测产品是否符合规格，质量检验的结果即合格品率。质量管理的关注焦点是产品的合格品率，而可靠性关注焦点是产品在用户使用过程中合格水平随着时间的保持能力，如下图所示 [^5]：

<img width="600" alt="质量与可靠性关系示意图" title="质量与可靠性关系示意图" src="https://static.nullwy.me/quality-vs-reliability.png">

# 故障与失效的区别

可靠性工程中有两个基本而重要的术语“fault”和“failure”。在我国的可靠性标准文档中，把“fault”翻译为“故障”，把“failure”翻译为“失效”或“故障”，也就是说，“失效”对应的英文只有“failure”；而“故障”对应的英文是“fault”或“failure”[^8][^9]。

我国第一个定义可靠性相关的常用术语的国家标准是 GB 3187-1982 《可靠性基本名词术语及定义》，该标准对英文术语“failure”的中文翻译是“失效”或“故障”，并把“mean time between failures”翻译是“平均无故障时间”：[^8]

> 2.2.1 失效（故障） failure：产品丧失规定的功能。对可修复产品通常也称故障。
> 2.5.5 平均寿命（平均无故障时间） mean life (mean time between failures)：寿命（无故障时间）的平均值。

替代 GB 3187-1982 的新国家标准是 [GB/T 3187-1994](https://std.samr.gov.cn/gb/search/gbDetailed?id=71F772D7962ED3A7E05397BE0A0AB82A) 《可靠性、维修性术语》（等同于 IEC 60050-191:1991），该标准把“failure”仅翻译为“失效”，不再又称“故障”。完整的术语定义如下：

> 4.1.1 失效 failure
> 产品终止完成规定功能的能力这样的事件。
> 4.2.1 故障 fault
> 产品不能执行规定功能的状态。预防性维修或共他计划性活动或缺乏外部资源的情况除外。故障通常是产品本身失效后的状态，但也可能在失效前就存在。

基于上述的定义，对于性能随时间逐渐退化的产品，故障（fault）与失效（failure）的区别，如下图所示[^10]：

<img width="600" alt="故障（fault）与失效（failure）的区别" title="故障（fault）与失效（failure）的区别" src="https://static.nullwy.me/reliability-fault-vs-failure.png">

国外历来都是将“fault”和“failure”的定义加以区分的，我国新的标准文档也区分翻译为“故障”和“失效”，但是实际上在我国的电工行业中的惯用情况是把“fault”和“failure”都翻译为“故障”，很多书籍资料也不严格区分。比如，中国质量协会的《可靠性工程师手册》（第 2 版 2017[^5])，不严格区分“故障”与“失效”术语，统一都使用“故障”，书中的解释如下：

> 在我国的可靠性工程应用中，一般不对故障与失效进行严格的区分，如失效树分析也称为故障树分析，故障模式、影响分析也称为失效模式、影响分析。因此本书也不做严格区分，多数情况下故障一词也可用失效代替。

一般而言，故障是产品本身失效后的状态，此时产品处于故障状态，这时故障和失效是不需要严格加以区分。对无故障容忍能力的产品而言，故障即失效。然而，对有故障容忍能力的产品，产品可以出故障，但不会失效，这时我们就必须区分失效和故障的概念[^9]。故障容忍（容错，[fault tolerance](https://en.wikipedia.org/wiki/Fault_tolerance)），是在某些故障出现时继续运行的能力，只有当所有冗余的硬件同时有故障时，产品才失效。

# 可靠性的度量

可靠性的度量指标是**可靠度**（[reliability](https://www.electropedia.org/iev/iev.nsf/display?openform&ievref=192-05-05)），根据国际电工委员会的标准文档，可靠度的定义是：在给定的条件下在时间区间 (t1, t2) 内按要求执行的概率。当 t1=0 和 t2=t，则 R(0,t) 可简化为 R(t)，并称为产品的**可靠度函数**（reliability function）。

若在 t=0 时产品的总数为$N_0$，在 0 ~ t 的时间内累计的故障数为$N_f(t)$，正常的产品数为$N_s(t)$，则有：

$N_0 = N_f(t) + N_s(t)$

产品在 t 时刻的可靠度的估计值为：

$\hat{R(t)}=\frac{N_s(t)}{N_0}$

显然，当 t=0 时，R(0) = 1，当 t=∞ 时，R(∞) = 0。

可靠度的反面是**不可靠度**（unreliability），含义是：在规定的条件下，在规定的时间内，不能完成规定功能的概率。**不可靠度函数**通常记为$F(t)$，不可靠度的估计值的计算公式为：

$\hat{F(t)} = \frac{N_f(t)}{N_0} = \frac{N_0 - N_s(t)}{N_0} = 1 - \hat{R(t)}$

**示例**：假设在 t=0，投入工作的 10000 只灯泡，以天作为度量时间的单位，在 t= 365 天时，发现有 300 只灯泡坏了，这时的可靠度和不可靠度的计算如下：

$\hat{R(365)} = \frac{1000 - 300}{1000} = 0.97$，$\hat{F(365)} = \frac{300}{1000} = 0.03$

对不可靠度函数求导，其导数称为**故障概率密度函数**，通常记为$f(t)$：

$f(t) = \lim_{\Delta t \to 0} \frac{F(t + \Delta t) - F(t)} {\Delta t}$

某时刻尚未发生故障的产品，在该时刻后单位时间内发生故障的概率，称为产品的**故障率**（[failure rate](https://en.wikipedia.org/wiki/Failure_rate)），记为记为$λ(t)$。故障率的计算公式如下：

$\hat{λ(t)} = \frac{N_s(t) - N_s(t + \Delta t)} {N_s(t) \Delta t} = \frac{间隔时间内的故障数}{间隔起点的存活数 × 时间间隔}$

$λ(t) = \lim_{\Delta t \to 0} \frac{N_s(t) - N_s(t + \Delta t)} {N_s(t) \Delta t} = \lim_{\Delta t \to 0} \frac{F(t + \Delta t) - F(t)}{R(t)\Delta t} = \frac{f(t)}{R(t)}$

在长期的可靠性实践中，人们发现许多产品的故障率随时间的变化曲线形似浴盆，所以习惯性的将故障率曲线称为“浴盆曲线”（[bathtub curve](https://en.wikipedia.org/wiki/Bathtub_curve)），如下图所示[^5]。大多数电子产品的故障率曲线的形状就是浴盆曲线。

<img width="600" alt="产品典型的故障率曲线" title="产品典型的故障率曲线" src="https://static.nullwy.me/reliability-bathtub-curve.png">

故障率随时间的变化大致可以分为以下三个阶段：早期故障期（[early failure period](https://www.electropedia.org/iev/iev.nsf/display?openform&ievref=192-02-28)）、偶然故障期（[random failure period](https://www.electropedia.org/iev/iev.nsf/display?openform&ievref=192-02-30)）、耗损故障期（[wear-out failure period](https://www.electropedia.org/iev/iev.nsf/display?openform&ievref=192-02-31)）。三个阶段有时也被称为：早夭期（infant mortality period）、使用寿命期（useful life period）和耗损期（wear-out period）。故障率曲线与人类的死亡率曲线相似，曲线的三个阶段分别对应人类生命周期的婴幼儿时期、壮年期以及老年期。在偶然故障期，产品的故障率可降到一个较低的水平，且基本处于平稳状态，可以近似认为故障率为常数。

如果产品的故障率为常数，那么其故障的概率分布可以用指数分布（[exponential distribution](https://zh.wikipedia.org/wiki/%E6%8C%87%E6%95%B0%E5%88%86%E5%B8%83)）描述，指数分布是唯一具有恒定故障率的连续概率分布。恒定故障率的特性也被称为“无记忆性”（[memorylessness](https://en.wikipedia.org/wiki/Memorylessness)），该特性说明故障率在任何时刻都与系统已工作过的时间长短没有关系。服从指数分布的概率密度函数为：

$f(t) = {\lambda} e^{-\lambda t}$

服从指数分布的可靠度函数、不可靠度函数和故障率函数依次为：

$R(t) = e^{-\lambda t}$，$F(t) = 1 - e^{-\lambda t}$，$\lambda(t) = \lambda$

公式中的$\lambda$为常数。

上面提到的这些指标是概率相关的度量指标。另外，还有一些是时间相关的度量指标：

- **平均故障间隔时间**（MTBF，mean time between failures）：相邻两次故障间的持续时间的平均值。失效间隔时间（[time between failures](https://www.electropedia.org/iev/iev.nsf/display?openform&ievref=192-05-03)）包括可用时间和不可用时间。MTBF 只能用于可修复产品。
- **平均故障间隔工作时间**（MTBF /[MOTBF](https://www.electropedia.org/iev/iev.nsf/display?openform&ievref=192-05-13)，mean operating time between failures）：相邻两次故障间的累计工作时间的平均值。MOTBF 只能用于可修复产品，该值也被成为可修复产品的平均寿命（mean life）。
- **平均故障前工作时间**（[MTTF]()，mean operating time to failure）：故障前工作时间的平均值。等同于，平均失效前时间（MTTF，mean time to failure）。MTTF 可以用于不可修复产品和可修复产品。MTTF 值也被成为不可修复产品的平均寿命（mean life）。
- **平均恢复时间**（[MTTR](https://www.electropedia.org/iev/iev.nsf/display?openform&ievref=192-07-23)，mean time to restoration）：恢复时间的平均值。最新的国际电工委员会 IEV 标准文档，废弃了平均恢复时间（mean time to recovery）和平均修理时间（mean time to repair）。平均恢复时间是维修性的主要度量指标。
- **可用性**（[availability](https://en.wikipedia.org/wiki/Availability)）：或者翻译为“可用度”，可以表示为平均可用时间除以平均可用时间与平均不可用时间之和。可用性是反映可靠性和维修的综合性指标。

根据国际电工委员会标准文档 IEC 60050-192:2015（等同的国家标准文档是 [GB/T 2900.99-2016](https://std.samr.gov.cn/gb/search/gbDetailed?id=71F772D81729D3A7E05397BE0A0AB82A)《电工术语 可信性》），MTBF 和 [MOTBF](https://www.electropedia.org/iev/iev.nsf/display?openform&ievref=192-05-13) 缩写代表的都是**平均故障间隔工作时间**（mean operating time between failures），统计的值是**故障间隔工作时间**（[operating time between failures](https://www.electropedia.org/iev/iev.nsf/display?openform&ievref=192-05-04)），即相邻两次故障间的累计工作时间；而**故障间隔时间**（[time between failures](https://www.electropedia.org/iev/iev.nsf/display?openform&ievref=192-05-03)），包括可用时间和不可用时间。所以，**平均故障间隔时间**和**平均故障间隔工作时间**的含义是不同的，前者同时统计包括工作时间（operating time）和非工作时间（non-operating time），而后者只统计工作时间（operating time）。但是在很多其他文档中，比如维基百科的词条“平均故障间隔时间”（[Mean time between failures](https://en.wikipedia.org/wiki/Mean_time_between_failures)）词条，“平均失效间隔时间”的缩写也是 MTBF。**也就说， MTBF 即是“平均故障间隔时间”的缩写，也是“平均故障间隔工作时间”的缩写，但两者含义却不同，需要读者自行辨别**。本文使用 MTBF，统一都表示“平均故障间隔时间”。

**MTBF 只能用于可修复产品，MTTF 用于不可修复产品和可修复产品**。对于不可修复产品，MTTF 等同于 MTBF，对于可修复产品，MTTF 等同于 MOTBF。MTBF 是 MTTR 和 MTTF 的总和，即$MTBF = MTTR + MTTF$。

产品的**平均寿命**（mean life）的理论值为故障概率密度函数的[期望值](https://zh.wikipedia.org/wiki/%E6%9C%9F%E6%9C%9B%E5%80%BC)，记为 θ。该期望值也是可修复产品的 MOTBF 的理论值，是不可修复产品的 MTTF 的理论值。平均寿命或 MTTF 的计算公式如下：

$MTTF = θ = E(T) = \int_{0}^\infty t f(t)\,\mathrm{d}x$

如果概率密度函数服从指数分布，那么平均寿命或 MTTF 值为：

$MTTF = θ = 1/\lambda$

对于服从指数分布的产品，当产品工作时间到达平均寿命时，可靠度的值 36.8%：

$R(θ) = e^{-1} = 36.8\%$

假设某可修复产品的正常工作时间为$\left\{ {TFF}_1, {TFF}_2 ...,  {TFF}_n \right\}$，故障时间为$\left\{ {TTR}_1, {TTR}_2 ..., {TTR}_n \right\}$，如下图所示：

<img width="600" alt="MTBF、MTTF 和 MTTR 示意图" title="MTBF、MTTF 和 MTTR 示意图" src="https://static.nullwy.me/reliability-mtbf-mttf-mttr.png">

MTBF、MTTF、MTTR 和可用性（availability）的估计值的计算公式如下：

$MTBF = \frac{\sum_{i=1}^n {TBF}_i}{n} = \frac{\text{total uptime + total downtime}}{\text{total number of failures}}$

$MTTF = \frac{\sum_{i=1}^n {TTF}_i}{n} = \frac{\text{total uptime}}{\text{total number of failures}}$

$MTTR = \frac{\sum_{i=1}^n {TTR}_i}{n} = \frac{\text{total downtime}}{\text{total number of failures}}$

$可用性 = \frac {MTTF} {MTTF + MTTR} = \frac {MTTF} {MTBF} = \frac{\text{total uptime}}{\text{total uptime} + \text{total downtime}}$

对于电子设备，MTBF 近似常量，比如，典型的企业级固态硬盘 SSD 的 MTBF 值可能是 200 万小时[^11]，也就是 228 年，年故障率 AFR（[Annualized Failure Rate](https://en.wikipedia.org/wiki/Annualized_failure_rate)） 为 0.44%（1/228）。

可以利用数学和统计方法对可靠性进行预计和度量，并分析可靠性数据。但是可靠性的量化涉及大量不确定性，由于可靠性经常关系到制造和使用产品的人，人的行为和表现不像植物对肥料的反应、气象模型对海洋温度的反应那样服从于数学分析和预测，而且产品可能在大范围变化的环境中工作，因此还会有其他不确定性因素被引入。数学和统计方法虽然在适当的场合是非常有价值的，但是由于涉及大量不确定性，在实际的可靠性工程中的作用有限，在实际的工程实践中优先需要确定的是故障的原因和解决方案[^12]。

# 软件可靠性工程

软件可靠性工程是从硬件可靠性工程发展而来的[^13]。在软件工程学建立初期，一些软件工程专家利用和改造硬件可靠性工程学的成果，使之移植到软件领域，从而开创了软件可靠性学科。

类似的，起源于 Google 的“站点可靠性工程”（SRE, [Site reliability engineering](https://en.wikipedia.org/wiki/Site_reliability_engineering)）也源自可靠性工程，区别是站点可靠性工程关注的是大型网站和网络服务这样的软件系统 [^14][^15]。

## 术语与概念

国家标准 [GB/T 11457-2006](https://std.samr.gov.cn/gb/search/gbDetailed?id=71F772D7805FD3A7E05397BE0A0AB82A) 《信息技术 软件工程术语》，对软件可靠性的定义如下：

> 2.1662 系统可靠性 system reliability
> 包括全部硬件和软件子系统在内的某个系统，在规定的环境及时间里正确执行所要求的任务或使命的概率。
> 2.1528 软件可靠性 software reliability
> a) 在规定条件下，在规定的时间内，软件不引起系统失效的概率。该概率是系统输人和系统使用的函数，也是软件中存在的缺陷的函数。系统输人将确定是否会遇到已存在的缺陷（如果有缺陷存在的话）。
> b) 在规定的时间周期内所述条件下程序执行所需要的功能的能力。

GB/T 11457-2006 吸收了 [IEEE Std 610.12-1990](https://standards.ieee.org/ieee/610.12/855/) 全部术语，包括术语“软件可靠性”，国标的术语定义只是对 IEEE 术语定义的中文翻译。IEEE 最早在标准文档中定义“软件可靠性”是在 [IEEE Std 729-1983](https://standards.ieee.org/ieee/729/967/)，这个标准之后被 IEEE Std 610.12-1990 替代。

IEEE 对“软件可靠性”术语给出两个定义，定义 a) 是定量的定义，也就是“可靠度”，定义 b) 是定性的定义。对比后容易发现，硬件可靠性和软件可靠性的定义是相同的。这种相容性，使这种可靠性定义能够用于既包括软件又包括硬件的系统。

故障相关的术语，国家标准 GB/T 11457-2006 《信息技术 软件工程术语》的定义是：

> **2.163 隐错 bug**
> 见：出错 error(2.561) 和故障 fault(2.609)。
> **2.421 缺陷 defect**
> 见：故障 fault(2.609)。
> **2.561 出错, 误差, 差错 error**
> a) 计算的、观察的或测量的值或条件与实际的、规定的或理论上正确的值或条件的差别。例如，在计算的结果和正确的结果之间差30m；
> b) 不正确的步骤、过程或数据定义。例如，在计算机程序中的不正确的指定；
> c) 不正确的结果。例如，当正确的结果是10，而计算的结果是12；
> d) 产生不正确结果的人为动作。例如，在编程或操作的一部分上的不正确动作。
> 注：当上述所有四种定义是公共使用时，一种区分赋给定义 a) 为字差错（error），定义 b) 为字过错（fault），定义 c) 为字失效（failure）和定义 d) 为字错误（mistake）。
> **2.601 失效 failure**
> 系统或部件不能按规定的性能要求执行它所要求的功能。注：故障容忍在人们的动作（弄错-mistake）、它的显示（硬件或软件故障 fault）、故障的结果（失效 failure）和不正确（差错－error）结果的总数之间进行区分
> **2.609 故障, 缺陷 fault**
> a) 硬件设备或部件中的缺陷。例如，短路或断线。
> b) 在计算机程序中不正确的步骤、过程或数据定义。注：此定义最初由容错（fault tolerance）系统使用。在通常用法中，术语“差错（error）”和“隐错（bug）”表示同样含义。

基于 IEEE 的定义，容易发现术语“fault、“[bug](https://en.wikipedia.org/wiki/Software_bug)”和“defect”是同义词，术语“error”同时具有“mistake”、“fault”和“failure”的含义。这些术语之间的因果关系是，开发者在软件开发过程中产生人为失误（mistake），导致在软件中存在缺陷（fault, bug, defect），在软件运行时如果用户遭遇缺陷（fault, bug, defect），会引发失效（failure），如下图所示：

<img width="600" alt="软件故障的因果关系" title="软件故障的因果关系" src="https://static.nullwy.me/reliability-software-failure-cause.png">

失效（failure）是指系统或部件在特定约束下不能完成所要求的功能，用户在测试或实际使用中会观察到失效（failure）。**失效（failure）是系统运行行为对用户要求的偏离，是一种面向用户的概念。故障（fault）是在系统运行时引起或可能潜在地引起失效（failure）的缺陷（defect），是一种面向开发者的概念。**

## 可靠性的度量

早期的软件可靠性度量工作试图将硬件可靠性理论中的数学公式外推来进行软件可靠性的预测。大多数与硬件相关的可靠性模型依据的是由于“磨损”而导致的故障，而不是由于设计缺陷而导致的故障。在硬件中，由于物理磨损（如温度、腐蚀、振动的影响）导致的故障远比与设计缺陷有关的故障多。不幸的是，软件恰好相反。实际上，所有软件故障都可以追溯到设计或实现问题，磨损根本没有影响[^16]。

软件系统的故障主要是人为差错造成，涉及大量不确定性，利用数学和统计方法对可靠性进行预计和度量，在实际的工程实践中的作用有限。少数常用的与可靠性相关的度量指标是：MTBF、MTTR 和可用性。

MTBF 指标衡量的是系统无故障运行的能力，也就是可靠性。MTTR 指标衡量的是系统快速从故障中恢复的能力，这种能力在硬件产品下被称为“维修性”（maintainability），但在软件系统下通常为称“韧性”（resilience）。术语“maintainability”，在硬件上下文中通常被翻译为“维修性”，而在软件上下文中通常被翻译为“维护性”或“可维护性”，软件可维护性指的是软件可被修改的能力，修改可能包括修复缺陷、增加或完善功能等。

MTBF 反映的是硬件产品的寿命，是硬件产品质量的最重要的指标之一。与硬件不同，软件的 MTBF 不可控，而故障恢复的工作流程清晰，可干预程度高，研发团队可以对各环节展开精细化管理，轻松、高效地达成 MTTR 优化目标，所以软件系统的 **MTTR 相对 MTBF 更加重要**[^17][^18]。

可用性是衡量可靠性和韧性的综合性指标。想要提高系统的可用性，需要做的是延长无故障时间（MTTF）和缩短故障恢复时间（MTTR）。

# 可靠性设计

Laprie 等人把提高系统可靠性的方法总结为[四种](https://en.wikipedia.org/wiki/Dependability#Means)：

- **故障避免（fault avoidonce，简称“避错”）**：在系统的设计和实现过程中使用一些开发方法来减少故障发生，并在系统投入使用之前发现系统中的故障。
- **故障排除（fault removal，简称“排错”）**：故障排除可以细分为两个子类别：开发期间的排除和使用期间的排除。在系统使用之前通过[验证和确认](https://zh.wikipedia.org/wiki/%E9%A9%97%E8%AD%89%E5%8F%8A%E7%A2%BA%E8%AA%8D)（V&V）来发现和去除系统中的故障；如果系统已经投入使用，通过维护周期将其消除。
- **故障容忍（[fault tolerance](https://en.wikipedia.org/wiki/Fault_tolerance)，简称“容错”）**：容错是使系统在其某些组件中出现一个或多个故障时能够继续提供服务的能力，尽管该服务可能处于降级级别。容错技术主要是采用**冗余**（[redundancy](https://en.wikipedia.org/wiki/Redundancy_(engineering))）方法来消除故障的影响，冗余的含义是指当系统无故障时取消冗余资源不会影响系统正常运行。
- **故障预报（fault forecasting）**：通过收集故障数据，建立可靠性建模，预测可能的故障。

针对软件可靠性设计，软件故障避免技术，包括采用优秀的软件设计方法、使用强类型的程序设计语言、全面的编译器检查等；软件故障排除技术，主要是代码评审和软件测试；故障预报，能提高硬件可靠性，但是很少应用于软件可靠性。

系统的资源包括硬件资源、软件资源、信息资源、时间资源，所以冗余区分 4 种方式：

- **硬件冗余**（hardware redundancy）：通过配置额外的硬件组件实现冗余。
- **软件冗余**（software redundancy）：通过配置额外的软件版本实现冗余，例如 N 版本编程（[NVP](https://en.wikipedia.org/wiki/N-version_programming)）。
- **信息冗余**（information redundancy）：通过对信息中外加一部分信息码或将信息存放在多个内存单元或将信息进行备份等实现冗余，例如循环冗余校验码、数据库备份等。
- **时间冗余**（time redundancy）：多次执行相同的操作（重试）实现冗余，例如多次执行程序或传输数据的多个副本。

硬件冗余和软件冗余被合称为结构冗余（structural redundancy）。相对与时间冗余，硬件冗余、软件冗余、信息冗余被合称为空间冗余（space redundancy）。硬件冗余比较常见，而软件冗余相对少见。

在工程领域，利用冗余提高可靠性的例子很多，比如汽车的备胎、大货车的多个轮子、飞机的四台发动机或双台发动机、火箭的多台引擎和多台计算机等。《像火箭科学家一样思考》书中的“为什么冗余不是多余的”小节[^19]中有这样一段阐述：

> 航天器上的计算机也使用冗余装置。在地球上，电脑往往免不了崩溃或死机，而在有压力的太空环境中，计算机发生故障的概率有增无减，因为计算机在太空中要经历无数振动、冲击、变化的电流和波动的温度。正因为如此，航天飞机的计算机是4倍冗余的，即飞机上有4台计算机在运行着同样的软件。这4台计算机会通过一个多数投票系统就下一步动作进行单独投票。如果其中一台计算机发生故障，开始乱输出数据，其他3台计算机就会投票将其排除在外（没错，伙计们，火箭科学比你想象的更民主）。
> 冗余装置要正常工作，就必须独立运行。一架航天飞机配备4台计算机，这听起来非常棒，但由于它们运行着相同的软件，所以只要一个软件出现错误，4台计算机就会同时瘫痪。因此，航天飞机还配备了第5个备用飞行系统。该系统安装有一款不同的软件，而这款软件由不同于其他4款软件的分包商提供。如果某个一般性的软件错误使4台相同的主计算机瘫痪，则备用系统将启动，并会将航天飞机送回地球。

在航天飞机上配备 4 台计算机属于硬件冗余，第 5 个备用飞行系统属于软件冗余。

IDC 数据中心等级划分（[data centre tiers](https://en.wikipedia.org/wiki/Data_centre_tiers)）主要是根据线路、电源、冷却等核心组件的冗余程度而划分的，不同的等级代表不同的可靠性和可用性。按 Uptime Institute 和 TIA-942 标准的建议，数据中心由低到高划分为 T1、T2、T3、T4 共 4 个等级，T1 级为基本型、T2 级为冗余型、T3 级为可并行维护冗余型、T4 级为容错型。4 个等级的冗余程度分别是，T1 级无冗余，T2 级部分 N+1 冗余，T3 级全部 N+1 冗余，T4 级 2N 或 2N+1 冗余。按我国国家标准《GB 50174-2017 数据中心设计规范》，数据中心由高到低划分为 A、B、C 三级，A 级为容错型，B 级为冗余型，C 级为基本型。不同的数据中心等级的对比，如下表所示（表格参考自[^20]）：

<img width="700" alt="数据中心等级划分" title="数据中心等级划分" src="https://static.nullwy.me/data-centre-tiers.png">

大多数商业数据中心都介于 T3 级和 T4 级之间，平衡了建设成本和可靠性。金融等行业的数据中心，如银行的数据中心，通常会同时遵循最高等级的国家标准 A 级和 Uptime Institute 的 Tier 4 级标准来建设。

在生物学中，冗余是生物体的一种重要特征。生物体中的冗余结构和功能可以提高生物体的适应性和生存能力。例如，人体的很多器官是冗余的，比如耳朵、眼睛、肾、肺。冗余的器官如果出现“故障”，虽然不会让人完全失去该器官的机能，但会导致机能降级。其中一个眼睛如果完全失去视力，不会让人失明，不过视觉上无法识别远近。其中一个耳朵如果完全没有听力，不会让人失聪，但无法透过耳朵识别声音的位置。

尽管冗余是一种很好的提高可靠性的措施，但额外的冗余增加到某种程度之后，就会无谓地增加设备的复杂性和成本，遵循边际效益递减规律。典型的例子是客机，随着发动机本身可靠性的提高，出于安全性和成本之间的权衡，之前的四发动机的客机越来越少见，逐渐被更省油、更低维护成本的双发动机的客机取代。

# 参考资料

[^1]: 秦咏红，吕乃基：工程系统可靠性的演进，东北大学学报，2011年第4期295-299，[cnki](https://kns.cnki.net/kcms2/article/abstract?v=zcLOVLBHd2wIt8gpzydgP6J3pB-hC88RrjSmTExnvcHvDDGG-WWagNHfY8JcfcgvEZjVrR4jQJ5Duh_KjbIUFWmw0rDVJqeHyfZG93hQC7B8vh91-Q5vECPlrcF0-t_l&uniplatform=NZKPT&flag=copy)，[cqvip](https://qikan.cqvip.com/Qikan/Article/Detail?id=38777595)
[^2]: 可靠性概论，工信部电子第五研究所潘勇，2015，[豆瓣](https://book.douban.com/subject/26870237/)：第11章 可靠性标准
[^3]: 2022-09 【标准解读】用标准语言解读装备“六性” <https://mp.weixin.qq.com/s/FqfLL_ovGIZ-GO998HB4uw>
[^4]: 2021-07 张健：装备通用质量特性关系概述 <https://mp.weixin.qq.com/s/nXmrTrz7c5EnEh4C_h4JEg>
[^5]: 可靠性工程师手册，中国质量协会，第2版 2017，[豆瓣](https://book.douban.com/subject/10607952/)
[^6]: 2018-01 北航康锐：可靠性的历史与今世 | 可靠性系统工程三部曲（上） <https://mp.weixin.qq.com/s/275pk9Z-V4T3lntI-ZzYCA>
[^7]: 可靠性工程（Reliability Engineering），Kapur & Pecht，2014，[豆瓣](https://book.douban.com/subject/30619305/)：第1章 21世纪的可靠性工程
[^8]: 2002-03 褚善元：failure和fault的定名问题 <http://www.term.org.cn/CN/abstract/abstract10076.shtml>
[^9]: 2002-03 朱美娴：关于failure和fault定义的研讨 <http://www.term.org.cn/CN/abstract/abstract10081.shtml>
[^10]: System Reliability Theory, Rausand, etc., 3rd 2020，[豆瓣](https://book.douban.com/subject/35473900/)：3 Failures and Faults, Figure 3.3 Illustration of the difference between failure and fault for a degrading item.
[^11]: 2021-09 揭秘：SSD 的“可靠性”到底可不可靠 <https://memblaze.com/innovate/technical-articles/169.html>
[^12]: 实用可靠性工程（Practical Reliability Engineering），O'Connor, etc.，第5版2012，[豆瓣](https://book.douban.com/subject/35263950/)
[^13]: 陈光宇，黄锡滋：软件可靠性学科发展现状及展望，电子科技大学学报社科版，2002年第3期99-102，[cnki](https://kns.cnki.net/kcms2/article/abstract?v=zcLOVLBHd2zqRn-SkRJjlRMqbVpWEZ7eixyUtZPjIkGDWM0ZZEly0jGgd6xE2xxgo1E7iX46uLXaMFUfS0eD7b9dTozKwVt_XXdd_svP-taFGshzUvzPODSCvVkd2rIz&uniplatform=NZKPT&flag=copy)，[cqvip](https://qikan.cqvip.com/Qikan/Article/Detail?id=6859748)
[^14]: SRE：Google运维解密，Beyer, etc. 2016，[豆瓣](https://book.douban.com/subject/26875239/)：序言
[^15]: SRE原理与实践：构建高可靠性互联网应用，张观石，2022，[豆瓣](https://book.douban.com/subject/36202918/)：第1章 互联网软件可靠性概论
[^16]: 软件工程：实践者的研究方法，Pressman，第8版2014，[豆瓣](https://book.douban.com/subject/26918148/)：第21章 软件质量保证，21.7 软件可靠性
[^17]: 2010-11 John Allspaw: MTTR is more important than MTBF (for most types of F) <https://www.kitchensoap.com/2010/11/07/mttr-mtbf-for-most-types-of-f/>
[^18]: 2023-07 LigaAI：研发质量指标大 PK：MTTR vs MTBF，谁是靠谱王？ <https://segmentfault.com/a/1190000043971564>
[^19]: 像火箭科学家一样思考，Ozan Varol，2020，[豆瓣](https://book.douban.com/subject/35228079/)：第1章 与不确定性共舞，为什么冗余不是多余的
[^20]: 2020-12 艾瑞咨询：2020年中国数据中心行业研究报告 <https://report.iresearch.cn/report/202012/3699.shtml>
