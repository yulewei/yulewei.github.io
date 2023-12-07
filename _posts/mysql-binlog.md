---
title: MySQL binlog：格式、增量恢复、闪回、Java 解析
date: 2018-06-18 15:09:51
categories: MySQL
tags: [MySQL, binlog, 数据库]
---

MySQL 的 [binlog](https://dev.mysql.com/doc/refman/5.7/en/binary-log.html) 日志文件，记录了数据库表的全部修改操作。本文简单整理 MySQL binlog 相关知识，以及如何使用 binlog 恢复或闪回数据库数据。


# STATEMENT 格式的 binlog

要想开启 binlog，需要在启动 MySQL 时传入 --[log-bin](https://dev.mysql.com/doc/refman/5.7/en/replication-options-binary-log.html#option_mysqld_log-bin) 参数。或者也可以在 MySQL 配置文件 `/etc/my.cnf`，设置 `log_bin` 开启 binlog。MySQL 5.7 开始，开启 binlog 后，`--server-id` 参数也必须指定，否则 MySQL 服务器会启动失败。

<!--more-->

`binlog_format` 支持 `STATEMENT`, `ROW`, `MIXED` 三种格式，MySQL 5.5 和 5.6 默认为 `STATEMENT`，MySQL 5.7.7 开始默认为 `ROW`。若 SQL 使用 UUID(), RAND(), VERSION() 等函数，或者使用存储过程、自定义函数，基于 STATEMENT 的主从复时，是不安全的（很多人可能会认为 NOW(), CURRENT_TIMESTAMP 这些函数也是不安全的，事实上是安全的）[ [doc1](https://dev.mysql.com/doc/mysql-replication-excerpt/5.7/en/replication-sbr-rbr.html), [doc2](https://mariadb.com/kb/en/library/unsafe-statements-for-statement-based-replication/) ]。基于 `ROW` 的主从复制，是最安全的复制方式。

现在先来看下 `STATEMENT` 格式的 binlog，`/etc/my.cnf` 文件修改的内容如下：

```
server_id = 1
log_bin = mysql-bin
binlog_format = STATEMENT
binlog_row_image=FULL
```

重启 MySQL 后，在数据目录 [datadir](https://dev.mysql.com/doc/refman/5.7/en/server-options.html#option_mysqld_datadir) 下，比如 `/var/lib/mysql/`，将会生成相应的 binlog 文件，`mysql-bin.index` 和 `mysql-bin.000001`。`.index` 后缀的文件保存全部 binlog 文件名。`mysql-bin.000001` 文件记录 binlog 内容。每次 MySQL 启动或者 flush 日志，都将按序号创建一个新的日志文件。另外，当日志文件大小超过 `max_binlog_size` 时，也会创建一个新的日志文件。

现在来试一试 binlog 功能。假设在 `testdb` 库在有 `hello` 表，并对其中某行做修改操作：

``` sql
mysql> select * from hello;
+----+-------+
| id | name  |
+----+-------+
|  1 | Andy  |
|  2 | Bill  |
|  3 | Candy |
+----+-------+
4 rows in set (0.00 sec)

mysql> update hello set name = 'Will' where id = 3;
Query OK, 1 row affected (0.02 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

binlog 为二进制文件，需要使用 `mysqlbinlog`（[doc](https://dev.mysql.com/doc/refman/5.7/en/mysqlbinlog.html), [man](https://www.mankier.com/1/mysqlbinlog)）命令查看：

```
$ sudo mysqlbinlog /var/lib/mysql/mysql-bin.000001  # 直接在 mysql 服务器上读取 binlog 文件
$ mysqlbinlog -R -h192.168.2.107 -uroot -p123456 mysql-bin.000001  # 或者，远程读取 binlog 文件
```

执行 `update` 后相应新增的 binlog 文件内容：

``` sql
# at 154
#180617 22:47:49 server id 1  end_log_pos 219 CRC32 0x4bd9d69b 	Anonymous_GTID	last_committed=0	sequence_number=1	rbr_only=no
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 219
#180617 22:47:49 server id 1  end_log_pos 302 CRC32 0x476fafc9 	Query	thread_id=2	exec_time=0	error_code=0
SET TIMESTAMP=1529246869/*!*/;
SET @@session.pseudo_thread_id=2/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1075838976/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=33/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
BEGIN
/*!*/;
# at 302
#180617 22:47:49 server id 1  end_log_pos 423 CRC32 0x7f2c2c7a 	Query	thread_id=2	exec_time=0	error_code=0
use `testdb`/*!*/;
SET TIMESTAMP=1529246869/*!*/;
update hello set name = 'Will' where id = 3
/*!*/;
# at 423
#180617 22:47:49 server id 1  end_log_pos 454 CRC32 0x68da744a 	Xid = 12
COMMIT/*!*/;
```

# ROW 格式的 binlog

修改 `/etc/my.cnf` 的 `binlog_format` 为 `ROW`，再重启 MySQL。格式修改后，会生成一个新的 binlog 文件 `mysql-bin.000002`。

``` sql
mysql> show create table hello;
+-------+-------------------------------------------------------------------------+
| Table | Create Table
+-------+-------------------------------------------------------------------------+
| hello | CREATE TABLE `hello` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(20) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8mb4 |
+-------+-------------------------------------------------------------------------+
1 row in set (0.00 sec)

mysql> select * from hello where id;
+----+------+
| id | name |
+----+------+
|  1 | Andy |
|  2 | Lily |
|  3 | Will |
+----+------+
1 row in set (0.00 sec)

mysql> update hello set name = 'David' where id = 3;
Query OK, 1 row affected (0.02 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```

查看 `ROW` 格式的 binlog，需要使用 `sudo mysqlbinlog -v --base64-output=DECODE-ROWS /var/lib/mysql/mysql-bin.000002` 命令。执行 `update` 后相应新增的 binlog 内容：

``` sql
# at 154
#180617 22:54:13 server id 1  end_log_pos 219 CRC32 0x2ce70d4d 	Anonymous_GTID	last_committed=0	sequence_number=1	rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 219
#180617 22:54:13 server id 1  end_log_pos 293 CRC32 0x8183fddf 	Query	thread_id=2	exec_time=0	error_code=0
SET TIMESTAMP=1529247253/*!*/;
SET @@session.pseudo_thread_id=2/*!*/;
SET @@session.foreign_key_checks=1, @@session.sql_auto_is_null=0, @@session.unique_checks=1, @@session.autocommit=1/*!*/;
SET @@session.sql_mode=1075838976/*!*/;
SET @@session.auto_increment_increment=1, @@session.auto_increment_offset=1/*!*/;
/*!\C utf8 *//*!*/;
SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=33/*!*/;
SET @@session.lc_time_names=0/*!*/;
SET @@session.collation_database=DEFAULT/*!*/;
BEGIN
/*!*/;
# at 293
#180617 22:54:13 server id 1  end_log_pos 346 CRC32 0x0fc7e1a4 	Table_map: `testdb`.`hello` mapped to number 110
# at 346
#180617 22:54:13 server id 1  end_log_pos 411 CRC32 0xb58e729d 	Update_rows: table id 110 flags: STMT_END_F
### UPDATE `testdb`.`hello`
### WHERE
###   @1=3
###   @2='Will'
### SET
###   @1=3
###   @2='David'
# at 411
#180617 22:54:13 server id 1  end_log_pos 442 CRC32 0xef964db8 	Xid = 13
COMMIT/*!*/;
```

若执行如下 SQL：

``` sql
mysql> insert hello (name) values ('Frank');
Query OK, 1 row affected (0.02 sec)
```

相应生成的 binlog 内容：

``` sql
# at 442
#180617 22:55:47 server id 1  end_log_pos 507 CRC32 0x79de08a7 	Anonymous_GTID	last_committed=1	sequence_number=2	rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 507
#180617 22:55:47 server id 1  end_log_pos 581 CRC32 0x56f9eb6a 	Query	thread_id=2	exec_time=0	error_code=0
SET TIMESTAMP=1529247347/*!*/;
BEGIN
/*!*/;
# at 581
#180617 22:55:47 server id 1  end_log_pos 634 CRC32 0xedb73620 	Table_map: `testdb`.`hello` mapped to number 110
# at 634
#180617 22:55:47 server id 1  end_log_pos 684 CRC32 0x525a6a70 	Write_rows: table id 110 flags: STMT_END_F
### INSERT INTO `testdb`.`hello`
### SET
###   @1=4
###   @2='Frank'
# at 684
#180617 22:55:47 server id 1  end_log_pos 715 CRC32 0x09a0d4de 	Xid = 14
COMMIT/*!*/;
```

若执行如下 SQL：

``` sql
mysql> delete from hello where id = 2;
Query OK, 1 row affected (0.02 sec)
```

相应生成的 binlog 内容：

``` sql
# at 715
#180617 22:56:44 server id 1  end_log_pos 780 CRC32 0x9f52450e 	Anonymous_GTID	last_committed=2	sequence_number=3	rbr_only=yes
/*!50718 SET TRANSACTION ISOLATION LEVEL READ COMMITTED*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 780
#180617 22:56:44 server id 1  end_log_pos 854 CRC32 0x0959bc8d 	Query	thread_id=2	exec_time=0	error_code=0
SET TIMESTAMP=1529247404/*!*/;
BEGIN
/*!*/;
# at 854
#180617 22:56:44 server id 1  end_log_pos 907 CRC32 0x2945260f 	Table_map: `testdb`.`hello` mapped to number 110
# at 907
#180617 22:56:44 server id 1  end_log_pos 956 CRC32 0xc70df255 	Delete_rows: table id 110 flags: STMT_END_F
### DELETE FROM `testdb`.`hello`
### WHERE
###   @1=2
###   @2='Bill'
# at 956
#180617 22:56:44 server id 1  end_log_pos 987 CRC32 0x0c98f18e 	Xid = 15
COMMIT/*!*/;
```

# 使用 binlog 增量恢复

MySQL 逻辑备份通常会结合**全量备份**和**增量备份**，使用 `mysqldump` 定期全量备份数据库，然后利用 binlog 保存增量数据。恢复数据时，就是用 `mysqldump` 备份的数据恢复到备份的时间点。数据库在备份时间点到当前时间的增量修改，则通过 `mysqlbinlog` 将 binlog 中的增量数据恢复到数据库。现在假设已经使用 `mysqldump` 将数据库还原到：

``` sql
mysql> select * from hello;
+----+------+
| id | name |
+----+------+
|  1 | Andy |
|  2 | Lily |
|  3 | Will |
+----+------+
3 rows in set (0.00 sec)
```

之后执行的 SQL：

``` sql
update hello set name = 'David' where id = 3;
insert hello (name) values ('Frank');
delete from hello where id = 2;
```

不管是使用 `STATEMENT` 还是 `ROW`，`mysqlbinlog` 命令都可以将 binlog 增量恢复到数据库 [ [doc](https://dev.mysql.com/doc/refman/5.7/en/point-in-time-recovery.html) ]。

观察 `binlog` 可以看到，从最开始的 `update hello set name = 'David' where id = 3;` 到最终的 `delete from hello where id = 2;`，时间上从 "2018-06-17 22:54:13" 到 "2018-06-17 22:56:44"，所以基于时间点恢复，命令如下：

```
$ sudo mysqlbinlog --start-datetime="2018-06-17 22:54:13" --stop-datetime="2018-06-17 22:56:44" mysql-bin.000002 | mysql -uroot -p123456
```

`binlog` 的事件位置号是从 "154" 到 "956"，但需要注意的是 用 `--start-position` 和 `--stop-position` 指定位置点范围，逻辑上对应的是 `start <= position < stop`，所以基于时间点恢复，命令如下：

```
$ sudo mysqlbinlog --start-position=154 --stop-position=957 mysql-bin.000002 | mysql -uroot -p123456
```

两种方式任意执行，都能将数据恢复到：

``` sql
mysql> select * from hello;
+----+-------+
| id | name  |
+----+-------+
|  1 | Andy  |
|  3 | David |
|  4 | Frank |
+----+-------+
3 rows in set (0.00 sec)
```

# 使用 binlog2sql 闪回

[binlog2sql](https://github.com/danfengcao/binlog2sql)，作者为曹单锋，大众点评 DBA。`binlog2sql`，从 MySQL binlog 解析出你要的 SQL。根据不同选项，你可以得到原始 SQL、回滚 SQL、去除主键的 INSERT SQL 等。`binlog2sql`，底层实现依赖 [python-mysql-replication](https://github.com/noplay/python-mysql-replication)，由该库完成 MySQL 复制协议和 binlog 格式的解析。

``` shell
$ python binlog2sql/binlog2sql.py -h192.168.2.107 -uroot -p123456 --start-position=154 --stop-position=957 --start-file='mysql-bin.000002'
UPDATE `testdb`.`hello` SET `id`=3, `name`='David' WHERE `id`=3 AND `name`='Will' LIMIT 1; #start 4 end 411 time 2018-06-17 22:54:13
INSERT INTO `testdb`.`hello`(`id`, `name`) VALUES (4, 'Frank'); #start 442 end 684 time 2018-06-17 22:55:47
DELETE FROM `testdb`.`hello` WHERE `id`=2 AND `name`='Bill' LIMIT 1; #start 715 end 956 time 2018-06-17 22:56:44
```

生成回滚 sql：

``` shell
$ python binlog2sql/binlog2sql.py --flashback -h192.168.2.107 -uroot -p123456 --start-position=154 --stop-position=956 --start-file='mysql-bin.000002'
INSERT INTO `testdb`.`hello`(`id`, `name`) VALUES (2, 'Bill'); #start 715 end 956 time 2018-06-17 22:56:44
DELETE FROM `testdb`.`hello` WHERE `id`=4 AND `name`='Frank' LIMIT 1; #start 442 end 684 time 2018-06-17 22:55:47
UPDATE `testdb`.`hello` SET `id`=3, `name`='Will' WHERE `id`=3 AND `name`='David' LIMIT 1; #start 154 end 411 time 2018-06-17 22:54:13
```

闪回的现实原理很简单，先通过 MySQL [复制协议](https://dev.mysql.com/doc/internals/en/replication-protocol.html)的 [com-binlog-dump](https://dev.mysql.com/doc/internals/en/com-binlog-dump.html) 命令 dump 出 binlog，然后按照 binlog 的[格式规范](https://dev.mysql.com/doc/internals/en/binary-log.html)解析 binlog，将 binlog 转换成 SQL，再将这些 SQL 转换反向逻辑的 SQL，最后再倒序执行。具体可以看，`binlog2sql` 作者的文章 [ [ref](https://github.com/danfengcao/binlog2sql/blob/master/example/mysql-flashback-priciple-and-practice.md) ]。


# Java 解析 binlog

上文中的 `binlog2sql` 其实底层依赖 `python-mysql-replication` 库，这是 Python 库。如果想使用 Java 解析 binlog 可以使用 `mysql-binlog-connector-java`（[github](https://github.com/shyiko/mysql-binlog-connector-java)）库。目前开源的 CDC 工具，如 Zendesk [maxwell](https://github.com/zendesk/maxwell)、Redhat [debezium](https://github.com/debezium/debezium)、LinkedIn [Databus](https://github.com/linkedin/databus) 等都底层依赖 `mysql-binlog-connector-java` 或者其前身 [open-replicator](https://github.com/zendesk/open-replicator)。使用 `mysql-binlog-connector-java` 的示例代码如下：

``` java
BinaryLogClient client = new BinaryLogClient("192.168.2.107", 3306, "root", "123456");
client.setBinlogFilename("mysql-bin.000001");
client.setBinlogPosition(4);
client.setBlocking(false);
client.registerEventListener(event -> {
    System.out.println(event);
});
client.connect();
```

输出（省略部分内容）：

``` java
...
Event{header=EventHeaderV4{timestamp=1529247253000, eventType=TABLE_MAP, serverId=1, headerLength=19, dataLength=34, nextPosition=346, flags=0}, data=TableMapEventData{tableId=110, database='testdb', table='hello', columnTypes=8, 15, columnMetadata=0, 40, columnNullability={1}}}
Event{header=EventHeaderV4{timestamp=1529247253000, eventType=EXT_UPDATE_ROWS, serverId=1, headerLength=19, dataLength=46, nextPosition=411, flags=0}, data=UpdateRowsEventData{tableId=110, includedColumnsBeforeUpdate={0, 1}, includedColumns={0, 1}, rows=[
    {before=[3, Will], after=[3, David]}
]}}
...
Event{header=EventHeaderV4{timestamp=1529247347000, eventType=TABLE_MAP, serverId=1, headerLength=19, dataLength=34, nextPosition=634, flags=0}, data=TableMapEventData{tableId=110, database='testdb', table='hello', columnTypes=8, 15, columnMetadata=0, 40, columnNullability={1}}}
Event{header=EventHeaderV4{timestamp=1529247347000, eventType=EXT_WRITE_ROWS, serverId=1, headerLength=19, dataLength=31, nextPosition=684, flags=0}, data=WriteRowsEventData{tableId=110, includedColumns={0, 1}, rows=[
    [4, Frank]
]}}
...
Event{header=EventHeaderV4{timestamp=1529247404000, eventType=TABLE_MAP, serverId=1, headerLength=19, dataLength=34, nextPosition=907, flags=0}, data=TableMapEventData{tableId=110, database='testdb', table='hello', columnTypes=8, 15, columnMetadata=0, 40, columnNullability={1}}}
Event{header=EventHeaderV4{timestamp=1529247404000, eventType=EXT_DELETE_ROWS, serverId=1, headerLength=19, dataLength=30, nextPosition=956, flags=0}, data=DeleteRowsEventData{tableId=110, includedColumns={0, 1}, rows=[
    [2, Bill]
]}}
```

# 参考资料

1. MySQL Replication: 5.1.1 Advantages and Disadvantages of Statement-Based and Row-Based Replication <https://dev.mysql.com/doc/mysql-replication-excerpt/5.7/en/replication-sbr-rbr.html>
2. Unsafe Statements for Statement-based Replication <https://mariadb.com/kb/en/library/unsafe-statements-for-statement-based-replication/>
3. MySQL 5.7 Reference Manual: 4.6.7 mysqlbinlog <https://dev.mysql.com/doc/refman/5.7/en/mysqlbinlog.html>
4. MySQL Internals Manual: 14.9 Replication Protocol <https://dev.mysql.com/doc/internals/en/replication-protocol.html>
5. MySQL Internals Manual: 20 The Binary Log <https://dev.mysql.com/doc/internals/en/binary-log.html>
6. MySQL闪回原理与实战 <https://github.com/danfengcao/binlog2sql/blob/master/example/mysql-flashback-priciple-and-practice.md>
7. <https://dev.mysql.com/doc/dev/mysql-server/8.0.11/classbinary__log_1_1Table__map__event.html#details>
8. <https://dev.mysql.com/doc/dev/mysql-server/8.0.11/classbinary__log_1_1Rows__event.html#details>

