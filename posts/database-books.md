---
title: 数据库书籍资料整理
date: 2016-11-26 15:56:02 
categories: 计算机系统
tags: [MySQL, 数据库, 书籍, CS]
---

本文整理一些经典的数据库教材以及MySQL相关书籍。

<!--more-->

[http://en.wikipedia.org/wiki/Database](http://en.wikipedia.org/wiki/Database)
[http://en.wikipedia.org/wiki/Template:Databases](http://en.wikipedia.org/wiki/Template:Databases)
[http://en.wikipedia.org/wiki/Template:SQL](http://en.wikipedia.org/wiki/Template:SQL)

Ask HN: Help me find a good databases textbook [https://news.ycombinator.com/item?id=377669](https://news.ycombinator.com/item?id=377669)
Database Internals - Where to Begin? [http://stackoverflow.com/q/770273](http://stackoverflow.com/q/770273)
Ask HN: good book or resources to get my SQL skills to the next level [https://news.ycombinator.com/item?id=1370721](https://news.ycombinator.com/item?id=1370721)
Ask HN: Is there a K&R for SQL? [https://news.ycombinator.com/item?id=5087439](https://news.ycombinator.com/item?id=5087439)
Tips for getting started with SQL? [http://stackoverflow.com/q/110124](http://stackoverflow.com/q/110124)

# 教材原理类书籍

1. 1986-2010，Silberschatz，《**数据库系统概念**》（Database System Concepts, 1st 1986, [6th](http://www.amazon.com/Database-System-Concepts-Abraham-Silberschatz/dp/0073523321) 2010），[home](http://codex.cs.yale.edu/avi/db-book/)，[豆瓣](http://book.douban.com/subject/10548379/)：对数据库**理论、概念**有很清晰地介绍
2. 1996、1999、2002，[Ramakrishnan](http://en.wikipedia.org/wiki/Raghu_Ramakrishnan) & [Gehrke](http://en.wikipedia.org/wiki/Johannes_Gehrke)，《**数据库管理系统：原理与设计** Cow Book》（Database Management Systems, [3rd](http://www.amazon.com/Database-Management-Systems-3rd-Edition/dp/0072465638) 2002），[home](http://pages.cs.wisc.edu/%7Edbbook/)，[豆瓣](http://book.douban.com/subject/1155934/)：第1作者2006年-2012年为Yahoo首席科学家，2012年跳槽到微软，该书为伯克利CS186、MIT、CMU教材
3. 2006、2010，Elmasri & Navathe，《**数据库系统基础**》（Fundamentals of Database Systems, 6th [2010](http://www.amazon.com/Fundamentals-Database-Systems-6th-Edition/dp/0136086209)），[图灵2007](http://www.ituring.com.cn/book/556)，第6版[豆瓣](http://book.douban.com/subject/6938465/)
4. 2005，[Hellerstein](http://en.wikipedia.org/wiki/Joseph_M._Hellerstein) & [Stonebraker](http://en.wikipedia.org/wiki/Michael_Stonebraker)，"**Readings in Database Systems**", 3th 1998, 4th 2005，[MITpress](http://mitpress.mit.edu/books/readings-database-systems)，[home](http://redbook.cs.berkeley.edu/)：被称为“Red Book”，经典论文收录集，第1作者是**Berkeley** Database Group的leader，第2作者是Ingres之父，Hellerstein的博士导师之一
5. 1975-2003，[C. J. Date](http://en.wikipedia.org/wiki/Christopher_J._Date)
，《**数据库系统导论**》（An Introduction to Database Systems, 2nd 1977, 8th 
[2003](http://www.amazon.com/Introduction-Database-Systems-8th-Edition/dp/0321197844)），[豆瓣](http://book.douban.com/subject/2179813/)：作者曾在IBM工作，该书是**最早**的数据库教材
6. 1999、2008，[Garcia-Molina](http://en.wikipedia.org/wiki/H%C3%A9ctor_Garc%C3%ADa-Molina) & [Ullman](http://en.wikipedia.org/wiki/Jeffrey_Ullman) & [Widom](http://en.wikipedia.org/wiki/Jennifer_Widom)，《数据库系统全书 DSCB》（Database Systems: The Complete Book, 2nd 2008），[home](http://infolab.stanford.edu/%7Eullman/dscb.html)，[豆瓣](http://book.douban.com/subject/1137262/)：内容偏向介绍数据库的底层实现原理，三位作者来自Stanford，并且均获得Codd奖，获奖年份依次为1999年、2006年、2007年
7. 1992，[Jim Gray](http://en.wikipedia.org/wiki/Jim_Gray_%28computer_scientist%29)，《**事务处理：概念与技术**》（Transaction Processing: Concepts and Techniques, [amazon](http://amzn.com/1558601902)）：介绍事务经典之作，不过略微过时。作者1998年图灵奖，1993年第2届Codd奖得主
8. 1997、2009，[Bernstein](http://en.wikipedia.org/wiki/Phil_Bernstein) & Newcomer，《**事务处理原理**》（Principles of Transaction Processing, [2nd](http://www.amazon.com/Principles-Transaction-Processing-Kaufmann-Management/dp/1558606238) 2009），[豆瓣](http://book.douban.com/subject/5412835/)：目前最新、全面的书籍。作者来自Microsoft研究院，1994年第3届Codd奖得主
9. 1999、2011，[M. Tamer Ozsu](https://cs.uwaterloo.ca/%7Etozsu/)，《**分布式数据库系统原理**》（Principles of Distributed Database Systems, [3rd](http://www.amazon.com/Principles-Distributed-Database-Systems-Tamer/dp/1441988335) 2011），[豆瓣](http://book.douban.com/subject/1099408/)

# SQL相关书籍

1. 2010，《SQL反模式》（SQL Antipatterns），[豆瓣](http://book.douban.com/subject/6800774/)
2. 2009，《SQL学习指南》（Learning SQL），[豆瓣](http://book.douban.com/subject/4872454/)
3. 1999-2008，《**SQL技术手册**》（SQL in a Nutshell, 2nd 2004, [3rd](http://www.amazon.com/SQL-Nutshell-In-OReilly/dp/B00CVDS62S) 2008），[豆瓣](http://book.douban.com/subject/4115916/)：根据SQL2003 ANSI标准，同时涵盖MySQL、Oracle、PostgreSQL及SQL Server
4. 1995-2010，[Joe Celko](http://en.wikipedia.org/wiki/Joe_Celko)，《**SQL权威指南**》（Joe Celko's SQL for Smarties: Advanced SQL Programming, [1st](http://amzn.com/1558603239) 1995, [4th](http://amzn.com/0123820227) 2010），[图灵2012](http://www.ituring.com.cn/book/969)，[豆瓣](http://book.douban.com/subject/20395440/)：全书652页，作者曾担任ANSl SQL标准委员会成员达10年之久，参与了SQL-89和SQL-92标准的制定
5. 2005，[Joe Celko](http://en.wikipedia.org/wiki/Joe_Celko)，《SQL编程风格》（Joe Celko's SQL Programming Style, [amazon](http://amzn.com/0120887975)）[，图灵2008](http://www.ituring.com.cn/book/434)，[豆瓣](http://book.douban.com/subject/3208673/)
6. 1996、2002，[Joe Celko](http://en.wikipedia.org/wiki/Joe_Celko)，《SQL解惑》（Joe Celko's SQL Puzzles and Answers, [2nd](http://amzn.com/0123735963) 2002），[图灵2008](http://www.ituring.com.cn/book/489)
7. 2005，Ben Forta，《MySQL必知必会》（MySQL Crash Course），[home](http://www.forta.com/books/0672327120/)，[豆瓣](http://book.douban.com/subject/3354490/) 


# MySQL书籍

[http://en.wikipedia.org/wiki/Template:MySQL](http://en.wikipedia.org/wiki/Template:MySQL)
[https://github.com/shlomi-noach/awesome-mysql](https://github.com/shlomi-noach/awesome-mysql)
[http://en.wikipedia.org/wiki/Database_engine](http://en.wikipedia.org/wiki/Database_engine)
2014-01 Any recommended MySQL books for more advanced stuff? (self.mysql) [http://redd.it/292u19](http://redd.it/292u19)
What resources exist for Database performance-tuning? [http://stackoverflow.com/q/761204](http://stackoverflow.com/q/761204)
2010-12 12 Best MySQL Database Books for Your Library [http://www.thegeekstuff.com/2010/12/12-best-mysql-books/](http://www.thegeekstuff.com/2010/12/12-best-mysql-books/)

## MySQL管理

1. 2008、2013，《深入浅出MySQL数据库：开发优化与管理维护》，[豆瓣](http://book.douban.com/subject/25817684/)：作者来自网易DBA小组
2. 2005、2008，《MySQL核心技术手册》（MySQL in a Nutshell, [2nd](http://www.amazon.com/MYSQL-Nutshell-In-OReilly/dp/B00CVDWK70) 2008），[豆瓣](http://book.douban.com/subject/4116793/)：作者MySQL公司知识库编辑
3. 1999-2013，[Paul DuBois](http://www.oreilly.com/pub/au/330)，《MySQL技术内幕》（MySQL: Developer's Library, [2nd](http://www.amazon.com/MySQL-Developers-Library-Paul-DuBois/dp/0735712123) 2003, [5th](http://amzn.com/0321833872) 2013），第4版2008[豆瓣](http://book.douban.com/subject/6524185/)，第5版[豆瓣](http://book.douban.com/subject/26436525/)：全书886页，作者是“MySQL参考手册”（MySQL Reference Manual）的主要贡献者之一，其他书“MySQL Cookbook”（[豆瓣](http://book.douban.com/subject/3045359/)）
4. 2002、2007、2014，[Paul DuBois](http://www.oreilly.com/pub/au/330)，《MySQL Cookbook 中文版》，[3rd](http://amzn.com/1449374026) 2014，第2版[豆瓣](http://book.douban.com/subject/3045359/)：全书948页

## MySQL调优

1. 2011，《**Effective MySQL之SQL语句最优化**》（Effective MySQL Optimizing SQL Statements, [amazon](http://www.amazon.com/Effective-MySQL-Optimizing-Statements-Oracle/dp/0071782796)），[豆瓣](http://book.douban.com/subject/20438822/)
2. 2009，简朝阳，《**MySQL性能调优与架构设计**》，[豆瓣](http://book.douban.com/subject/3729677/)：作者目前就职于阿里巴巴
3. 2012，Sevta Smirnova，《MySQL排错指南》（MySQL Troubleshooting），[豆瓣](http://book.douban.com/subject/26591051/)：Oracle公司MySQL部门bug分析支持团队的首席技术支持工程师
4. 2014，李海翔，《**数据库查询优化器的艺术**》，[豆瓣](http://book.douban.com/subject/25815707/)：作者现任职于Oracle公司MySQL全球开发团队，从事查询优化技术的研究和MySQL查询优化器的开发工作
5. 2014，贺春旸，《Mysql管理之道：性能调优、高可用与监控》，[豆瓣](http://book.douban.com/subject/25801248/)
6. 2004、2008、2012，[Schwartz](http://en.wikipedia.org/wiki/Baron_Schwartz) & [Zaitsev](http://www.mysqlperformanceblog.com/author/admin/) & Tkachenko，《**高性能MySQL：优化，备份，复制**》（High Performance MySQL, [3rd](http://amzn.com/1449314287) 2012），[home](http://www.highperfmysql.com/)，[豆瓣](http://book.douban.com/subject/23008813/)：深入MySQL类top1书籍，第2作者和第3作者来自MySQL AB公司高性能开发组，2006年创办Percona公司
7. 2010，Charles Bell & Kindahl & Thalmann，《**高可用MySQL：构建健壮的数据中心**》（MySQL High Availability, [amazon](http://amzn.com/0596807309)），[豆瓣](http://book.douban.com/subject/6847455/)，第2版[豆瓣](http://book.douban.com/subject/26630834/)：3位作者全部来自MySQL复制和备份小组，是复制、备份方面的开发人员及专家

## MySQL源码架构
1. 2010、2013，[姜承尧](http://www.innomysql.net/about)，《MySQL技术内幕：InnoDB》，第2版[豆瓣](http://book.douban.com/subject/24708143/)：2011至今，网易杭州研究院，数据库技术组 技术经理
2. 2007、2012，[Charles Bell](https://www.apress.com/index.php/author/author/view/id/3873)，《**深入理解MySQL**》（Expert MySQL, [2nd](http://www.amazon.com/Expert-MySQL-Experts-Voice-Source/dp/1590597419) 2012），[图灵2009](http://www.ituring.com.cn/book/224)，[豆瓣](http://book.douban.com/subject/4188364/)：全书466页，MySQL源代码分析，作者是MySQL核心开发人员
3. 2007，[Sasha Pachev](http://www.oreilly.com/pub/au/2761)，《**深入理解MySQL核心技术**》（Understanding MySQL Internals, [amazon](http://www.amazon.com/Understanding-MySQL-Internals-Sasha-Pachev/dp/0596009577)），[豆瓣](http://book.douban.com/subject/4022870/)：全书246页，纯粹的MySQL源代码分析，作者是前MysQL开发团队成员


