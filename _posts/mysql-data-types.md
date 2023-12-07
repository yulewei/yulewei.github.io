---
title: MySQL数据库字段类型杂记
date: 2017-01-04 23:26:28 
categories: MySQL
tags: [MySQL, 数据库]
---

文本整理MySQL字段类型的常见问题和用法。

# 数值类型

## int和int(3)
新手在定义整数字段时，常常想当然通过，如`int(3)`，来限制整数的有效长度，然而这样仅仅只是指定了显示宽度。选择有效长度不同的整数，需要使用`tinyint`（1个字节）、`smallint`（2个字节）、`mediumint`（3个字节）、`int`（4个字节）或`bigint`（8个字节）。MySQL的相关文档如下[[doc](http://dev.mysql.com/doc/refman/5.7/en/numeric-type-attributes.html)]：

<!--more-->

> MySQL supports an extension for optionally specifying the *display width* of integer data types in parentheses following the base keyword for the type. For example, [INT(4)](http://dev.mysql.com/doc/refman/5.7/en/integer-types.html) specifies an [INT](http://dev.mysql.com/doc/refman/5.7/en/integer-types.html) with a display width of four digits.

> The display width does *not* constrain the range of values that can be stored in the column. Nor does it prevent values wider than the column display width from being displayed correctly. For example, a column specified as [SMALLINT(3)](http://dev.mysql.com/doc/refman/5.7/en/integer-types.html) has the usual [SMALLINT](http://dev.mysql.com/doc/refman/5.7/en/integer-types.html) range of `-32768` to `32767`, and values outside the range permitted by three digits are displayed in full using more than three digits.

> When used in conjunction with the optional (nonstandard) attribute ZEROFILL, the default padding of spaces is replaced with zeros. For example, for a column declared as [INT(4) ZEROFILL](http://dev.mysql.com/doc/refman/5.7/en/integer-types.html), a value of `5` is retrieved as `0005`.

如下示例，字段`bar`类型是`int(3)`，但依然能够正确保存数值`12345`：

```sql
mysql> create table test (foo int(3) zerofill, bar int(3) zerofill, baz int);
Query OK, 0 rows affected (0.16 sec)
 
mysql> insert into test values (42, 12345, 12345);
Query OK, 1 row affected (0.00 sec)
 
mysql> select * from test;
+------+-------+-------+
| foo  | bar   | baz   |
+------+-------+-------+
|  042 | 12345 | 12345 |
+------+-------+-------+
1 row in set (0.00 sec)
```

## bit和bool
MySQL同时支持`bit`和`bool`类型，但`bool`仅仅是`tinyint(1)`的同义词，创建的字段值的范围并不是`0`和`1`或`true`和`false`，而是`-128`到`127`。如下所示：

```sql
mysql> create table test (foo bool, bar bit);
Query OK, 0 rows affected (0.10 sec)
 
mysql> desc test;
+---------+------------+--------+-------+-----------+---------+
| Field   | Type       | Null   | Key   |   Default | Extra   |
|---------+------------+--------+-------+-----------+---------|
| foo     | tinyint(1) | YES    |       |    <null> |         |
| bar     | bit(1)     | YES    |       |    <null> |         |
+---------+------------+--------+-------+-----------+---------+
2 rows in set (0.00 sec)
```
 
# 字符串类型
## 长度和编码

从MySQL 4.1开始（2004年10月），用**字符**单位解释在字符列定义中的长度规范。(以前的一些MySQL版本以**字节**解释长度）。官方文档描述如下[[doc](https://docs.oracle.com/cd/E17952_01/mysql-5.0-en/string-type-overview.html)]
> In MySQL 4.1 and up , string data types include some features that you may not have encountered in working with versions of MySQL prior to 4.1: 

>    MySQL interprets length specifications in character column definitions in character units. (Before MySQL 4.1, column lengths were interpreted in bytes.) This applies to `CHAR`, `VARCHAR`, and the `TEXT` types. 

MySQL服务器默认的*字符集*是`latin1`，使用的*校对规则*是`latin1_swedish_ci` [[doc](http://dev.mysql.com/doc/refman/5.7/en/charset-server.html)]（*校对规则*是在字符集内用于比较字符的一套规则）。若要保存中文，典型的做法是使用`utf8`编码。但MySQL的`utf8`编码最多只能保存使用`utf8`编码后长度是3个字节的字符，即只支持[基本多文种平面](https://zh.wikipedia.org/wiki/Unicode%E5%AD%97%E7%AC%A6%E5%B9%B3%E9%9D%A2%E6%98%A0%E5%B0%84)。使用MySQL的utf8保存常见的字符基本上没有问题，但对于生僻字或[emoji](https://en.wikipedia.org/wiki/Emoji)字符就无能为力了。emoji的中的`笑脸（grinning face）`的Unicode编码，如下 [[ref1](http://www.iemoji.com/view/emoji/885/smileys-people/grinning-face)][[ref2](https://en.wikibooks.org/wiki/Unicode/Character_reference/1F000-1FFFF)]：

| 表情 | Unicode | UTF-16 | UTF8 |
| ---  | ---       | ---       | ---   |
| 😀   | U+1F604 | 0xD83D 0xDE04 | 0xF0 0x9F 0x98 0x84 |

emoji位于辅助多文种平面，`utf8`需要4个字节保存。为了解决这个问题，MySQL 5.5.3开始支持`utf8mb4`，支持辅助多文种平面，每个字符最大4个字节 [[doc](http://dev.mysql.com/doc/refman/5.5/en/charset-unicode-utf8mb4.html)]。除了`utf8`，MySQL还支持`ucs2`、`utf16`、`utf32`等，完整列表如下 [[ref](http://dev.mysql.com/doc/refman/5.5/en/charset-unicode.html)]

| 字符集    | 支持的字符     | 每个字符需要的存储空间  |
| ---       | ---           | ---                                    |
| `utf8`      | 基本多文种平面 | 1, 2, 或 3 个字节           |
| `ucs2`     | 基本多文种平面 | 2字节                           |
| `utf8mb4` | 基本多文种平面和辅助多文种平面 | 1, 2, 3,或4字节 |
| `utf16`   | 基本多文种平面和辅助多文种平面 | 2或4字节 |
| `utf16le` | 基本多文种平面和辅助多文种平面 | 2或4字节 |
| `utf32`    | 基本多文种平面和辅助多文种平面 | 4字节 |

##varchar和text

MySQL支持多种字符串类型， 如下表所示：

| 类型      | 最大字节长度 | 最大`utf8`字符数           |
| ---       | ---              | ---                                |
| `char(M)` | M，M为0~255之间的整数 | 85个`utf8`字符 |
| `varchar(M)` | M，M为0~65,535之间的整数 | 21,844个`utf8`字符 |
| `tinytext ` | 255 (2<sup>8</sup>−1) 字节 | 85个`utf8`字符 |
| `text`       | 65,535 (2<sup>16</sup>−1) 字节 = 64 KB  | 21,844个`utf8`字符 |
| `mediumtext` | 16,777,215 (2<sup>24</sup>−1) 字节 = 16 MB | 5,592,405个`utf8`字符 |
| `longtext` | 4,294,967,295 (2<sup>32</sup>−1) 字节 = 4 GB | 1,431,655,765个`utf8`字符 |

字符串类型实际支持的最大字符数与编码有关。比如，`varchar`类型在`utf8`编码下最大支持保存21,844 (65,535 / 3 = 21,844) 个字符，而`utf8mb4`编码下最大支持保存16,383 (65,535 / 4 = 16,383) 个字符。

另外，MySQL表行最大总长度为65,535字节 [[doc](http://dev.mysql.com/doc/refman/5.7/en/column-count-limit.html)]， 所以`varchar`类型字段的最大字符数会被表中其他字段所占用的存储空间挤掉。`blob`或`text`类型的字段不会受表行总长度的限制，因为字段存储的实际内容和表行是**分离**的，只会占用表行的9到12个字节。

```sql
mysql> create table test ( foo varchar(21845) character set utf8 );
ERROR 1118 (42000): Row size too large. The maximum row size for the used table type, not counting BLOBs, is 65535. This includes storage overhead, check the manual. You have to change some columns to TEXT or BLOBs
mysql> create table test ( foo varchar(21844) character set utf8 );
Query OK, 0 rows affected (0.19 sec)
 
mysql> create table test2 ( foo varchar(16384) character set utf8mb4 );
ERROR 1074 (42000): Column length too big for column 'foo' (max = 16383); use BLOB or TEXT instead
```
 
# 日期和时间类型
 MySQL支持的日期和时间类型如下表所示 [[doc](http://dev.mysql.com/doc/refman/5.7/en/date-and-time-type-overview.html)]：

| 日期和时间类型 | 字节 | 最小值 | 最大值 |
| ---                 | ---  | ---     | ---     |
| `date`               | 4     | `1000-01-01` | `9999-12-31` |
| `datetime`        | 8     | `1000-01-01 00:00:00.000000` | `9999-12-31 23:59:59.999999` |
| `timestamp`     | 4      | `1970-01-01 00:00:01.000000` | `2038-01-19 03:14:07.999999` |
| `time`              |    3 | `-838:59:59.000000` | `838:59:59.000000` |
| `year`              | 1   |  `1901` |  `2155`  |

# 参考资料

1. MySQL 5.7 Reference Manual, ***12 Data Types*** <http://dev.mysql.com/doc/refman/5.7/en/data-types.html>
2. MySQL 5.7 Reference Manual, ***10.1 Character Set Support*** <http://dev.mysql.com/doc/refman/5.5/en/charset.html>
3. MySQL 5.7 Reference Manual, ***C.10.4 Limits on Table Column Count and Row Size*** <http://dev.mysql.com/doc/refman/5.7/en/column-count-limit.html>
4. Difference between “int” and “int(3)” data types in mysql <http://stackoverflow.com/q/5562322>
5. Which MySQL Datatype to use for storing boolean values? <http://stackoverflow.com/q/289727>
6. MySQL: Large VARCHAR vs. TEXT? <http://stackoverflow.com/q/2023481>
7. TINYTEXT, TEXT, MEDIUMTEXT, and LONGTEXT maximum storage sizes <http://stackoverflow.com/q/13932750>

