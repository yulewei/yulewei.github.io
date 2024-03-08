---
title: 流行互联网网站技术栈整理（万字长文）
date: 2023-12-03 23:37:00
categories: 架构
tags: [技术栈, 架构, 分布式, 微服务, 可伸缩性, 数据库, MySQL, Java]
---

本文整理总结主要的流行互联网网站技术栈，以及这些网站的技术栈和架构的历史演进过程。涉及的网站大部分都是当前或曾经访问量或月活用户量 Top 的网站（参见 Similarweb 网站的统计[^1]，或访问量 Top 10 网站的历史演变[^2]，或 wiki 整理的至少 1 亿月活用户量的社交平台[^3]）。国内网站或 APP 涵盖了主流[^4]的阿里、腾讯、百度、美团、字节、京东等大厂的互联网产品。整理的技术栈主要是流行网站的服务端业务系统的技术栈，包括编程语言、数据库、RPC 框架等，同时也简单整理了大数据技术栈，前端和客户端技术栈等不涉及。除了对大部分流行网站的技术栈做系统性梳理外，本文还挑选部分有代表性的网站，对这些网站的技术栈和架构的历史演进做详细解析。

<!--more-->

技术栈是构建应用的技术集合，由编程语言、框架、库、服务器、数据库、工具等组合而成。组成技术栈的技术是与具体业务无关的基础软件。互联网公司选择的技术栈，倾向于使用开源软件，相对于专有软件，开源软件具有高质量、免费、开放、灵活等优势。互联网的早期开拓者 Yahoo 的技术栈选择是经典案例，受开源运动的影响，在 2000 左右 Yahoo 从最初基于自定义的专属软件迁移了到 LAMP 技术栈。


在“开源”（open source）一词出现之前，技术社区的黑客选择使用“自由软件”（free software）这个词。但是“自由软件”这个词与对知识产权的敌意、共产主义和其它观点相联系，几乎不受管理者和投资者的欢迎，于是 1998 年 2 月 3 日在由 [Eric Raymond](https://en.wikipedia.org/wiki/Eric_S._Raymond) 等人参加的会议上“开源”一词诞生，2 月下旬开源软件促进会成立 [OSI](https://opensource.org/history/)，Eric Raymond 担任主席。自由软件和开源软件被合称为 [FOSS](https://en.wikipedia.org/wiki/Free_and_open-source_software)。缩写“[LAMP](https://en.wikipedia.org/wiki/LAMP_%28software_bundle%29)”代表的是 Linux-Apache-MySQL-PHP，这些软件都是自由软件或开源软件。

# 案例汇总与解析

## 技术发展时间线

在互联网诞生早期，开源技术栈、开源社区尚未成熟，互联网公司不得不自研专有软件，随着开源软件的成熟，技术栈的选择开始从专有软件逐渐转向开源软件。先来看下，主要 Web 技术和服务端技术的发展时间线：

- 1994.03，Linux 1.0 对外发布，源码采用 GPL 协议。
- 1994.10，网景公司的 Web 浏览器 [Netscape](https://en.wikipedia.org/wiki/Netscape_%28web_browser%29) 首次对外发布。
- 1995.02，[Apache HTTP Server](https://en.wikipedia.org/wiki/Apache_HTTP_Server) 项目创立，4 月首次对外开源发布，版本为 0.6.2，源码采用 [Apache 协议](https://en.wikipedia.org/wiki/Apache_License)。创建 Apache HTTP Server 项目的[核心成员](https://httpd.apache.org/ABOUT_APACHE.html)包括 [Brian Behlendorf](https://en.wikipedia.org/wiki/Brian_Behlendorf)、[Roy Fielding](https://en.wikipedia.org/wiki/Roy_Fielding) 等。
- 1995.12，[JavaScript](https://en.wikipedia.org/wiki/JavaScript) 语言诞生，创造者为来自网景公司的 Brendan Eich。
- 1995.05，[Java](https://en.wikipedia.org/wiki/Java_%28programming_language%29) 语言诞生，创造者为 Sun 公司。Java 平台早期并不真正开源，虽然 1998 年 JDK 1.2 开始以 [SCSL](https://en.wikipedia.org/wiki/Sun_Community_Source_License) 协议开放源代码，但 SCSL 协议限制太大，饱受批评，直到 [2006.11](https://web.archive.org/web/0/http://www.sun.com/2006-1113/feature/story.jsp) 才以 GPL 协议真正开源。
- 1995.06，[PHP](https://en.wikipedia.org/wiki/PHP) 语言诞生，源码采用 [PHP](https://www.php.net/license/) 协议（BSD 风格的协议）。
- 1995.12，[HTML](https://en.wikipedia.org/wiki/HTML) 标准规范首次发布，版本为 HTML 2.0
- 1996.05，HTTP/1.0 规范发布（[RFC1945](https://datatracker.ietf.org/doc/html/rfc1945)），第一作者 [Tim Berners-Lee](https://en.wikipedia.org/wiki/Tim_Berners-Lee)，第二作者 [Roy Fielding](https://en.wikipedia.org/wiki/Roy_Fielding)。1997 年 HTTP 1.1 规范发布（[RFC2068](https://datatracker.ietf.org/doc/html/rfc2068)），1999 年 HTTP 1.1 规范发布更新版（[RFC2616](https://datatracker.ietf.org/doc/html/rfc2616)）。HTTP 1.1 的主要设计者是 Roy Fielding，他也是 Apache HTTP Server 项目的[主要创建者](https://httpd.apache.org/ABOUT_APACHE.html)之一。基于 HTTP 的设计思想，Roy Fielding 在 2000 年的博士论文创造了 [REST](https://en.wikipedia.org/wiki/REST) 架构风格。
- 1996.05，Sun 公司首次对外发布 [Java Servlet](https://en.wikipedia.org/wiki/Jakarta_Servlet) API。
- 1996.10，[MySQL](https://en.wikipedia.org/wiki/MySQL) 首次公开对外发布，最初的公开发行版仅提供了 Solaris 的二进制发行版，一个月后，源代码和 Linux 二进制文件发布，源码采用的是专有协议“MySQL Free Public License”，到 [2000.06](https://web.archive.org/web/0/http://www.mysql.com/news/article-23.html) 改为 GPL 协议。
- 1996.10，[PostgreSQL](https://en.wikipedia.org/wiki/PostgreSQL) 的诞生日是在 10 月 22 日，在这一天 PostgreSQL.org 网站上线。PostgreSQL 之前的项目名为 Postgres，之所以改名是为了反映其对 SQL 的支持。源码采用 [PostgreSQL 协议](https://www.postgresql.org/about/licence/)（类似 BSD 或 MIT 协议）。
- 1996.12，微软发布 [ASP](https://en.wikipedia.org/wiki/Active_Server_Pages) 技术，到 2002 年 ASP 被 ASP.NET 替代。
- 1998.12，缩写“[LAMP](https://en.wikipedia.org/wiki/LAMP_%28software_bundle%29)”诞生，代表 Linux-Apache-MySQL-PHP。
- 1999.12，Sun 公司首次对外发布 [J2EE](https://en.wikipedia.org/wiki/Jakarta_EE)，版本为 1.2，包括 Servlet、JSP、EJB 等技术。
- 2001.04，Douglas Crockford 创造了 [JSON](https://en.wikipedia.org/wiki/JSON) 缩写和数据格式。2006.07，JSON 规范发布（[RFC4627](https://datatracker.ietf.org/doc/html/rfc4627)）。
- 2002.08，[Nginx](https://en.wikipedia.org/wiki/Nginx) 首次对外发布，源码采用 BSD 协议。最初，研发 Nginx 目的是为了解决 [C10k](https://en.wikipedia.org/wiki/C10k_problem) 问题。之后，Nginx 逐渐替代 Apache HTTP Server，LAMP 技术栈改为 LNMP 技术栈。
- 2003.06，Java 的 [Spring](https://en.wikipedia.org/wiki/Spring_Framework) 框架首次对外发布，版本为 0.9，源码采用 Apache 协议。
- 2004.08，[Ruby on Rails](https://en.wikipedia.org/wiki/Ruby_on_Rails) 框架首次对外发布，源码采用 MIT 协议。
- 2004.10，首届 [Web 2.0 Conference](https://web.archive.org/web/20041001082754/http://www.web2con.com/) 举办，[Web 2.0](https://en.wikipedia.org/wiki/Web_2.0) 概念开始流行。
- 2005.02，“[Ajax](https://en.wikipedia.org/wiki/Ajax_%28programming%29)”术语诞生，缩写代表的是“Asynchronous JavaScript + XML”。早期使用 Ajax 技术的经典案例是 Google 在 2004 年发布的 Gmail、2004 年 12 月发布的 Google 搜索的 [Google Suggest](https://en.wikipedia.org/wiki/Timeline_of_Google_Search) 特性、2005 年发布的 Google Maps，以及 2004 年发布的 Flickr 等。早期 Ajax API 响应的数据格式是 XML，从 2005 年 [del.icio.us API](https://inkdroid.org/2005/09/21/delicious-json/) 和 [Yahoo! Web Services](https://simonwillison.net/2005/Dec/16/json/) 支持 [JSON](https://en.wikipedia.org/wiki/JSON) 格式等代表性转变开始，XML 逐渐被 JSON 取代。
- 2005.07，Python 的 [Django](https://en.wikipedia.org/wiki/Django_%28web_framework%29) 框架首次对外发布，源码采用 BSD 协议。
- 2006.06，Rails 的创造者 DHH 在 RailsConf 2006 会议上做了题为“Discovering a world of Resources on Rails”的演讲，介绍了将要发布的 Rails 1.2（2007.01 正式发布）对 REST 开发的支持。Rails 框架支持 REST 开发，大力推动了 REST 的普及。受 Rails 的影响，其他编程语言的 Web 开发框架[模仿 Rails 的方式](https://groups.google.com/g/rest_in_action/c/G7sBHhkNdsw)开始对支持 REST 开发。
- 2007.02，[HBase](https://en.wikipedia.org/wiki/Apache_HBase) 宣布在 Hadoop 项目中成立，成为 Hadoop 的子项目。HBase 是 Google 的 [BigTable](https://en.wikipedia.org/wiki/Bigtable)（OSDI'2006）论文的开源实现。
- 2007.01，iPhone 手机首次对外发布，同年 6 月 iPhone 正式发售，11 月 Android 系统首次对外公布，移动互联网开始大爆发。
- 2008.07，Facebook 对外开源 [Cassandra](https://en.wikipedia.org/wiki/Apache_Cassandra) 项目，项目受 Amazon 的 [Dynamo](https://en.wikipedia.org/wiki/Dynamo_%28storage_system%29)（SOSP'2007）论文和 Google 的 BigTable（OSDI'2006）论文的。
- 2009.06，“[NoSQL](https://en.wikipedia.org/wiki/NoSQL)”术语诞生。“NoSQL”术语诞生于在旧金山举行的一场关于“开源、分布式、非关系数据库”的技术聚会，该技术聚会被命名为“[NoSQL meetup](https://web.archive.org/web/20110710205509/http://nosql.eventbrite.com/)”。NoSQL 的两大起源是 BigTable 和 Dynamo。
- 2009.11，Google 对外公开 [Go](https://en.wikipedia.org/wiki/Go_%28programming_language%29) 语言，BSD 协议开源。
- 2011.04，451 Group 咨询公司的分析师 Matthew Aslett 创造“[NewSQL](https://en.wikipedia.org/wiki/NewSQL)”术语，是对用于 OLTP 的分布式关系型数据库的统称。最具代表性的 NewSQL 是 Google 的 [Spanner](https://en.wikipedia.org/wiki/Spanner_%28database%29)（OSDI'2012）和 Amazon 的 [Aurora](https://en.wikipedia.org/wiki/Amazon_Aurora)（2014.10 发布的云原生数据库产品）。

## 技术栈案例汇总
 
按编程语言区别，主要流行的互联网产品的创建时间和早期的技术栈选择（也可以参见 [wiki](https://en.wikipedia.org/wiki/Programming_languages_used_in_most_popular_websites)）：

* **C/C++ 技术栈**：
    * **Yahoo!**（1994，2002 转向 PHP）、**Amazon**（1994，之后转向以 Java 为主）、**eBay**（1995，2002 全面转向 Java）、**Google 搜索**（1998）[^5][^6]、**新浪网**（1998，之后转 PHP，2008 年 PHP 占据 90% 以上的 Web 开发）[^7]、**腾讯 QQ**（1999）、**QQ 空间**（2005）、**微信**（2011）[^8]、**百度搜索**（2000）、**Yandex 搜索**（2000）[^9]等 
* **PHP（LAMP 或 LNMP）技术栈**：
    * **Wikipedia**（2001）[^10]、**Facebook**（2004，2006 核心服务转向 C++）、**Flickr**（2004）[^11]、**Etsy**（2005）[^12]、**[WordPress](https://en.wikipedia.org/wiki/WordPress)**（2005）、**新浪博客**（2005）[^13]、**百度百科**（2006，2006 开始百科、知道、文库、贴吧等产品以 PHP 为主实现）[^14]，**Tumblr**（2007，2011 转向 Scala）[^15]、**哔哩哔哩**（2009，2017 转向 Golang）[^16]、**新浪微博**（2009，2011 转向 Java）[^17]、**腾讯微博**（2010）[^18]、**美团**（2010，2011 转向 Java）[^19]、**滴滴**（2012，2017 业务系统转向 Golang）[^20]等
* **Ruby on Rails 技术栈**：
    * **Twitter**（2006，2009 核心服务转向 Scala 和 Java）、**Shopify**（2006）[^21]、**SoundCloud**（2007，2012 业务转向多种 JVM 语言和基础设施转向 Golang）[^22][^23][^24]、**Heroku**（2007）[^25]、**GitHub**（2008）[^26]、**Airbnb**（2008，2016 开始服务化并部分转向 Java）[^27][^28]等
* **Python 技术栈**：
    * **Reddit**（2005，少部分高性逻辑用 C 语言）[^29]、**YouTube**（2005）[^30]、**豆瓣**（2005）[^31]、**Disqus**（2007）[^32]、**Dropbox**（2007，2013 基础设施转向 Golang）[^33][^34][^35]、**Quora**（2009，高性能服务用 C++）[^36]、**Pinterest**（2009，部分服务转向 Java）[^37][^38]、**Instagram**（2010，少部分高性逻辑用 C++）[^39][^40][^41]、**Uber**（2010，2015 转向 Golang 和 Java 等）[^42]、**知乎**（2011，2018 核心业务转向 Golang）[^43]、**字节跳动**（2012，今日头条、抖音等，2014 开始引入 Golang，2016 业务大规模转向 Golang）[^44][^45]等
* **Java 技术栈**：
    * **淘宝**（2003，2004 PHP 全面转向 Java）、**LinkedIn**（2003）、**Google Gmail**（2004）[^46]、**Google Docs**（2006）、**Google+**（2011）[^47]、**人人网**（2005）[^48]、**Netflix**（2007）[^49]等
* **.NET 技术栈**
    * **携程**（1999，2017 全面转向 Java）[^50]、**京东**（2004，2012 全面转向 Java）[^51]、**Stack Overflow**（2008）、**Bing 搜索**（2009）等

值得一提的是，PHP 之父 [Rasmus Lerdorf](https://en.wikipedia.org/wiki/Rasmus_Lerdorf)，在 2002 年至 2009 年期间为供职于 Yahoo!，2011 年起至今供职于 Etsy。Python 之父 [Guido van Rossum](https://en.wikipedia.org/wiki/Guido_van_Rossum)，在 2005 年至 2012 年期间供职于 Google，在 2013 年至 2019 年期间供职于 Dropbox。国内最有影响力的 PHP 技术专家是[惠新宸](https://baike.baidu.com/item/%E6%83%A0%E6%96%B0%E5%AE%B8/8222328)，是加入 PHP 语言官方开发组的首位国人，曾先后供职于雅虎中国、百度、新浪微博、链家网等公司。

## 编程语言选择

2000 年前创建的网站，因为开源技术栈尚未成熟，编程语言基本上都是选择 C++ 开发。在开源技术栈成熟后，多数网站会选择拥抱开源。具体选择哪个编程语言，PHP、Ruby、Python、Java 等，主要由技术负责人的技术偏好决定。根据 [w3techs](https://w3techs.com/technologies/history_overview/programming_language/ms/y) 统计，历年网站使用的服务端编程语言统计占比，2012 年至今 10 多年，PHP 占比稳居榜首，每年都是 75% 以上。大型网站通常都是公司内的技术团队研发的，技术栈由技术团队选择，而很多小型网站，很可能会直接使用开源的 CMS 系统搭建。根据 [w3techs](https://w3techs.com/technologies/overview/content_management) 统计，前 100 万网站中有 43.0% 使用 WordPress 构建。WordPress 的服务端编程语言就是 PHP，数据库是 MySQL。大型网站选择 PHP 越来越少，因为 PHP 是解释型脚本语言，相对编译型编程语言有性能劣势。另外，PHP 的优势是快速开发 Web 动态网页，但是随着前后端分离开发模式的流行，Web 页面从服务端渲染逐渐转向客户端渲染，PHP 的优势不再，后端工程师只需要向前端提供 REST API 接口，展示层的实现完全由前端工程师实现。

早期部分网站选择 .NET 技术栈，比如**京东**、**携程**。但是，Java 平台生态更完善，有非常多的经验可以借鉴。另外，.NET 平台本身虽然不收费，但是 Windows 操作系统是收费的，开发工具也不便宜。于是京东在 2012 年从 .NET 迁移到 Java，携程在 2017 左右从 .NET 迁移到 Java。目前在国内，多数互联网大厂都选择 Java 技术栈，如淘宝、美团、京东、微博、携程等，Java 相对来说是主流选择。

编程语言的另外一个流行趋势是 Go 语言。Go 语言诞生于 Google，在 2009 年 11 月对外公开。在发明 Go 语言前，Google 内部主要使用的语言是 C++、Java 和 Python 等，但是 Go 发明者认为这些语言无法同时满足高效编译、高效执行和易于编程的特性诉求，所以创造了 Go 语言[^52]。Go 比 C++ 能更高效编译和易于编程，比 Java 更易于编程，比 Python 能更高效执行。在 GoCon Tokyo 2014 会议上，Go 语言研发团队的 Brad Fitzpatrick 对各种编程语言在编程乐趣和执行速度（Fun vs. Fast）的对比[^53]，如下图所示：

<img width="600" alt="编程语言 Fun vs. Fast" title="编程语言 Fun vs. Fast" src="https://static.nullwy.me/gocon-tokyo-2014-fitzpatrick-go-fun-fast.png">

早期由 Go 语言实现的经典开源项目主要是基础设施软件，比如 YouTube 的 [Vitess](https://github.com/vitessio/vitess) 数据库分片中间件（[2012.02](https://www.reddit.com/r/programming/comments/qakq4) 对外开源）、[Docker](https://en.wikipedia.org/wiki/Docker_%28software%29) 容器（2013.03 对外开源）、Red Hat 的 CoreOS 团队的 [Etcd](https://github.com/etcd-io/etcd) 分布式配置服务（[2013.07](https://web.archive.org/web/0/http://coreos.com/blog/distributed-configuration-with-etcd/) 对外开源，实现 [Raft](https://en.wikipedia.org/wiki/Raft_%28algorithm%29) 共识协议）、前 Google 工程师创建的 [CockroachDB](https://en.wikipedia.org/wiki/CockroachDB) 分布式数据库（2014.02 对外开源，受 Google Spanner 启发）、Google 的 [Kubernetes](https://en.wikipedia.org/wiki/Kubernetes) 容器编排系统（2014.06 对外开源）、SoundCloud 的 [Prometheus](https://en.wikipedia.org/wiki/Prometheus_%28software%29) 监控工具（2015.01 对外开源）等。大规模使用 Go 语言的代表性的国外的互联网公司有 **SoundCloud**[^23][^24]、**Dropbox**[^34][^35]、**Uber**[^42] 等，国内的有**七牛云**、**字节跳动**[^44][^45]、**哔哩哔哩**[^16]、**滴滴**[^20]、**知乎**[^43]、**百度**[^54][^55]、**腾讯**[^56]等，在这些互联网公司 Go 语言通常主要被用于构建基础设施软件（网关、数据存储、监控、视频处理等）或高性能要求的业务服务，部分公司的大量核心业务也从 Python、PHP 等迁移到 Go 语言。使用 Go 语言的公司的更加完整的列表，可以参见官方整理的“[GoUsers](https://go.dev/wiki/GoUsers)”。

## 数据库选择

数据库方面，根据 [DB-Engines Ranking](https://db-engines.com/en/ranking_trend/relational+dbms) 的统计，流行的关系数据库主要是 4 个，开源免费的 MySQL 和 PostgreSQL，专有收费的 Oracle 和 Microsoft SQL Server。多数互联网产品使用的数据库是 MySQL，部分是 PostgreSQL 或 Oracle。一些早期使用 Oracle 数据库的网站选择部分或完全从 Oracle 数据库中迁出，代表性的案例是，Amazon 从 Oracle 完全迁移到 Amazon RDS 和 NoSQL，淘宝从 Oracle 完全迁移到 MySQL。以 PostgreSQL 为主数据库的互联网应用有 Skype、Reddit、Spotify、Disqus、Heroku、Instagram 等[^57]。

MySQL 相对 PostgreSQL 更加流行的主要原因是[^58][^59]，在早期 MySQL 入门槛更低。MySQL 很早就支持在 Windows 下安装，而 PostgreSQL 只能在 Cygwin 下安装，MySQL 对 Windows 平台的支持使得初学者更容易上手，并且 MySQL 更加易于管理，能够快速简单地启动和使用，而且 MySQL 拥有一个非常简洁、易于导航和用户友好的在线文档和支持社区。另外一个重要原因是，MySQL 很早就默认支持复制（replication），而 PostgreSQL 的复制是第三方的，而且极其难用。之后的几年即便 PostgreSQL 完善了不足，但错过了流行的时间窗口，MySQL 成为了主流选择，有更完善的生态。不过，受 MySQL 被 Oracle 公司收购的影响，近几年 PostgreSQL 开始越来越流行，根据 DB-Engines Ranking 的统计，最新的 PostgreSQL 的流行度分数是 10 年前的三倍，流行度与前三名越来越接近。

传统关系数据库，因为在面对大数据量和高负载量时的性能和可扩展性较差，以及对数据模型的支持有限，逐渐演变出了分布式非关系数据库 NoSQL 和分布式关系数据库 NewSQL。NoSQL 数据库的按数据模型分类包括：键值数据库（比如 Redis、DynamoDB 等）、列族数据库（比如 Cassandra、HBase 等）、文档数据库（比如 MongoDB）、图数据库（比如 Neo4j）等。

## 技术栈和架构演进

容易发现多数网站的技术栈的演进模式类似。在创建网站早期通常选择使用 PHP、Ruby、Python 脚本语言和框架来开发，这些脚本语言和框架具有更快的开发效率，能快速交付上线。随着网站流量的增长，面临性能和可扩展性问题，通常会将网站服务化，从单体架构向面向服务的 SOA 和微服务架构演进，一大部分网站在服务化过程中会选择将核心业务改由其他性能更佳的编译型编程语言来实现，如 Java、Scala、C++、Go，原先的脚本语言可能全部废弃，也可能仅用于前端展示层的 Web 动态页面渲染，比如 Facebook、Twitter 等。当然也有一部分网站，演进为 SOA 和微服务架构后，业务逻辑的实现一直以最初的脚本语言为主。

在架构微服务化后，很多网站实现的微服务之间采用基于 HTTP 协议的 REST 方式通信（或者使用的 RPC 框架支持跨语言 RPC 调用），各个微服务的实现在理论上编程语言不需要统一，可以根据该微服务的性能要求或负责该微服务的团队的技术偏好自由选择，所以真实世界的互联网公司内部各个业务子系统的技术栈可能不统一。但是内部技术栈不统一会带来额外的维护成本。另外，考虑技术栈的迁移重构成本，新的业务子系统使用新的技术栈，而有些旧的业务子系统很可能继续使用旧的技术栈。除了业务子系统外，很可能团队内部有自研的中间件、运维工具等，这些组件由中间件团队、运维团队研发，很可能会选择与业务系统不一样的技术栈。

**注意，解决网站的可扩展性问题，不一定需要演进为服务化架构**。架构服务化意味着将完整的单体服务按业务的功能领域做垂直拆分，而实际上在应用服务层可以通过部署多个相同副本的单体服务的方式实现系统的水平扩展，网站的可扩展性问题主要在数据存储层上。**拆分应用服务的好处更多在于能实现组织团队的规模化，拆分后的小团队独立维护各自的微服务，能有效提升研发效率**。解决数据库的扩展性的策略有，数据复制（数据缓存、数据库主从读写分离）、数据垂直拆分、数据水平拆分（也叫数据分片，sharding）。数据库被拆分后，如果单个事务内的数据分散在多个节点就要解决分布式事务问题，但是实现分布式事务代价太大，通常的选择是牺牲一致性，仅满足最终一致性（[BASE](https://en.wikipedia.org/wiki/Eventual_consistency)）。对于无或弱事务要求的非关系型的数据，也可以选择存储在可扩展性能力更强的 NoSQL 数据库。

没有拆分应用服务，始终采用单体架构的经典案例是 Instagram，2019 年 Instagram 在技术博客上有这样一段话[^41]：

> Our server app is a monolith, one big codebase of several million lines and a few thousand Django endpoints, all loaded up and served together. A few services have been split out of the monolith, but we don’t have any plans to aggressively break it up.

Instagram 解决可扩展性问题，主要在数据存储层[^39][^40]。在 Instagram，PostgreSQL 被用于存储用户信息、媒体元数据、用户关系等数据，照片媒体数据存储在亚马逊 S3 服务上。Instagram 对 PostgreSQL 数据库做了主从读写分离、数据垂直拆分和数据水平分片。另外，Instagram 从 2012 年开始使用 Cassandra 数据库，Cassandra 被用于存储 Feed 流、活动等数据。类似的，Reddit 也是单体架构，在数据存储层做了可扩展性改造[^29]。对 PostgreSQL 数据库做了主从读写分离和垂直拆分，拆分为四个主数据库，链接、帐户、子版块、评论、投票和杂项，每个主数据库都从数据库。另外，投票数据存储在 Cassandra 数据库。

# Yahoo!（1994）

Yahoo!，1994 年创立，早期网站操作系统是 FreeBSD，Web 服务器是自研的 [Filo Server](https://en.wikipedia.org/wiki/David_Filo)，数据库是自研的 DBM 文件，Web 动态页面脚本是自研的 yScript，业务逻辑编程语言是 C++。1996 年 Web 服务器改用 Apache HTTP Server，1999 年部分数据库改用 MySQL（同时也使用 Oracle 数据库），2002 年编程语言从 yScript 和 C++ 改为 PHP，操作系统也从 FreeBSD 逐渐转向 Linux[^60][^61][^62]。在 2002.10 的 PHPCon 2002 会议上，Yahoo! 工程师 Michael Radwin 介绍了 Yahoo! 从专有软件向开源软件转向的原因和演进过程，以及选择 PHP 而不选择 ASP、Perl、JSP 等其他技术的原因[^60]。Yahoo! 转向开源软件的原因主要是，避免维护专有软件的成本，开源软件从早期的不成熟最终变得成熟，具有更好的性能，更容易与第三方软件集成，以及开源社区逐渐发展壮大。2005 年，Yahoo! 的技术架构如下图所示[^61]：

<img width="600" alt="Yahoo! 技术架构（2005）" title="Yahoo! 技术架构（2005）" src="https://static.nullwy.me/yahoo-architecture-2005.png">

2000 年 ~ 2010 年，Yahoo! 是世界上流量最大的网站，虽然期间被 Google 短暂超越，但随后又反超，一直到到 2010 年后才被 Google 彻底超越[^2]。2000 年前后 Yahoo! 转向到 MySQL 和 PHP，并在技术上助力 LAMP 生态的发展壮大，对 LAMP 技术栈的流行起到巨大的推动作用。值得一提的是，知名 MySQL 专家 [Jeremy Zawodny](https://en.wikipedia.org/wiki/Jeremy_Zawodny)，《高性能MySQL》2004 年第 1 版（[豆瓣](https://book.douban.com/subject/1495763/)）的作者，在 1999.12 ~ 2008.06 期间为 Yahoo! 工程师。PHP 之父 [Rasmus Lerdorf](https://en.wikipedia.org/wiki/Rasmus_Lerdorf) 在 2002.09 ~ 2009.11 期间为 Yahoo! 工程师。在前端领域，JSON 创造者 [Douglas Crockford](https://en.wikipedia.org/wiki/Douglas_Crockford)，在 2005 年至 2012 年期间供职于 Yahoo!。

# Amazon（1994）

Amazon，1994 年创立，早期是单服务、单数据库的单体架构，使用的编程语言是 C++，2000 年开始从单体架构向 SOA 和微服务架构演进，使用最多的语言变为 Java，C++ 被用在高性能要求的系统和底层基础设施组件。Amazon 架构演进过程如下图所示[^63]：

<img width="600" alt="Amazon 架构演进" title="Amazon 架构演进" src="https://static.nullwy.me/amazon-architecture-evolution.png">

Amazon 网站的技术栈演进过程：

+ 1995 ~ 2000：单体架构，Unix（Sun）、Obidos、Oracle、C++
  - Obidos 是亚马逊内部 Web 动态页面渲染引擎，编程语言是 C++。
+ 2000 ~ 2006：SOA 架构，Linux、Obidos、Oracle、C++
  - 2000 年，将 Sun/Unix 服务器替换为 HP/Linux 服务器。
  - 2000 年，拆分应用服务，向 SOA 架构演进。
  - 2005 年初，开始将 Obidos 框架替换为 Gurupa 框架，Gurupa 是 Web 页面渲染引擎，同时也是 SOA 框架。Web 动态页面的编程语言从 C++ 替换为 Perl/Mason，同时应用服务开始被更细粒度的拆分。
+ 2006 年开始：微服务架构，Linux、Gurupa、Oracle、Perl & Java & C++ 
  - 2019.10，彻底去掉 Oracle 数据库，迁移到 Amazon RDS 和 NoSQL。

当前 Amazon 网站的主要技术栈[^63][^64][^65][^66]：

- **应用服务**：
   - **展示层**：Perl/Mason
   - **业务逻辑层**：Java（主要）、C++ 等
   - **RPC框架**：Gurupa 框架（自研闭源）
      - 2006 年更早之前使用 [Obidos](https://en.wikipedia.org/wiki/Obidos_%28software%29) 框架
   - **消息队列MQ**：[Amazon SQS](https://en.wikipedia.org/wiki/Amazon_Simple_Queue_Service)
- **数据存储**[^67]：
   - **关系数据库**：Amazon RDS for PostgreSQL[^68]、Amazon Aurora (PostgreSQL)
      - 2019 年 10 月彻底去掉 Oracle 数据库，迁移到 Amazon RDS 和 NoSQL。
   - **键值存储**：[Amazon DynamoDB](https://en.wikipedia.org/wiki/Amazon_DynamoDB)
   - **缓存**：Amazon ElastiCache for Redis[^69]
   - **Blob文件存储**：[Amazon S3](https://en.wikipedia.org/wiki/Amazon_S3)
- **数据分析**：[Amazon Kinesis](https://en.wikipedia.org/wiki/Amazon_Kinesis)、Amazon EMR、[AWS Lambda](https://en.wikipedia.org/wiki/AWS_Lambda)、[Amazon Redshift](https://en.wikipedia.org/wiki/Amazon_Redshift) 等

# eBay（1995）

eBay，1995 年创立，早期使用的编程语言是 Perl，1997 年迁移到 C++，2002 年迁移到 Java。eBay 的技术栈演进过程[^70][^71]：

+ 1995 ~ 1997：FreeBSD、Apache、GDBM、Perl
+ 1997 ~ 2002：Windows、IIS、Oracle、C++
  - 1999.02，开始应用服务器池化和水平扩展；数据库服务器垂直扩展（替换为高端服务器 Sun E10000）
  - 1999.11，开始数据库水平扩展，按功能垂直拆分、数据水平分片。
+ 2002 ~ 2006：Windows、IBM WebSphere、Oracle、Java（J2EE）
  - 2002 年，开始将 C++ 替换为 Java，使用 J2EE 框架
+ 2006 年开始：Ubuntu、Tomcat、Oracle、Java（Spring/Spring Boot）
  - 2007 年初，开始向 SOA 架构演进，再然后演进为微服务架构[^72][^73]

2016 年，eBay 的技术架构如下图所示[^73]：

<img width="600" alt="eBay 技术架构（2016）" title="eBay 技术架构（2016）" src="https://static.nullwy.me/ebay-architecture-2016.png">

当前 eBay 的主要技术栈[^70][^71][^73][^74]：

- **应用服务**：
   - **展示层**：Java、Spring MVC
   - **业务逻辑层**：Java、Spring、Spring Boot
   - **RPC框架**：Raptor、Raptor.io（自研闭源，基于 Spring、Spring Boot）
   - **消息队列MQ**：BES (business event stream)（自研闭源）、BES2（自研闭源，基于 Kafka 实现，BES2 API 完全兼容 BES1）[^75]
- **数据存储**[^76]：
   - **关系数据库**：Oracle（主）、MySQL（次）
   - **NoSQL**：Apache Cassandra、MongoDB
   - **搜索引擎**：Elasticsearch[^77]
- **数据分析**[^78]：Apache Kafka、Apache Flink、[Apache Kylin](https://en.wikipedia.org/wiki/Apache_Kylin)（自研开源，MOLAP 数仓）、[ClickHouse](https://en.wikipedia.org/wiki/ClickHouse)（ROLAP 实时数仓） 等
     - Apache Kylin，2014.10 对外开源，2014.11 成为 Apache 孵化器项目，2015.12 从 Apache 孵化器毕业。

# 淘宝（2003）

淘宝技术架构[^79][^80][^81][^82][^83]，是从 LAMP 架构演变而来的。在 2003 年 5 月最早上线时，淘宝是对购买得到的采用 LAMP 架构网站源码进行二次开发的网站。为了应对网站流量的增长，2003 年底数据库从 MySQL 切换到 Oracle，2004 年初编程语言从 PHP 迁移到 Java，同时硬件演变为 IBM 小型机和 EMC 高端硬件存储，依赖的基础设施被统称为“IOE”（IBM 小型机、Oracle 数据库、EMC 存储设备），这个架构被称为 2.0 架构。

2007 年底开始拆分第一个业务中心是用户中心（UIC，User Information Center），在 2008 年初上线。2008 年初，启动“千岛湖”项目，拆分出了交易中心（TC，Trade Center）和类目属性中心（Forest）。2008 年 10 月，为了打通淘宝网和淘宝商城（后来的天猫）的数据，启动“五彩石”项目，拆分出了店铺中心（SC，Shop Center）、商品中心（IC，Item Center）、评价中心（RC，Rate Center）。到 2009 年初，“五彩石”项目结束，阿里电商架构也从之前的单体架构演进为了 SOA 服务化架构，被称为 3.0 架构，同时淘宝网和淘宝商城也完成了数据整合，并淘宝网和淘宝商城的共享业务沉淀到多个业务中心。在 2009 年，共享业务事业部成立，在组织架构上共享业务事业部和淘宝、天猫平级。在业务中心的上层业务服务有，交易管理（TM，Trade Manager）、商品管理（IM，Item Manager）、店铺系统（ShopSystem）、商品详情（Detail）、评价管理（RateManager）等。2009 年淘宝的服务化拆分如下图所示[^81]：

<img width="600" alt="淘宝的服务化拆分（2009）" title="淘宝的服务化拆分（2009）" src="https://static.nullwy.me/taobao-architecture-2009.jpg">

在各个应用服务拆分的同时，数据库也按业务功能做了垂直拆分，2008 年 Oracle 被拆分为商品、交易、用户、评价、收藏夹等 10 来套。当时的淘宝的分布式架构设计，包括 Java + Oracle 技术栈的选择，以及应用服务和数据库的拆分策略，很大程度上参考的是 eBay 的架构[^79][^82]。对照 eBay 架构师在 2006 年 11 月对外分享的 eBay 架构的 slides[^70]，容易发现两者大同小异。

因为使用 Oracle 成本太高，2008 年 8 月淘宝数据库设计了异构数据库读写分离架构，写库为集中式的 Oracle，读库使用分库分表的 MySQL，该数据库架构方案基于自研的数据库中间件 TDDL 实现[^84]。之后的几年，数据库逐渐向 MySQL 迁移。2008 年 9 月前微软亚洲研究院常务副院长[王坚](https://baike.baidu.com/item/%E7%8E%8B%E5%9D%9A/8451588)博士加盟阿里巴巴集团，担任首席架构师，2009 年 11 月时任阿里 CTO 的王坚博士决策启动阿里“去IOE”工程。淘宝的“去IOE”工程的关键时点[^85]：

- 2009.07，王坚博士开始担任阿里首席技术官 CTO。
- 2009.11，王坚博士决策启动阿里“去IOE”工程。
- 2010.01，大淘宝核心系统“去IOE”工作启动。
- 2010.07，淘宝商品库完成“去I”。
- 2011.07，淘宝商品库从 Oracle 迁移到 MySQL，完成“去IOE”。
- 2011.09，淘宝交易库从 Oracle 迁移到 MySQL，完成“去IOE”。
- 2012.12，完成大淘宝“去IOE”。

值得注意的是，在 2011 年淘宝商品库和交易库“去IOE”的同时，为了满足数据库的性能要求，底层硬件从 HDD 机械硬盘改为 SSD 固态硬盘。而庆幸的是，当时的前几年正好是 SSD 大爆发的时间点[^86]。在消费级 SSD 市场，2010 年苹果的 MacBook Air 开始全面改用 SSD，2012 年苹果的 MacBook Pro 开始全面改用 SSD。在数据库服务器层面，因为 SSD 在随机读写的性能具有巨大优势，2010 年开始机械硬盘改用 SSD 成为趋势，当时的 MySQL 新版代码也专门针对 SSD 做了性能优化[^87]。

2009 年初淘宝演变为 SOA 架构后，服务拆分粒度越来越细，到 2009 年底拆分出的总服务数达到 187 个，到 2010 年底达到 329 个[^81]。因为系统依赖关系越来越复杂，当时最大问题是稳定性问题，系统的稳定性建设成为重点工作。到 2013 年阿里电商系统开始 4.0 架构改造，即异地多活架构改造，内部称为单元化项目。2013 年 8 月完成杭州同城双活，2014 年双 11 前的 10 月完成杭州和上海近距离两个单元的异地双活，2015 年双 11 完成三地四单元的异地多活[^80][^83]，至此也完成了 3.0 到 4.0 架构的升级。在阿里电商系统迁移到阿里云公共云方面，2015 年阿里电商系统开始采用混合云弹性架构，当阿里本地保有云无法支撑时，就快速在公有云上扩建新的单元，当流量过去后，再还资源给公有云。2015 年 10% 的双 11 大促流量使用了阿里云的机器来支撑，2016 年 50% 以上的双 11 大促流量使用了阿里云的机器来支撑[^80]。2019 年初，阿里电商开启了全面上公共云的改造，到 2019 年双 11，阿里电商系统实现了全部核心应用上公共云，2021 年双 11，阿里电商系统实现了 100% 上公共云。

当前阿里电商系统（淘宝、天猫等）的主要技术栈[^82][^83]：

- **应用服务**：
   - **展示层**：Java
   - **业务逻辑**：Java、Spring、Spring Boot 等
   - **RPC框架**：HSF（2008 自研闭源）、Apache Dubbo（2008 自研，2011.10 对外开源）
   - **消息队列MQ**：[Apache RocketMQ](https://en.wikipedia.org/wiki/Apache_RocketMQ)（自研，2013.09 对外开源）
     - 2007 年自研 Notify；2011 年自研 MetaQ；2013.09，对外开源发布 RocketMQ 3.0。RocketMQ 3.0 和 MetaQ 3.0 等价，阿里内部使用的称为 MetaQ 3.0，外部开源称之为 RocketMQ 3.0。
   - **链路追踪**：鹰眼 EagleEye（2012 自研闭源）
      - 2012 年内部发布鹰眼 1.0，参考 Google 2010 年的 [Dapper](https://research.google/pubs/pub36356/) 论文
   - **注册和配置服务**：ConfigServer（2008 自研闭源）、Nacos（自研，[2018.07](https://mp.weixin.qq.com/s/fnrGjiywiySA8iAZh_cF0Q) 对外开源）
      - 2008 年，淘宝团队内部自研 ConfigServer 并发布 1.0，被用于 HSF 的服务发现。2018.07，Nacos 对外开源，是 ConfigServer 的开源实现，在阿里内部 Nacos 被用在钉钉、考拉、饿了么、优酷等业务线。
- **数据存储**：
   - **关系数据库**：MySQL (阿里云 RDS，底层基于 MySQL 自研 AliSQL)、自研 TDDL 中间件、阿里云分布式式数据库 [PolarDB-X](https://help.aliyun.com/zh/polardb/polardb-for-xscale/what-is-polardb-for-xscale)、MySQL 的 [X-Engine](https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/introduction-to-x-engine) 存储引擎
      - 2008 年，淘宝团队内部自研 TDDL 中间件，用于实现数据库的读写分离、分库分表。
      - 2014 年，淘宝 TDDL 团队和阿里云 RDS 团队合作，在阿里云上输出产品分布式数据库服务 DRDS，2014.06 DRDS 开始公测，2014.12 正式上线。DRDS 脱胎于阿里 B2B 团队开源的 [Cobar](https://github.com/alibaba/cobar) 分布式数据库引擎（2012.06 对外开源），同时借鉴了淘宝 TDDL 丰富的分布式数据库实践经验。
      - 2020 年，阿里云发布分布式式数据库 PolarDB-X，融合了分布式 SQL 引擎 DRDS 与分布式自研存储 X-DB（X-Cluster）。
      - MySQL 的 [X-Engine](https://github.com/polardb/polardbx-engine/wiki/2-X-Engine-Overview) 存储引擎，早期代码源自 Facebook 的 RocksDB 4.8.1（2016.07 发布），被应用在阿里的交易历史库、钉钉历史库等核心系统。
   - **缓存**：[Tair](https://www.alibabacloud.com/help/zh/redis/product-overview/overview-1)（2009 自研，2010.06 对外开源，分布式键值存储，支持基于内存和文件的两种存储方式）
      - 2009.04，Tair 1.0 正式诞生，并被应用于淘宝核心系统、MDB缓存、用户中心等业务。
      - 2010.06，Tair 对外开源。
      - 2014.05，阿里云推出基于 Tair 的缓存产品 OCS。
      - 2019.11，阿里云发布 Redis 企业版，即 Tair 3.0，兼容 Redis 5.0。
   - **列族数据库**：Apache [HBase](https://help.aliyun.com/zh/hbase/product-overview/what-is-apsaradb-for-hbase)，为淘宝推荐、手淘消息等提供支撑的数据库
      - 早期淘宝交易订单的所有数据都存储在单独的 Oracle 数据库；2010 年拆分为在线库和历史库，三个月之前的历史订单迁移进单独的 Oracle 历史库；2011 年交易历史库整体迁移到 HBase；2018 年历史库迁移到基于 X-Engine 存储引擎的 PolarDB-X 集群[^88]。
   - **搜索引擎**：Havenask（自研，[2022.12](https://www.infoq.cn/article/myy674gdqphqgvm8u2h5) 对外开源）
      - 支持了淘宝、天猫、菜鸟、优酷、高德、饿了么等在内整个阿里的搜索业务。阿里云的 [OpenSearch](https://help.aliyun.com/zh/open-search/) 产品底层基于 Havenask 实现。
   - **Blob文件存储**：阿里云 OSS
      - 早期采用 NetApp 提供的 NAS 存储设备；2007.06 开始基于自研的 [TFS](https://github.com/alibaba/tfs)（Taobao File System）；2010.09 对外开源 TFS；2016 迁移到阿里云 OSS[^89]。
- **数据分析**：阿里云 [Flink](https://help.aliyun.com/zh/flink/)、阿里云 [MaxCompute](https://help.aliyun.com/zh/maxcompute/)（离线数仓）、阿里云 [Hologres](https://cn.aliyun.com/product/bigdata/hologram)（实时数仓）等
   - 早期阿里内部三大实时计算引擎并存，Galaxy、JStorm、Blink（[Apache Flink](https://en.wikipedia.org/wiki/Apache_Flink) 的分支），[2017](https://flink-learning.org.cn/article/detail/b91ee62ecc12107ed15ccd0c97d33cd5) 年开始统一到 Blink。2019.01，阿里收购 Apache Flink 母公司 Data Artisans，随后阿里的 Blink 正式开源，之后 Blink 和 Flink 完成合并。2021 年开始，阿里内部的实时计算使用的是 Blink 团队与 Flink 创始团队联合打造了 [Flink 企业版](https://help.aliyun.com/zh/flink/)平台，即 VVP（Ververica Platform）。

2016 年，阿里产品专家倪超对外介绍的阿里电商技术架构，如下图所示[^90]：

<img width="600" alt="阿里电商技术架构（2016）" title="阿里电商技术架构（2016）" src="https://static.nullwy.me/alibaba-architecture-2016.png">

类似的，阿里巴巴中间件首席架构师《企业IT架构转型之道》（[豆瓣](https://book.douban.com/subject/27039508/)）作者钟华在 2017 年对外介绍的阿里电商技术架构图[^91]：

<img width="600" alt="阿里电商中台架构（2017）" title="阿里电商中台架构（2017）" src="https://static.nullwy.me/alibaba-middle-platform-2017.png">

2017 年，阿里中间件技术部专家谢吉宝对外介绍的阿里中间件技术大图[^83]：

<img width="600" alt="阿里中间件技术大图（2017）" title="阿里中间件技术大图（2017）" src="https://static.nullwy.me/alibaba-middleware-2017.png">

阿里 HSF 与 Dubbo 的历史演进时间线：
- 2008 年，阿里淘宝团队自研 HSF 框架，5 月发布 HSF 1.1。
- 2009 年初，由阿里 B2B 团队研发的 Dubbo 发布 1.0 版本。阿里内部淘系（淘宝、天猫、一淘等）主要使用的服务框架是 HSF，而阿里 B2B 使用的则是 Dubbo，二者独立，各行其道，彼此不通[^92]。
- 2010.04，重构后发布 Dubbo 2.0 版本。
- [2011.10](https://www.iteye.com/topic/1116866)，Dubbo 对外开源，版本为 2.0.7。因为在功能完善性、架构优雅性、使用简便性等方面有其相对独特的优势，开源后被国内很多互联网公司广泛使用[^93]，包括去哪儿，京东、当当网、网易考拉、有赞等。
- 2012 年，阿里内部架构调整，开始统一技术基础设施，合并重复项目，决定把 Dubbo 合并到 HSF 里面去[^94]。随后，HSF 发布 2.0 的版本，兼容 Dubbo 的协议，HSF 推出后很快就在阿里集团全面铺开[^92]。Dubbo 项目在阿里内部被抛弃，在 2012.10 发布 2.5.3 版本之后就停止更新，2013 年和 2014 年更新了 2 次 Dubbo 2.4 的维护版本，然后停止了所有维护工作。
- [2014.10](https://www.infoq.com/cn/news/2014/10/dubbox-open-source)，当当网开源 Dubbox，基于 Dubbo 2.5.3 的代码，为 Dubbo 新增了 REST 风格远程调用、Kryo/FST 序列化等特性。
- [2016.01](https://mp.weixin.qq.com/s/P5p7jMcaZVW15sknDbgdRQ)，阿里云互联网中间件产品 [EDAS](https://help.aliyun.com/zh/edas/product-overview/what-is-edas) 正式商用，EDAS 所提供的分布式服务框架源自阿里的 HSF。
- [2017.09](https://www.oschina.net/news/88477)，Dubbo 重启维护，对外发布恢复维护后的第一个版本是 2.5.4。之所以重启是因为阿里云上的客户大部分使用 Dubbo，阿里云想要将基于 Dubbo 的解决方案作为自己的一个产品，卖给这些客户[^92][^94]。
- 2018.01，Dubbo 2.6.0 发布，合并了当当网提供的 Dubbox 分支。
- 2018.02，阿里宣布将 Dubbo 捐献给 Apache，进入 Apache 孵化器。
- [2018.11](https://developer.aliyun.com/article/672608)，阿里云 EDAS 产品版本升级，新版本实现 SpringCloud 和 Dubbo 用户代码零侵入就能迁移至 EDAS。
- 2018.12，Dubbo 3.0 正式进入开发阶段，Dubbo 3.0 是 HSF 与 Dubbo 的融合版本。
- 2019.01，Dubbo 2.7.0 发布，包名切换到 org.apache，全面拥抱 Java 8。
- [2019.05](https://www.infoq.cn/article/LHqFdI_X9kdHdeXkZ7xf)，Dubbo 从 Apache 毕业，成为 Apache 的顶级项目。
- 2020 年，在双十一前在阿里内部率先由阿里考拉实现了 Dubbo 3.0 的全面部署。
- 2021.06，Dubbo 3.0 正式发布。
- 2022 年，阿里集团几乎所有的业务线（包括淘宝、天猫、饿了么、钉钉等）全面从 HSF2 迁移到 Dubbo3。阿里内部使用的是 HSF3，HSF3 与以往的 HSF2 完全不同，HSF3 完全就是基于标准 Dubbo3 的 SPI 扩展库[^95][^96]。

阿里 RocketMQ 的历史演进时间线：
- 2007，自研 Notify，最早底层的消息存储采用本地文件存储，参考 ActiveMQ 实现了单机 kv 存储引擎，2008 年底层的消息存储改用 Oracle，2010 年从 Oracle 迁移到高可用 MySQL 存储集群[^97]。
- 2011.01，基于 Kafka 的设计用 Java 完全重写并发布 MetaQ 1.0。
- [2012.03](https://www.infoq.cn/article/2012/03/metamorphosis)，淘宝对外开源 MetaQ 1.x，项目名为 Metamorphosis（[淘蝌蚪](https://web.archive.org/web/20120312015328/http://code.taobao.org/p/metamorphosis/wiki/intro/)、[GitHub](https://github.com/killme2008/Metamorphosis)），版本号为 1.4.0。
  - Metamorphosis [1.0.1](https://web.archive.org/web/20120312015318/http://code.taobao.org/p/metamorphosis/wiki/changelist/) 开始实现高可用的 HA 方案，支持同步和异步复制，复制特性类似于 MySQL 的主从复制。
  - Kafka 的复制特性，直到 2013.12 发布的 0.8.0 版本才开始支持。Kafka 实现的复制是集群间的分区复制（Intra-cluster Replication），复制的副本粒度是分区（partition），参见 [KAFKA-50](https://issues.apache.org/jira/browse/KAFKA-50)。
- 2012.09，淘宝内部发布 MetaQ 2.0 版本，MetaQ 2.0 对架构进行了重新设计，为了解决分区文件数增加后的性能下降问题，对消息日志文件存储目录结构做了改造[^98]。改造后的 MetaQ 架构与 Kafka 存在很大差异，这个版本的 MetaQ 可以认为是第一代的 RocketMQ。
- 2013.07，淘宝内部发布 MetaQ 3.0 版本。
- 2013.09，对外开源发布 RocketMQ 3.0。RocketMQ 3.0 和 MetaQ 3.0 等价，阿里内部使用的称为 MetaQ 3.0，外部开源称之为 RocketMQ 3.0[^99]。
- 2014.10，阿里云消息队列 [ONS](https://help.aliyun.com/zh/apsaramq-for-rocketmq/)（云开放消息服务，Open Notification Service）对外公测，ONS 是基于阿里消息中间件 MetaQ（RocketMQ）打造的云消息产品。
- [2016.11](https://www.oschina.net/news/89061)，阿里巴巴将 RocketMQ 捐赠给 Apache，成为 Apache 孵化器项目。
- [2017.09](https://www.oschina.net/news/89061)，RocketMQ 从孵化器毕业，正式成为 Apache 顶级项目。


# LinkedIn（2003）

LinkedIn，2003 年创立，采用纯 Java 技术栈，早期是单体架构，到 2008 年演化为 SOA 架构[^100]。到 2010 年拆分的服务数超过 150 个，到 2015 年拆分的服务数超过 750 个[^101]。

2008 年，LinkedIn 对外介绍的技术架构如下图所示[^100]：

<img width="600" alt="LinkedIn 技术架构（2008）" title="LinkedIn 技术架构（2008）" src="https://static.nullwy.me/linkedIn-architecture-2008.png">

2012 年，LinkedIn 对外介绍的关注在数据基础设施上的技术架构图[^102]：

<img width="600" alt="LinkedIn 数据基础设施架构（2012）" title="LinkedIn 数据基础设施架构（2012）" src="https://static.nullwy.me/linkedIn-data-infrastructure-icde2012.jpg">

当前 LinkedIn 的主要技术栈[^100][^101]：

- **应用服务**：
   - **展示层**：Java、Spring MVC、
   - **业务逻辑层**：Java、Spring
   - **RPC框架**：
      - [Rest.li](https://github.com/linkedin/rest.li)（自研开源），REST + JSON 框架。[2023.04](https://engineering.linkedin.com/blog/2023/linkedin-integrates-protocol-buffers-with-rest-li-for-improved-m)，对外宣布 Rest.li 项目不再活跃，为了提升性能，迁移到 Protobuf + gRPC。
   - **消息队列MQ**：ActiveMQ
- **数据存储**[^102][^103][^104]：
   - **关系数据库**：Oracle（主）、MySQL（次）
   - **缓存**：Memcached
   - **键值存储**：[Voldemort](https://en.wikipedia.org/wiki/Voldemort_%28distributed_data_store%29)（自研，2009.01 对外开源）
      - [2009.01](https://blog.linkedin.com/2009/03/20/project-voldemort-scaling-simple-storage-at-linkedin) 对外开源，设计受 Amazon [Dynamo](https://en.wikipedia.org/wiki/Dynamo_%28storage_system%29) 论文影响。2018 年之后 LinkedIn 不在使用 Voldemort，数据迁移到 LinkedIn 的 Venice 数据库
   - **文档数据库**：[Espresso](https://dbdb.io/db/espresso)（自研闭源），分布式文档数据库
      - 2011 年初开始计划和设计，2012.06 在生产环境部署，底层实现基于 MySQL、Lucene、ZooKeeper。
   - **图数据库**：LIquid（自研闭源）
   - **搜索引擎**：基于 Lucene
   - **Blob文件存储**：[Ambry](https://github.com/linkedin/ambry)（自研，[2016.05](https://engineering.linkedin.com/blog/2016/05/introducing-and-open-sourcing-ambry---linkedins-new-distributed-) 对外开源）
- **数据分析**[^102][^103][^104]：
   - **数据采集**：
      - [Databus](https://github.com/linkedin/databus)（自研，[2013.02](https://engineering.linkedin.com/data-replication/open-sourcing-databus-linkedins-low-latency-change-data-capture-system) 对外开源）
         - 数据变更抓取 CDC 工具，支持 Oracle 和 MySQL，2012.10 在 SoCC 2012 会议上发表[论文](https://dblp.org/rec/conf/cloud/DasBSGVNZGWGSTPSS12.html)，2013.02 对外开源。
      - [Apache Kafka](https://en.wikipedia.org/wiki/Apache_Kafka)（自研，[2011.01](https://blog.linkedin.com/2011/01/11/open-source-linkedin-kafka) 对外开源）
         - 2011.07 成为 Apache 孵化器项目，2012.10 从 Apache 孵化器毕业
   - **计算引擎**：Apache Hadoop、[Apache Samza](https://en.wikipedia.org/wiki/Apache_Samza)（自研开源）、Apache Spark、Apache Flink
      - Apache Samza，2013.09 对外开源，2015.01 从 Apache 孵化器毕业
   - **数据仓库**：Apache Hive、Presto、[Venice](https://github.com/linkedin/venice) 等

# Facebook（2004）

Facebook，2004 年创立，早期采用 LAMP 技术栈，为了应对负载增长开始服务化，将核心业务从 LAMP 中移出到新的服务，新的服务改用使用 C++、Java 等编写。为了解决 PHP 进程与非 PHP 进程的 RPC 通信问题，[2006](https://engineering.fb.com/2014/02/20/open-source/under-the-hood-building-and-open-sourcing-fbthrift/) 年内部研发了跨语言的 RPC 库 [Thrift](https://en.wikipedia.org/wiki/Apache_Thrift)，2007.04 对外开源。PHP 代码用于实现前端展示层的 Web 动态页面渲染，以及对服务层的数据聚合。

2010 年，Facebook 工程师 Aditya Agarwal 对外介绍的 Facebook 技术架构如下图所示[^105]：

<img width="600" alt="Facebook 技术架构（2010）" title="Facebook 技术架构（2010）" src="https://static.nullwy.me/facebook-architecture-2010.png">

Facebook 的技术栈图[^106]：

<img width="600" alt="Facebook 技术架构（2008）" title="Facebook 技术架构（2008）" src="https://static.nullwy.me/facebook-architecture-2008.png">

Facebook 的 NewsFeed 服务的架构图[^106]：

<img width="600" alt="Facebook NewsFeed 技术架构（2008）" title="Facebook NewsFeed 技术架构（2008）" src="https://static.nullwy.me/facebook-newsfeed-architecture-2008.png">

Facebook 的 Search 服务的架构图[^106]：

<img width="600" alt="Facebook Search 技术架构（2008）" title="Facebook Search 技术架构（2008）" src="https://static.nullwy.me/facebook-search-architecture-2008.png">

当前 Facebook 的主要技术栈[^105][^106][^107]：
- **应用服务**：
   - **展示层**：PHP（早期采用 LAMP 技术栈）、[HHVM](https://en.wikipedia.org/wiki/HHVM)（[2011.12](https://www.facebook.com/notes/10158791564347200/) 对外开源）、Hack（2014.03 对外发布）
   - **业务逻辑层**：Hack、C++（主要用于实现性能敏感服务，包括 Ads、News Feed、Search 等）、Rust、Java 等[^108][^109]
   - **RPC框架**：[Apache Thrift](https://en.wikipedia.org/wiki/Apache_Thrift)（自研开源）
      - 2005 年末开始研发；2007.04 对外开源；[2008.05](https://thrift.apache.org/about) 成为 Apache 孵化器项目；2010.10 从 Apache 孵化器毕业。
   - **消息队列MQ**：Wormhole（自研闭源）
      - [2013.06](https://engineering.fb.com/2013/06/13/core-infra/wormhole-pub-sub-system-moving-data-through-space-and-time/) 对外介绍；2015.05 在 [NSDI 2015](https://www.usenix.org/conference/nsdi15/technical-sessions/presentation/sharma) 会议上发表论文。Wormhole 是发布-订阅系统，消息发布者直接读取数据存储系统的写事务日志，投递给对消息感兴趣的订阅者，订阅者可能是 News Feed、Cache、Index Servers 等。
- **数据存储**：
   - **关系数据库**：MySQL、MySQL 存储引擎 [MyRocks](https://en.wikipedia.org/wiki/MyRocks)（自研，[2015.11](https://github.com/facebook/mysql-5.6/wiki) 对外开源）
      - [RocksDB](https://en.wikipedia.org/wiki/RocksDB) 是 Facebook 在 2012.04 创建的内嵌的键值数据库，基于 Google 开源的 [LevelDB](https://en.wikipedia.org/wiki/LevelDB) 代码。
      - [MyRocks](https://en.wikipedia.org/wiki/MyRocks) 是 Facebook 基于 RocksDB 数据库实现的 MySQL 存储引擎，2014 年开始研究 RocksDB 和 MySQL 的集成，2015.11 对外开源，在 [2016.04](https://www.slideshare.net/matsunobu/myrocks-deep-dive) 在 Percona Live 2016 会议上最早对外介绍。
      - [2017.08](https://engineering.fb.com/2017/09/25/core-infra/migrating-a-database-from-innodb-to-myrocks/)，Facebook 将 UDB 数据库（User Database，存储社交活动数据）从 InnoDB 完全迁移到了 MyRocks，迁移后存储空间使用量减少了一半，而 CPU 和 IO 使用率没有明显变化。
      - [2018.06](https://engineering.fb.com/2018/06/26/core-infra/migrating-messenger-storage-to-optimize-performance/)，Facebook 在博客上对外介绍已经将 Messenger 应用的数据从 HBase 迁移到了 MyRocks，迁移后存储空间使用量减少 90%，读取延迟比以前的系统低 50 倍。值得注意的是，Messenger 应用的数据最早存储在 Facebook 自研的 [Cassandra](https://en.wikipedia.org/wiki/Apache_Cassandra) 上，[2010](https://engineering.fb.com/2010/11/15/core-infra/the-underlying-technology-of-messages/) 年底发布新版的 Messenger 开始，改为存储在 HBase 上。
   - **键值存储**：[ZippyDB](https://engineering.fb.com/2021/08/06/core-infra/zippydb/)（自研闭源），基于 RocksDB 实现的分布式键值数据库，2013 年开始在 Facebook 部署使用
   - **缓存**：[Mcrouter](https://github.com/facebook/mcrouter)（自研开源），实现分布式扩展的 Memcached
      - Facebook 早在 2005.04 就已经开始使用 Memcached；[2013.04](https://www.facebook.com/notes/10158791580232200/) 在 [NSDI 2013](https://www.usenix.org/conference/nsdi13/technical-sessions/presentation/nishtala) 会议上发表论文，描述对 Memcached 的分布式扩展性改造。
   - **图数据库**：[TAO](https://engineering.fb.com/2013/06/25/core-infra/tao-the-power-of-the-graph/)（自研闭源），底层是 Memcache 和 MySQL
   - **搜索引擎**：[Unicorn](https://research.facebook.com/publications/unicorn-a-system-for-searching-the-social-graph/)（自研闭源），基于标准的搜索引擎的概念，但增加了社交图谱搜索的特性。Facebook 搜索功能（包括 [Facebook Graph Search](https://en.wikipedia.org/wiki/Facebook_Graph_Search) 产品）底层实现基于 Unicorn。
      - 2013.03 Facebook Graph Search 产品对外推出，2014.12 开始原始的图谱搜索公开可见度降低，2019.06 几乎完全弃用。
      - 2015 年初，Instagram 的搜索引擎从 Elasticsearch 迁移到 Unicorn[^110]。
   - **Blob文件存储**：Haystack（自研闭源）
      - [2009.04](https://engineering.fb.com/2009/04/30/core-infra/needle-in-a-haystack-efficient-storage-of-billions-of-photos/) 对外介绍；2010.10 在 [OSDI 2010](https://dblp.org/rec/conf/osdi/BeaverKLSV10.html) 会议上发表论文。基于 Haystack 论文的开源实现包括：[SeaweedFS](https://github.com/seaweedfs/seaweedfs)、bilibili 的 [BFS](https://github.com/Terry-Mao/bfs) 等。
- **数据分析**[^111][^112]：
   - **日志采集**：[Scribe](https://en.wikipedia.org/wiki/Scribe_%28log_server%29)（自研，[2008.10](https://engineering.fb.com/2008/10/24/web/facebook-s-scribe-technology-now-open-source/) 对外开源）
   - **计算引擎**：XStream（Puma、Stylus、Swift）
   - **数据仓库**：[Apache Hive](https://en.wikipedia.org/wiki/Apache_Hive)（自研，2008.08 对外开源）、[Presto](https://en.wikipedia.org/wiki/Presto_%28SQL_query_engine%29)（自研，2013.11 对外开源）、Laser、Scuba
 
# Twitter（2006）

Twitter，2006 年创立，早期采用 Ruby on Rails 技术栈，数据库是 MySQL，2009 年开始服务化，将核心业务从 Ruby on Rails 中移出到新的服务，新的服务用 Scala、Java 编写[^113][^114]。

2013 年，Twitter 工程师总结的 Twitter 从单体架构向分布式架构演进的过程，如下图所示[^114]：

<img width="600" alt="Twitter 单体架构" title="Twitter 单体架构" src="https://static.nullwy.me/twitter-architecture-rails.png">

<img width="600" alt="Twitter 服务化架构（2013）" title="Twitter 服务化架构（2013）" src="https://static.nullwy.me/twitter-architecture-2013.png">

当前 Twitter 的主要技术栈[^113][^114]：
- **应用服务**：
   - **展示层**：Ruby on Rails
   - **业务逻辑层**：Scala（主要）、Java
   - **RPC框架**：[Finagle](https://github.com/twitter/finagle)（自研，[2011.08](https://blog.twitter.com/engineering/en_us/a/2011/finagle-a-protocol-agnostic-rpc-system) 对外开源）
      - 使用 Finagle 框架的公司包括：Foursquare（[2010](https://www.scala-lang.org/old/node/5130) PHP 转向 Scala）、Pinterest（Python + Java）、SoundCloud（Ruby + JVM）、Tumblr（PHP + Scala） 等。
   - **消息队列MQ**：Kestrel（[2008.11](https://robey.livejournal.com/53832.html) 对外开源） -> DistributedLog -> Kafka
   - **链路追踪**：[Zipkin](https://github.com/openzipkin/zipkin)（自研开源）
      - [2012.06](https://blog.twitter.com/engineering/en_us/a/2012/distributed-systems-tracing-with-zipkin)，对外开源，是 Google [Dapper](https://research.google/pubs/pub36356/) 链路追踪系统的开源实现。
      - 2015 年中，项目从 Twitter 转交到 OpenZipkin 开源组织。
- **数据存储**[^115]：
   - **关系数据库**：MySQL、Manhattan（自研闭源，基于 MySQL 的分布式数据库）
      - 最初所有数据都存储在 MySQL 上。
      - [2009](https://news.ycombinator.com/item?id=987693) 年底，计划从 MySQL 迁移到 Cassandra，打算用 Cassandra 存储推文 Tweets 数据，但最终改变了策略，继续使用 MySQL，Cassandra 仅被用于存储分析数据[^116]。
      - 2010.04，对外开源数据库分片框架 [Gizzard](https://en.wikipedia.org/wiki/Gizzard_%28Scala_framework%29)，基于 Gizzard 的分布式数据库 T-bird 被用于存储推文 Tweets 数据[^117]。
      - [2014.04](https://blog.twitter.com/engineering/en_us/a/2014/manhattan-our-real-time-multi-tenant-distributed-database-for-twitter-scale)，对外介绍自研的基于 MySQL 的分布式数据库 Manhattan，被用于存储推文 Tweets 等主要数据，Cassandra 被废弃。
      - [2018](https://blog.twitter.com/engineering/en_us/topics/infrastructure/2021/adopting-rocksdb-within-manhattan) 年中，开始将 Manhattan 底层使用的 MhBtree 存储引擎改为基于 RocksDB 存储引擎。
   - **缓存和键值存储**：Redis、Memcached
   - **图数据库**：[FlockDB](https://en.wikipedia.org/wiki/FlockDB)（自研开源）
   - **搜索引擎**：Earlybird[^118]（自研闭源）、Elasticsearch[^119]，用于推文 Tweets、用户、消息等搜索
      - [2010](https://blog.twitter.com/engineering/en_us/a/2010/twitters-new-search-architecture) 年，自研基于 Lucene 实现的 Earlybird 搜索引擎，替代旧的基于 MySQL 实现的搜索[^118]
   - **Blob文件存储**：Blobstore（自研闭源）
      - 2011 年，使用 [Photobucket](https://en.wikipedia.org/wiki/Photobucket)；[2012.09](https://blog.twitter.com/engineering/en_us/a/2012/blobstore-twitter-s-in-house-photo-storage-system) 开始改为使用自研的 Blobstore。
- **数据分析**[^120]：Kafka、Cloud BigTable、Cloud BigQuery 等


# 参考资料

[^1]: Similarweb: Top Websites Ranking <https://www.similarweb.com/top-websites/>
[^2]: 2020-08 Ranked: The Most Popular Websites Since 1993 (Captain Gizmo) <https://www.visualcapitalist.com/most-popular-websites-since-1993/>
[^3]: List of social platforms with at least 100 million active users <https://en.wikipedia.org/wiki/List_of_social_platforms_with_at_least_100_million_active_users>
[^4]: 2023-02 艾媒金榜｜2022年度中国APP月活排行榜 <https://www.iimedia.cn/c880/91597.html>

[^5]: 2007 Jeff Dean: Software Engineering Advice from Building Large-Scale Distributed Systems (Stanford CS295 class lecture, Spring, 2007) <https://research.google.com/people/jeff/stanford-295-talk.pdf>
[^6]: 2008-11 Google Architecture <http://highscalability.com/blog/2008/11/22/google-architecture.html>
[^7]: 2008-06 新浪夏清然、徐继哲：自由软件和新浪网. 程序员 2008年第6期54-55，[cnki](https://kns.cnki.net/kcms2/article/abstract?v=zcLOVLBHd2yRQ8cenitMd2JyjVqLnTRrIbCiNVgweUi0oksDMHi6mpnw3tjzJCetgm1nmkLf3WsQz_LDhH0H4WbuX5fX2pEZlAPiMqseXpMsfZMncrPMYOQ77jJKNZBm&uniplatform=NZKPT&flag=copy)，[cqvip](http://lib.cqvip.com/Qikan/Article/Detail?id=27523653) <http://www.zeuux.com/blog/content/3620/>
[^8]: 2016-01 微信张文瑞：从无到有：微信后台系统的演进之路 <https://www.infoq.cn/article/the-road-of-the-growth-weixin-background>
[^9]: 2008-02 Yandex Architecture http://highscalability.com/yandex-architecture
[^10]: 2007-08 Wikimedia architecture <http://highscalability.com/wikimedia-architecture>
[^11]: 2007-11 Flickr Architecture <http://highscalability.com/flickr-architecture>
[^12]: 2016-03 What does Etsy's architecture look like today? <http://highscalability.com/blog/2016/3/23/what-does-etsys-architecture-look-like-today.html>
[^13]: 2011-05 新浪刘晓震：新浪博客应用架构分享 (PHPChina 2011) <https://web.archive.org/web/0/http://www.phpchina.com/?action-viewnews-itemid-38418> <https://www.modb.pro/doc/121035>
[^14]: 2013-11 百度张东进：百度PHP基础设施构建思路 (QCon上海2013, slides, 30p) <https://www.modb.pro/doc/121042>
[^15]: 2013-05 The Tumblr Architecture Yahoo Bought for a Cool Billion Dollars <http://highscalability.com/blog/2013/5/20/the-tumblr-architecture-yahoo-bought-for-a-cool-billion-doll.html>
[^16]: 2017-06 B站任伟：B站高性能微服务架构 <https://zhuanlan.zhihu.com/p/33247332>
[^17]: 2021-05 微博刘道儒：十年三次重大架构升级，微博应对“极端热点”的进阶之路 <https://www.infoq.cn/article/qgwbh0wz5bvw9apjos2a>
[^18]: 2012-08 腾讯张松国：腾讯微博架构介绍 (ArchSummit深圳2012, slides, 59p) <https://www.slideshare.net/dleyanlin/08-13994311>
[^19]: 2016-02 美团夏华夏：从技术细节看美团架构 (ArchSummit北京2014) <https://www.infoq.cn/article/see-meituan-architecture-from-technical-details> <http://bj2014.archsummit.com/node/596/> <https://www.modb.pro/doc/8311>
[^20]: 2019-06 滴滴杜欢：大型微服务框架设计实践 (Gopher China 2019) <https://www.infoq.cn/article/EfOlY8_rubh4LfoXzF8B> <https://www.modb.pro/doc/35485>
[^21]: 2018-08 E-Commerce at Scale: Inside Shopify's Tech Stack - Stackshare.io <https://shopify.engineering/e-commerce-at-scale-inside-shopifys-tech-stack>
[^22]: 2013-03 Phil Calçado @ SoundCloud: From a monolithic Ruby on Rails app to the JVM (slides, 75p) <https://www.slideshare.net/pcalcado/from-a-monolithic-ruby-on-rails-app-to-the-jvm>
[^23]: 2012-07 Go at SoundCloud <https://developers.soundcloud.com/blog/go-at-soundcloud>
[^24]: 2014-04 Peter Bourgon @ SoundCloud: Go: Best Practices for Production Environments (GopherCon 2014) <http://peter.bourgon.org/go-in-production/>
[^25]: 2009-04 Heroku - Simultaneously Develop and Deploy Automatically Scalable Rails Applications in the Cloud <http://highscalability.com/blog/2009/4/24/heroku-simultaneously-develop-and-deploy-automatically-scala.html>
[^26]: 2021-07 GitHub’s Journey from Monolith to Microservices <https://www.infoq.com/articles/github-monolith-microservices/>
[^27]: 2017-12 Building Services at Airbnb, Part 1 <https://medium.com/airbnb-engineering/c4c1d8fa811b>
[^28]: 2020-11 Jessica Tai @ Airbnb: How to Tame Your Service APIs: Evolving Airbnb’s Architecture <https://www.infoq.com/presentations/airbnb-api-architecture/>
[^29]: 2013-08 Reddit: Lessons Learned from Mistakes Made Scaling to 1 Billion Pageviews a Month <http://highscalability.com/blog/2013/8/26/reddit-lessons-learned-from-mistakes-made-scaling-to-1-billi.html>
[^30]: 2012-03 7 Years of YouTube Scalability Lessons in 30 Minutes <http://highscalability.com/blog/2012/3/26/7-years-of-youtube-scalability-lessons-in-30-minutes.html>
[^31]: 2016-04 豆瓣田忠博：豆瓣的服务化体系改造 (QCon北京2016, slides, 37p) <http://2016.qconbeijing.com/presentation/2834/>
[^32]: 2010-09 Disqus: Scaling the World's Largest Django App (DjangoCon 2010, slides) <https://www.slideshare.net/zeeg/djangocon-2010-scaling-disqus>
[^33]: 2021-05 Dropbox: Atlas: Our journey from a Python monolith to a managed platform <https://dropbox.tech/infrastructure/atlas--our-journey-from-a-python-monolith-to-a-managed-platform>
[^34]: 2014-07 Dropbox: Open Sourcing Our Go Libraries <https://dropbox.tech/infrastructure/open-sourcing-our-go-libraries>
[^35]: 2017-07 Tammy Butow @ Dropbox: Go Reliability and Durability at Dropbox <https://about.sourcegraph.com/blog/go/go-reliability-and-durability-at-dropbox-tammy-butow>
[^36]: 2011-02 Adam D'Angelo: Why did Quora choose C++ over C for its high performance services? Does the Quora codebase use a limited subset of C++? <https://qr.ae/pKwCE3>
[^37]: 2012-02 A Short on the Pinterest Stack for Handling 3+ Million Users <http://highscalability.com/blog/2012/2/16/a-short-on-the-pinterest-stack-for-handling-3-million-users.html>
[^38]: 2015-08 Finagle and Java Service Framework at Pinterest (slides) <https://www.slideshare.net/yongshengwu/finaglecon2015pinterest>
[^39]: 2013-03 Instagram 5位传奇工程师背后的技术揭秘 <https://web.archive.org/web/0/http://www.csdn.net/article/2013-03-28/2814698-The-technologie-%20behind-Instagram>
[^40]: 2017-03 Lisa Guo: Scaling Instagram Infrastructure（QCon London 2017, 87p） <https://www.infoq.com/presentations/instagram-scale-infrastructure/>
[^41]: 2019-08 Static Analysis at Scale: An Instagram Story <https://instagram-engineering.com/8f498ab71a0c>
[^42]: 2016-07 The Uber Engineering Tech Stack, Part I: The Foundation <https://web.archive.org/web/0/https://eng.uber.com/tech-stack-part-one/>
[^43]: 2018-11 知乎社区核心业务 Golang 化实践 <https://zhuanlan.zhihu.com/p/48039838>
[^44]: 2022-10 字节马春辉：字节大规模微服务语言发展之路 <https://www.infoq.cn/article/n5hkjwfx1gxklkh8iham>
[^45]: 2021-07 字节成国柱：字节跳动微服务架构体系演进 <https://mp.weixin.qq.com/s/1dgCQXpeufgMTMq_32YKuQ>
[^46]: Why does Google use Java instead of Python for Gmail? <https://qr.ae/pKkduC>
[^47]: 2011-07 揭秘Google+技术架构 <http://www.infoq.com/cn/news/2011/07/Google-Plus>
[^48]: 2009-12 人人网架构师张洁：人人网使用的开源软件列表 <https://web.archive.org/web/0/http://ugc.renren.com/2009/12/13/a-list-of-open-source-software-in-renren>
[^49]: 2018-12 Netflix OSS and Spring Boot — Coming Full Circle <https://netflixtechblog.com/4855947713a0>
[^50]: 携程为什么突然技术转型从 .NET 转 Java？ <https://www.zhihu.com/question/56259195>
[^51]: 京东技术解密，2014，[豆瓣](https://book.douban.com/subject/26264767/)：12 少年派的奇幻漂流——从.Net到Java

[^52]: Golang Frequently Asked Questions: Why did you create a new language? <https://go.dev/doc/faq#creating_a_new_language>
[^53]: 2014-05 Brad Fitzpatrick: Go: 90% Perfect, 100% of the time: Fun vs. Fast <https://go.dev/talks/2014/gocon-tokyo.slide#28>
[^54]: 2021-08 百度Geek说：短视频go研发框架实践 <https://my.oschina.net/u/4939618/blog/5191598>
[^55]: 2023-07 百度Geek说：从 php5.6 到 golang1.19 - 文库 App 性能跃迁之路 <https://my.oschina.net/u/4939618/blog/10086661>
[^56]: 2022-03 2021年，腾讯研发人员增长41%，Go首次超越C++成为最热门语言 <https://mp.weixin.qq.com/s/zj-DhASG4S-3z56GTYjisg>
[^57]: 2020-05 Which Major Companies Use PostgreSQL? What Do They Use It for? <https://learnsql.com/blog/companies-that-use-postgresql-in-business/>
[^58]: 2009-05 Why is MySQL more popular than PostgreSQL? <https://news.ycombinator.com/item?id=619871>
[^59]: Why is MySQL more popular than PostgreSQL? <https://qr.ae/pKPJcE>

[^60]: 2002-10 Michael Radwin: Making the Case for PHP at Yahoo! (PHPCon 2002, slides) <https://web.archive.org/web/0/http://www.php-con.com/2002/view/session.php?s=1012> <https://speakerdeck.com/xy/making-the-case-for-php-at-yahoo>
[^61]: 2005-10 Michael Radwin: PHP at Yahoo! (PHP Conference 2005, slides) <https://speakerdeck.com/xy/php-at-yahoo-zend2005>
[^62]: 2007-06 Federico Feroldi: PHP in Yahoo! (slides) <https://www.slideshare.net/fullo/federico-feroldi-php-in-yahoo>

[^63]: 2021-02 AWS re:Invent 2020: Amazon’s architecture evolution and AWS strategy <https://www.youtube.com/watch?v=HtWKZSLLYTE>
[^64]: What programming languages are used at Amazon? <https://qr.ae/pKFwnw>
[^65]: 2011-04 Charlie Cheever: How did Google, Amazon, and the like initially develop and code their websites, databases, etc.? <https://qr.ae/pKKyB0>
[^66]: 2016-04 Is Amazon still using Perl Mason to render its content? <https://qr.ae/pKFwFm>
[^67]: 2019-10 Migration Complete – Amazon’s Consumer Business Just Turned off its Final Oracle Database <https://aws.Amazon/blogs/aws/migration-complete-amazons-consumer-business-just-turned-off-its-final-oracle-database/?nc1=h_ls>
[^68]: Amazon RDS for PostgreSQL customers <https://aws.Amazon/rds/postgresql/customers/?nc1=h_ls>
[^69]: Amazon ElastiCache for Redis customers <https://aws.Amazon/elasticache/redis/customers/?nc1=h_ls>

[^70]: 2006-11 Randy Shoup & Dan Pritchett: The eBay Architecture (SDForum2006, slides, 37p) <http://www.addsimplicity.com/downloads/eBaySDForum2006-11-29.pdf>
[^71]: 2007-05 eBay Architecture <http://highscalability.com/blog/2008/5/27/ebay-architecture.html>
[^72]: 2011-10 Tony Ng: eBay Architecture (slides, 46p) <https://www.slideshare.net/tcng3716/ebay-architecture>
[^73]: 2016-11 Ron Murphy: Microservices at eBay (slides, 20p) <https://www.modb.pro/doc/120378>
[^74]: 2020-09 High Efficiency Tool Platform for Framework Migration <https://innovation.ebayinc.com/tech/engineering/high-efficiency-tool-platform-for-framework-migration/>
[^75]: 2023-10 BES2：打造eBay下一代高可靠消息中间件 <https://mp.weixin.qq.com/s/ThhkO1WM7ck1WO8RqjTCJA>
[^76]: 2013-06 Cassandra at eBay - Cassandra Summit 2013 (slides) <https://www.slideshare.net/jaykumarpatel/cassandra-at-ebay-cassandra-summit-2013>
[^77]: 2017-03 Sudeep Kumar: 'Elasticsearch as a Service' at eBay <https://www.elastic.co/elasticon/conf/2017/sf/elasticsearch-as-a-service-at-ebay>
[^78]: 2016-08 How eBay Uses Apache Software to Reach Its Big Data Goals <https://www.linux.com/news/how-ebay-uses-apache-software-reach-its-big-data-goals/>

[^79]: 淘宝技术这十年，子柳，2013，[豆瓣](https://book.douban.com/subject/24335672/)
[^80]: 尽在双11：阿里巴巴技术演进与超越，2017，[豆瓣](https://book.douban.com/subject/26998040/)：第1章 阿里技术架构演进
[^81]: 2011-06 淘宝吴泽明范禹：淘宝业务发展及技术架构 (slides, 43p) <https://www.modb.pro/doc/116697>
[^82]: 2014-12 阿里云王宇德、张文生：淘宝迁云架构实践 <https://web.archive.org/web/1/http://www.csdn.net/article/a/2014-12-09/15821474>
[^83]: 2017-07 阿里谢吉宝唐三：阿里电商架构演变之路 (首届阿里巴巴中间件技术峰会, slides, 33p) <https://www.modb.pro/doc/121185>
[^84]: 2010-11 淘宝赵林丹臣：淘宝数据库架构演进历程 (iDataForum2010, slides, 38p) <https://www.slideshare.net/ssuser1646de/ss-10163048>
[^85]: 2019-10 阿里刘振飞：十年磨一剑：从2009启动“去IOE”工程到2019年OceanBase拿下TPC-C世界第一 <https://mp.weixin.qq.com/s/7B6rp17XVhpAWZr1-6DHqQ> <https://developer.aliyun.com/article/722414>
[^86]: 2016-02 SSD的30年发展史 <https://mp.weixin.qq.com/s/JsHKFilB5fvLY9V9z_xsXw>
[^87]: 2010-04 Yoshinori Matsunobu: SSD Deployment Strategies for MySQL (slides, 52p) <https://www.slideshare.net/matsunobu/ssd-deployment-strategies-for-mysql>
[^88]: 2020-04 阿里王剑英、和利：淘宝万亿级交易订单背后的存储引擎 <https://mp.weixin.qq.com/s/MkX1Pr8tERrzK29XG9zMUQ> <https://help.aliyun.com/zh/rds/apsaradb-rds-for-mysql/storage-engine-that-processes-trillions-of-taobao-order-records>
[^89]: 2022-03 阿里云罗庆超：阿里巴巴集团上云之 TFS 迁移 OSS 技术白皮书 <https://developer.aliyun.com/article/876301>
[^90]: 2016-12 阿里倪超：阿里巴巴Aliware十年微服务架构演进历程中的挑战与实践 <https://www.sohu.com/a/121588928_465959>
[^91]: 2017-10 阿里钟华古谦：企业IT架构转型之道 (杭州云栖大会2017, slides, 18p) <https://www.modb.pro/doc/121695>
[^92]: 2021-09 阿里郭浩：Dubbo 和 HSF 在阿里巴巴的实践：携手走向下一代云原生微服务 <https://mp.weixin.qq.com/s/_Ug3yEh9gz5mLE_ag1DjwA>
[^93]: 2012-11 阿里巴巴分布式服务框架 Dubbo 团队成员梁飞专访 <http://www.iteye.com/magazines/103>
[^94]: 2019-01 Dubbo 作者梁飞亲述：那些辉煌、沉寂与重生的故事 <https://www.infoq.cn/article/3F3Giujjo-QwSw2wEz7u>
[^95]: Dubbo 用户案例：阿里巴巴升级 Dubbo3 全面取代 HSF2 <https://cn.dubbo.apache.org/zh-cn/users/alibaba/> <https://github.com/apache/dubbo-website/blob/ee727525aa61d68b397cc6ddedb322001f0ca4da/content/zh/users/alibaba.md>
[^96]: 2023-01 阿里巴巴升级 Dubbo3 全面取代 HSF2 <https://cn.dubbo.apache.org/zh-cn/blog/2023/01/16/%e9%98%bf%e9%87%8c%e5%b7%b4%e5%b7%b4%e5%8d%87%e7%ba%a7-dubbo3-%e5%85%a8%e9%9d%a2%e5%8f%96%e4%bb%a3-hsf2/>
[^97]: 2017-11 阿里林清山隆基：阿里消息中间件架构演进之路：notify和metaq <https://zhuanlan.zhihu.com/p/302600352>
[^98]: 2013-07 淘宝张乐伟韩彰：淘宝消息中间件技术演变：MetaQ 1.0、MetaQ 2.0、MetaQ 3.0（slides, 30p）<https://www.modb.pro/doc/109298>
[^99]: 2017-03 阿里冯嘉鼬神：Apache RocketMQ背后的设计思路与最佳实践 <https://developer.aliyun.com/article/71889>

[^100]: 2008-05 LinkedIn - A Professional Network built with Java Technologies and Agile Practices (JavaOne2008, slides) <https://www.slideshare.net/linkedin/linkedins-communication-architecture>
[^101]: 2015-07 Josh Clemm: A Brief History of Scaling LinkedIn <https://engineering.linkedin.com/architecture/brief-history-scaling-linkedin> <https://www.slideshare.net/joshclemm/how-linkedin-scaled-a-brief-history>
[^102]: Aditya Auradkar, Chavdar Botev, etc.: Data Infrastructure at LinkedIn. ICDE 2012: 1370-1381，[dblp](https://dblp.org/rec/conf/icde/AuradkarBDMFGGGGHKKKLNNPPQQRSSSSSSSSTTVWWZZ12.html), [semanticscholar](https://www.semanticscholar.org/paper/a89f1ca0846f39e71b6b4452951f2fd19c2ae3b5), [slides](https://www.slideshare.net/amywtang/icde)
[^103]: 2012-06 Sid Anand: LinkedIn Data Infrastructure Slides (Version 2) (slides) <https://www.slideshare.net/r39132/linkedin-data-infrastructure-slides-version-2-13394853>
[^104]: 2021-12 Evolving LinkedIn’s analytics tech stack <https://engineering.linkedin.com/blog/2021/evolving-linkedin-s-analytics-tech-stack>

[^105]: 2010-05 Aditya Agarwal: Scale at Facebook (Qcon London 2010) <https://www.infoq.com/presentations/Scale-at-Facebook/>
[^106]: 2008-11 Aditya Agarwal: Facebook architecture (QCon SF 2008, slides, 34p) <https://www.slideshare.net/mysqlops/facebook-architecture> <https://www.infoq.com/presentations/Facebook-Software-Stack/>
[^107]: 2011-04 Michaël Figuière: What is Facebook's architecture? <https://qr.ae/pKYg12>
[^108]: 2013-10 Where does Facebook use C++? <https://qr.ae/pKCIee>
[^109]: 2022-07 Programming languages endorsed for server-side use at Meta <https://engineering.fb.com/2022/07/27/developer-tools/programming-languages-endorsed-for-server-side-use-at-meta/>
[^110]: 2015-07 Instagram: Search Architecture <https://instagram-engineering.com/eeb34a936d3a>
[^111]: Guoqiang Jerry Chen, Janet L. Wiener, etc.: Realtime Data Processing at Facebook. SIGMOD Conference 2016: 1087-1098，[dblp](https://dblp.org/rec/conf/sigmod/ChenWIJLSWWWY16.html), [semanticscholar](https://www.semanticscholar.org/paper/4ce25286205c62fffda7d685a916cf4508149245)
[^112]: 2021-10 XStream: stream processing platform at facebook (Flink Forward Global 2021, slides) <https://www.slideshare.net/aniketmokashi/xstream-stream-processing-platform-at-facebook>

[^113]: 2013-01 Johan Oskarsson: The Twitter stack <https://blog.oskarsson.nu/post/40196324612/the-twitter-stack>
[^114]: 2013-10 Chris Aniszczyk: Evolution of The Twitter Stack (slides) <https://www.slideshare.net/caniszczyk/twitter-opensourcestacklinuxcon2013>
[^115]: 2017-01 The Infrastructure Behind Twitter: Scale <https://blog.twitter.com/engineering/en_us/topics/infrastructure/2017/the-infrastructure-behind-twitter-scale>
[^116]: 2010-07 Cassandra at Twitter Today <https://blog.twitter.com/engineering/en_us/a/2010/cassandra-at-twitter-today>
[^117]: 2011-12 How Twitter Stores 250 Million Tweets a Day Using MySQL <http://highscalability.com/blog/2011/12/19/how-twitter-stores-250-million-tweets-a-day-using-mysql.html>
[^118]: 2013-11 Michael Busch: Search at Twitter (Lucene Revolution 2013, slides) <https://www.slideshare.net/lucenerevolution/twitter-search-lucenerevolutioneu2013-copy> <https://www.youtube.com/watch?v=AguWva8P_DI>
[^119]: 2022-10 Stability and scalability for search <https://blog.twitter.com/engineering/en_us/topics/infrastructure/2022/stability-and-scalability-for-search>
[^120]: 2021-10 Processing billions of events in real time at Twitter <https://blog.twitter.com/engineering/en_us/topics/infrastructure/2021/processing-billions-of-events-in-real-time-at-twitter->