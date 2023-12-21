---
title: 亚马逊网站架构演进
date: 2023-11-04 12:45:00
categories: 架构
tags: [架构, 可伸缩性, 可扩展性, 技术栈, 微服务, SOA]
---

# SOA 与微服务

Amazon，1994 年创立，早期网站是单服务、单数据库的单体架构的系统[^1][^2][^3]，全部代码由 C++ 编写，编译成单个二进制文件，整个代码仓库被命名为 [Obidos](https://en.wikipedia.org/wiki/Obidos_(software))。Obidos 是底层是一个 Web 页面渲染引擎，是一个框架，业务逻辑基于这个框架开发，Obidos 渲染引擎和业务逻辑共同组成整个代码仓库。随着时间的推移，Obidos 变得越来越复杂，编译 Obidos 整个代码库耗时 12 小时，开发调试效率低下[^2]。另外，全部业务逻辑在单个二进制文件中，导致紧耦合，新功能特性无法快速发布上线。1995 年，Amazon 网站的技术架构，如下图所示[^3]：

<!--more-->

<img width="600" alt="Amazon 网站技术架构（1995）" title="Amazon 网站技术架构（1995）" src="https://static.nullwy.me/amazon-architecture-1995.png">

到 2000 年，为了应对流量增长，解决网站的可扩展性问题，开始拆分 Obidos，向可扩展的 SOA 架构演进。拆分出的服务有，用户服务（customer service）、订单服务（order service）、商品服务（item service）等，并且每个服务都拥有各自独立的数据库。2005 年 1 月，Amazon 网站开始从 Obidos 引擎迁移到 Gurupa 引擎。Gurupa 是一个 Web 页面渲染引擎，同时也是一个 SOA 框架，Gurupa 被用于集成背后的数百个服务，使用 Perl 的 Mason 模板将各个服务响应的数据渲染成 Web 页面。到 2006 年完成了到 Gurupa 的全部迁移[^4]。

Amazon 网站技术架构演进过程，如下图所示[^2]：

<img width="600" alt="Amazon 网站技术架构演进" title="Amazon 网站技术架构演进" src="https://static.nullwy.me/amazon-architecture-evolution.png">

需要注意的是，图中认为从 2006 年开始 Amazon 网站架构从 SOA 演进为微服务，但是事实上，“微服务”这个术语诞生的时间是 2012 年。2006 年的架构实现与之前的 SOA 架构实现的不同点是，开始更细粒度的服务拆分。

本质上来看，微服务架构是一种特殊的 SOA 架构，在“微服务”术语诞生之前，亚马逊的 SOA 架构实现被认为是“SOA done right”（正确实现的 SOA）[^5]。Netflix 的 SOA 架构实现也在“微服务”术语诞生之前，Netflix 认为自己实现架构的是“fine-grained SOA”（细粒度的 SOA）[^6]。James Lewis 和 Martin Fowler 等人是“微服务”概念的早期提倡者，他们将亚马逊和 Netflix 的 SOA 架构实现归类为微服务的经典案例[^7]。所以，SOA 和微服务的关系可以简单理解为，微服务是“fine-grained SOA”或“SOA done right”。

根据 Martin Fowler 的解释[^7][^8]，SOA 与微服务的关系，如下图所示：

<img width="400" alt="SOA 与微服务的关系" title="SOA 与微服务的关系" src="https://static.nullwy.me/soa-vs-microservices.svg">

在 SOA 架构的具体实践上，根据前亚马逊工程师 Steve Yegge 的 2011 年的文章[^9][^10]，在 2002 年左右，亚马逊创始人兼 CEO Jeff Bezos 向全公司发布了一道指令（这个指令被外界称为“API Mandate”），具体内容如下：

> 1. 从今天起，所有的团队都要以服务接口的方式提供数据和各种功能。
> 2. 团队之间必须通过接口来通信。
> 3. 不允许任何其他形式的互操作：不允许直接链接，不允许直接读其他团队的数据，不允许共享内存，不允许任何形式的后门。唯一许可的通信方式就是通过网络调用服务。
> 4. 至于具体的技术则不做规定。HTTP、Corba、Pubsub、自定义协议都可以。贝索斯不关心这个。
> 5. 所有的服务接口，必须从一开始就要以可以公开为设计导向，没有例外。这就是说，团队必须在设计的时候就计划好，接口要可以对外面的开发人员开放。没有讨价还价的余地。
> 6. 不听话的人会被炒鱿鱼。

# 技术栈演进 

Amazon 网站的技术栈演进过程[^2][^4][^11]：

+ 1995 ~ 2000：单体架构，Unix（Sun）、Obidos、Oracle、C++
  - Obidos 是亚马逊内部 Web 动态页面渲染引擎，编程语言是 C++。
+ 2000 ~ 2006：SOA 架构，Linux、Obidos、Oracle、C++
  - 2000 年，将 Sun/Unix 服务器替换为 HP/Linux 服务器[^12]。
  - 2000 年，拆分应用服务，向 SOA 架构演进。
  - 2005 年初，开始将 Obidos 框架替换为 Gurupa 框架，Gurupa 是 Web 页面渲染引擎，同时也是 SOA 框架。Web 动态页面的编程语言从 C++ 替换为 Perl/Mason，同时应用服务开始被更细粒度的拆分。
+ 2006 年开始：微服务架构，Linux、Gurupa、Oracle、Perl & Java & C++
  - 2019.10，彻底去掉 Oracle 数据库，迁移到 Amazon RDS 和 NoSQL[^13]。

当前 Amazon 网站的主要技术栈：

- **应用服务**[^4][^11]：
   - **展示层**：Perl/Mason[^14]
   - **业务逻辑层**：Java（主要）、C++ 等
   - **RPC框架**：Gurupa 框架（自研闭源）
      - 2006 年更早之前使用 [Obidos](https://en.wikipedia.org/wiki/Obidos_%28software%29) 框架
   - **消息队列MQ**：[Amazon SQS](https://en.wikipedia.org/wiki/Amazon_Simple_Queue_Service)
- **数据存储**[^13]：
   - **关系数据库**：Amazon RDS for PostgreSQL[^15]、Amazon Aurora (PostgreSQL)
      - 2019 年 10 月彻底去掉 Oracle 数据库，迁移到 Amazon RDS 和 NoSQL。
   - **键值存储**：[Amazon DynamoDB](https://en.wikipedia.org/wiki/Amazon_DynamoDB)
   - **缓存**：Amazon ElastiCache for Redis[^16]
   - **Blob文件存储**：[Amazon S3](https://en.wikipedia.org/wiki/Amazon_S3)

# AWS 的诞生

Amazon 网站架构演进的过程，伴随着 AWS 云服务的诞生，促使 AWS 云服务诞生的因素主要有[^17][^18]：

- **技术价值**：Amazon 网站在向可扩展架构演进的过程中，其基础设施团队开始善于运营像计算、存储和数据库这样的基础设施，团队已经变得能非常熟练地运营可靠的、可扩展的、具有成本效益的数据中心，这些专业能力推动了亚马逊电商平台的发展。将维护可靠、可扩展的基础设施专业能力通过服务的方式对外交付，能有效节省第三方企业或初创公司的成本，当时亚马逊预计基础设施的成本可能会从 70% 降低至 30% 或更低[^19]。
- **原始动机**：Amazon 网站的流量有很强的季节性，每年 11 月和 12 月（尤其是在[感恩节](https://zh.wikipedia.org/wiki/%E9%BB%91%E8%89%B2%E6%98%9F%E6%9C%9F%E4%BA%94_(%E8%B4%AD%E7%89%A9))和圣诞节前后）流量都会飙升。为了应对购物季，不得不扩容，增加服务器资源。在购物季结束后，服务器资源被大量闲置。零售电商的利润很薄，却要面对数据中心基础设施不断扩展所带来的成本压力。所以，很多人（包括部分亚马逊员工）[^20][^21]认为推出 AWS 产品的一个重要原因是为了出租 SOA 网站在销售淡季时的过剩服务器容量。不过，出租过剩容量的故事是一个神话，因为不可能在每年购物季时把开发商从服务器里踢出去，而且实际上在推出 EC2 云服务后的 2 个月内，AWS 就已经烧掉了过剩的 Amazon 网站容量[^19]。
- **商业理念**：[Jeff Bezos](https://en.wikipedia.org/wiki/Jeff_Bezos)，将亚马逊定位为一家技术公司，而不仅仅是一家在线零售商，所以在亚马逊的核心业务之外进行很多投资尝试，AWS 产品就是其中之一[^17]。

于是，在 2003 年亚马逊团队内部逐渐开始形成销售基础设施服务的设想。2003 年 9 月[^22]，[Andy Jassy](https://en.wikipedia.org/wiki/Andy_Jassy) 写了 6 页纸的关于 AWS 的愿景文档（vision document）并提交给管理团队，愿景文档中提议了设想的 AWS 业务，并概述了 AWS 提供的初始的基础设施服务集，首批提供的服务包括存储、计算和数据库等。同年，Jassy 组建了由 57 人组成的 AWS 团队，AWS 团队的 CEO 由 Jassy 担任（Jassy 担任 AWS CEO 一直到 2021 年，2021 年 7 月开始任职亚马逊 CEO）。**三年之后，2006 年 3 月 AWS 对外发布 S3 云存储，8 月对外发布 EC2 弹性云计算服务器，S3 和 EC2 是行业内最早的云服务产品**。S3 发布的刚开始几个月，并没有引起太大的关注。EC2 发布后，大量开发商开始飞速涌入。在没有其他类似云服务产品最初几年，几乎每一家创业公司都在亚马逊的服务器上构建自己的系统。能吸引大量开发商的原因主要是，按需使用和收费的商业模式，以及亚马逊故意压力利润的价格策略[^17]。

# 迁移到 AWS

2011 年 11 月，Amazon 网站全部都迁移到了 AWS 云服务器上[^23]。迁移到 AWS 上最大的动机是能利用  AWS 云服务器的弹性伸缩能力，从而节省成本。如果没有弹性伸缩能力，在淡季时，总体上服务器资源容量的利用率是 61%，无法有效利用的容量是 39%，到购物季的 11 月，无法有效利用的容量高到 76%。引入弹性伸缩技术后，可以按网站的实际流量负载情况，供应恰当的容量，避免资源浪费。Amazon 网站在淡季和购物季时的静态伸缩与弹性伸缩，如下图所示[^23][^24]。

<img width="600" alt="Amazon 网站的典型的周流量分布" title="Amazon 网站的典型的周流量分布" src="https://static.nullwy.me/amazon-typical-weekly-traffic.png">

<img width="600" alt="Amazon 网站的静态伸缩" title="Amazon 网站的静态伸缩" src="https://static.nullwy.me/amazon-november-traffic-static-scaling.png">

<img width="600" alt="Amazon 网站的弹性伸缩" title="Amazon 网站的弹性伸缩" src="https://static.nullwy.me/amazon-november-traffic-elastic-scaling.png">

# 参考资料

[^1]: 2006-05 ACM Queue Interview: A Conversation with Amazon CTO Werner Vogels <https://queue.acm.org/detail.cfm?id=1142065>
[^2]: 2021-02 Amazon’s architecture evolution and AWS strategy (AWS re:Invent 2020) <https://www.youtube.com/watch?v=HtWKZSLLYTE>
[^3]: 2022-11 Reliable scalability: How Amazon scales in the cloud (AWS re:Invent 2022) <https://www.youtube.com/watch?v=_AhfV5LZmvo>
[^4]: 2011-04 Charlie Cheever: How did Google, Amazon, and the like initially develop and code their websites, databases, etc.? <https://qr.ae/pKKyB0>
[^5]: 2007-06 SOA done right: the Amazon strategy <https://www.zdnet.com/article/soa-done-right-the-amazon-strategy/>
[^6]: 2011-05 How the cloud helps Netflix (interview Adrian Cockcroft) <http://radar.oreilly.com/2011/05/netflix-cloud.html>
[^7]: 2014-03 James Lewis & Martin Fowler: Microservices <https://martinfowler.com/articles/microservices.html>
[^8]: 2014-11 Microservices • Martin Fowler • GOTO 2014 <https://youtu.be/wgdBVIX9ifA?t=880>
[^9]: 2011-10 Steve's Google Platform rant <https://gist.github.com/chitchcock/1281611>
[^10]: 2016-09 亚马逊如何变成 SOA（面向服务的架构）？（摘录自Steve Yegg的《程序员的呐喊》） <https://www.ruanyifeng.com/blog/2016/09/how_amazon_take_soa.html>

[^11]: What programming languages are used at Amazon? <https://qr.ae/pKFwnw>
[^12]: 2001-10 How Linux saved Amazon millions <https://web.archive.org/web/0/http://news.com.com/2100-1001-275155.html>
[^13]: 2019-10 Migration Complete – Amazon’s Consumer Business Just Turned off its Final Oracle Database <https://aws.Amazon/blogs/aws/migration-complete-amazons-consumer-business-just-turned-off-its-final-oracle-database/?nc1=h_ls>
[^14]: 2016-04 Is Amazon still using Perl Mason to render its content? <https://qr.ae/pKFwFm>
[^15]: Amazon RDS for PostgreSQL customers <https://aws.Amazon/rds/postgresql/customers/?nc1=h_ls>
[^16]: Amazon ElastiCache for Redis customers <https://aws.Amazon/elasticache/redis/customers/?nc1=h_ls>

[^17]: 一网打尽：贝佐斯与亚马逊时代，Brad Stone，2013，[豆瓣](https://book.douban.com/subject/25766700/)：第7章 一家技术公司，而非零售商
[^18]: 2016-07 AWS CEO Andy Jassy: How AWS came to be <https://techcrunch.com/2016/07/02/andy-jassys-brief-history-of-the-genesis-of-aws/>
[^19]: 2011-01 Amazon CTO Werner Vogels: How and why did Amazon get into the cloud computing business? <https://qr.ae/pKscWd>
[^20]: 2016-07 前亚马逊员工在 Reddit 上对文章“How AWS came to be”的评论 <https://www.reddit.com/r/programming/comments/4qxthq/comment/d4wrnk7/>
[^21]: 2021-01 前亚马逊员工 Dan Rose：全球最大云厂商AWS是如何诞生的？ <https://mp.weixin.qq.com/s/C7Mqeh1hyT6k5BQOzpWL9w>
[^22]: 2013-11 Andy Jassy's Book Review of "The Everything Store" <https://www.Amazon/review/R1Q4CQQV1ALSN0/>

[^23]: 2011-07 2011 AWS Tour Australia, Closing Keynote: How Amazon migrated to AWS, by Jon Jenkins <https://www.slideshare.net/AmazonWebServices/2011-aws-tour-australia-closing-keynote-how-amazoncom-migrated-to-aws-by-jon-jenkins>
[^24]: 2017-03 AWS: Elasticity and Management <https://www.slideshare.net/AmazonWebServices/elasticity-and-management>


