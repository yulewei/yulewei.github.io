---
title: RocketMQ 和 Kafka 的数据分片和复制策略
date: 2024-01-18 20:36:00
categories: 架构
tags: [架构, 中间件, 分布式, 可伸缩性, 可扩展性, 可靠性, 可用性, RocketMQ, Kafka]
---

为了提升系统的**可扩展性**（scalability），分布式数据库或分布式存储系统通常支持数据**分区**（partitioning）或**分片**（sharding），即将完整的数据拆分存放在多个服务器节点上，拆分后的部分数据称为“partition”或“[shard](https://en.wikipedia.org/wiki/Shard_%28database_architecture%29)”。数据被拆分后多个服务器节点能分摊负载压力，从而提升系统性能。“分区”和分片”，这两个术语，在很多情况下不区分，可以混用。如果严格区分的话，**分片**拆分的数据分布在多个服务器节点上，而**分区**拆分的数据在单个服务器节点。另外，**复制**（[replication](https://en.wikipedia.org/wiki/Replication_%28computing%29)）也典型的分布式技术，多个数据副本能实现读请求的负载均衡，提升系统性能。同时复制也提供了冗余容错的能力，提升系统的**可用性**（availability）。本文关注消息中间件的消息存储系统，解析并对比 RocketMQ 和 Kafka 的消息数据的分片和复制的具体实现策略。

<!--more-->

MySQL 等传统关系数据库支持**表分区**（[partition](https://dev.mysql.com/doc/refman/8.0/en/partitioning.html)），但原生不支持**分片**（sharding），拆分后的表分区都分布在同一个服务器节点上。为了解决数据库的水平扩展问题，出现很多数据库分片方案。其中一类是基于传统关系数据库的“分库分表”中间件，如 [Vitess](https://github.com/vitessio/vitess)、[ShardingSphere](https://github.com/apache/shardingsphere)、阿里 TDDL 和 DRDS 等。另外一类是非关系型的 [NoSQL](https://en.wikipedia.org/wiki/NoSQL) 数据库，如 BigTable、Dynamo、HBase、Cassandra 等。以及采用全新架构的 [NewSQL](https://en.wikipedia.org/wiki/NewSQL) 数据库，如 Google Spanner、CockroachDB、TiDB 等；或基于云服务的 NewSQL 数据库，如 Amazon Aurora、阿里 PolarDB 等。

术语分片（shard）或分区（partition），在具体的不同系统下有着不同的称呼，例如它对应于 MongoDB、Elasticsearch 和 SolrCloud 中的 `shard`，HBase 中的 `region`，Bigtable 中的 `tablet`，Cassandra 和 Riak 中的 `vnode`，以及 Couchbase 中 的 `vBucket`。总体而言，分片和分区使用最普遍。

分布式数据库不是本文关注的主题，不再展开。消息中间件的消息存储系统与分布式数据库系统类似，为了系统可扩展性和可用性，也需要支持数据分片和复制特性。

# 历史演进时间线

RocketMQ 和 Kafka 的历史演进时间线：
- 2007，淘宝自研 Notify，最早底层的消息存储采用本地文件存储，参考 ActiveMQ 实现了单机 kv 存储引擎，2008 年底层的消息存储改用 Oracle，2010 年从 Oracle 迁移到高可用 MySQL 存储集群[^1]。
- [2011.01](https://www.linkedin.com/blog/member/archive/open-source-linkedin-kafka)，LinkedIn 公司在 Github 上开源 Kafka 项目，项目地址 kafka-dev/[kafka](https://github.com/kafka-dev/kafka)。
  - 同年，淘宝基于 Kafka 的设计用 Java 完全重写并内部发布 MetaQ 1.0。
- 2011.07，Kafka 成为 Apache 孵化器项目。
- [2012.03](https://www.infoq.cn/article/2012/03/metamorphosis)，淘宝对外开源 MetaQ 1.x，项目名为 Metamorphosis（[淘蝌蚪](https://web.archive.org/web/20120312015328/http://code.taobao.org/p/metamorphosis/wiki/intro/)、[GitHub](https://github.com/killme2008/Metamorphosis)），版本号为 1.4.0。
  - Metamorphosis [1.0.1](https://web.archive.org/web/20120312015318/http://code.taobao.org/p/metamorphosis/wiki/changelist/) 开始实现高可用的 HA 方案，支持同步和异步复制，复制特性类似于 MySQL 的主从复制。
  - Kafka 的复制特性，直到 2013.12 发布的 0.8.0 版本才开始支持。Kafka 实现的复制是集群间的分区复制（Intra-cluster Replication），复制的副本粒度是分区（partition），参见 [KAFKA-50](https://issues.apache.org/jira/browse/KAFKA-50)。
- 2012.09，淘宝内部发布 MetaQ 2.0 版本，MetaQ 2.0 对架构进行了重新设计，为了解决分区文件数增加后的性能下降问题，对消息日志文件存储目录结构做了改造[^2]。改造后的 MetaQ 架构与 Kafka 存在很大差异，这个版本的 MetaQ 可以认为是第一代的 RocketMQ。
- 2012.10，Kafka 从孵化器项目毕业，成为 Apache 顶级项目。
- 2013.07，淘宝内部发布 MetaQ 3.0 版本。
- 2013.09，淘宝对外开源发布 RocketMQ 3.0，项目地址 alibaba/[RocketMQ](https://web.archive.org/web/20170506102903/https://github.com/alibaba/RocketMQ)。RocketMQ 3.0 和 MetaQ 3.0 等价，阿里内部使用的称为 MetaQ 3.0，外部开源称之为 RocketMQ 3.0[^3]。
- 2013.12，Kafka 发布版本 0.8.0，开始支持集群间的分区复制。
- [2016.11](https://www.oschina.net/news/89061)，RocketMQ 成为 Apache 孵化器项目。
- [2017.09](https://www.oschina.net/news/89061)，RocketMQ 从孵化器毕业，正式成为 Apache 顶级项目。
- [2019.04](https://www.oschina.net/news/105805)，RocketMQ 4.5 发布，开始支持 Borker 节点的自动选主，实现自动故障转移，自动选主模块被命名为 DLedger，DLedger 是基于 Raft 协议实现的轻量级 [Java Library](https://github.com/openmessaging/dledger/wiki)，被集成到各个 Borker 节点的进程中。
- 2019.10，Kafka 社区开始尝试用基于 Raft 的控制器替换基于 ZooKeeper 的控制器，新控制器叫作 KRaft，KRaft 模块被集成到 Borker 节点的进程中，去掉了对 ZooKeeper 的依赖，简化了整体架构，具体参见 [KIP-500](https://issues.apache.org/jira/browse/KAFKA-9119)。
  - 2021.04，Kafka 2.8 发布，KRaft 模式的早期访问版可用。
  - 2022.10，Kafka 3.3 发布，KRaft 模式被标记为生产环境可用。
  - 2023.06，Kafka 3.5 发布，ZooKeeper 模式被标记为废弃，计划在 Kafka 4.0 删除。
- 2022.09，RocketMQ 5.0 发布，自动选主开始支持 DLedger Controller 模式，Controller 可以独立部署，也可以嵌入在 Nameserver 中，具体参见 [RIP-44](https://github.com/apache/rocketmq/wiki/RIP-44-Support-DLedger-Controller)。

# RocketMQ

**RocketMQ 的数据分片和复制策略**[^4]：
- **分片策略**：
  - **分片术语命名**：[消息队列](https://rocketmq.apache.org/zh/docs/domainModel/03messagequeue/)（message queue）
    - 将单个 Topic 的消息日志拆分到多个消息队列中。
  - **键-分片的分配关系**：默认轮询（round-robin）分配
    - 默认按 Topic 消息的写入次序轮询分配给各个消息队列，也可以自定义消息队列选择器（MessageQueueSelector）。
    - 相关源码：DefaultMQProducerImpl#[sendDefaultImpl](https://github.com/apache/rocketmq/blob/rocketmq-all-4.9.0/client/src/main/java/org/apache/rocketmq/client/impl/producer/DefaultMQProducerImpl.java#L554)、DefaultMQProducerImpl#[sendSelectImpl](https://github.com/apache/rocketmq/blob/rocketmq-all-4.9.0/client/src/main/java/org/apache/rocketmq/client/impl/producer/DefaultMQProducerImpl.java#L1134)
  - **分片-机器的分配关系**[^5]：可配置某 Topic 在某 Borker 服务器节点的消息队列数。
    - 若发送消息时自动创建 Topic，配置项 `autoCreateTopicEnable` 开启，会在发送消息时轮询选择其中一台 Master Borker，在该 Borker 上分配消息队列。消息队列数由全局配置项 `defaultTopicQueueNums` 控制，默认值 `4`。
      - 相关源码：MQClientInstance#[updateTopicRouteInfoFromNameServer](https://github.com/apache/rocketmq/blob/rocketmq-all-4.9.0/client/src/main/java/org/apache/rocketmq/client/impl/factory/MQClientInstance.java#L605)、AbstractSendMessageProcessor#[msgCheck](https://github.com/apache/rocketmq/blob/rocketmq-all-4.9.0/broker/src/main/java/org/apache/rocketmq/broker/processor/AbstractSendMessageProcessor.java#L166)、TopicConfigManager#[createTopicInSendMessageMethod](https://github.com/apache/rocketmq/blob/rocketmq-all-4.9.0/broker/src/main/java/org/apache/rocketmq/broker/topic/TopicConfigManager.java#L156)
    - 若预先手动创建 Topic，执行 `mqadmin updateTopic` 命令，可以通过命令行参数指定在某个 Master Borker 上分配消息队列。也可以通过命令行参数指定 cluster，在 cluster 下的全部的 Master Borker 上分配消息队列，每个 Borker 的消息队列的数量相同。默认队列数 `8`。
      - 相关源码：[UpdateTopicSubCommand](https://github.com/apache/rocketmq/blob/rocketmq-all-4.9.0/tools/src/main/java/org/apache/rocketmq/tools/command/topic/UpdateTopicSubCommand.java#L90)
    - Topic 的消息队列的主副本分布在各个 Master Borker，某 Topic 的分区总数量是该 Topic 分布在各个 Master Borker 上的消息队列的数量的总和。
  - **分片再均衡策略**：手动再均衡
    - 在扩容添加新 Broker 节点后，在创建新 Topic 时，可以自动或指定在新 Broker 节点上分配消息队列，而旧的 Topic 也可以通过执行 `mqadmin updateTopic` 命令，在新的 Broker 节点上分配消息队列。
- **复制策略**：类似于 MySQL 的主从复制
  - **复制单位**：以机器为单位，副本 Borker 节点之间的数据完全相同
  - **复制系数**：由消息队列所属的 Broker Group 的下 Borker 总数量决定（每个 Broker Group 下有一个 Master Borker 和零到若干个 Slave Borker）
  - **副本更新传播策略**：
    - Borker 节点分为主从（master-slave）两种角色，支持异步复制（默认）和同步复制两种复制模式。配置项 `brokerRole` 用于配置节点的主从角色和复制模式，默认值为 `ASYNC_MASTER`，可配置为 `SYNC_MASTER`/`ASYNC_MASTER`/`SLAVE`。
  - **主从读写分离**[^6]：Master Borker 可写可读，Slave Borker 只允许读。配置项 `slaveReadEnable` 用于配置是否允许消息从从节点读取，默认 `false`。如果 `slaveReadEnable=true`，并且当前消息堆积量超过物理内存 40%（由配置项 `accessMessageInMemoryMaxRatio` 控制），则建议从 Slave Borker 拉取消息，否则还是从 Master Borker 拉取消息。
    - 相关源码：PullMessageProcessor#[processRequest](https://github.com/apache/rocketmq/blob/rocketmq-all-4.9.0/broker/src/main/java/org/apache/rocketmq/broker/processor/PullMessageProcessor.java#L266)
  - **消息可靠性**[^7][^8]：主要影响的配置项是主从节点的副本复制方式和磁盘刷盘方式。
    - 对于 `Borker` 单点故障情况，若采用主从异步复制，可保证 99% 的消息不丢，但是仍然会有极少量的消息可能丢失。若采用主从同步复制可以完全避免单点，但相对损失影响性能，适合对消息可靠性要求极高的场合。
    - 配置项 `FlushDiskType` 用于控制磁盘刷盘方式，可配置为异步刷盘 `ASYNC_FLUSH`（默认）和同步刷盘 `SYNC_FLUSH`。同步刷盘会损失很多性能，但是也更可靠。
    - 生产环境下的**推荐配置**是[^9]，把主从节点的磁盘刷盘方式都配置为**异步刷盘**，主从节点之间复制方式配置为**同步复制**，这种配置方式是相对兼顾了性能和可靠性。如果对消息丢失零容忍，则建议配置为同步复制、同步刷盘方式。
    - 对于副本系统来说，在系统设计或配置时，必须要在副本一致性和延迟（性能）之间做**权衡**，参见 [PACELC](https://en.wikipedia.org/wiki/PACELC_theorem) 理论（CAP 理论的扩展版）。
- **配置和协调服务**：
  - NameServer 负责存储消息队列路由信息、Borker 集群注册信息等元数据，是 ZooKeeper 的轻量级替代。
  - **自动选举主节点**[^10][^11]：
    - **Raft 模式**：RocketMQ 4.5 开始，DLedger 模块被集成到各个 Borker 节点的进程中，用于 Borker 节点的自动选主，实现自动故障转移，自动选主基于 Raft 协议。
    - **Controller 模式**：RocketMQ 5.0 开始，自动选主支持 DLedger Controller 模式，Controller 可以独立部署，也可以嵌入在 Nameserver 中。

RocketMQ 架构，以及各个 Borker 下的分区和副本分布示例，如下图所示：

<img width="800" alt="RocketMQ 架构与分区和副本分布示例" title="RocketMQ 架构与分区和副本分布示例" src="https://static.nullwy.me/rocketmq-architecture.png">

# Kafka

**Kafka 的数据分片和复制策略**[^12][^13]：
- **分片策略**：
  - **分片术语命名**：分区（partition）
    - 将单个 Topic 的消息日志拆分到多个分区
  - **键-分片的分配关系**：按 Hash 拆分或轮询分配。
    - 若消息 key 有值，按 key 的 Hash 值拆分；若消息 key 值为 null 时，轮询分配给各个分。也可以自定义分区策略。Hash 拆分具体实现是，根据 murmur2 算法计算消息 key 的 Hash 值，然后对总分区数求模得到消息要被发送到的目标分区号。
      - 相关源码：[DefaultPartitioner](https://github.com/apache/kafka/blob/2.3.0/clients/src/main/java/org/apache/kafka/clients/producer/internals/DefaultPartitioner.java)。
  - **分片-机器的分配关系**：可配置某 Topic 的分区总数量。
    - 在创建 Topic 时把各个分区和分区的副本**轮询分配**给各个 `Broker` 节点。
      - 相关源码：AdminUtils#[assignReplicasToBrokers](https://github.com/apache/kafka/blob/3.6.0/server-common/src/main/java/org/apache/kafka/admin/AdminUtils.java#L46)
    - 若发送消息时自动创建 Topic，由配置项 `num.partitions` 控制 Topic 的默认分区总数量，默认值 `1`。
    - 若预先手动创建 Topic，执行 `kafka-topics.sh --create` 命令，由 `--partitions` 命令行参数控制该 Topic 的分区总数量。
  - **分片再均衡策略**：手动再均衡
    - 在扩容添加新 Broker 节点后，新的分区和分区副本能自动分配到新的 Broker 节点上，但已有的旧分区和节点的分配关系的固定的。如果要让旧的分区和分区副本能分配新的 Broker 节点，需要手动执行分区重分配命令 `kafka-reassign-partitions.sh`。
- **复制策略**：
  - **复制单位**：以分区为单位
  - **复制系数**：
    - 自动创建 Topic 时，由配置项 `default.replication.factor` 全局控制 Topic 的默认副本个数，默认值 `1`。
    - 手动创建 Topic 时，执行 `kafka-topics.sh --create` 命令，由 `--replication-factor` 命令行参数控制该 Topic 的分区副本的复制系数。
    - 复制系数必须等于或小于可用 Broker 节点数，如果大于可用 Broker 节点数，在创建 Topic 时会报异常。
    - 推荐的复制系数的配置值是 >= 3，通常配置为 `3`。复制系数配置为 >= 3 的原因是，允许集群内同时发生一次计划内停机和一次计划外停机，配置为 `3` 是在避免消息丢失和过度复制之间的常见的权衡选择。HBase（基于 HDFS）和 Cassandra 等分布式存储系统默认的复制系数也是 `3`。
  - **副本更新传播策略**：类似于微软的 PacificA 复制协议，Elasticsearch 的[分片复制](https://www.elastic.co/guide/en/elasticsearch/reference/8.12/docs-replication.html)也采用 PacificA 协议
    - 副本分为主从（leader-follower）两种角色。Kafka 动态维护**同步副本集合**（in-sync replica set），简称 **ISR 集合**。如果一个 follower 副本落后 leader 的时间超过 `replica.lag.time.max.ms` 配置值（Kafka 2.5 开始从默认 10 秒改为 30 秒），那么该 follower 副本会被认为是“不同步副本”（out-of-sync replica，OSR），会被移除 ISR 集合。在消息 commit 之前必须保证 ISR 集合中的全部节点都完成同步复制。这种机制确保了只要 ISR 中有一个或者以上的 follower，一条被 commit 的消息就不会丢失。ISR 集合大小由 Broker 端的配置项 `min.insync.replicas` 控制，默认值 `1`，即只需要 leader。
    - Producer 端的配置项 `acks`，用于控制在确认一个请求发送完成之前需要收到的反馈信息的数量。`min.insync.replicas` 配置项只有在 `acks=all` 时才生效。
      - `acks=0`：表示 Producer 不等待 Broker 返回确认消息。
      - `acks=1`（Kafka < v3.0 默认）：表示 leader 节点会将记录写入本地日志，并且在所有 follower 节点反馈之前就先确认成功。
      - `acks=all`（Kafka >= v3.0 默认）：表示 leader 节点会等待所有同步中的副本（ISR集合）确认之后再确认这条记录是否发送完成。
    - 与异步复制、半同步复制、同步复制的对应关系：
      - 当 `acks=0` 或 `acks=1` 时，相当于**异步复制**。
      - 当 `acks=all` 并且 `min.insync.replicas` 值大于 `1` 并小于 Broker 节点总数时，相当于**半同步复制**。
      - 当 `acks=all` 并且 `min.insync.replicas` 值等于 Broker 节点总数时，相当于**全同步复制**。
  - **主从读写分离**：
    - Kafka 2.4 之前，leader 副本可写可读，follower 副本不可读，仅用于备份。消息消费者只允许读取 leader 副本，follower 副本不处理来自消费者的请求。当 leader 所在的节点发生崩溃，其中一个 follower 就会被 Controller 选举为新 leader。
    - Kafka 2.4 开始（2019.12 发布）支持读取 follower 副本来消费消息，参见 [KIP-392](https://issues.apache.org/jira/browse/KAFKA-8443)。
  - **消息可靠性**：
    - 优先考虑消息可靠性（无消息丢失）又同时兼顾性能的常用的配置是，复制系数的配置值为 `3`，ISR 集合大小的配置值为 `min.insync.replicas=2`，消息发送确认的配置值为 `acks=all`[^14][^15]。
    - Kafka 是默认异步刷盘的，没有直接的同步刷盘相关配置项。Kafka 会在重启之前和关闭日志片段（默认1 GB 大小时关闭）时将消息冲刷到磁盘上，或者等到 Linux 系统页面缓存被填满时冲刷。虽然 Kafka 提供刷盘的时间间隔和刷盘的消息条数的配置项，但是官方文档不建议设置，推荐将刷盘的工作交给操作系统完成[^16]。相对于刷盘，复制提供了更强的可靠性保障。
- **配置和协调服务**：
  - **ZooKeeper 模式**[^17]：ZooKeeper 负责存储元数据，包括 Broker、Topic、分区、副本、路由等信息，以及负责选举 Controller 角色的 Broker，整个集群只有一个 Controller 角色的 Broker。Controller 角色的 Broker 节点的主要职责是 Broker 集群成员管理、Topic 管理（创建、删除、增加分区）、分区重分配、选举新的分区 leader 副本等，这些职责的实现重度依赖 ZooKeeper。
  - **KRaft 模式**[^18]：Kafka 2.8 开始，Kafka 开始用基于 Raft 的控制器替换基于 ZooKeeper 的控制器，新控制器叫作 KRaft。KRaft 模块被集成在 Borker 节点的进程中，去掉了对 ZooKeeper 的依赖，简化了整体架构。

Kafka 在 ZooKeeper 模式下的架构图，以及各个 Borker 下的分区和副本分布示例，如下图所示：

<img width="800" alt="Kafka 架构与分区和副本分布示例" title="Kafka 架构与分区和副本分布示例" src="https://static.nullwy.me/kafka-architecture.png">

Kafka 在 KRaft 模式下的架构图，如下图所示[^18]：

<img width="600" alt="Kafka 的 KRaft 模式" title="Kafka 的 KRaft 模式" src="https://static.nullwy.me/kafka-kraft-mode.png">

# 参考资料

[^1]: 2017-11 阿里林清山隆基：阿里消息中间件架构演进之路：notify和metaq <https://zhuanlan.zhihu.com/p/302600352>
[^2]: 2013-07 淘宝张乐伟韩彰：淘宝消息中间件技术演变：MetaQ 1.0、MetaQ 2.0、MetaQ 3.0（slides, 30p）<https://www.modb.pro/doc/109298>
[^3]: 2017-03 阿里冯嘉鼬神：Apache RocketMQ背后的设计思路与最佳实践 <https://developer.aliyun.com/article/71889>
[^4]: Apache RocketMQ 4.9.x开发者指南 <https://github.com/apache/rocketmq/blob/4.9.x/docs/cn>
[^5]: 2019-03 张乘辉：深度解析RocketMQ Topic的创建机制 <https://objcoding.com/2019/03/31/rocketmq-topic/>
[^6]: 2019-09 张乘辉：RocketMQ主从读写分离机制 <https://objcoding.com/2019/09/22/rocketmq-read-write-separation/>
[^7]: Apache RocketMQ 4.9.x开发者指南：特性：4 消息可靠性 <https://github.com/apache/rocketmq/blob/4.9.x/docs/cn/features.md>
[^8]: 2016-04 Kafka vs RocketMQ——单机系统可靠性 <https://web.archive.org/web/0/http://jm.taobao.org/2016/04/28/kafka-vs-rocktemq-4>
[^9]: 2018-12 How much memory should we use for broker and namesrv when using cluster mode? #614 <https://github.com/apache/rocketmq/issues/614>
[^10]: 2019-08 金融通、武文良：RocketMQ 实现高可用多副本架构的关键：DLedger—基于raft协议的commitlog存储库 <https://mp.weixin.qq.com/s/0nmWq29FN17vNzt0njRE-Q> <https://www.infoq.cn/article/7xeJrpDZBa9v*GDZOFS6>
[^11]: 2022-09 金融通：RocketMQ 5.0：面向消息与流的云原生高可用架构 <https://mp.weixin.qq.com/s/bb6cGUxpsAoU-IqBgmSJHw>

[^12]: Kafka 2.8 权威指南，第2版2021，[豆瓣](https://book.douban.com/subject/36161660/)
[^13]: Kafka 文档 <https://kafka.apachecn.org/> <https://kafka.apache.org/36/documentation.html>
[^14]: Optimize Confluent Cloud Clients for Durability <https://docs.confluent.io/cloud/current/client-apps/optimizing/durability.html>
[^15]: 2019-06 胡夕：Kafka 2.3 核心技术与实战：11 | 无消息丢失配置怎么实现？ <https://time.geekbang.org/column/article/102931>
[^16]: Kafka Documentation: Application vs. OS Flush Management <https://kafka.apache.org/36/documentation.html#appvsosflush>
[^17]: 2019-08 胡夕：Kafka 2.3 核心技术与实战：26 | 你一定不能错过的Kafka控制器（Controller） <https://time.geekbang.org/column/article/111339>
[^18]: 2022-04 Jun Rao: The Apache Kafka Control Plane (ZooKeeper vs. KRaft) <https://developer.confluent.io/courses/architecture/control-plane/>