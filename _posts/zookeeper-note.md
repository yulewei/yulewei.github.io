---
title: ZooKeeper 学习笔记
date: 2017-11-23 15:52:51
categories: 架构
tags: [ZooKeeper, 分布式, 共识, Paxos, 架构]
---


# ZooKeeper 介绍

ZooKeeper（[wiki](https://en.wikipedia.org/wiki/Apache_ZooKeeper)，[home](http://zookeeper.apache.org/)，[github](https://github.com/apache/zookeeper)） 是用于分布式应用的开源的分布式协调服务。通过暴露简单的原语，分布式应用能在之上构建更高层的服务，如同步、配置管理和组成员管理等。在设计上易于编程开发，并且数据模型使用了熟知的文件系统目录树结构 [ [doc](http://zookeeper.apache.org/doc/current/zookeeperOver.html) ]。

<!--more-->

## 共识与 Paxos

在介绍 ZooKeeper 之前，有必要了解下 Paxos 和 Chubby。2006 年 Google 在 OSDI 发表关于 [Bigtable](https://academic.microsoft.com/paper/2624304035) 和 [Chubby](https://academic.microsoft.com/paper/1992479210) 的两篇会议论文，之后再在 2007 年 PODC 会议上发表了论文“[Paxos Made Live](https://academic.microsoft.com/paper/2143149536)”，介绍 Chubby 底层实现的共识（[consensus](https://en.wikipedia.org/wiki/Consensus_%28computer_science%29)）协议 Multi-Paxos，该协议对 Lamport 的原始 Paxos 算法做了改进，提高了运行效率 [ [ref](https://zhuanlan.zhihu.com/p/21466932) ]。Chubby 作为锁服务被 Google 应用在 GFS 和 Bigtable 中。受 Chubby 的影响，来自 Yahoo 研究院的 Benjamin Reed 和 Flavio Junqueira 等人开发了被业界称为开源版的 Chubby 的 ZooKeeper（内部实现事实上稍有不同 [ [ref](https://cwiki.apache.org/confluence/display/ZOOKEEPER/Zab+vs.+Paxos) ]），底层的共识协议为 ZAB。Lamport 的 Paxos 算法出了名的难懂，如何让算法更加可理解（understandable），便成了 Stanford 博士生 Diego Ongaro 的研究课题。Diego Ongaro 在 2014 年发表了介绍 [Raft](https://academic.microsoft.com/paper/2156580773) 算法的论文，“In search of an understandable consensus algorithm”。Raft 是可理解版的 Paxos，很快就成为解决共识问题的流行协议之一。这些类 Paxos 协议和 Paxos 系统之间的关系，如下 [ [Ailijiang2016](https://academic.microsoft.com/paper/2414647798) ]：

<img width="500" alt="Paxos 协议和 Paxos 系统" title="Paxos 协议和 Paxos 系统" src="https://static.nullwy.me/paxos-protocols-vs-paxos-systems.png">

Google 的 Chubby 没有开源，在云计算和大数据技术的风口下，Yahoo 开源的 ZooKeeper 便在工业界流行起来。ZooKeeper 重要的时间线如下：
- 2007 年 11 月 ZooKeeper 1.0 在 SourceForge 上发布 [ [ref](https://sourceforge.net/p/zookeeper/news/) ]
- 2008 年 6 月开始从 SourceForge 迁移到 Apache [ [ref](https://issues.apache.org/jira/browse/ZOOKEEPER-1) ]，在 10 月 Zookeeper 3.0 发布，并成为 Hadoop 的子项目 [ [ref1](https://sourceforge.net/p/zookeeper/mailman/message/20659436/) [ref2](https://issues.apache.org/jira/projects/ZOOKEEPER/versions/12313216) ]

关于 ZooKeeper 名字的来源，Flavio Junqueira 和 Benjamin Reed 在介绍 [ZooKeeper 的书](https://book.douban.com/subject/26766807/)中有如下阐述：
> ZooKeeper 由雅虎研究院开发。我们小组在进行 ZooKeeper 的开发一段时间之后，开始推荐给其他小组，因此我们需要为我们的项目起一个名字。与此同时，小组也一同致力于 Hadoop 项目，参与了很多动物命名的项目，其中有广为人知的 Apache Pig 项目（<http://pig.apache.org>）。我们在讨论各种各样的名字时，一位团队成员提到我们不能再使用动物的名字了，因为我们的主管觉得这样下去会觉得我们生活在动物园中。大家对此产生了共鸣，分布式系统就像一个动物园，混乱且难以管理，而 ZooKeeper 就是将这一切变得可控。

## 体系结构

ZooKeeper 服务由若干台服务器构成，其中的一台通过 ZAB 原子广播协议选举作为主控服务器（leader），其他的作为从属服务器（follower）。客户端可以通过 TCP 协议连接任意一台服务器。如果客户端是读操作请求，则任意一个服务器都可以直接响应请求；如果是更新数据操作（写数据或者更新数据）。则只能由主控服务器来协调更新操作；如果客户端连接的是从属服务器，则从属服务器会将更新据请求转发到主控服务器，由其完成更新操作。主控服务器将所有更新操作序列化，利用 ZAB 协议将数据更新请求通知所有从属服务器，ZAB 保证更新操作。

![zookeeper-service](https://static.nullwy.me/zookeeper-service.png 'ZooKeeper 架构图')


读和写操作，如下图所示 [ [Haloi2015](https://book.douban.com/subject/26336214/) ]：
<img width="600" alt="ZooKeeper 读和写操作" title="ZooKeeper 读和写操作" src="https://static.nullwy.me/zookeeper-read-write.png">

ZooKeeper 的任意一台服务器都可以响应客户端的读操作，这样可以提高吞吐量。Chubby 在这点上与 ZooKeeper 不同，所有读/写操作都由主控服务器完成，从属服务器只是为了提高整个协调系统的可用性，即主控服务器发生故障后能够在从属服务器中快速选举出新的主控服务器。在带来高吞吐量优势的同时，ZooKeeper 这样做也带来潜在的问题：客户端可能会读到过期数据，因为即使主控服务器已经更新了某个内存数据，但是 ZAB 协议还未能将其广播到从属服务器。为了解决这一问题，在 ZooKeeper 的接口 API 函数中提供了 sync 操作，应用可以根据需要在读数据前调用该操作，其含义是：接收到 sync 命令的从属服务器从主控服务器同步状态信息，保证两者完全一致。这样如果在读操作前调用 sync 操作，则可以保证客户端一定可以读取到最新状态的数据。

## 数据模型

ZooKeeper 所提供的命名空间跟标准文件系统很相似。路径中一系列元素是用斜杠（/）分隔的。每个节点在 ZooKeper 命名空间中是用路径来识别的。在 ZooKeeper 术语下，节点被称为 znode。默认每个 znode 最大只能存储 1M 数据（可以通过配置参数修改），这与 Chubby 一样是出于避免应用将协调系统当作存储系统来用。znode 只能使用绝对路径，相对路径不能被 ZooKeeper 识别。znode 命名可以是任意 Unicode 字符。唯一的例外是，名称"/zookeeper"。命名为"/zookeeper"的 znode，由 ZooKeeper 系统自动生成，用配额（quota）管理。

<img width="300" alt="ZooKeeper 数据模型" title="ZooKeeper 数据模型" src="https://static.nullwy.me/zookeeper-data-model.jpg">


# ZooKeeper 使用

## 安装与配置

ZooKeeper 安装与启动：

``` sh
$ brew info zookeeper
zookeeper: stable 3.4.10 (bottled), HEAD
Centralized server for distributed coordination of services
https://zookeeper.apache.org/
... 省略
$ brew install zookeeper
$ zkServer start  # 启动
$ zkServer stop   # 终止
$ zkServer help
ZooKeeper JMX enabled by default
Using config: /usr/local/etc/zookeeper/zoo.cfg
Usage: ./zkServer.sh {start|start-foreground|stop|restart|status|upgrade|print-cmd}
```

若不修改配置文件，默认是**单机模式**启动。若要使用**集群模式**，需要修改 `/usr/local/etc/zookeeper/zoo.cfg`（默认路径）。示例 `zoo.cfg` [ [doc](http://zookeeper.apache.org/doc/current/zookeeperStarted.html#sc_RunningReplicatedZooKeeper) ]：

```
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
initLimit=5
syncLimit=2
server.1=192.168.211.11:2888:3888
server.2=192.168.211.12:2888:3888
server.3=192.168.211.13:2888:3888
```

`clientPort`：客户端连接 Zookeeper 服务器的端口，Zookeeper 会监听这个端口，接受客户端的访问请求。
`server.X=YYY:A:B`

- X：表示的服务器编号；
- YYY：表示服务器的ip地址；
- A：表示服务器节点间的通信端口，用于 follower 和 leader 节点的通信；
- B：表示选举端口，表示选举新 leader 时服务器间相互通信的端口，当 leader 挂掉时，其余服务器会相互通信，选择出新的 leader。

若想在单台主机上试验集群模式，可以将 `YYY` 都修改为 `localhost`，并且让两个端口 `A:B` 也相互不同（比如：2888:3888, 2889:3889, 2890:3890），即可实现**伪集群模式**。示例 `zoo.cfg` 如下 [ [doc](http://zookeeper.apache.org/doc/current/zookeeperStarted.html#sc_RunningReplicatedZooKeeper) ]：

``` 
server.1=localhost:2888:3888
server.2=localhost:2889:3889
server.3=localhost:2890:3890
```

zkCli 支持的全部命令：

``` sh
$ zkCli help
ZooKeeper -server host:port cmd args
	stat path [watch]
	set path data [version]
	ls path [watch]
	delquota [-n|-b] path
	ls2 path [watch]
	setAcl path acl
	setquota -n|-b val path
	history
	redo cmdno
	printwatches on|off
	delete path [version]
	sync path
	listquota path
	rmr path
	get path [watch]
	create [-s] [-e] path data acl
	addauth scheme auth
	quit
	getAcl path
	close
	connect host:port
```

## 节点类型及其操作

Zookeeper 支持两种类型节点：持久节点（persistent znode）和临时节点（ephemeral znode）。持久节点不论客户端会话情况，一直存在，只有当客户端显式调用删除操作才会消失。而临时节点则不同，会在客户端会话结束或者发生故障的时候被 ZooKeeper 系统自动清除。另外，这两种类型的节点都可以添加是否是顺序（sequential）的特性，这样就有了：持久顺序节点和临时顺序节点。

**(1) 持久节点（persistent znode）**

使用 `create` 创建节点（默认持久节点），以及使用 `get` 查看该节点：

``` sh
$ zkCli -server 127.0.0.1  # 启动客户端
[zk: localhost:2181(CONNECTED) 1] create /zoo 'hello zookeeper'
Created /zoo
[zk: localhost:2181(CONNECTED) 2] get /zoo
hello zookeeper
cZxid = 0x8d
ctime = Thu Nov 08 20:42:55 CST 2017
mZxid = 0x8d
mtime = Thu Nov 08 20:42:55 CST 2017
pZxid = 0x8d
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 15
numChildren = 0
```

`create` 创建子节点，以及使用 `ls` 查看全部子节点：

``` sh
[zk: localhost:2181(CONNECTED) 3] create /zoo/duck ''
Created /zoo/duck
[zk: localhost:2181(CONNECTED) 4] create /zoo/goat ''
Created /zoo/goat
[zk: localhost:2181(CONNECTED) 5] create /zoo/cow ''
Created /zoo/cow
[zk: localhost:2181(CONNECTED) 6] ls /zoo
[cow, goat, duck]
```

`delete` 删除节点，以及使用 `rmr` 递归删除：

``` sh
[zk: localhost:2181(CONNECTED) 7] delete /zoo/duck
[zk: localhost:2181(CONNECTED) 8] ls /zoo
[cow, goat]
[zk: localhost:2181(CONNECTED) 9] delete /zoo
Node not empty: /zoo
[zk: localhost:2181(CONNECTED) 10] rmr /zoo
[zk: localhost:2181(CONNECTED) 11] ls /zoo
Node does not exist: /zoo
```

**(2) 临时节点（ephemeral znode）**

和持久节点不同，临时节点不能创建子节点：

``` sh
$ zkCli  # 启动第1个客户端
[zk: localhost:2181(CONNECTED) 0] create -e /node 'hello'
Created /node
[zk: localhost:2181(CONNECTED) 40] get /node
hello
cZxid = 0x97
ctime = Thu Nov 08 21:01:25 CST 2017
mZxid = 0x97
mtime = Thu Nov 08 21:01:25 CST 2017
pZxid = 0x97
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x161092a0ff30000
dataLength = 5
numChildren = 0
[zk: localhost:2181(CONNECTED) 1] create /node/child ''
Ephemerals cannot have children: /node/child
```

临时节点在客户端会话结束或者发生故障的时候被 ZooKeeper 系统自动清除。现在试验下的针对临时节点自动清除的监视：

``` sh
$ zkCli  # 启动第2个客户端
[zk: localhost:2181(CONNECTED) 0] create -e /node 'hello'
Node already exists: /node
[zk: localhost:2181(CONNECTED) 1] stat /node true
cZxid = 0x97
ctime = Thu Nov 08 21:01:25 CST 2017
mZxid = 0x97
mtime = Thu Nov 08 21:01:25 CST 2017
pZxid = 0x97
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x161092a0ff30000
dataLength = 5
numChildren = 0
```

若客户端1，退出 `quit` 或崩溃，客户端2将收到监视事件：

``` sh
[zk: localhost:2181(CONNECTED) 2]
WATCHER::

WatchedEvent state:SyncConnected type:NodeDeleted path:/node
```

**(3) 顺序节点（sequential znode）**

顺序节点在其创建时 ZooKeeper 会自动在 znode 名称上附加上顺序编号。顺序编号，由父 znode 维护，并且单调递增。顺序编号，由 4 字节的有符号整数组成，并被格式化为 0 填充的 10 位数字。

``` sh
[zk: localhost:2181(CONNECTED) 1] create /test ''
Created /test
[zk: localhost:2181(CONNECTED) 2] create -s /test/seq ''
Created /test/seq0000000000
[zk: localhost:2181(CONNECTED) 3] create -s /test/seq ''
Created /test/seq0000000001
[zk: localhost:2181(CONNECTED) 4] create -s /test/seq ''
Created /test/seq0000000002
[zk: localhost:2181(CONNECTED) 5] ls /test
[seq0000000000, seq0000000001, seq0000000002]
```

## 客户端 API

<style>
  table {
    font-family: consolas, Menlo
  }
</style>

ZooKeeper 提供的主要 znode 操作 API 如下表所示：

| API 操作 | 描述 | CLI 命令 |
| --- | --- | --- |
| create | 创建 znode | create |
| delete | 删除 znode | delete/rmr/delquota |
| exists | 检查 znode 是否存在 | stat |
| getChildren | 读取 znode 全部的子节点 | ls/ls2 |
| getData | 读取 znode 数据 | get/listquota |
| setData | 设置 znode 数据 | set/setquota |
| getACL | 读取 znode 的 ACL | getACL |
| setACL | 设置 znode 的 ACL | setACL |
| sync | 同步 | sync |

Java 的 [ZooKeeper](https://static.javadoc.io/org.apache.zookeeper/zookeeper/3.4.11/org/apache/zookeeper/ZooKeeper.html) 类实现了上述提供的 API。

Zookeeper 底层是 Java 实现，`zkCli` 命令行工具底层也是 Java 实现，对应的 Java 实现类为 `org.apache.zookeeper.ZooKeeperMain` [ [src1](https://github.com/apache/zookeeper/blob/release-3.5.3/bin/zkCli.sh) [src2](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/ZooKeeperMain.java) ]。ZooKeeper 3.5.x 下，CLI 命令与底层实现 API 对应关系：

| 命名 CLI | Java API ([ZooKeeper](https://static.javadoc.io/org.apache.zookeeper/zookeeper/3.4.11/org/apache/zookeeper/ZooKeeper.html) 类) |
| --- | --- |
| [addauth](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/cli/AddAuthCommand.java) scheme auth | public void addAuthInfo(String scheme, byte[] auth) |
| [close](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/cli/CloseCommand.java) | public void close() |
| [create](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/cli/CreateCommand.java) [-s] [-e] path data acl | public String create(final String path, byte data[], List<ACL> acl, CreateMode createMode) |
| [delete](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/cli/DeleteCommand.java) path [version] | public void delete(String path, int version)   |
| [delquota](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/cli/DelQuotaCommand.java) [-n&#124;-b] path | public void delete(String path, int version)  |
| [get](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/cli/GetCommand.java) path [watch] | public byte[] getData(String path, boolean watch, Stat stat) |
| [getAcl](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/cli/GetAclCommand.java) path | public List<ACL> getACL(final String path, Stat stat)   |
| [listquota](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/cli/ListQuotaCommand.java) path | public byte[] getData(String path, boolean watch, Stat stat) |
| [ls](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/cli/LsCommand.java) path [watch] | public List<String> getChildren(String path, Watcher watcher, Stat stat)|
| [ls2](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/cli/Ls2Command.java) path [watch] | -   |
| quit | public void close() |
| [rmr](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/cli/DeleteAllCommand.java) path |  public void delete(final String path, int version) |
| [set](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/cli/SetCommand.java) path data [version] | public Stat setData(String path, byte[] data, int version) |
| [setAcl](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/cli/SetAclCommand.java) path acl |      public Stat setACL(final String path, List<ACL> acl, int aclVersion)|
| [setquota](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/cli/SetQuotaCommand.java) -n&#124;-b val path | public Stat setData(String path, byte[] data, int version)   |
| [stat](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/cli/StatCommand.java) path [watch] | public Stat exists(String path, boolean watch) |  
| [sync](https://github.com/apache/zookeeper/blob/release-3.5.3/src/java/main/org/apache/zookeeper/cli/SyncCommand.java) path | public void sync(String path, AsyncCallback.VoidCallback cb, Object ctx) |

## 监视点（watch）

ZooKeeper 提供了处理变化的重要机制一一监视点（watch）。通过监视点，客户端可以对指定的 znode 节点注册一个通知请求，在发生变化时就会收到一个单次的通知。当应用程序注册了一个监视点来接收通知，匹配该监视点条件的第一个事件会触发监视点的通知，并且最多只触发一次。例如，当 znode 节点也被删除，客户端需要知道该变化，客户端在 /z 节点执行 exists 操作并设置监视点标志位，等待通知，客户端会以回调函数的形式收到通知。

ZooKeeper 的 API 中的读操作：getData、getChildren 和 exists，均可以选择在读取的 znode 节点上设置监视点。使用监视点机制，我们需要实现 [Watcher](https://static.javadoc.io/org.apache.zookeeper/zookeeper/3.4.11/org/apache/zookeeper/Watcher.html) 接口类，该接口唯一方法为 process：

``` java
void process(WatchedEvent event)
```

[WatchedEvent](https://static.javadoc.io/org.apache.zookeeper/zookeeper/3.4.11/org/apache/zookeeper/WatchedEvent.html) 数据结构包括以下信息：

 - ZooKeeper会话状态（KeeperState)：Disconnected、SyncConnected、AuthFailed、ConnectedReadOnly 、SaslAuthenticated、Expired。
 - 事件类型（EventType)：NodeCreated 、NodeDeleted 、NodeDataChanged、NodeChildrenChanged 和 None 。
 - 若事件类型不是 None，还包括 znode 路径。

若收到 WatchedEvent， 在 zkCli 中会输出类似如下结果：

``` 
WatchedEvent state:SyncConnected type:NodeDeleted path:/node
```

监视点有两种类型：数据监视点和子节点监视点。创建、删除或设置一个 znode 节点的数据都会触发数据监视点，exists 和 getData 这两个操作可以设置数据监视点。只有 getChildren 操作可以设置子节点监视点，这种监视点只有在 znode 子节点创建或删除时才被触发。对于每种事件类型，我们通过以下调用设置监视点：

NodeCreated
&nbsp;&nbsp;&nbsp;通过 exists 调用设置一个监视点。
NodeDeleted
&nbsp;&nbsp;&nbsp;通过 exists 或 getData 调用设置监视点。
NodeDataChanged
&nbsp;&nbsp;&nbsp;通过 exists 或getData 调用设置监视点。
NodeChildrenChanged
&nbsp;&nbsp;&nbsp;通过 getChildren 调用设置监视点。

## Java 示例代码

在 Java 下使用 ZooKeeper 需要先添加如下 [maven](http://mvnrepository.com/artifact/org.apache.zookeeper/zookeeper) 依赖：

``` xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.4.11</version>
    <type>pom</type>
</dependency>
```

`ZookeeperDemo` 示例，展示了建立连接会话，以及对 znode 的创建、读取、修改、删除和设置监视点等操作：

``` java
import java.io.IOException;
import org.apache.commons.lang3.time.DateFormatUtils;
import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;

public class ZookeeperDemo {

    public static void main(String[] args) throws KeeperException, InterruptedException, IOException {
        // 创建服务器连接
        ZooKeeper zk = new ZooKeeper("127.0.0.1:2181", 100, new Watcher() {
            // 监控所有被触发的事件
            public void process(WatchedEvent event) {
                System.out.printf("WatchedEvent state:%s type:%s path:%s\n", event.getState(), event.getType(), event.getPath());
            }
        });

        // 创建节点
        zk.create("/zoo", "hello ZooKeeper".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        // 读取节点数据
        Stat stat = new Stat();
        System.out.println(new String(zk.getData("/zoo", false, stat)));
        printStat(stat);

        // 创建子节点
        zk.create("/zoo/duck", "hello duck".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        zk.create("/zoo/goat", "hello goat".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        zk.create("/zoo/cow", "hello cow".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);

        // 读取子节点列表，并设置监视点
        System.out.println(zk.getChildren("/zoo", true));

        // 读取子节点数据，并设置监视点
        System.out.println(new String(zk.getData("/zoo/duck", true, null)));

        // 修改子节点数据
        zk.setData("/zoo/duck", "hi duck".getBytes(), -1);
        // 读取修改后的子节点数据
        System.out.println(new String(zk.getData("/zoo/duck", true, null)));

        // 删除子节点
        zk.delete("/zoo/duck", -1);
        zk.delete("/zoo/goat", -1);
        zk.delete("/zoo/cow", -1);
        // 删除父节点
        zk.delete("/zoo", -1);

        // 关闭连接
        zk.close();
    }

    private static void printStat(Stat stat) {
        System.out.println("cZxid = 0x" + Long.toHexString(stat.getCzxid()));
        System.out.println("ctime = " + DateFormatUtils.format(stat.getCtime(), "yyyy-MM-dd HH:mm:ss"));
        System.out.println("mZxid = 0x" + Long.toHexString(stat.getMzxid()));
        System.out.println("mtime = " + DateFormatUtils.format(stat.getMtime(), "yyyy-MM-dd HH:mm:ss"));
        System.out.println("pZxid = 0x" + Long.toHexString(stat.getPzxid()));
        System.out.println("cversion = " + stat.getCversion());
        System.out.println("dataVersion = " + stat.getVersion());
        System.out.println("aclVersion = " + stat.getAversion());
        System.out.println("ephemeralOwner = 0x" + Long.toHexString(stat.getEphemeralOwner()));
        System.out.println("dataLength = " + stat.getDataLength());
        System.out.println("numChildren = " + stat.getNumChildren());
    }
}
```

输出结果：

```
WatchedEvent state:SyncConnected type:None path:null
hello ZooKeeper
cZxid = 0x1e1
ctime = 2017-11-20 12:18:36
mZxid = 0x1e1
mtime = 2017-11-20 12:18:36
pZxid = 0x1e1
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 15
numChildren = 0
[cow, goat, duck]
hello duck
WatchedEvent state:SyncConnected type:NodeDataChanged path:/zoo/duck
hi duck
WatchedEvent state:SyncConnected type:NodeDeleted path:/zoo/duck
WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/zoo
```

## ZooInspector

ZooInspector 是 ZooKeeper 3.3.0 开始官方提供的可视化查看和编辑 ZooKeeper 实例的工具 [ [ZOOKEEPER-678](https://issues.apache.org/jira/browse/ZOOKEEPER-678) ]。源码位于目录 `src/contrib/zooinspector` 下，GitHub 地址为：[link](https://github.com/apache/zookeeper/tree/master/src/contrib/zooinspector)。可以根据 `README.txt` 的说明运行使用。或者可以直接用 ZOOKEEPER-678 下提供的可执行 jar 包。

<img width="500" alt="ZooInspector" title="ZooInspector" src="https://static.nullwy.me/zookeeper-ZooInspector.png">

# 参考资料

1. 官方文档：ZooKeeper <http://zookeeper.apache.org/doc/current/index.html>
2. 2010-11 许令波：分布式服务框架 Zookeeper <https://www.ibm.com/developerworks/cn/opensource/os-cn-zookeeper/>
3. ZooKeeper：分布式过程协同技术详解，Benjamin Reed & Flavio Junqueira，2013，[豆瓣](https://book.douban.com/subject/26766807/)
4. Apache ZooKeeper Essentials, Haloi 2015，[豆瓣](https://book.douban.com/subject/26336214/)
5. 从Paxos到Zookeeper，阿里倪超 2015，[豆瓣](https://book.douban.com/subject/26292004/)
6. 大数据日知录：架构与算法，张俊林 2014，第5章 分布式协调系统，[豆瓣](https://book.douban.com/subject/25984046/)
7. 2010，Patrick Hunt, Mahadev Konar, Flavio Paiva Junqueira, Benjamin Reed: **ZooKeeper: Wait-free Coordination for Internet-scale Systems**. USENIX ATC 2010，[dblp](http://dblp.org/rec/conf/usenix/HuntKJR10)，[msa](https://academic.microsoft.com/paper/192446467)，[usenix](https://www.usenix.org/conference/usenix-atc-10/zookeeper-wait-free-coordination-internet-scale-systems)

