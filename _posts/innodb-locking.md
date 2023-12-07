---
title: InnoDB 的并发控制：锁与 MVCC
date: 2023-07-10 21:20:00
categories: MySQL
tags: [MySQL, InnoDB, 数据库, 锁, MVCC]
---

目前主流数据库事务的并发控制实现，如 MySQL InnoDB、PostgreSQL、Oracle，都使用两阶段封锁 [2PL](https://en.wikipedia.org/wiki/Two-phase_locking) 与 [MVCC](https://en.wikipedia.org/wiki/Multiversion_concurrency_control) 技术，但具体实现细节上存在差异。InnoDB 是在以封锁技术为主体的情况下，用 MVCC 技术辅助实现读-写、写-读操作的并发。PostgreSQL 的并发控制技术是以 MVCC 技术为主，封锁技术为辅。本文主要关注 InnoDB 事务的并发控制实现。

<!--more-->

<style>
  .nav-number {
    display: none;
  }
</style>

# 背景知识

并发控制，是数据库系统的 ACID 特性中的隔离性（[Isolation](https://en.wikipedia.org/wiki/Isolation_(database_systems))）的保障。所谓隔离性，就是事务的执行不应受到其他并发执行事务的干扰，事务的执行看上去应与其他事务是隔离的。被隔离的执行，等价于事务的某种串行执行，或者说，它等价于一个没有并发的执行。保证串行性可能只允许极小的并发度，采用较弱隔离性，能带来更高的并发度，是并发事务的正确性和性能之间的妥协。

早期各大数据库厂商实现并发控制时多采用基于封锁的并发控制技术，所以在基于封锁的技术背景下，才在 [ANSI SQL-92](https://en.wikipedia.org/wiki/SQL-92) 标准中提出了四种隔离级别：未提交读（Read Uncommitted）、己提交读（Read Committed）、可重复读（Repeatable Read）、可串行化（Serializable）（附注：为了书写简便本文将各个隔离级别依次缩写为 RU、RC、RR、SER）。ANSI SQL-92 标准的四种隔离级别，是根据三种读异常现象（phenomena）定义的，隔离级别和异常现象的关系如下：

| **隔离级别** | **P1 脏读** | **P2 不可重复读** | **P4 幻读** |
| --- | --- | --- | --- |
| **Read Uncommitted** | 可能 | 可能 | 可能 |
| **Read Committed** | 避免 | 可能 | 可能 |
| **Repeatable Read** | 避免 | 避免 | 可能 |
| **Serializable** | 避免 | 避免 | 避免 |

ANSI SQL-92 标准文档对三种读异常现象的定义原文如下 [[ref](http://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt)]：

> The isolation level specifies the kind of phenomena that can occur during the execution of concurrent SQL-transactions. The following phenomena are possible:
> **1) P1 ("Dirty read")**: SQL-transaction T1 modifies a row. SQL-transaction T2 then reads that row before T1 performs a COMMIT. If T1 then performs a ROLLBACK, T2 will have read a row that was never committed and that may thus be considered to have never existed.
> **2) P2 ("Non-repeatable read")**: SQL-transaction T1 reads a row. SQL-transaction T2 then modifies or deletes that row and performs a COMMIT. If T1 then attempts to reread the row, it may receive the modified value or discover that the row has been deleted.
> **3) P3 ("Phantom")**: SQL-transaction T1 reads the set of rows N that satisfy some <search condition>. SQL-transaction T2 then executes SQL-statements that generate one or more rows that satisfy the <search condition> used by SQL-transaction T1. If SQL-transaction T1 then repeats the initial read with the same <search condition>, it obtains a different collection of rows.

除了脏读、不可重复读和幻读这 3 种读数据异常外，还有写数据异常，即脏写和丢失更新。各个异常的含义如下：
  - **P0 脏写（Dirty Write）**：事务 T1 写某数据项，并且未提交或回滚，事务 T2 也写该数据项，然后事务 T1 或事务 T2 回滚，回滚导致另外一个事务的修改被连带回滚。脏写异常会导致事务无法回滚，原子性无法得到保障，所以全部隔离级别下都应该避免。脏写也可以叫回滚丢失。
  - **P1 脏读（Dirty Read）**：读到了其他事务还未提交的数据。
  - **P2 不可重复读（Non-Repeatable）**：事务 T1 读取某数据项，事务 T2 修改 update 或删除 delete 该数据项，事务 T1 再次读取该数据项，结果不同。
  - **P3 幻读（Phantom）**：事务 T1 读取满足某条件的数据项集合，事务 T2 生成新的满足该条件的数据项，事务 T2 再次读取满足该条件的数据项集合，结果不同。
  - **P4 丢失更新（Lost Update）**：事务 T1 读取某数据项，事务 T2 更新该数据项并提交，事务 T1 忽略事务 T2 的更新，直接基于最初的读取数据项做更新并提交，导致事务 T2 的更新丢失。丢失更新也可以叫覆盖丢失。

各个异常的读写操作序列的简化符号表示如下 [Berenson 1995]：

```
P0: w1[x]...w2[x]...(c1 or a1)                 事务 T2 脏写
A1: w1[x]...r2[x]...(a1 and c2 in any order)   事务 T2 脏读，r2[x] 为脏读
A2: r1[x]...w2[x]...c2...r1[x]...c1            事务 T1 不可重复读，两次 r1[x] 结果不同
A3: r1[P]...w2[y in P]...c2...r1[P]...c1       事务 T1 幻读，两次 r1[P] 结果不同
P4: r1[x]...w2[x]...w1[x]...c1                 事务 T2 更新丢失，c1 导致 w2[x] 丢失
```

其中 w1[x] 表示事务 T1 写入记录 x，r1[x] 表示事务 T1 读取记录 x，c1 表示事务 T1 提交，a1 表示事务 T1 回滚，r1[P] 表示事务 T1 按照谓词 P 的条件读取若干条记录，w1[y in P] 表示事务 T1 写入记录 y 满足谓词 P 的条件。

Berenson 的论文评判了 ANSI SQL-92 标准的异常定义。ANSI SQL-92 标准的异常的定义存在歧义，可以严格解释，也可以宽松解释，A1、A2 和 A3 的符号表示为严格解释，按严格解释，某些特殊的异常无法囊括，所以推荐宽松解释。按照标准的定义，容易引起误解的是，在排除 P1 脏读、P2 不可重复、P3 幻读这三种读异常后就会得到可串行化隔离级别，但是事实并非如此。标准没有定义 P0 脏写和 P4 更新丢失异常。另外，基于 MVCC 技术实现的快照隔离（[Snapshot Isolation](https://en.wikipedia.org/wiki/Snapshot_isolation)），能避免标准定义的 P1 脏读、P2 不可重复、P3 幻读，并且避免 P0 脏写和 P4 更新丢失，但还存在写偏序（Write Skew）异常。

**不可重复读和幻读的区别：**
  - 不可重复读对于事务 T2 的写操作是更新或删除操作，而幻读对于事务 T2 的写操作是插入（插入的新数据满足条件）或更新（使不满足条件的数据在更新后满足条件）操作。
  - 对于幻读现象中事务 T2 的操作，如果操作是对现有数据的更新或删除操作，则表明这样的操作等同于不可重复读，即是在多个行数据上进行更新或删除，**即在多个行数据上批量化重演了不可重复读现象**。
  - 不可重复读和幻象最大的区别就是前者只需要“锁住”（考虑）已经读过的数据，而幻读需要**对“还不存在的数据“做出预防**。不可重复读现象中事务 T2 着眼于**对现有数据进行操作**；而幻读现象中事务 T2 **着眼于对新增**（或不在锁定范围内已经存在的数据上做更新后而得的数据满足了谓词条件）**数据**。

**异常由并发冲突引起，对应关系如下：**
  + 写写冲突：P0 脏写、P4 丢失更新
  + 写读冲突：P1 脏读
  + 读写冲突：P2 不可重复读、P3 幻读

早期各大数据库厂商实现并发控制时多采用基于封锁的并发控制技术，所以在基于封锁的技术背景下，才在ANSI SQL 标准中提出了四种隔离级别。基于锁的并发控制技术的加锁方式与隔离级别的关系表 [Berenson 1995]：

| **隔离级别** | **写锁** | **数据项的读锁** | **谓词的读锁** |
| --- | --- | --- | --- |
| **Read Uncommitted** | 长写锁 | 无锁要求 | 无锁要求 |
| **Read Commited** | 长写锁 | 短读锁 | 短谓词锁 |
| **Repeatable Read** | 长写锁 | 长读锁 | 短谓词锁 |
| **Serializable** | 长写锁 | 长读锁 | 长谓词锁 |

**说明：**
  +  短锁（short duration lock），当前正在执行的语句持有锁，语句执行完毕锁被释放。长锁（long duration lock），当锁被持有后，直到事务提交之后才被释放。
  + RU 隔离级别，阻止 P0，长写锁
  + RC 隔离级别，阻止 P0、P1，长写锁 + 短读锁 + 短谓词锁
  + RR 隔离级别，阻止 P0、P1、P4、P2，长写锁 + 长写锁 + 短谓词锁
  + SER 隔离级别，阻止 P0、P1、P4、P2、P3，长写锁 + 长写锁 + 长谓词锁

基于锁的并发控制下，隔离级别和异常现象的关系：

| **隔离级别** | **P0 脏写** | **P1 脏读** | **P4 丢失更新** | **P2 不可重复读** | **P4 幻读** |
| --- | --- | --- | --- | --- | --- |
| **Read Uncommitted** | 避免 | 可能 | 可能 | 可能 | 可能 |
| **Read Committed** | 避免 | 避免 | 可能 | 可能 | 可能 |
| **Repeatable Read** | 避免 | 避免 | 避免 | 避免 | 可能 |
| **Serializable** | 避免 | 避免 | 避免 | 避免 | 避免 |

各个隔离级别在基于锁的并发控制技术下的具体的实现说明（参考自腾讯李海翔的《数据库事务处理的艺术》第 2 章）：

<img width="700" alt="基于锁的并发控制" title="基于锁的并发控制" src="https://static.nullwy.me/db-lock-based.png">

基于锁的并发控制，读-读操作可以并发执行，但读-写、写-读、写-写操作无法并发执行，阻塞等待。MVCC 结合封锁技术，使得读－写、写－读操作互不阻塞，即只有写－写操作不能并发，并发度被提高到 75%，这就是 MVCC 被广为使用的原因。

InnoDB 的并发控制以封锁技术为主，MVCC 技术为辅助。让我们先看下 InnoDB 的封锁技术。

# 共享锁与排他锁

InnoDB 存储引擎实现两种标准的**行级锁**模式，共享锁（读锁）和排他锁（写锁）[[doc](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)]：

- 共享锁（shared lock，S）：允许事务读一行数据。
- 排他锁（exclusive Lock，X）：允许事务删除或更新一行数据。

如果事务 T1 持有行 r 上的共享锁（S），则来自某个不同事务 T2 的对行 r 上的锁的请求将按如下方式处理：

- T2 对 S 锁的请求可以立即被授予。因此，T1 和 T2 持有 r 上的锁。
- T2 对 X 锁的请求不能立即被授予。

如果事务 T1 持有行 r 上的排他锁（X），则某个不同事务 T2 对 r 上任一类型的锁的请求无法立即被授予。相反，事务 T2 必须等待事务 T1 释放其对行 r 的锁定。

**共享锁和排他锁的兼容性：**

| 待申请 \\ 已持有 | 共享锁 S |  排他锁 X |
| --- | --- | --- |
| 共享锁 S | 兼容 | 冲突 |
| 排他锁 X | 冲突 | 冲突 |

**区分共享锁（读锁）和排它锁（写锁）后，读锁与读锁的并发可被允许进行，并发能力得以提高。**

对于 `update`、`delete` 和 `insert` 语句，InnoDB 会自动给涉及数据集加排他锁（X）；对于普通 `select` 语句，InnoDB 不会加任何锁（`SERIALIZABLE`隔离级别下除外）；事务可以通过以下语句显式给查询 `select` 显式加共享锁或排他锁：

- 共享锁（S）：`select ... for share`
- 排他锁（X）：`select ... for update`

现在让我们来试验下共享锁和排他锁。创建 `tbl` 表，并添加表数据：

```sql
create table tbl 
  (a int, b int, c int, d int, primary key(a), unique key(b), key(c));
insert into tbl values
  (10, 10, 10, 10), (20, 20, 20, 20), (30, 30, 30, 30), 
  (40, 40, 40, 40), (50, 50, 50, 50), (60, 60, 60, 60), 
  (70, 70, 70, 70), (80, 80, 80, 80), (90, 90, 90, 90),
  (100, 100, 100, 100);
```

InnoDB 的排它锁示例，如下：

| **事务1** | **事务2** |
|---|---|
| mysql> begin; | mysql> begin; |
| -- 在 a = 10 的索引记录上添加排他锁<br>mysql> select * from tbl where a = 10 for update; |  |
|  | -- 阻塞，获取 a = 10 的排他锁超时<br>mysql> update tbl set b = 42 where a = 10;<br>ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction |
|  | -- 阻塞，获取 a = 10 的排他锁超时<br>mysql> update tbl set b = 42 where a >= 10; |
|  | -- 阻塞，获取 a = 10 的排他锁超时<br>mysql> delete from tbl where a = 10; |
|  | -- 阻塞，获取 a = 10 的排他锁超时<br>mysql> select * from tbl where a = 10 for update; |
|  | -- 更新成功，可以获得其他记录的排他锁<br>mysql> update tbl set b = 42 where a = 20; |
| mysql> commit; |  |
|  | -- 更新成功，在事务 1 释放锁后，其他事务可以获取排他锁<br>mysql> update tbl set b = 42 where a = 10; |

InnoDB的共享锁示例，如下：

| 事务1 | 事务2 |
|---|---|
| mysql> begin; | mysql> begin; |
| -- 在 a = 10 的索引记录上添加共享锁<br>mysql> select * from tbl where a = 10 for share; |  |
|  | -- 获取 a = 10 的共享锁成功<br>mysql> select * from tbl where a = 10 for share; |
|  | -- 阻塞，获取 a = 10 的排他锁超时<br>mysql> update tbl set b = 42 where a = 10; |
| mysql> commit; |  |
|  | -- 更新成功，在事务 1 释放锁后，其他事务可以获取排他锁<br>mysql> update tbl set b = 42 where a = 10; |

# 多粒度与意向锁

InnoDB 存储引擎支持多粒度锁定（[multiple granularity locking](https://en.wikipedia.org/wiki/Multiple_granularity_locking)），这种锁定允许事务在行级上的锁和表级上的锁同时存在。为了支持在不同粒度上进行加锁操作，InnoDB 存储引擎支持一种额外的锁方式，称之为意向锁（Intention Lock）。意向锁是将锁定的对象分为多个层次，意向锁意味着事务希望在更细粒度上进行加锁。

若将上锁的对象看成一棵树，那么对最下层的对象上锁，也就是对最细粒度的对象进行上锁，那么首先需要对粗粒度的对象上锁。如果需要对页上的记录 r 进行上 X 锁，那么分别需要对数据库 A、表、页上意向锁 IX，最后对记录 r 上 X 锁。若其中任何一个部分导致等待，那么该操作需要等待粗粒度锁的完成。

在一个对象加锁之前，该对象的全部祖先节点均加上了意向锁。希望给某个记录加锁的事务必须遍历从根到记录的路径。在遍历树的过程中，该事务给各节点加上意向锁。

<img width="600" alt="粒度层次图" title="粒度层次图" src="https://static.nullwy.me/db-lock-hierarchy.png">

举例来说，假设在表 1 的记录 r 上持有 X 锁，表 1 上必定持有 IX 锁。如果其他事务想在表 1 上加 S 表锁或 X 表锁，但与已有 IX 锁不兼容，所以该事务需要等待。再举例，假设表 1  持有 S 锁，如果其他事务想在表 1 的记录 r 上加 X 锁，需要先获得表 1 的 IX 锁，但与已有 S 锁不兼容，所以该事务需要等待。有了意向锁之后，就能快速判断行锁和表锁之间是否兼容。

InnoDB 存储引擎支持意向锁设计比较简练，其意向锁即为**表级别的锁**，两种意向锁 [[doc](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html#innodb-intention-locks)]：

- 意向共享锁（IS）：事务打算给数据行加行共享锁（S），事务在给一个数据行加共享锁（S）前必须先取得该表的 IS 锁。
- 意向排他锁（IX）：事务打算给数据行加行排他锁（X），事务在给一个数据行加排他锁（X）前必须先取得该表的 IX 锁。

**IS、IX、S、X 锁的兼容性：**

| 待申请 \\ 已持有 | **IS** | **IX** | **S** | **X** |
| --- | --- | --- | --- | --- |
| **IS** | 兼容 | 兼容 | 兼容 | 冲突 |
| **IX** | 兼容 | 兼容 | 冲突 | 冲突 |
| **S** | 兼容 | 冲突 | 兼容 | 冲突 |
| **X** | 冲突 | 冲突 | 冲突 | 冲突 |

**兼容关系**：各种意向锁（IS、IX）之间全部兼容，意向共享锁 IS 和共享锁 S 兼容，共享锁 S 和共享锁 S 兼容，其他冲突。

SQL 语句可以分为数据定义语言（DDL）、数据控制语言（DCL）、数据查询语言（DQL）、数据操纵语言（DML）四种类型的语句，前两种语句，涉及的对象在数据之上，所以加锁的范围，通常是表级，对应表级锁。后两种语句操作的对象是数据，加锁的范围，通常是数据级，这就对应行级锁。

# 三种行锁：记录锁、间隙锁和 next-key 锁

InnoDB 行锁分为 3 种类型 [[doc](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)]：

- **记录锁（record lock）**：对索引记录项加锁。
- **间隙锁（gap lock）**：间隙锁，对索引记录项之间的“间隙”、笫一条记录前的“间隙”或最后一条记录后的“间隙“加锁。锁定一个范围，但不包含索引记录本身。
- **next-key 锁（next-key Lock）**：前两种锁的组合，记录锁 + 间隙锁，锁定一个范围，并且锁定索引记录本身。（中文世界有时将 next-key lock 翻译为“临键锁”）

如果索引上包含 10, 20, 30, 40, 50 这些记录，那么可能的 next-key 锁的锁区间（[interval](https://zh.wikipedia.org/wiki/%E5%8D%80%E9%96%93)），如下：

```
(-无穷, 10]     即，间隙锁 (-无穷, 10) + 记录锁 10。区间为，左开右闭区间
(10, 20]       即，间隙锁 (10, 20) + 记录锁 20
(20, 30]       即，间隙锁 (20, 30) + 记录锁 30
(30, 40]       即，间隙锁 (30, 40) + 记录锁 40
(40, 50]       即，间隙锁 (40, 50) + 记录锁 50
(50, +无穷]     即，间隙锁 (50, +无穷)
```

最后一个锁区间 `(50, +无穷]`，对应的是上界伪记录（`supremum pseudo-record`），不是真实存在的记录。这个锁区间用于防止在最大值 50 之后插入记录。

记录锁总是会去锁住索引记录，如果 InnoDB 存储引擎表在建立的时候没有设置任何一个索引，那么这时 InnoDB 存储引擎会使用隐式的主键来进行锁定。

MySQL 默认的事务隔离级别是可重复读（REPEATABLE-READ），**如果把事务隔离级别改成已提交读（READ-COMMITTED），间隙锁会被禁用**。禁用间隙锁后，幻读异常会出现，因为其他事务可以在间隙中插入新行。**InnoDB 的间隙锁，就是为了解决幻读异常而引入的**。关于幻读异常，参见官方文档 [doc](https://dev.mysql.com/doc/refman/8.0/en/innodb-next-key-locking.html)。

RR 隔离级别下，InnoDB 的锁通常使用 next-key 锁。但是，**在唯一索引（和主键索引）上的等值查询**，**next-key 锁退化为记录锁**，间隙锁并不需要，即仅锁住索引本身，而不是范围。**如果在唯一索引（和主键索引）上做范围查询，间隙锁依然需要**。官方文档描述如下 [[doc](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)]：

> Gap locking is not needed for statements that lock rows using a unique index to search for a unique row. (This does not include the case that the search condition includes only some columns of a multiple-column unique index; in that case, gap locking does occur.)

**间隙锁是“纯抑制性的”，间隙锁唯一的作用就是为了防止其他事务的插入到间隙中。间隙锁和间隙锁之间是互不冲突的，所以间隙共享 S 锁和间隙排他 X 锁没有任何区别。**

另外，还有一种锁叫**插入意向锁（insert intention lock）**，基于间隙锁，专门用于 insert 操作。在执行 insert 操作时，需要先申请获取插入意向锁，也就是说，需要先检查当前插入位置上的下一条记录上是否持有间隙锁，如果被间隙锁锁住，则锁冲突，插入被阻塞。多个事务做 insert 操作，被相同的间隙锁阻塞，如果插入的值各不相同，这些事务的 insert 操作之间不阻塞。

所以，间隙锁与插入意向锁的兼容关系是，已持有的间隙锁与待申请的插入意向锁冲突，而插入意向锁之间兼容，在一个间隙锁锁上可以有多个意向锁等待。

**IS、IX、X、S 锁和记录锁、间隙锁、next-key 锁的关系：**

- IS、IX、S、X 是锁模式（lock mode）（源码层面上对应 [lock_mode](https://github.com/mysql/mysql-server/blob/mysql-8.0.30/storage/innobase/include/lock0types.h#L51) 枚举）。
- 记录锁、间隙锁、next-key 锁、插入意向锁是行锁类型（record lock type）。
- 每一种行锁类型，都包含 IS、IX、S、X 锁模式，如共享的记录锁、排他的记录锁、共享的间隙录锁、排他的间隙锁等。

# 锁监控：data_locks 和 data_lock_waits 表

MySQL 8.0 之前，`information_schema` 库提供 `innodb_trx`、`innodb_locks` 和 `innodb_lock_waits` 三张表，用来监控事务和诊断潜在的锁问题，具体介绍可以参见官方 5.7 文档 [doc](https://dev.mysql.com/doc/refman/5.7/en/innodb-information-schema-transactions.html)。

- `innodb_trx`：当前事务表
- `innodb_locks`：锁等待中的锁信息表
- `innodb_lock_waits`：锁等待中的事务表

在 MySQL 8.0 之前，要想获得当前已持有的锁信息，需要开启参数 `innodb_status_output_locks` 并且执行命令 `show engine innodb status`，具体介绍可以参见官方文档“15.17 InnoDB Monitors”，[doc](https://dev.mysql.com/doc/refman/8.0/en/innodb-monitors.html)。

MySQL 8.0 开始，`innodb_locks` 表和 `innodb_lock_waits` 表，被 `performance_schema` 库的 `data_locks` 表和 `data_lock_waits` 表替代。其中值得注意的不同点是，新的 `data_locks` 表，同时包含了已持有的锁和请求中的锁的信息，这样查看当前已持有的锁信息更加方便。相关 SQL 示例：

```sql
-- 查询全部锁信息
select * from performance_schema.data_locks \G
-- 查询全部记录锁的锁信息
select * from performance_schema.data_locks where LOCK_TYPE = 'RECORD' \G
-- 查询等待中的锁信息
select * from performance_schema.data_locks where LOCK_STATUS = 'WAITING' \G
-- 查询锁等待中的事务
select * from performance_schema.data_lock_waits \G
-- 使用 sys 库的 innodb_lock_waits 视图
-- 查询锁等待中的事务
select * from sys.innodb_lock_waits \G
```

命令 `show engine innodb status` 的输出和 `data_locks` 表的对应关系，可以参考文章 [link](https://hackmysql.com/post/mysql-data-locks-mapping-80-to-57/)。

# 行锁加锁案例分析

## RR 隔离级别

本文的全部案例采用的 MySQL 版本为 8.0.30。MySQL 的默认事务隔离级别是 `REPEATABLE-READ`（可重复读），事务隔离级别可以通过系统变量 [transaction_isolation](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_transaction_isolation) 控制。

```sql
-- 事务隔离级别，默认为可重复读（Repeatable Read）
mysql> select @@global.transaction_isolation, @@transaction_isolation;
+--------------------------------+-------------------------+
| @@global.transaction_isolation | @@transaction_isolation |
+--------------------------------+-------------------------+
| REPEATABLE-READ                | REPEATABLE-READ         |
+--------------------------------+-------------------------+
1 row in set (0.00 sec)
```

`tbl` 表的数据如下：

```sql
mysql> select * from tbl;
+-----+------+------+------+
| a   | b    | c    | d    |
+-----+------+------+------+
|  10 |   10 |   10 |   10 |
|  20 |   20 |   20 |   20 |
|  30 |   30 |   30 |   30 |
|  40 |   40 |   40 |   40 |
|  50 |   50 |   50 |   50 |
|  60 |   60 |   60 |   60 |
|  70 |   70 |   70 |   70 |
|  80 |   80 |   80 |   80 |
|  90 |   90 |   90 |   90 |
| 100 |  100 |  100 |  100 |
+-----+------+------+------+
10 rows in set (0.00 sec)
```

### (a1) 主键索引上的等值查询

**SQL 语句：**

```sql
select * from tbl where a = 10 for update;
```

`data_locks` 表中的行锁数据：

```sql
mysql> select * from performance_schema.data_locks where LOCK_TYPE = 'RECORD' \G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 4816436360:185:4:3:140408965390368
ENGINE_TRANSACTION_ID: 56664
            THREAD_ID: 367
             EVENT_ID: 22
        OBJECT_SCHEMA: testdb
          OBJECT_NAME: tbl
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140408965390368
            LOCK_TYPE: RECORD
            LOCK_MODE: X,REC_NOT_GAP
          LOCK_STATUS: GRANTED
            LOCK_DATA: 10
```

**加锁情况：**

- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）

**其他 SQL 语句的加锁情况（通过查 `data_locks` 表确认）：**

```sql
-- 在 a = 10 的索引记录上添加共享记录锁（S,REC_NOT_GAP）
select * from tbl where a = 10 for share;

-- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
update tbl set b = 42 where a = 10;

-- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
delete from tbl where a = 10;
```

加锁与锁冲突 SQL 演示：

| 事务1 | 事务2 |
|---|---|
| mysql> begin; | mysql> begin; |
| -- 在 a = 10 的索引记录上添加排他记录锁<br>mysql> select * from tbl where a = 10 for update; |  |
|  | -- 阻塞，因为 a = 10 上存在排他记录锁<br>mysql> select * from tbl where a = 10 for update;<br>-- 阻塞，因为 a = 10 上存在排他记录锁<br>mysql> insert into tbl (a) values (10); |
|  | -- 插入成功<br>mysql> insert into tbl (a) values (9);<br>-- 插入成功<br>mysql> insert into tbl (a) values (11); |
| mysql> rollback; | mysql> rollback; |

### (a2) 唯一索引上的等值查询

**SQL 语句的加锁情况（通过查 `data_locks` 表确认）：**

```sql
-- 在 b = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
select * from tbl where b = 10 for update;

-- 在 b = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 覆盖索引，但系统会认为接下来要更新数据，因此会顺便给主键索引上满足条件的行加上行锁
select a from tbl where b = 10 for update;

-- 在 b = 10 的索引记录上添加共享记录锁（S,REC_NOT_GAP）
-- 在 a = 10 的索引记录上添加共享记录锁（S,REC_NOT_GAP）
select * from tbl where b = 10 for share;

-- 在 b = 10 的索引记录上添加共享记录锁（S,REC_NOT_GAP）
-- 覆盖索引，所以只在字段 b 上加锁
select a from tbl where b = 10 for share;

-- 在 b = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
update tbl set b = 42 where b = 10;

-- 在 b = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
delete from tbl where b = 10;
```

上面的全部 SQL，除了走覆盖索引的 `select for share` 外，其他的加锁范围都相同。

### (a3) 非唯一索引上的等值查询

**SQL 语句的加锁情况（通过查 `data_locks` 表确认）：**

```sql
-- 在 c = 10 的索引记录上添加排他 next-key 锁，区间为 (-无穷, 10]（X）
-- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 c = 20 的索引记录上添加排他间隙锁，区间为 (10, 20)（X,GAP）
select * from tbl where c = 10 for update;

-- 在 c = 10 的索引记录上添加共享 next-key 锁，区间为 (-无穷, 10]（S）
-- 在 a = 10 的索引记录上添加共享记录锁（S,REC_NOT_GAP）
-- 在 c = 20 的索引记录上添加共享间隙锁，区间为 (10, 20)（S,GAP）
select * from tbl where c = 10 for share;

-- 在 c = 10 的索引记录上添加共享 next-key 锁，区间为 (-无穷, 10]（S）
-- 在 c = 20 的索引记录上添加共享间隙锁，区间为 (10, 20)（S,GAP）
-- 覆盖索引，所以只在字段 c 上加锁
select a from tbl where c = 10 for share;

-- 在 c = 10 的索引记录上添加排他 next-key 锁，区间为 (-无穷, 10]（X）
-- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 c = 20 的索引记录上添加排他间隙锁，区间为 (10, 20)（X,GAP）
update tbl set c = 42 where c = 10;

-- 在 c = 10 的索引记录上添加排他 next-key 锁，区间为 (-无穷, 10]（X）
-- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 c = 20 的索引记录上添加排他间隙锁，区间为 (10, 20)（X,GAP）
delete from tbl where c = 10;
```

上面的全部 SQL，除了走覆盖索引的 `select for share` 外，其他的加锁范围都相同。

加锁与锁冲突 SQL 演示：

| 事务1 | 事务2 |
|---|---|
| mysql> begin; | mysql> begin; |
| -- 在 c = 10 的索引记录上添加排他 next-key 锁，区间为 (-无穷, 10]<br>-- 在 a = 10 的索引记录上添加排他记录锁<br>-- 在 c = 20 的索引记录上添加排他间隙锁，区间为 (10, 20)<br>mysql> select * from tbl where c = 10 for update; |  |
|  | -- 阻塞，因为 c = 10 上存在排他 next-key 锁<br>mysql> select * from tbl where c = 10 for update;<br>-- 阻塞，因为 a = 10 上存在排他记录锁<br>mysql> select * from tbl where a = 10 for update; |
|  | -- 阻塞，因为 c = 10 上存在排他 next-key 锁，区间为 (-无穷, 10]<br>mysql> insert into tbl (a, c) values (1, 9);<br>-- 阻塞，因为 c = 10 上存在排他 next-key 锁，区间为 (-无穷, 10]<br>mysql> insert into tbl (a, c) values (1, 10); |
|  | -- 阻塞，因为 c = (10, 20) 区间存在间隙锁<br>mysql> insert into tbl (a, c) values (1, 11);<br>-- 插入成功<br>mysql> insert into tbl (a, c) values (1, 21); |
| mysql> rollback; | mysql> rollback; |


### (a4) 无索引的等值查询

**SQL 语句的加锁情况（通过查 `data_locks` 表确认）：**

```sql
-- 在 a 主键的全部索引记录上添加排他 next-key 锁
select * from tbl where d = 10 for update;
```

因为字段 d 上没有索引，这个 SQL 语句，只能在聚簇索引上全表扫描。加锁情况，在 a 主键的全部索引记录上添加排他 next-key 锁。表 `tbl` 共 10 条记录，全部的持有的 next-key 锁的锁区间，如下：

```
(-无穷, 10]
  (10, 20]
  (20, 30]
  (30, 40]
  (40, 50]
  (50, 60]
  (60, 70]
  (70, 80]
  (80, 90]
  (90, 100]
(100, +无穷]
```

### (a5) 值不存在的等值查询

**SQL 语句的加锁情况（通过查 `data_locks` 表确认）：**

```sql
---- 主键索引上的值不存在的等值查询
-- 在 a = 100 的索引记录上添加排他间隙锁，区间为 (90, 100)（X,GAP）
select * from tbl where a = 95 for update;
-- 在 a 的索引记录上添加排他 next-key 锁，区间为 (100, +无穷]（X）
select * from tbl where a = 105 for update;

---- 唯一索引上的值不存在的等值查询
-- 在 b = 100 的索引记录上添加间隙锁，区间为 (90, 100)（X,GAP）
select * from tbl where b = 95 for update;
-- 在 b 的索引记录上添加排他 next-key 锁，区间为 (100, +无穷]（X）
select * from tbl where b = 105 for update;

---- 非唯一索引上的值不存在的等值查询
-- 在 c = 100 的索引记录上添加间隙锁，区间为 (90, 100)
select * from tbl where c = 95 for update;
-- 在 c 的索引记录上添加排他 next-key 锁，区间为 (100, +无穷]（X）
select * from tbl where c = 105 for update;
```

### (b1) 主键索引上的范围查询

**SQL 语句的加锁情况（通过查 `data_locks` 表确认）：**

```sql
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 100 的索引记录上添加排他 next-key 锁，区间为 (90, 100]（X）
-- 在 a 的索引记录上添加排他 next-key 锁，区间为 (100, +无穷]（X）
select * from tbl where a >= 90 for update;

-- 在 a = 100 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a 的索引记录上添加排他 next-key 锁，区间为 (100, +无穷]（X）
select * from tbl where a >= 100 for update;

-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 100 的索引记录上添加排他间隙锁，区间为 (90, 100)（X,GAP）
-- 附注：与主键上的等值查询 `a = 90` 的加锁范围的区别是额外加了区间为 (90, 100) 间隙锁
select * from tbl where a >= 90 and a < 91 for update;

-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 100 的索引记录上添加排他间隙锁，区间为 (90, 100)（X,GAP）
-- 附注：与相同查询条件的 `select for update`，加锁范围相同
update tbl set d = 42 where a >= 90 and a < 91;

-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 100 的索引记录上添加排他间隙锁，区间为 (90, 100)（X,GAP）
-- 附注：与相同查询条件的 `select for update` 的 SQL，加锁范围相同
delete from tbl where a >= 90 and a < 91;
```

### (b2) 唯一索引上的范围查询

**SQL 语句的加锁情况（通过查 `data_locks` 表确认）：**

```sql
-- 在 b = 90 的索引记录上添加排他 next-key 锁，区间为 (80, 90]（X）
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 b = 100 的索引记录上添加排他 next-key 锁，区间为 (90, 100]（X）
-- 在 a = 100 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 b 的索引记录上添加排他 next-key 锁，区间为 (100, +无穷]（X）
select * from tbl where b >= 90 for update;

-- 在 b = 90 的索引记录上添加排他 next-key 锁，区间为 (80, 90]（X）
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 b = 100 的索引记录上添加排他 next-key 锁，区间为 (90, 100]（X）（不必要的记录锁）
select * from tbl where b >= 90 and b < 91 for update;

-- 在 b = 90 的索引记录上添加排他 next-key 锁，区间为 (80, 90]（X）
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 b = 100 的索引记录上添加排他 next-key 锁，区间为 (90, 100]（X）（不必要的记录锁）
-- 在 a = 100 的索引记录上添加排他记录锁（X,REC_NOT_GAP）（不必要的记录锁）
update tbl set d = 42 where b >= 90 and b < 91;

-- 在 b = 90 的索引记录上添加排他 next-key 锁，区间为 (80, 90]（X）
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 b = 100 的索引记录上添加排他 next-key 锁，区间为 (90, 100]（X）（不必要的记录锁）
-- 在 a = 100 的索引记录上添加排他记录锁（X,REC_NOT_GAP）（不必要的记录锁）
delete from tbl where b >= 90 and b < 91;
```

加锁与锁冲突 SQL 演示：

| 事务1 | 事务2 |
|---|---|
| mysql> begin; | mysql> begin; |
| -- 在 b = 90 的索引记录上添加排他 next-key 锁，区间为 (80, 90]<br>-- 在 a = 90 的索引记录上添加排他记录锁<br>-- 在 b = 100 的索引记录上添加排他 next-key 锁，区间为 (90, 100]<br>mysql> select * from tbl where b >= 90 and b < 91 for update; |  |
|  | -- 阻塞，因为 b = 90 上存在排他 next-key 锁<br>mysql> select * from tbl where b = 90 for update;<br>-- 阻塞，因为 b = 100 上存在排他 next-key 锁（不必要的记录锁）<br>mysql> select * from tbl where b = 100 for update;<br>-- 阻塞，因为 a = 90 上存在排他记录锁<br>mysql> select * from tbl where a = 90 for update; |
| mysql> rollback; | mysql> rollback; |

### (b3) 非唯一索引上的范围查询

**SQL 语句的加锁情况（通过查 `data_locks` 表确认）：**

```sql
-- 在 b = 90 的索引记录上添加排他 next-key 锁，区间为 (80, 90]（X）
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 b = 100 的索引记录上添加排他 next-key 锁，区间为 (90, 100]（X）
-- 在 a = 100 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 b 的索引记录上添加排他 next-key 锁，区间为 (100, +无穷]（X）
select * from tbl where c >= 90 for update;

-- 在 c = 90 的索引记录上添加排他 next-key 锁，区间为 (80, 90]（X）
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 c = 100 的索引记录上添加排他 next-key 锁，区间为 (90, 100]（X）（不必要的记录锁）
select * from tbl where c >= 90 and c < 91 for update;

-- 在 b = 90 的索引记录上添加排他 next-key 锁，区间为 (80, 90]（X）
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 b = 100 的索引记录上添加排他 next-key 锁，区间为 (90, 100]（X）（不必要的记录锁）
-- 在 a = 100 的索引记录上添加排他记录锁（X,REC_NOT_GAP）（不必要的记录锁）
update tbl set d = 42 where c >= 90 and c < 91;

-- 在 b = 90 的索引记录上添加排他 next-key 锁，区间为 (80, 90]（X）
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 b = 100 的索引记录上添加排他 next-key 锁，区间为 (90, 100]（X）（不必要的记录锁）
-- 在 a = 100 的索引记录上添加排他记录锁（X,REC_NOT_GAP）（不必要的记录锁）
delete from tbl where c >= 90 and c < 91;
```

对比，唯一索引上的范围查询的加锁情况，容易得出结论，**唯一索引和普通索引上的范围查询的加锁规则相同**。

## RC 隔离级别

把事务隔离级别修改为已提交读（Read Committed）：

```sql
-- 事务隔离级别，修改为已提交读（Read Committed）
mysql> set @@transaction_isolation = 'READ-COMMITTED';
```

### (a1) 主键索引上的等值查询

**SQL 语句的加锁情况（通过查 `data_locks` 表确认）：**

```sql
-- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
select * from tbl where a = 10 for update;

-- 在 a = 10 的索引记录上添加共享记录锁（S,REC_NOT_GAP）
select * from tbl where a = 10 for share;

-- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
update tbl set b = 42 where a = 10;

-- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
delete from tbl where a = 10;
```

结论：因为主键索引上的等值查询不涉及间隙锁，所以 RR 和 RC 隔离级别下的加锁规则相同。

### (a2) 唯一索引上的等值查询

同样的，因为唯一索引上的等值查询不涉及间隙锁，所以 RR 和 RC 隔离级别下的加锁规则相同。

### (a3) 非唯一索引上的等值查询

```sql
-- 在 c = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
select * from tbl where c = 10 for update;

-- 在 c = 10 的索引记录上添加排他记录锁（S,REC_NOT_GAP）
-- 在 a = 10 的索引记录上添加排他记录锁（S,REC_NOT_GAP）
select * from tbl where c = 10 for share;

-- 在 c = 10 的索引记录上添加排他记录锁（S,REC_NOT_GAP）
-- 覆盖索引，所以只在字段 b 上加锁
select a from tbl where c = 10 for share;

-- 在 c = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
update tbl set c = 42 where c = 10;

-- 在 c = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
delete from tbl where c = 10;
```

### (a4) 无索引的等值查询

```sql
-- 在 a = 10 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
select * from tbl where d = 10 for update;
```

### (a5) 值不存在的等值查询

```sql
---- 主键索引上的值不存在的等值查询
-- 无锁
select * from tbl where a = 95 for update;
-- 无锁
select * from tbl where a = 105 for update;

---- 唯一索引上的值不存在的等值查询
-- 无锁
select * from tbl where b = 95 for update;
-- 无锁
select * from tbl where b = 105 for update;

---- 非唯一索引上的值不存在的等值查询
-- 无锁
select * from tbl where c = 95 for update;
-- 无锁
select * from tbl where c = 105 for update;
```

### (b1) 主键索引上的范围查询

```sql
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 100 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
select * from tbl where a >= 90 for update;

-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
select * from tbl where a >= 90 and a < 91 for update;

-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
update tbl set d = 42 where a >= 90 and a < 91;

-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
delete from tbl where a >= 90 and a < 91;
```

### (b2) 唯一索引上的范围查询

```sql
-- 在 b = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 b = 100 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 100 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
select * from tbl where b >= 90 for update;

-- 在 b = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
select * from tbl where b >= 90 and b < 91 for update;

-- 在 b = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
update tbl set d = 42 where b >= 90 and b < 91;

-- 在 b = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
delete from tbl where b >= 90 and b < 91;
```

### (b3) 非唯一索引上的范围查询

加锁情况，和 RC 隔离级别的唯一索引上的范围查询完全相同。

# 行锁加锁规则总结

**RC 隔离级别时的加锁规则：**

- 间隙锁被禁用，只有记录锁，没有间隙锁和 next-key 锁。
- 对全部满足查询条件的索引记录加记录锁。如果查询条件满足覆盖索引，就只对辅助索引加记录锁。如果需要回表，对辅助索引记录和聚簇索引记录引同时加锁。
- 如果不存在满足查询条件的索引记录，就不加锁。

**RR 隔离级别时的加锁规则：**

- 加锁的基本单位是 next-key 锁。
- 对全部满足查询条件的索引记录加 next-key 锁。如果查询条件满足覆盖索引，就只对辅助索引加记录锁。如果需要回表，对辅助索引记录和聚簇索引记录引同时加锁。
- 范围查询时，或值不存在的等值查询时，在从右扫描到的最后的不满足查询条件的记录上加**间隙锁**。如果索引的最大记录值，满足查询条件，则在上界伪记录（supremum pseudo-record）上加 next-key 锁（相当于间隙锁）。
- 等值查询时，在主键索引和唯一索引上加锁，next-key 锁退化为记录锁。

## 范围查询时的不必要加锁 bug
注意，RR 隔离级别时，在主键索引上的范围查询时，确实是按上文的规则加**间隙锁**。但实际验证发现，在辅助索引（包括唯一索引和普通索引）上的范围查询时，在最后的不满足查询条件的记录上实际加的是 **next-key 锁**。这样加锁的问题是，会在不满足查询条件的记录上记录锁，这个记录锁其实是不必要的，是一个 bug。

其实，这个不必要的记录锁 bug，在 MySQL 8.0.18 之前，主键索引的场景下也存在，MySQL 8.0.18 修复了，但只修复了主键索引的场景，辅助索引的场景未修复。修复对应 bug 为“Bug #29508068 UNNECESSARY NEXT-KEY LOCK TAKEN”，修复提交记录见 [github](https://github.com/mysql/mysql-server/commit/d1b0afd75ee669f54b70794eb6dab6c121f1f179)。

在 MySQL 8.0.19 版本上，有人再次提了 bug，“Bug #[98639](https://bugs.mysql.com/bug.php?id=98639) Redundant row-level locking for secondary index”。不过 MySQL 官方认为“Not a Bug”。然后，提 bug 的人，也只好妥协认为这个是“performance issue”。

对比下面这 3 个 SQL 的加锁情况，可以发现后 2 个 SQL 存在不必要的加锁问题。

```sql
-- 主键索引上的范围查询
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 a = 100 的索引记录上添加排他间隙锁，区间为 (90, 100)（X,GAP）
select * from tbl where a >= 90 and a < 91 for update;

-- 唯一索引上的范围查询
-- 在 b = 90 的索引记录上添加排他 next-key 锁，区间为 (80, 90]（X）
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 b = 100 的索引记录上添加排他 next-key 锁，区间为 (90, 100]（X）（不必要的记录锁）
select * from tbl where b >= 90 and b < 91 for update;

-- 非唯一索引上的范围查询
-- 在 c = 90 的索引记录上添加排他 next-key 锁，区间为 (80, 90]（X）
-- 在 a = 90 的索引记录上添加排他记录锁（X,REC_NOT_GAP）
-- 在 c = 100 的索引记录上添加排他 next-key 锁，区间为 (90, 100]（X）（不必要的记录锁）
select * from tbl where c >= 90 and c < 91 for update;
```

# 隔离级别的实现

上文提到，PostgreSQL 的并发控制技术是以 MVCC 技术为主，封锁技术为辅。先看下 PostgreSQL 对隔离级别的实现 [[doc](https://www.postgresql.org/docs/15/transaction-iso.html)]：

- PostgreSQL 支持 SQL 标准的 4 种隔离级别，默认的隔离级别是 RC。但 PostgreSQL 内部只实现 3 种隔离级别 RC、RR 和 SER。若选择 RU 隔离，实际上是 RC。
- PostgreSQL 的 RR 隔离级别，底层是基于 MVCC 技术实现的快照隔离（SI，[Snapshot Isolation](https://en.wikipedia.org/wiki/Snapshot_isolation)）。快照隔离下，能避免 SQL-92 定义的三种异常，脏读、不可重复读和幻读异常，但是可能会出现写偏序（Write Skew）异常。
- PostgreSQL 的 SER 隔离，底层是可串行化的快照隔离（SSI，Serializable Snapshot Isolation）。

InnoDB 的并发控制以封锁技术为主，MVCC 技术辅助，各个隔离级别的具体实现是：

- RC 隔离级别：快照读 + 写长锁
   - 快照读，能避免脏读
- RR 隔离级别：快照读 + 写长锁 + 间隙锁（没有实现真正的快照隔离 SI）
   - 快照读，能一定程度避免不可重复读和幻读异常，但因为 InnoDB 的刷新快照的特殊实现，不能完全避免
   - 间隙锁，能避免幻读异常，只有锁定读时才会申请获取间隙锁
- 可串行化隔离级别：读长锁 + 写长锁 + 间隙锁
   - 完全基于锁实现串行化，并发度很低，性能不好

InnoDB 实现的 MVCC 技术，能让事务以**快照读**的方式执行查询。**快照读**（[snapshot read](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_consistent_read)），或者叫一致性非锁定读（consistent nonlocking read），或者一致性读（consistent read），即使用多版本技术实现的读取数据在某个时间点的快照的查询。在 RR 和 RC 隔离级别下，一致性读是普通的 `select` 语句的默认模式。快照读**避免加锁**，从而提高并发度。在 RR 和 RC 隔离级别下快照读的区别：

- RR 隔离级别时，事务中的所有一致性读都会读取该事务中第一次此类读取建立的快照。
- RC 隔离级别时，事务中的每个一致性读都会设置并读取其自己的最新快照，快照是最新已提交的数据。

如果事务在查询数据后，要对该数据做修改操作，快照读无法提供足够的保护，因为其他事务可以对这些数据做修改操作。为了提供额外的保护，InnoDB 提供**锁定读**（[locking read](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_locking_read)），即同时执行锁定操作的 `select` 语句，锁持有直到事务结束。锁定读分两种：

- `select ... for share` 是加共享锁的查询数据
- `select ... for update` 是加排他锁的查询数据

## RR 隔离级别下的不可重复读和幻读异常

上文提到，“快照读，能一定程度避免不可重复读和幻读异常，但因为 InnoDB 的刷新快照的特殊实现，不能完全避免”。现在来看下 RR 隔离级别下的不可重复读异常的示例：

| 事务1 | 事务2 |
|---|---|
| mysql> begin; | mysql> begin; |
| -- 返回值为 10，快照读<br>mysql> select b from tbl where a = 10; |  |
|  | -- 把 b 值修改为 0<br>mysql> update tbl set b = 0 where a = 10;<br>mysql> commit; |
| -- 返回值为 10，没有出现不可重复读异常<br>mysql> select b from tbl where a = 10; |  |
| -- update 会读取 a = 10 的已提交的最新值<br>-- 同时 a = 10 记录的快照会被刷新<br>mysql> update tbl set b = b + 1 where a = 10; |  |
| -- 返回值为 1，出现不可重复读异常<br>mysql> select b from tbl where a = 10; |  |

这个问题在 MySQL 的 Bug 系统中可以找到，参见：Bug #[57973](https://bugs.mysql.com/bug.php?id=57973)、Bug #[63870](https://bugs.mysql.com/bug.php?id=63870) 等。官方认为，这不是 Bug，InnoDB 就是按这种方式设计。Bug #57973 下 MySQL 工程师 Kevin Lewis 对这个问题的解答 [[ref](https://bugs.mysql.com/bug.php?id=57973#c403965)]：

> **[16 Aug 2013 19:23] Kevin Lewis**
> Rejecting this bug because InnoDB is working as designed for the following reason;
> ...
> But when InnoDB Repeatable Read transactions modify the database, it is possible to get phantom reads added into the static view of the database, just as the ANSI description allows.  Moreover, **InnoDB relaxes the ANSI description for Repeatable Read isolation in that it will also allow non-repeatable reads during an UPDATE or DELETE. Specifically, it will write to newly committed records within its read view**.  And because of gap locking, it will actually wait on other transactions that have pending records that may become committed within its read view.  So not only is an UPDATE or DELETE affected by pending or newly committed records that satisfy the predicate, but also 'SELECT … LOCK IN SHARE MODE' and 'SELECT … FOR UPDATE'.
> This WRITE COMMITTED implementation of REPEATABLE READ is not typical of any other database that I am aware of.  But it has some real advantages over a standard 'Snapshot' isolation.  When an update conflict would occur in other database engines that implement a snapshot isolation for Repeatable Read, an error message would typically say that you need to restart your transaction in order to see the current data. So the normal activity would be to restart the entire transaction and do the same changes over again.  But InnoDB allows you to just keep going with the current transaction by waiting on other records which might join your view of the data and including them on the fly when the UPDATE or DELETE is done.  This WRITE COMMITTED implementation combined with implicit record and gap locking actually adds a serializable component to Repeatable Read isolation.

就是说，InnoDB 实现的 RR 隔离级别，放松了 SQL 标准对 RR 隔离级别的要求。事务 T1 在快照读后，如果其他事务 T2 修改了快照对应的记录并提交，之后事务 T1 执行涉及快照的 DML 语句（update、delete、insert）或锁定读，会触发快照刷新，事务 T2 最新提交的修改会刷新进快照。最终导致事务 T1 再次执行相同条件的快照读，读取结果不同，出现不可重复读或幻读异常。**简单概括就是，在快照失效后，又刷新快照，导致两次读到的快照不同。另外，如果实现上选择不刷新快照，并且事务 T1 正常执行，会出现 P4 丢失更新异常。**

**不可重复读异常的避免（一定程度上避免，但没有完全避免）：**

- 如果事务重复的两次读都是快照读（普通 `select` 语句），并且中间没有执行涉及快照的 DML 或锁定读，这样两次读到的是相同的快照读，所以不会出现不可重复读异常。
- 如果事务重复的两次读都是当前读（`select for update/share`），因为第一次加锁，其他事务无法更新该记录，所以也不会出现不可重复读异常。
- 如果事务重复的两次读都是快照读，但是中间执行涉及快照的 DML 或锁定读，触发了快照刷新，如果快照被更新，就会出现不可重复读异常。

**幻读异常的避免（一定程度上避免，但没有完全避免）：**

- 如果事务重复的两次读都是快照读（普通 `select` 语句），并且中间没有执行涉及快照的 DML 或锁定读，这样两次读到的是相同的快照读，所以不会出现幻读异常。
- 如果事务重复的两次读都是当前读（`select for update/share`），因为第一次当前读加间隙锁，其他事务无法插入，被阻塞，所以也不会出现幻读异常。
- 如果事务重复的两次读都是快照读，但是中间执行涉及快照的 DML 或锁定读，触发了快照刷新，如果快照被更新，就会出现幻读异常。

上述的**快照失效**的场景，PostgreSQL 的处理方式是，事务会被回滚并报错提示，应用程序收到这个报错，可以尝试重试，重试的事务读到的快照是最新的，这样即避免丢失更新异常，也避免了幻读和不可重复读异常（参见官方文档 [doc](https://www.postgresql.org/docs/15/transaction-iso.html#XACT-REPEATABLE-READ)）。

# 参考资料

**MySQL 8.0 Reference Manual**

- 15.7 InnoDB Locking and Transaction Model [https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-transaction-model.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking-transaction-model.html)
   - 15.7.1 InnoDB Locking [https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
   - 15.7.2 InnoDB Transaction Model [https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-model.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-model.html)
   - 15.7.3 Locks Set by Different SQL Statements in InnoDB [https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html](https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html)
   - 15.7.4 Phantom Rows [https://dev.mysql.com/doc/refman/5.7/en/innodb-next-key-locking.html](https://dev.mysql.com/doc/refman/5.7/en/innodb-next-key-locking.html)
- 15.15 InnoDB INFORMATION_SCHEMA Tables
   - 15.15.2 InnoDB INFORMATION_SCHEMA Transaction and Locking Information [https://dev.mysql.com/doc/refman/8.0/en/innodb-information-schema-transactions.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-information-schema-transactions.html)
- 15.17 InnoDB Monitors [https://dev.mysql.com/doc/refman/8.0/en/innodb-monitors.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-monitors.html)

**其他参考资料：**

- Hal Berenson, [Philip A. Bernstein](https://en.wikipedia.org/wiki/Phil_Bernstein), [Jim Gray](https://en.wikipedia.org/wiki/Jim_Gray_%28computer_scientist%29), Jim Melton, Elizabeth J. O'Neil, Patrick E. O'Neil: **A Critique of ANSI SQL Isolation Levels**. SIGMOD Conference 1995: 1-10（隔离级别的经典论文，其中作者 Jim Gray 因“对数据库和事务处理研究的开创性贡献以及系统实现方面的技术领导力”而于 1998 年获得图灵奖）
- 数据库事务处理的艺术，腾讯李海翔 2017
- MySQL技术内幕：InnoDB存储引擎，姜承尧 第2版2013：第6章 锁
- 2013-12 何登成：MySQL 加锁处理分析 [https://web.archive.org/web/0/http://hedengcheng.com/?p=771](https://web.archive.org/web/0/http://hedengcheng.com/?p=771) [https://bit.ly/44rsCH7](https://bit.ly/44rsCH7)
- 2019-04 阿里王德浩/孟勃荣：开发者都应该了解的数据库隔离级别 [https://mp.weixin.qq.com/s/bFg8XFYd9HLvEoYyzAD3jg](https://mp.weixin.qq.com/s/bFg8XFYd9HLvEoYyzAD3jg)
- 2021-07 MySQL Data Locks: Mapping 8.0 to 5.7 [https://hackmysql.com/post/mysql-data-locks-mapping-80-to-57/](https://hackmysql.com/post/mysql-data-locks-mapping-80-to-57/)
- 2019-07 Bug #29508068 UNNECESSARY NEXT-KEY LOCK TAKEN (MySQL 8.0.18 发布) [https://github.com/mysql/mysql-server/commit/d1b0afd75ee669f54b70794eb6dab6c121f1f179](https://github.com/mysql/mysql-server/commit/d1b0afd75ee669f54b70794eb6dab6c121f1f179)
- 2020-02 Bug #98639 Redundant row-level locking for secondary index (8.0.19) [https://bugs.mysql.com/bug.php?id=98639](https://bugs.mysql.com/bug.php?id=98639)
- 2010-11 Bug #57973 UPDATE tries to lock rows not visible to read view [https://bugs.mysql.com/bug.php?id=57973#c403965](https://bugs.mysql.com/bug.php?id=57973#c403965)
- 2011-12 Bug #63870 Repeatable-read isolation violated in UPDATE (5.1.42, 5.5.20) [https://bugs.mysql.com/bug.php?id=63870](https://bugs.mysql.com/bug.php?id=63870)
- PostgreSQL Documentation: 13.2. Transaction Isolation <https://www.postgresql.org/docs/15/transaction-iso.html>

