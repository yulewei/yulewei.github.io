---
title: 大型网站的稳定性、可靠性和韧性
date: 2023-12-25 10:36:00
categories: 架构
tags: [架构, 稳定性, 可靠性, 韧性, SRE]
---

为了应对负载的增长，提升系统性能，目前大型网站普遍都是分布式架构，采用微服务架构风格。分布式系统的最重要的架构特性是**伸缩性**（scalability），伸缩性的系统具备应对增长的工作负载的能力。关于性能和伸缩性，可以参阅笔者的文章《[大型网站的性能和可伸缩性](https://nullwy.me/2023/12/website-performance-scalability/)》。相对于采用单体架构的系统，分布式系统中有大量的服务器及设备，各服务之间存在错综复杂的依赖关系，存在更多的不确定性。整个系统的故障率会随服务节点的增加而呈指数级增加，单一节点问题可能会被无限放大，日常运行过程中一定会伴随故障发生。所以构建分布式系统需要关注的另外一个重要架构特性是**稳定性**（stability）。有关减少系统故障以及快速从故障中恢复的工程实践，国内通常称为“稳定性建设”，而国外类似的工程实践更多称为“站点可靠性工程”（SRE, [Site reliability engineering](https://en.wikipedia.org/wiki/Site_reliability_engineering)）。稳定性（stability）、可靠性（reliability）、韧性（resilience）、可用性（availability）等架构特性，相似并且相关，虽然严格区分的话，含义并不相同，但是很多时候在探讨这些架构特性时往往涵盖的是类似的内容。本文的内容主要是总结稳定性、可靠性、韧性这些架构特性的内涵，以及如何建设分布式系统的这些特性。

<!--more-->

性能、伸缩性、稳定性、可靠性、韧性、可用性等特性都属于系统的**架构特性**（architecture characteristics）（或翻译为架构特征），这些特性实现的是系统的非功能性需求，是影响系统是否成功的至关重要的关注点[^1]。除了被统称为架构特性外，很多时候也被统称为系统的**质量属性**（[quality attributes](https://en.wikipedia.org/wiki/List_of_system_quality_attributes)）。

# 术语与概念

## 稳定性概念

先来看下术语“稳定性”（[stability](https://en.wikipedia.org/wiki/Stability)）的含义。国家标准 [GB/T 11457-2006](https://std.samr.gov.cn/gb/search/gbDetailed?id=71F772D7805FD3A7E05397BE0A0AB82A)《信息技术 软件工程术语》对术语“稳定性”的定义如下：

> 2.1559 稳定性 stability
> a) 在有干扰或破坏事件影响下仍能保持不变的能力。
> b) 在干扰或破坏性事件之后返回到原始状态的能力。

这个“稳定性”定义区分 a) 和 b) 两种类型的稳定性，关注点分别是“保持”和“恢复”。

除了软件系统涉及“稳定性”概念外，在其他领域的系统也会涉及“稳定性”概念。1995 年出版的经典著作《系统论：系统科学哲学》的第 15 章“系统稳定性原理”中，对“稳定性”概念有如下阐述[^2]：

> 系统稳定性原理指的是，在外界作用下开放系统具有一定的自我稳定能力，能够有一定范围内自我调节，从而保持和恢复原来的有序状态、保持和恢复原有的结构和功能。

可以看到，这个定义同时涉及“保持”和“恢复”两个动词，正好对应了国家标准 GB/T 11457 的“稳定性”定义中的 a) 和 b) 两种类型的稳定性。

这两种类型的稳定性，在生态系统领域下，对应[生态稳定性](https://zh.wikipedia.org/wiki/%E7%94%9F%E6%85%8B%E7%A9%A9%E5%AE%9A%E6%80%A7)的两种类型，并有专门的术语，分别是“[抵抗力稳定性](https://zh.wikipedia.org/wiki/%E6%8A%B5%E6%8A%97%E5%8A%9B%E7%A9%A9%E5%AE%9A%E6%80%A7)”（resistance stability）和“[恢复力稳定性](https://zh.wikipedia.org/wiki/%E6%81%A2%E5%BE%A9%E5%8A%9B%E7%A9%A9%E5%AE%9A%E6%80%A7)”（resilience stability）。英文术语“[resilience](https://en.wikipedia.org/wiki/Resilience_(engineering_and_construction))”被翻译为“恢复力”，但在软件工程领域，更加常见的是把“resilience”被翻译为“韧性”或“弹性”。

**简单概括来看，稳定性是承受干扰的能力，分为两种类型，抵抗力稳定性和和恢复力稳定性**。对软件系统来说，干扰指的是故障的组件、瞬时高负载、持续高负载等。

对于稳定性的概念，也可以从系统的业务数据指标角度来理解[^3]：

> 稳定性是指在一定工作条件下，业务服务成功率（反向对应失败率）、业务量的指标（如订单数、营收、PCU、QPS等）保持一致可预期的趋势，指标曲线呈现周期性或稳定在同一水平。与稳定性相对应的是异常波动性。系统稳定性代表软件系统能抵御各种异常因素造成系统核心指标抖动的能力。异常波动性度量一般是看实际曲线偏离预期稳定曲线的程度。

如图，曲线 A 的指标呈现周期性变化，表明系统能保持稳定；而曲线 B 的指标在时间 t2 至 t3 之间出现异常波动，表明系统无法保持稳定，是不稳定的。稳定性的反义词是波动性（volatility）。

<img width="600" alt="稳定性与异常波动" title="稳定性与异常波动" src="https://static.nullwy.me/stability-vs-volatility.png">

如果业务核心指标出现异常波动，说明很可能出现了故障。互联网应用的典型的故障定级标准的依据就是对业务核心指标的影响情况的判断。阿里的技术文档，给出了故障等级的参考定义[^4]，故障区分 4 个等级，P1 是最严重的故障，P4 是最轻微的故障。当业务量级为大体量时：对于核心功能，P1 级故障是成功率下跌 30% 及以上，P2 级故障是成功率下跌 20% ~ 30%，P3 级故障的定级标准是成功率下跌 20% 以下；对于非核心功能，P2 级故障是成功率下跌 30% 及以上，P3 级故障是成功率下跌 20%～30%，P4 级故障是成功率下跌 20% 以下。

## 稳定性与可靠性

由中国信息通信研究院牵头，并联合多个行业的多家单位（其中互联网公司包括阿里云、华为云、百度、蚂蚁、腾讯、字节跳动、京东、哈啰等），共同参与编制《分布式系统稳定性建设指南》[^5]，在 2022 年 6 月发布。该指南对系统稳定性有如下描述：

> **三、分布式系统稳定性建设目标**
> **(一) 稳定性建设目标**
> 稳定性建设目标：稳定性工作贯穿软件生命周期的全过程，从故障的视角来看稳定性建设的最终目标是“降发生”和“降影响”，稳定性建设目标可以通过评价指标实现量化。
> ...
> 降发生，即降低故障发生的概率。
> 降影响，即降低故障发生后的影响范围。

中国信通院的指南，是对“系统稳定性”的权威阐述。**对稳定性建设的阐述，是从故障的视角看的，稳定性建设的目标是故障的“降发生”和“降影响”**。这两个目标也与上文的稳定性定义的“保持”和“恢复”相对应。

<img width="500" alt="中国信通院的分布式系统稳定性建设目标" title="中国信通院的分布式系统稳定性建设目标" src="https://static.nullwy.me/stability-goals-caict.png">

故障的“降发生”和“降影响”，其实也是“可靠性工程”（[reliability engineering](https://en.wikipedia.org/wiki/Reliability_engineering)）的目标。对“可靠性工程”的理解，区分狭义可靠性工程或者广义可靠性工程。狭义可靠性工程的目标仅是提高系统无故障运行的能力，即提高可靠性。而广义可靠性工程的目标除了提高可靠性外，还包括提高从故障中恢复运行能力，即维修性（maintainability），同时还包括其他围绕故障展开的各种能力，如可用性（availability）、保障性（supportability）等。**通常提及“可靠性工程”时，大都是指广义可靠性工程，而提及“可靠性”时，更多是指狭义可靠性，即系统无故障运行的能力**。从故障中恢复运行的能力，对于硬件产品通常被称为“维修性”（maintainability），但在软件系统下通常称为“**韧性**”（resilience）。术语“maintainability”，在硬件上下文中通常被翻译为“维修性”，而在软件上下文中通常被翻译为“维护性”或“可维护性”，软件可维护性指的是软件可被修改的能力，修改可能包括修复缺陷、增加或完善功能等。关于“可靠性工程”更加全面的阐述，可以参阅笔者的文章《[可靠性工程概述](https://nullwy.me/2023/10/reliability-engineering/)》。

## 韧性概念

上文对韧性做了简单解释，现在让我们再展开来看看韧性的含义。根据维基百科的“[Resilience](https://en.wikipedia.org/wiki/Resilience_(engineering_and_construction))”词条的解释，在字典中，韧性（resilience）是“从困难或干扰中恢复的能力”（the ability to recover from difficulties or disturbance）。韧性一词的根源在拉丁语“resilio”中找到，意思是回到一个状态或反弹。但是很多时候术语“韧性”所代表的含义是对原始含义扩展后的含义。维基百科对术语“Resilience”的完整定义是：

> the ability to respond, absorb, and adapt to, as well as recover in a disruptive event
> 在破坏性事件中做出响应、承受和适应以及恢复的能力

“AWS Well-Architected Framework”文档，对“Resiliency”术语的定义是[^6]：

> The ability for a system to recover from a failure induced by load, attacks, and failures.

微软的 Azure 韧性白皮书，对“韧性”的解释[^7]：

> 韧性指系统从故障中恢复正常并继续工作的能力。它不仅指避免故障的能力，还包括在故障发生时避免停机或数据丢失的故障响应能力。韧性的目标是避免故障，并在无法避免故障时，使应用程序恢复至故障发生前完全正常的状态。
> Resiliency is the ability of a system to recover from failures and continue to function. It's not just about avoiding failures but responding to failures in a way that avoids downtime or data loss. The goal of resiliency is to avoid failures and if they still occur, return the application to a fully functioning state following an occurrence. 

Google 的《构建安全可靠的系统》一书对术语“韧性”（弹性）的解释是[^8]：

> “弹性”代表的是系统承受重大故障或中断的能力。具备弹性的系统可以自动从系统的部分故障（或者整个系统的故障）中恢复，并在问题解决后恢复正常运行。理想情况下，弹性系统中的服务在整个事件过程中保持运行状态，但可能处于降级模式。将弹性嵌入到系统中每一层的设计中，有助于保护系统免受意外故障和攻击的影响。

综合概括来看，**狭义韧性，指的是自动或快速从故障中恢复运行的能力；而广义韧性，除了从故障中恢复运行的能力外，还包括故障容忍能力**。故障容忍（[fault tolerance](https://en.wikipedia.org/wiki/Fault_tolerance)，简称“容错”），是使系统在其某些组件中出现一个或多个故障时能够继续提供服务的能力，从客户的角度来看，该服务仍能完全正常运行，或可能降级运行。AWS 对韧性定义是属于狭义韧性，而维基百科、微软和 Google 对韧性定义都属于广义韧性。另外，值得注意的是，广义韧性的故障“恢复”和“容忍”的这两种能力，其实也对应着上文在解释“稳定性”术语时提到的恢复力和抵抗力。**广义韧性的含义与稳定性的含义相似**。

综合上文对各个概念的解释，**可以将稳定性简单理解为，稳定性 = (狭义) 可靠性 + (狭义) 韧性**。稳定性、可靠性与韧性的区别，如下图所示：

<img width="400" alt="稳定性、可靠性与韧性的区别" title="稳定性、可靠性与韧性的区别" src="https://static.nullwy.me/stability-vs-reliability-vs-resilience.svg">

可靠性和韧性的侧重点不同。**可靠性工程的目标是尽可能减少系统中的故障，保证系统无故障运行。而韧性工程，接受故障总会发生的现实，关注的是如何降低故障带来的损失以及如何从故障中恢复**。分布式系统，100% 的可靠性是不存在的，必须拥抱故障，提升系统的韧性，可靠性和韧性必须同时关注。不过在将“可靠性”和“韧性”的含义扩展后，很多时候这两个术语背后的内涵是等价的。

## 可用性概念

可用性（[availability](https://en.wikipedia.org/wiki/Availability)）是衡量可靠性和韧性的综合性指标，可以表示为总可用时间除以总可用时间与总不可用时间之和，也可以通过平均无故障时间（MTTF，Mean Time To Failure）和平均故障恢复时间（MTTR，Mean Time To Repair）计算。可用性的计算公式如下：

$\text{availability} = \frac{\text{total uptime}}{\text{total uptime} + \text{total downtime}} = \frac {MTTF} {MTTF + MTTR}$

高可用性（[high availability](https://en.wikipedia.org/wiki/High_availability)）的系统，通常将可用性目标设定为 N 个 9，常见的可用性目标和不可用分钟数，如下表所示：

| 可用性 | 年不可用分钟数 | 月不可用分钟数 |
|---|---|---|
| 99.5% (2.5 个 9) | 2635 | 219 |
| 99.9% (3 个 9) | 526 | 43.83 |
| 99.95% (3.5 个 9) | 263 | 21.92 |
| 99.99% (4 个 9) | 52.60 | 4.38 |
| 99.995% (4.5 个 9) | 26.30 | 2.19 |

很多云服务平台在服务级别协议（SLA）中规定了服务可用性承诺，比如 AWS EC2 服务 [SLA 协议](https://aws.amazon.com/cn/compute/sla/)，区域级 SLA 承诺的每月可用性至少是 99.99%，实例级 SLA 承诺的每月可用性至少是 99.5%，若低于该承诺值，会作相应的赔偿，可用性越低，赔偿额度越高，若低于 95% 全额赔偿。一些互联网公司会在公司内部设定自己业务系统的 SLA 可用性目标，比如笔者所在的公司设定的年度 SLA 可用性目标是 99.99%，然后考核技术团队绩效的依据之一就是对这个可用性目标的达成情况。

想要提高系统的可用性，需要做的是**延长无故障时间（MTTF）和缩短故障恢复时间（MTTR）**。

## 站点可靠性工程

站点可靠性工程（SRE, [Site reliability engineering](https://en.wikipedia.org/wiki/Site_reliability_engineering)），起源于 Google，2003 年 Ben Treynor Sloss 在加入 Google 后组建了最早的 SRE 团队。在《SRE：Google运维解密》[^9]（2016 年出版，该书是关于 SRE 的第一本书）中 Ben Treynor Sloss 解释了 SRE 的内涵：

> SRE 究竟是如何在 Google 起源的呢？其实我的答案非常简单：SRE 就是让软件工程师来设计一个新型运维团队的结果。... 从本质上来说，SRE 就是在用软件工程的思想和方法论完成以前由运维团队手动完成的任务。这些 SRE 倾向于通过设计、构建自动化工具来取代人工操作。... 我们可以认为 DevOps 是 SRE 核心理念的普适版，可以用于更广范围内的组织结构、管理结构和人员安排。同时，SRE 是 DevOps 模型在 Google 的具体实践，带有一些特别的扩展。

对于 Google 来说，SRE 的内涵主要局限在 IT 运维（IT operations，也翻译为 IT 运营）实践上，SRE 是 DevOps 在 Google 的具体实践，即“class SRE implements interface DevOps”。不过，很多资料扩展了站点可靠性工程的内涵[^10][^11][^3][^12]，典型的代表是微软，微软对站点可靠性工程的定义是[^11]：

> Site Reliability Engineering is an engineering discipline devoted to helping an organization sustainably achieve the appropriate level of reliability in their systems, services, and products.
> 站点可靠性工程是一门工程学科，致力于帮助组织可持续地实现其系统、服务和产品的适当可靠性水平。

可以看到，这个定义没有与“运维”强绑定，实现可靠性水平相关的工程实践都属于站点可靠性工程。扩展内涵后的站点可靠性工程，除了用软件工程方式完成运维任务外，还涉及韧性架构设计，可靠性工作贯穿软件生命周期的全过程[^3][^12]。SRE 工程师的技能要求，如下图所示[^12]。

<img width="550" alt="SRE 工程师的技能要求" title="SRE 工程师的技能要求" src="https://static.nullwy.me/sre-skills.jpg">

**可以认为，Google 诠释的站点可靠性工程是狭义站点可靠性工程，而扩展后的站点可靠性工程是广义站点可靠性工程。简单来说，广义站点可靠性工程等价于中国信通院的指南的分布式系统稳定性建设，两者涵盖相同的内容。**

《SRE：Google运维解密》最早介绍了 SRE 的可靠性层级模型。前 Google SRE 工程师 Nat Welch 在《SRE生存指南》（Real World SRE, [2018](https://www.packtpub.com/product/real-world-sre/9781788628884)）一书中对可靠性层级模型做了更加系统全面的阐述。可靠性层级模型是前 Google SRE 工程师 [Mikey Dickerson](https://en.wikipedia.org/wiki/Mikey_Dickerson) 为了解释如何提高系统可靠性而提出来的。模型的各个层次从低到高分别是：韧性架构设计（design resilient architecture）、监控（monitoring）、事故响应（incident response）、事后回顾（postmortems）、测试与发布（test/release processes）、容量规划（capacity planning）、开发（development）、用户体验（UX）。类似于[马斯洛需求层次模型](https://zh.wikipedia.org/wiki/%E9%9C%80%E6%B1%82%E5%B1%82%E6%AC%A1%E7%90%86%E8%AE%BA)，可靠性层级模型的不同层次代表不同的优先级，低层是基本需求，高层是高级需求，在满足可靠性需求时，需要按部就班，在到达更高层次之前必须先满足每个低层次的需求。SRE 的可靠性层级模型，如下图所示。

<img width="450" alt="SRE 的可靠性层级模型" title="SRE 的可靠性层级模型" src="https://static.nullwy.me/sre-reliability-hierarchy.svg">

需要注意的是，Dickerson 原始的可靠性层级模型的最低层是监控，而不是韧性架构设计，之所以加上了韧性架构设计，参考的是微软的技术培训师 Unai Huete Beloki 写的关于 SRE 的书籍[^13]。

# 可靠性和韧性设计

基于 AWS 的经验，对于分布式系统的故障，亚马逊 CTO Werner Vogels 有如下总结[^14]：

> Failures are a given and everything will eventually fail over time: from routers to hard disks, from operating systems to memory units corrupting TCP packets, from transient errors to permanent failures. This is a given, whether you are using the highest-quality hardware or lowest cost components. ... We needed to build systems that embrace failure as a natural occurrence even if we did not know what the failure might be. Systems need to keep running even if the “house is on fire.” It is important to be able to manage pieces that are impacted without the need to take the overall system down.
> 故障是注定的；随着时间的流逝，一切终将归于失败：从路由器到硬盘，从操作系统到存储单元损坏的TCP数据包，从瞬时误差到永久失效，无论你用的是最高质量的硬件还是最低成本的组件，这都是理所当然的。... 因此，我们需要构建的是将故障视为自然发生的系统，即使我们并不知道故障是什么。这个系统应该要做到，即使在“后院已经着火”的情况下依然可以继续运行。重要的是在不需要引起整个系统宕机的情况下就能管理好受影响的局部组件。

Werner Vogels 的名言是，“Everything fails, all the time”。**分布式系统，100% 的可靠性是不存在的，必须拥抱故障，假设一切都会失败，面向故障设计，这样没什么会真正失败（design for failure and nothing will really fail）**。面向故障设计（design for failure），或翻译为“面向失败设计”、“防故障设计”等，接受故障总会发生的现实，以提升系统的韧性为目标，也叫做**韧性设计**（design for resilience）。面向故障设计，是亚马逊 AWS 的关于在云环境下构建应用的白皮书“Architecting for The Cloud: Best Practices”中总结的第一条最佳架构实践[^15][^16]。目前这个白皮书已经被最新的“AWS Well-Architected Framework”白皮书替代。

AWS 的 [Well-Architected 框架](https://aws.amazon.com/cn/architecture/well-architected/)，最早在 2015 年 10 月发布[^17]，描述了用于在云中设计和运行工作负载的关键概念、设计原则和架构最佳实践。受亚马逊 AWS 的影响和启发，其他云平台也相继发布类似的在云环境下的架构最佳实践的框架，[Google Cloud 架构框架](https://cloud.google.com/architecture/framework?hl=zh-cn)（2015.10）、[Microsoft Azure Well-Architected 框架](https://learn.microsoft.com/zh-cn/azure/well-architected/)（2020.08）、[阿里云卓越架构](https://help.aliyun.com/product/2362200.html)（2023.06）。这些框架都由五个或六个支柱组成，内容上大同小异。AWS 的 Well-Architected 框架基于六大支柱，其中两个支柱是**可靠性支柱（Reliability Pillar）**和**卓越运营支柱（Operational Excellence Pillar）**。可靠性支柱侧重于执行预期职能的工作负载，以及如何从故障快速恢复以满足需求。类似的，Google Cloud 架构框架由六大支柱组成，其中两个支柱是可靠性和卓越运营。Azure 架构良好的框架的由五大要素组成，其中两个要素是可靠性和卓越运营。阿里云卓越架构包含五个架构最佳实践支柱，其中两个支柱是**稳定性**和**卓越运营**。可靠性或稳定性支柱涵盖的内容相对偏向上文提到的**面向故障设计**或**韧性架构设计**，而卓越运营支柱涵盖的内容相对偏向上文提到的**狭义 SER** 或 **DevOps**，不过两个支柱涉及的内容有很重叠的部分。

**按故障的根因（root cause）分类**，主要有如下类型：硬件故障（hardware failure）、网络故障（network failure）、软件 bug（software bug）、配置错误（misconfiguration）、运维操作错误（operator error）、过载（overload）、依赖服务（dependency service）等。故障原因的分类，不同的组织各有不同，有些分类可能会把配置错误和运维操作错误一起归类为人为错误（human error）。另外，本质上来看，软件 bug 也是开发时的人为错误引入的，但一般都不把人为错误和软件故障区分为两种类型的故障。

下图所示的是造成谷歌某大型互联网服务可检测到的服务中断所有事件的一个粗略分类，以及故障原因的分布比例[^18]。容易发现，故障更多是由软件错误、错误的配置和人为错误造成的，而非机器或网络故障。由硬件故障导致的服务级别故障占比之所以很低，主要不是依靠这些系统硬件组件的可靠性，而是因为容错技术在防止组件故障影响上层系统行为方面是相当成功的。硬件设备故障以外的因素更容易导致服务级别中断，是因为构建能容忍已知硬件故障的服务相对容易，而处理一般的软件错误和运维人员误操作则比较难。

<img width="550" alt="谷歌某一主要服务最可能的故障原因分布" title="谷歌某一主要服务最可能的故障原因分布" src="https://static.nullwy.me/stability-google-service-failures-distribution.png">

提高系统可靠性的方法分为：
- **故障避免（fault avoidonce，简称“避错”）**：在系统的设计和实现过程中使用一些开发方法来减少故障发生，并在系统部署使用之前进行验证和确认来发现和去除程序中的故障。避错技术包括通过优秀的软件设计方法、编译器检查、技术评审、代码评审、测试等。
- **故障容忍（fault tolerance，简称“容错”）**：容错是使系统在其某些组件中出现一个或多个故障时能够继续提供服务的能力，尽管该服务可能处于降级级别。容错技术主要是采用**冗余**（[redundancy](https://en.wikipedia.org/wiki/Redundancy_(engineering))）方法来消除故障的影响，冗余的含义是指当系统无故障时取消冗余资源不会影响系统正常运行。

系统的资源包括硬件资源、软件资源、信息资源、时间资源，所以冗余区分 4 种方式：

- **硬件冗余**（hardware redundancy）：通过配置额外的硬件组件实现冗余。
- **软件冗余**（software redundancy）：通过配置额外的软件版本实现冗余，例如 N 版本编程（[NVP](https://en.wikipedia.org/wiki/N-version_programming)）。
- **信息冗余**（information redundancy）：通过对信息中外加一部分信息码或将信息存放在多个内存单元或将信息进行备份等实现冗余，例如循环冗余校验码、数据复制、数据库备份等。
- **时间冗余**（time redundancy）：多次执行相同的操作（重试）实现冗余，例如多次执行程序或传输数据的多个副本。

硬件冗余和软件冗余被合称为结构冗余（structural redundancy）。相对与时间冗余，硬件冗余、软件冗余、信息冗余被合称为空间冗余（space redundancy）。硬件冗余比较常见，而软件冗余相对少见。

**应对各种故障的具体典型的可靠性和韧性策略**[^19][^5][^7]：

- **硬件和网络故障**：
  - 避错：通过提高硬件的**质量**实现避错
  - 容错：通过**冗余**实现容错，具体的措施包括硬件冗余、数据复制（replication）、数据库备份（backup）、重试（retry）等
  - 快恢：自动主备、主从或多活**流量切换**
- **软件 bug**：
  - 避错：技术评审、代码评审、测试等实现避错
  - 快恢：通过**重启**处理导致崩溃（[crash](https://en.wikipedia.org/wiki/Crash_%28computing%29)）、夯死（[hang](https://en.wikipedia.org/wiki/Hang_%28computing%29)）的软件故障，通过**回滚**代码快速修复 bug
- **人为错误**：包括配置错误和运维操作错误
  - 避错：更好的人；消除人为因素，即自动化；预先检测等
  - 快恢：通过**回滚**配置或操作修复错误
- **服务过载**：典型的是在**大促**期间可能出现服务过载
  - 避错：通过提前的**容量规划**（capacity planning）、**压测**（stress testing）实现避错
  - 容错：通过**弹性扩容**（elastic scaling）实现负载均衡，通过**限流**（rate limiting）、**优雅降级**（graceful degradation）来降低负载
- **依赖服务**：主要策略是**故障隔离**，将故障的影响限制在较小的范围内，避免发生**连锁故障**（[cascading failure](https://en.wikipedia.org/wiki/Cascading_failure)）
  - 避错：通过**服务功能拆分**、**服务依赖资源隔离**、**服务强弱依赖治理**等实现故障隔离
  - 容错：通过**熔断**（circuit breaker）实现故障隔离，通过快速失败（fail fast）的方式，避免请求大量阻塞，从而保护调用方

云环境的硬件基础设施，比如 AWS、Azure、阿里云等，按物理隔离程度区分**可用区**（[Availability Zone](https://en.wikipedia.org/wiki/Availability_zone), AZ）和**地域**（Region，也叫区域）。**地域**指数据中心所在的地理区域，通常按照数据中心所在的城市划分。例如阿里云[^20]，华北 1（青岛）地域表示数据中心所在的城市是青岛。**可用区**是指在同一地域内独立的物理分区，每个可用区包含一个或多个数据中心，这些数据中心配置独立电源、冷却和网络。例如阿里云，华北 1（青岛）地域支持 2 个可用区，包括青岛可用区 B 和青岛可用区 C。在同一地域内，可用区与可用区之间内网互通。各可用区之间可以实现故障隔离，即如果一个可用区出现故障，则不会影响其他可用区的正常运行。

按故障的影响范围，可以区分组件级、可用区级和地域级共三个级别的故障，这三个级别故障的具体的容错措施是：
 - 组件级故障：实现组件冗余，避免单点故障
 - 可用区级故障：实现跨可用区冗余，复制组件和数据到其他可用区，经典案例是同城灾备、同城双活/多活
 - 地域级故障：实现跨地域冗余，复制组件和数据到其他区域，经典案例是异地灾备、异地双活/多活

另外，从故障的**直接原因**角度来看，故障主要由**变更**触发。根据 Google SRE 经验，由变更触发的生产事故占比大概 70%[^9]。为了提高系统稳定性，Google SRE 总结了**变更管理**（change management）的三点最佳实践：
  - 采用渐进式发布机制
  - 迅速而准确地检测到问题的发生
  - 当出现问题时，安全迅速地回退改动

阿里将这三点变更管理最佳实践总结概括为简单易记的“**变更三板斧**”，可灰度、可监控、可回滚[^21][^22]。另外一个提高系统稳定性的变更管理最佳实践是，在重保活动等重要事件的时候开启**封版**（[change freeze](https://en.wikipedia.org/wiki/Freeze_%28software_engineering%29)）策略，在封版期间除了特殊的紧急发布外禁止生产环境的全部变更。

在**故障响应**（incident response）方面，提高系统稳定性的最核心的目标就是**缩短故障恢复时间（MTTR）**。阿里的稳定性实践是把这个目标量化，提出“1-5-10 故障快恢”目标，1 分钟发现及启动响应，5 分钟定位，10 分钟恢复[^21][^22]。阿里的 1-5-10 能力图谱，如下图所示[^23]：

<img width="750" alt="阿里“1-5-10 故障快恢”能力图谱" title="阿里“1-5-10 故障快恢”能力图谱" src="https://static.nullwy.me/stability-response-alibaba-1-5-10.png">

类似的，哈啰的故障响应目标是 5-5-10：5 分钟响应、5 分钟定位、10 分钟恢复[^24]。

# 参考资料

[^1]: 软件架构：架构模式、特征及实践指南，Mark Richards & Neal Ford，2020，[豆瓣](https://book.douban.com/subject/35487561/)：第4章 现有的架构特征
[^2]: 系统论：系统科学哲学，曾国屏、魏宏森，1995，[豆瓣](https://book.douban.com/subject/1008370/)：第三篇 系统论的基本原理，15 系统稳定性原理
[^3]: SRE原理与实践：构建高可靠性互联网应用，张观石，2022，[豆瓣](https://book.douban.com/subject/36202918/)：第1章 互联网软件可靠性概论
[^4]: 阿里云卓越架构：卓越运营支柱：故障管理：故障等级定义的制定和录入 <https://help.aliyun.com/document_detail/2536143.html>
[^5]: 2022-06 中国信通院：分布式系统稳定性建设指南（2022年） <http://www.caict.ac.cn/kxyj/qwfb/ztbg/202206/t20220620_404604.htm> <https://mp.weixin.qq.com/s/OkG3_pjtaQcB-cOupCNe-w>
[^6]: AWS Well-Architected Framework: Concepts: Resiliency <https://wa.aws.amazon.com/wellarchitected/2020-07-02T19-33-23/wat.concept.resiliency.en.html>
[^7]: 2022-01 Microsoft Azure 韧性白皮书（Resilience in Azure whitepaper） <https://www.modb.pro/doc/109965> <https://web.archive.org/web/0/https://azure.microsoft.com/en-us/resources/resilience-in-azure-whitepaper/>
[^8]: Google 构建安全可靠的系统，2021，[豆瓣](https://book.douban.com/subject/35585206/)：第8章 弹性设计
[^9]: SRE：Google运维解密，Beyer, etc. 2016，[豆瓣](https://book.douban.com/subject/26875239/)、[英文版](https://sre.google/sre-book/table-of-contents/)
[^10]: The Art of Site Reliability Engineering (SRE) with Azure, Unai Huete Beloki, 2022, [springer](https://link.springer.com/book/10.1007/978-1-4842-8704-0): Chapter 1: The Foundation of Site Reliability Engineering
[^11]: Microsoft Azure: Site reliability engineering documentation <https://learn.microsoft.com/en-us/azure/site-reliability-engineering/>
[^12]: Becoming a Rockstar SRE, Proffitt & Anami, 2023, [packtpub](https://www.packtpub.com/product/becoming-a-rockstar-sre/9781803239224): Chapter 1: SRE Job Role – Activities and Responsibilities
[^13]: The Art of Site Reliability Engineering (SRE) with Azure, Unai Huete Beloki, 2022, [springer](https://link.springer.com/book/10.1007/978-1-4842-8704-0): Chapter 4: Architecting Resilient Solutions in Azure, 4.1 What Is Resiliency?: Figure 4-1. Customized hierarchy of reliability
[^14]: 2016-03 Amazon CTO Werner Vogels: 10 Lessons from 10 Years of Amazon Web Services <https://www.allthingsdistributed.com/2016/03/10-lessons-from-10-years-of-aws.html> <https://aws.amazon.com/cn/blogs/china/10-lessons-from-10-years-of-aws/>
[^15]: 2010-01 Jinesh Varia: Architecting for the Cloud: Best Practices (AWS whitepaper) <https://web.archive.org/web/0/https://aws.amazon.com/blogs/aws/new-whitepaper-architecting-for-the-cloud-best-practices/>
[^16]: 2010-04 Jinesh Varia: Architecting for the Cloud: Best Practices <https://www.slideshare.net/AmazonWebServices/aws-architectingdesantislondon>
[^17]: 2015-10 The AWS Well-Architected Framework <https://www.infoq.com/news/2015/10/aws-well-architected-framework/>
[^18]: 数据中心一体化最佳实践，Barroso, Hölzle, Ranganathan，第3版2018，[豆瓣](https://book.douban.com/subject/34950732/)：第7章 故障处理与维修
[^19]: 云系统管理：大规模分布式系统设计与运营，[Tom Limoncelli](https://en.wikipedia.org/wiki/Tom_Limoncelli)，2014，[豆瓣](https://book.douban.com/subject/26865122/)：第6章 弹性设计模式
[^20]: 阿里云：地域和可用区 <https://help.aliyun.com/document_detail/40654.html>
[^21]: 2020-03 阿里陈鑫：阿里巴巴DevOps文化浅谈 <https://mp.weixin.qq.com/s/h-F8dopr23pgvSoXjWfE8A>
[^22]: 阿里云卓越架构：稳定性支柱：稳定性设计方案 <https://help.aliyun.com/document_detail/2573820.html>
[^23]: 2021-05 阿里暴晓亚若厉：阿里巴巴GOC稳定性保障介绍（slides, 26p） <https://www.modb.pro/doc/31443>
[^24]: 2022-04 哈啰技术：稳定性建设系列文章1_大纲&方法论 <https://segmentfault.com/a/1190000041671012>