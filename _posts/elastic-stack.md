---
title: Elastic Stack 日志分析平台搭建笔记
date: 2019-01-14 14:58:20
categories: 架构
tags: [ELK, Elastic, 日志, 架构]
---

Elastic Stack（旧称 ELK Stack）是最受欢迎的开源日志平台 [ [ref](https://www.elastic.co/cn/solutions/logging) ]。Elastic Stack 由 Elasticsearch、Logstash、Kibana 和 Beats 四个组件组成：

- [Beats](https://www.elastic.co/cn/products/beats)，是轻量型采集器的平台，从边缘机器向 Logstash 和 Elasticsearch 发送数据。
- [Logstash](https://www.elastic.co/cn/products/logstash)，集中、转换和存储数据，是动态数据收集管道，拥有可扩展的插件生态系统，能够与 Elasticsearch 产生强大的协同作用。
- [Elasticsearch](https://www.elastic.co/cn/products/elasticsearch)，搜索、分析和存储您的数据，是基于 JSON 的分布式搜索和分析引擎，专为实现水平扩展、高可靠性和管理便捷性而设计。
- [Kibana](https://www.elastic.co/cn/products/kibana)，实现数据可视化，导览 Elastic Stack。能够以图表的形式呈现数据，并且具有可扩展的用户界面，供您全方位配置和管理 Elastic Stack。

<!--more-->

Elastic Stack 是逐步发展而来的，一开始只有 Elasticsearch，专注做搜索引擎，2013 年 1 月 Kibana 及其作者 Rashid Khan [加入](https://www.elastic.co/blog/welcome-drew-rashid) Elasticsearch 公司，同年 8 月 Logstash 及作者 Jordan Sissel 也[加入](https://www.elastic.co/blog/welcome-jordan-logstash)，原本的非官方的 ELK Stack，正式成为官方用语。2015 年 3 月，旧金山举行的第 1 届 Elastic{ON} 大会上，Elasticsearch 公司改名为 Elastic。两个月后，Packetbeat 项目也[加入](https://www.elastic.co/blog/welcome-packetbeat-tudor-monica) Elastic，Packetbeat 和 Filebeat（之前叫做 [Logstash-forwarder](https://github.com/elastic/logstash-forwarder)，由 Logstash 作者 Jordan Sissel 开发）项目被整合改造为 Beats。加上 Beats 以后，官方不知道如何将 “B” 和 E-L-K 组合在一起（用过 ELKB 或 BELK），ELK Stack 于是改名为 Elastic Stack，并在 2016 年 10 月正式发布 Elastic Stack 5.0 [ [ref1](https://www.elastic.co/cn/about/history-of-elasticsearch), [ref2](https://www.elastic.co/elk-stack), [ref3](https://www.elastic.co/cn/blog/elastic-stack-5-0-0-released) ]。

# 使用 Logstash

Logstash（[home](https://www.elastic.co/cn/products/logstash), [github](https://github.com/elastic/logstash)）最初是来自 Dreamhost 运维工程师 Jordan Sissel 的开源项目，是管理事件和日志的工具，能够用于采集数据，转换数据，然后将数据发送到您最喜欢的“存储库”中（比如 Elasticsearch）。Logstash 使用 JRuby 开发，2011 年 5 月发布 1.0 版本。2013 年 8 月 Jordan Sissel 带着 Logstash 加入 Elasticsearch 公司，Logstash 成为 Elastic Stack 一员。

在 Ubuntu 下[安装](https://www.elastic.co/guide/en/logstash/6.5/installing-logstash.html)、[启动](https://www.elastic.co/guide/en/logstash/6.5/running-logstash.html) Logstash 可以使用下面命令：

``` sh
$ sudo apt-get install logstash                # 安装 logstash
$ sudo systemctl start logstash.service        # 系统服务方式启动 logstash
$ /usr/share/logstash/bin/logstash --version   # 查看 logstash 版本
logstash 6.5.4
$ sudo vim /etc/logstash/logstash.yml          # 查看默认 logstash.yml
$ sudo vim /etc/logstash/logstash-sample.conf  # 查看示例 logstash-sample.conf
```

安装完成后，二进制文件在 `/usr/share/logstash/bin` 目录下，配置文件位于 `/etc/logstash` 目录，日志输出位于 `/var/log/logstash` 目录，其他详细的目录位置的分布情况，可以阅读[官方文档](https://www.elastic.co/guide/en/logstash/6.5/dir-layout.html)。

## 最简单的示例

Logstash 管道（pipeline）由 `input`、`filter` 和 `output` 三个元素组成，其中 `input` 和 `output` 必须设置，而 `filter` 可选。`input` 插件从数据源中消费数据，`filter` 插件按指定方式修改数据，`output` 插件将数据写入特定目标中 [ [doc](https://www.elastic.co/guide/en/logstash/6.5/first-event.html) ]。

<img width="600" alt="Logstash 管道" title="Logstash 管道" src="https://static.nullwy.me/basic_logstash_pipeline.png">

现在来试试 Logstash 下 Hello Wolrd 级别的示例，运行下面命令：

``` sh
$ sudo /usr/share/logstash/bin/logstash -e 'input { stdin { } } output { stdout {} }'
```

`-e` 选项能够使 Logstash 直接在[命令行](https://www.elastic.co/guide/en/logstash/6.5/running-logstash-command-line.html)中设置管道配置。示例中 `input` 插件为 [stdin](https://www.elastic.co/guide/en/logstash/6.5/plugins-inputs-stdin.html) （标准输入），`output` 插件是 [stdout](https://www.elastic.co/guide/en/logstash/6.5/plugins-outputs-stdout.html) （标准输出）。若在控制台输入 `hello world`，相应的控制台将输出：

``` json
{
      "@version" => "1",
          "host" => "ubuntu109",
    "@timestamp" => 2019-01-12T05:52:56.291Z,
       "message" => "hello world"
}
```

值得注意的是，输出内容编码格式默认为 `rubydebug`，使用 Ruby 的 "[awesome_print](https://github.com/awesome-print/awesome_print)" 库打印。另外，响应输出 `@timestamp` 字段的值为 `2019-01-12T05:52:56.291Z`，这是 [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) 时间格式，时区是 0（zero），和北京时间相差 8 个小时。

输出内容编码格式，可以通过 `codec` 指令指定。除了默认的 `rubydebug` 外，官方还支持其他 20 多种编码格式，参见 [doc](https://www.elastic.co/guide/en/logstash/6.5/codec-plugins.html)。若把编码格式改为 `json`，即 `stdout { codec => json }`，输出结果将变成：

``` json
{"message":"hello world","@version":"1","@timestamp":"2019-01-12T05:52:56.291Z","host":"ubuntu109"}
```

管道配置也可以保存到文件中，以 `*.conf` 作为文件后缀，比如保存为 `test-stdin-stdout.conf`：

``` conf
input {
  stdin { }
}
output {
  stdout { }
}
```

启动 logstash 时，`-f` 选项用于指定管道配置文件的路径：

``` sh
$ sudo /usr/share/logstash/bin/logstash -f ~/test-stdin-stdout.conf
```

默认情况下，在启动 logstash 后，若再修改管道配置文件，新的修改需要重启 logstash 才能加载生效。在开发调试时，不太方便。解决这个问题，可以使用 `-r` 命令行[选项](https://www.elastic.co/guide/en/logstash/6.5/running-logstash-command-line.html)。开启这个选项后，只要确定配置文件已经发生变更，便会自动重新加载配置文件。

## file 输入插件

上文日志是控制台输入的，但真实情况日志在日志文件中，要想使用 Logstash 读取日志文件，官方提供了 [file](https://www.elastic.co/guide/en/logstash/6.5/plugins-inputs-file.html) 输入插件。管道配置文件示例 `test-file-stdout.conf`，如下：

``` conf
input {
 file {
   path => ["/home/yulewei/test.log"]
   sincedb_path => "/dev/null"
   start_position => "beginning"
  }
}
filter {
}
output {
  stdout {
    codec => rubydebug
  }
}
```

启动 Logstash：

```
$ sudo /usr/share/logstash/bin/logstash -r -f ~/test-file-stdout.conf
```

配置文件中的 `path` 指令用于指定日志文件的路径。`start_position` 指令设置 Logstash 启动时读取日志文件的位置，可以设置为 `beginning` 或者 `end`，默认是 `end`，即文件末尾。

为了跟踪每个输入文件中已处理了哪些数据，Logstash 文件输入插件会使用名为 sincedb 的文件来记录现有位置。由于我们的配置用于开发，所以我们希望能够重复读取文件，并进而希望禁用 sincedb 文件。在 Linux 系统上，将 `sincedb_path` 指令设置为 “/dev/null” 即可禁用。若没有禁用，默认保存的 sincedb 文件将在 `/usr/share/logstash/data/plugins/inputs/file/` 目录下。


## grok 过滤插件

上文的例子，做的核心事情就是把日志行转换到 `message` 字段，并附加某些元数据，如 `@timestamp`。如果要想解析自己的日志，把非结构化日志转换结构换日志，有两个过滤器特别常用：[dissect](https://www.elastic.co/guide/en/logstash/6.5/plugins-filters-dissect.html) 会根据分界符来解析日志，而 [grok](https://www.elastic.co/guide/en/logstash/6.5/plugins-filters-grok.html) 则会根据正则表达式匹配来运行。

如果数据结构定义非常完善，dissect 过滤插件的运行效果非常好，而且运行速度非常快捷高效。同时，其也更加容易上手，对于不熟悉正则表达式的用户而言，更是如此。通常而言，grok 的功能更加强大，而且可以处理各种各样的数据。然而，正则表达式匹配会耗费更多资源，而且速度也会慢一些，如果未能正确进行优化的话，尤为如此。

现在先来看下 grok 过滤插件。grok 模式的基本语法是 `%{SYNTAX:SEMANTIC}`，`SYNTAX` 是用于匹配数据的模式（或正则表达式）名称，`SEMANTIC` 是标识符或字段名称。Logstash 提供了超过 120 种默认的 grok 模式，全部预定义的模式可以在 [github](https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns) 上找到。典型的预定义模式（非完整列表）[ [github](https://github.com/logstash-plugins/logstash-patterns-core/blob/v4.1.2/patterns/grok-patterns) ]：

- WORD - 匹配单个词汇的模式
- NUMBER - 匹配整数或浮点数（正值或负值均可）的模式
- POSINT - 匹配正整数的模式
- EMAILADDRESS - 邮箱地址
- IP - 匹配 IPv4 或 IPv6 IP 地址的模式
- URI - URI 地址
- TIMESTAMP_ISO8601 - [ISO8601](https://en.wikipedia.org/wiki/ISO_8601) 格式的时间
- NOTSPACE - 匹配非空格的任何内容的格式
- SPACE - 匹配任何数量的连续空格的模式
- DATA - 匹配任何数据类型的限定数量的模式
- GREEDYDATA - 匹配剩余所有数据的格式


比如，`3.44` 可以使用 `NUMBER` 模式进行匹配，`192.168.2.104` 可以使用 `IP` 模式。`%{NUMBER:num} %{IP:client}` 模式，将会用 `NUMBER` 模式把 `3.44` 识别为 `num` 字段，用 `IP` 模式把 `192.168.2.104` 识别为 `client` 字段。默认情况识别获得的字段值是字符串类型，grok 插件支持把类型转换为 `int` 或 `float`。要想把 `3.44` 转换为浮点数，可以使用 `%{NUMBER:num:float}`。

假设有如下日志内容：

``` txt
Will, yulewei@gmail.com, 42, 1024, 3.14
```

grok 匹配表达式可以写成这样：

```
%{WORD:name}, %{EMAILADDRESS:email}, %{NUMBER:num1}, %{NUMBER:num2:int}, %{NUMBER:pi:float}
```

即，在这一行日志中依次提取出，`name`、`email`、`num1`、`num2` 和 `pi` 字段。完整的 `filter` 过滤器配置的写法：

``` conf
filter {
  grok {
    match => { "message" => "%{WORD:name}, %{EMAILADDRESS:email}, %{NUMBER:num1}, %{NUMBER:num2:int}, %{NUMBER:pi:float}" }
  }
}
```

启动 logstash：

``` sh
$ sudo /usr/share/logstash/bin/logstash -r -f ~/test-file-grok-stdout.conf
```

控制台将输出：

``` json
{
      "@version" => "1",
       "message" => "Will, yulewei@gmail.com, 42, 1024, 3.14",
          "num1" => "42",
            "pi" => 3.14,
    "@timestamp" => 2019-01-12T08:12:49.581Z,
          "path" => "/home/yulewei/test-grok.log",
          "host" => "ubuntu109",
         "email" => "yulewei@gmail.com",
          "name" => "Will",
          "num2" => 1024
}
```

Logstash 提供了超过 120 种默认的 grok 模式，基本上满足大多数使用场景。若没有符合要求的预定义的模式，可以使用 [Oniguruma](https://www.elastic.co/guide/en/logstash/6.5/plugins-filters-grok.html#_regular_expressions) 语法指定正则表达式：

```
(?<field_name>the pattern here)
```

上文中的 `%{WORD:name}` 和 `%{EMAILADDRESS:email}`，等价的正则表达式的写法如下 [ [ref1](https://github.com/logstash-plugins/logstash-patterns-core/blob/v4.1.2/patterns/grok-patterns), [ref2](https://stackoverflow.com/a/8829363) ]：

```
(?<name>\w+)
(?<email>[a-zA-Z][a-zA-Z0-9_.+-=:]+@[0-9A-Za-z][0-9A-Za-z-]{0,62}\.[0-9A-Za-z][0-9A-Za-z-]{0,62})
```

调试 grok 匹配表达式容易出错，官方提供可视化的 [Grok Debugger](https://www.elastic.co/guide/en/kibana/6.5/grokdebugger-getting-started.html) 工具，提高调试效率。

<img width="800" alt="Grok Debugger" title="Grok Debugger" src="https://static.nullwy.me/kibana-grokdebugger.png">


### 解析 http 服务器日志

现在来看下，真实的日志解析案例，使用 grok 过滤插件解析 http 服务器日志。通用日志格式（[Common Log Format](https://en.wikipedia.org/wiki/Common_Log_Format)），是 http 服务器的标准的日志格式。对通用日志格式扩展，加上额外的 referer 和 user-agent 字段，称为组合日志格式（Combined Log Format）。两种日志格式包含的字段如下：

```
# 通用日志格式
%remote-host %ident %authuser %timestamp "%request" %status %bytes
# 组合日志格式
%remote-host %ident %authuser %timestamp "%request" %status %bytes "%referer" "%user-agent"
```

[Apache](https://httpd.apache.org/docs/2.4/logs.html#accesslog) 和 [Nginx](https://nginx.org/en/docs/http/ngx_http_log_module.html#log_format) 服务器默认的日志格式，采用的就是通用日志格式或者组合日志格式。解析这两种日志格式，Logstash 提供预定义模式 [COMMONAPACHELOG](https://github.com/logstash-plugins/logstash-patterns-core/blob/v4.1.2/patterns/httpd#L14) 和 [COMBINEDAPACHELOG](https://github.com/logstash-plugins/logstash-patterns-core/blob/v4.1.2/patterns/httpd#L15)。

典型的 nginx 日志例子：

``` log
192.168.2.104 - - [13/Jan/2019:02:01:15 +0800] "GET /images/avatar.png HTTP/1.1" 200 266975 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.13; rv:64.0) Gecko/20100101 Firefox/64.0"
```

管道配置文件示例 `test-file-grokhttp-stdout.conf`，如下：

``` conf
input {
  file {
    path => ["/var/log/nginx/access.log"]
    sincedb_path => "/dev/null"
    start_position => "beginning"
  }
}
filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
    remove_field => ["message"]
  }
}
output {
  stdout { }
}
```

示例中使用了 grok 的预定义模式 [COMBINEDAPACHELOG](https://github.com/logstash-plugins/logstash-patterns-core/blob/v4.1.2/patterns/httpd#L15)。另外，`remove_field` [指令](https://www.elastic.co/guide/en/logstash/6.5/plugins-filters-grok.html#plugins-filters-grok-remove_field)用于把输出事件中某字段删除，示例中是 `message` 字段。

启动 logstash：

``` sh
$ sudo /usr/share/logstash/bin/logstash -r -f ~/test-file-grokhttp-stdout.conf
```

解析获得的结构化数据，如下：

``` json
{
           "auth" => "-",
           "host" => "ubuntu109",
           "verb" => "GET",
       "clientip" => "192.168.2.104",
       "@version" => "1",
    "httpversion" => "1.1",
     "@timestamp" => 2019-01-13T13:35:52.983Z,
          "bytes" => "266975",
      "timestamp" => "13/Jan/2019:02:01:15 +0800",
        "request" => "/images/avatar.png",
       "response" => "200",
       "referrer" => "\"-\"",
           "path" => "/var/log/nginx/access.log",
          "agent" => "\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10.13; rv:64.0) Gecko/20100101 Firefox/64.0\"",
          "ident" => "-"
}
```

## dissect 过滤插件

和 grok 过滤插件使用正则表达式提取字段不同，dissect 过滤插件使用分界符切割来提取字段。由于没有使用正则表达式，运行速度相对 grok 快很多。使用 [dissect](https://www.elastic.co/guide/en/logstash/6.5/plugins-filters-dissect.html) 过滤插件时，需要指明提取字段的顺序，还要指明这些字段之间的分界符。过滤插件会对数据进行单次传输，并匹配模式中的分界符。同时，过滤插件还会将分界符之间的数据分配至指定字段。过滤插件不会对所提取数据的格式进行验证。 

现在看下示例，`test-dissect.log` 文件内容如下：

``` txt
Will, yulewei@gmail.com, 42, 1024, 3.14
```

dissect 匹配规则可以写成这样：

```
%{name}, %{email}, %{num1}, %{num2}, %{num3}
```

完整的配置文件，`test-file-dissect-stdout.conf`：

``` conf
input {
 file {
   path => ["/home/yulewei/test-dissect.log"]
   sincedb_path => "/dev/null"
   start_position => "beginning"
  }
}
filter {
  dissect {
    mapping => {
      "message" => "%{name}, %{email}, %{num1}, %{num2}, %{num3}"
    }
    convert_datatype => {
      "num2" => "int"
      "num3" => "float"
    }
  }
}
output {
  stdout { }
}
```


和 grok 插件一样，默认提取的字段是字符串类型。配置文件中的 `convert_datatype` 指令用于将类型转为 `int` 或 `float`。


启动 logstash：

``` sh
$ sudo /usr/share/logstash/bin/logstash -r -f ~/test-file-dissect-stdout.conf
```

输出结果：

``` json
{
          "host" => "ubuntu109",
          "num1" => "42",
       "message" => "Will, yulewei@gmail.com, 42, 1024, 3.14",
      "@version" => "1",
          "name" => "Will",
          "num2" => 1024,
    "@timestamp" => 2019-01-13T13:32:10.900Z,
          "path" => "/home/yulewei/test-dissect.log",
         "email" => "yulewei@gmail.com",
          "num3" => 3.14
}
```


# 输出到 Elasticsearch

上文举的例子全部都是，输出控制台，使用 `stdout` 输出插件，没有实用价值。Elastic Stack 的核心其实是 Elasticsearch，使用Elasticsearch 搜索和分析日志。想要将数据发送到 Elasticsearch，可以使用 [elasticsearch](https://www.elastic.co/guide/en/logstash/6.5/plugins-outputs-elasticsearch.html) 输出插件。

## 安装 Elasticsearch

如果没有安装 Elasticsearch，参考[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/deb.html)按步骤安装。唯一要注意的是，在执行 `apt install` 命令前，必须先添加 elastic 的软件源地址，否则无法正常启动。核心命令如下：

``` sh
$ sudo apt-get install elasticsearch           # 安装 elasticsearch
$ sudo systemctl start elasticsearch.service   # 系统服务方式启动 elasticsearch
$ curl http://localhost:9200/                  # 用 rest 接口查看 elasticsearch
{
  "name" : "1t9JXt5",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "yQgVsvupSqGCGQbwqnanIg",
  "version" : {
    "number" : "6.5.4",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "d2ef93d",
    "build_date" : "2018-12-17T21:17:40.758843Z",
    "build_snapshot" : false,
    "lucene_version" : "7.5.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

默认情况下 elasticsearch 服务器绑定的 ip 地址是回环地址 `127.0.0.1`，若想绑定特定 ip 地址，可以修改 `/etc/elasticsearch/elasticsearch.yml` 配置文件中的 [network.host](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/network.host.html) 选项：

``` yml
network.host: 192.168.2.109
```

修改完成并重启后，elasticsearch 服务器访问地址从 `http://localhost:9200/` 变成 `http://192.168.2.109:9200/`。


## 输出到 Elasticsearch

现在来看下 [elasticsearch](https://www.elastic.co/guide/en/logstash/6.5/plugins-outputs-elasticsearch.html) 输出插件。示例，`test-file-elasticsearch.conf`：

``` conf
input {
 file {
   path => ["/home/yulewei/test.log"]
   sincedb_path => "/dev/null"
   start_position => "beginning"
  }
}
output {
  elasticsearch {
    hosts => ["http://192.168.2.109:9200"]
    index => "logstash-%{+YYYY.MM.dd}"
  }
}
```

示例的 `elasticsearch` 输出插件使用了 `hosts` 和 `index` 指令。`hosts` 指令，用于指定 elasticsearch 服务器的地址。而 `index` 指令，用于指定 elasticsearch 索引的名称模式，该指令默认值为 `logstash-%{+YYYY.MM.dd}`。在字符串内部的 `%{...}`，是 Logstash 字符串插值语法，官方称之为 `sprintf format` [ [doc](https://www.elastic.co/guide/en/logstash/6.5/event-dependent-configuration.html) ]。`+YYYY.MM.dd`，用于指定 `@timestamp` 的格式化的格式。`logstash-%{+YYYY.MM.dd}`，格式化后最终生成的值可能将是 `logstash-2019.01.13`。

启动 logstash：

```
$ sudo /usr/share/logstash/bin/logstash -r -f ~/test-file-elasticsearch.conf
```

查看在 elasticsearch 上新创建的 `logstash-*` 索引以及从 logstash 同步过来的日志数据：

``` sh
$ curl http://192.168.2.109:9200/_cat/indices
yellow open logstash-2019.01.13       tFjc5xL_QYeNw4oqe4odeg 5 1     3 0 15.5kb 15.5kb
$ curl 'http://192.168.2.109:9200/logstash-*/_search?pretty' -H 'Content-Type: application/json' -d'{"size": 1}'
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 3,
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "logstash-2019.01.13",
        "_type" : "doc",
        "_id" : "RUHNRWgBLIBntI4FV8Rf",
        "_score" : 1.0,
        "_source" : {
          "path" : "/home/yulewei/test.log",
          "@timestamp" : "2019-01-13T06:01:51.303Z",
          "@version" : "1",
          "host" : "ubuntu109",
          "message" : "hello world"
        }
      }
    ]
  }
}
```

可以看到，新的索引名为 `logstash-2019.01.13`。同步过来的日志记录全部有 5 条，第 1 条日志的 `message` 内容是 `hello world`。


## 使用 Kibana

Kibana，能够对 Elasticsearch 中的数据进行可视化，是 Elastic Stack 的窗口。在 Ubuntu 下[安装](https://www.elastic.co/guide/en/kibana/6.5/deb.html) Kibana 可以使用下面命令：

``` sh
$ sudo apt-get install kibana          # 安装 kibana，当前最新版为 6.5.4
$ sudo systemctl start kibana.service  # 系统服务方式启动 kibana
```

默认配置下，kibana 服务访问地址是 `http://localhost:5601/`，连接的 elasticsearch 地址是 `http://localhost:9200`，这两个配置分别由 `server.host` 和 `elasticsearch.url` 控制 [ [doc](https://www.elastic.co/guide/en/kibana/6.5/settings.html) ]。上文尝试过把 elasticsearch 服务 ip 地址绑定到 `192.168.2.109`。现在来试下绑定 ip 地址到 kibana，编辑配置文件 `/etc/kibana/kibana.yml`，修改为：

``` yml
server.host: "192.168.2.109"
elasticsearch.url: "http://192.168.2.109:9200"
```

使用 `sudo systemctl restart kibana.service` 重启后，kibana 服务访问地址变成 `http://192.168.2.109:5601/`。

elasticsearch 服务器上存在索引 `logstash-2019.01.13`，要想把这个索引导入到 kibana，参考官方[教程](https://www.elastic.co/guide/en/kibana/6.5/tutorial-define-index.html)即可。点击 `Management` 菜单，然后创建索引模式（index pattern）。索引模式可以直接为 `logstash-2019.01.13`，这样匹配单个索引。若要匹配多个时间的 logstash 索引，可以使用通配符 `*`，比如 `logstash-*`。如果要匹配全部 2019 年 01 月的索引，可以写成 `logstash-2019.01*`。完成索引模式定义后，便可以在 `Discover` 菜单下查看索引，如图：

<img width="800" alt="Kibana" title="Kibana" src="https://static.nullwy.me/logstash-kibana.png">


# 使用 Filebeat

Filebeat，轻量型日志采集器 [ [home](https://www.elastic.co/cn/products/beats/filebeat) ]。其前身是由 Logstash 作者 Jordan Sissel 开发的 [Logstash Forwarder](https://github.com/elastic/logstash-forwarder)。Logstash Forwarder 项目因为和收购过来的 Packetbeat 项目功能相近，并且都是 Go 语言开发，就一起被整合改造为 Beats [ [ref](https://www.elastic.co/blog/beats-1-0-0) ]。

我们知道，Logstash 使用 JRuby 开发，运行依赖 JVM，会消耗较多的系统资源。为了减少系统资源（CPU、内存和网络）的使用，Logstash Forwarder 改用 Go 语言开发。另外，在功能上也做了精简，只做单一的数据传输，不像 Logstash 有数据过滤能力。Logstash 类似于功能多样的“瑞士军刀”，能提供从多个数据源加载数据的功能，使用各种强大的插件来处理日志，并提供将多个来源的输出数据进行存储的功能。简而言之，Logstash 提供数据 ETL（数据的提取、变换和加载）的功能；而 Beats 是轻量级的数据传输工具，能将数据传输到 Logstash 或 Elasticsearch 中，其间没有对数据进行任何转换 [ [Gupta2017](https://book.douban.com/subject/30326542/) ]。Filebeat 和 Logstash 的关系如下图所示 [ [logz.io](https://logz.io/blog/filebeat-vs-logstash/) ]：

<img width="600" alt="Filebeat 和 Logstash 的关系" title="Filebeat 和 Logstash 的关系" src="https://static.nullwy.me/filebeat-and-logstash.png">

安装 filebeat 很简单，参考[官方文档](https://www.elastic.co/guide/en/beats/filebeat/6.5/setup-repositories.html)即可，核心命令如下：

``` sh
$ sudo apt-get install filebeat           # 安装 filebeat
$ filebeat version                        # 查看 filebeat 版本
filebeat version 6.5.4 (amd64), libbeat 6.5.4 [bd8922f1c7e93d12b07e0b3f7d349e17107f7826 built 2018-12-17 20:22:29 +0000 UTC]
$ sudo systemctl start filebeat.service   # 系统服务方式启动 filebeat
```


## 输出到 Elasticsearch

修改 filebeat 配置文件 /etc/filebeat/[filebeat.yml](https://github.com/elastic/beats/blob/6.5/filebeat/filebeat.yml)，示例如下：

``` yml
filebeat.inputs:
- type: log
  paths:
    - /home/yulewei/test.log

output.elasticsearch:
  hosts: ["192.168.2.109:9200"]
```


`filebeat.inputs` 选项用于配置日志的[输入方式](https://www.elastic.co/guide/en/beats/filebeat/6.5/configuration-filebeat-options.html)。子选项 `type` [支持](https://www.elastic.co/guide/en/beats/filebeat/6.5/configuration-filebeat-options.html#filebeat-input-types) `log`、`stdin`、`redis`、`udp`、`tcp` 等，示例中使用了 `log`，表明从日志文件输入。

`output` 选项用于配置日志的[输出方式](https://www.elastic.co/guide/en/beats/filebeat/6.5/configuring-output.html)，配置支持 elasticsearch、logstash、kafka、redis、file、console 等，一次只能选择配置其中某一个。示例配置了 `output.elasticsearch.hosts`，指定日志输出目标 elasticsearch 的主机地址。`output.elasticsearch.index` 可以用来指定索引 index 名称模式，默认是 `filebeat-%{[beat.version]}-%{+yyyy.MM.dd}`（比如 `filebeat-6.5.4-2019.01.12`）[ [doc](https://www.elastic.co/guide/en/beats/filebeat/6.5/elasticsearch-output.html#index-option-es) ]。

完成 `filebeat.yml` 修改后，重启 filebeat，将可以看到，在 elasticsearch 上新创建的 `filebeat-*` 索引：

``` sh
$ curl http://192.168.2.109:9200/_cat/indices
yellow open filebeat-6.5.4-2019.01.12 B4JbQDnZQuK5XvsQ77uedA 3 1 11043 0  1.7mb  1.7mb
```

## 输出到 Logstash

上文的示例直接把 Filebeat 采集的日志传输到 Elasticsearch，日志数据并没有被解析或者转换。若想解析和转换日志，需要在Filebeat 和 Elasticsearch 中间引入 Logstash。现在看下把日志输出到 Logstash 的示例配置文件，`filebeat.yml` 示例：

``` yml
filebeat.inputs:
- type: log
  paths:
    - /home/yulewei/test.log

output.logstash:
  hosts: ["localhost:5044"]
```

`filebeat.inputs` 和上文的示例一样。不同的是，把 `output.elasticsearch.hosts` 改成了 `output.logstash.hosts`，指定日志输出目标 [Logstash](https://www.elastic.co/guide/en/beats/filebeat/6.5/logstash-output.html) 的主机地址。`5044` 这个端口是 Logstash 用于监听 Filebeat 的端口。

现在来看下 Logstash 的管道配置文件，示例 `test-beats-stdout.conf`：

``` conf
input {
  beats {
    port => 5044
  }
}
output {
  stdout {
    codec => rubydebug
  }
}
```

示例中，使用了 [beats](https://www.elastic.co/guide/en/logstash/6.5/plugins-inputs-beats.html) 输入插件，配置的端口就 `filebeat.yml` 中指定的 `5044`。输出插件为 `stdout`，即把 Logstash 采集到日志输出到控制台。

重启 filebeat 和 logstash：

``` sh
$ cat test.log                              # 查看日志文件内容
hello world
$ sudo systemctl restart filebeat.service   # 重启 filebeat
$ sudo /usr/share/logstash/bin/logstash -r -f ~/test-beats-stdout.conf
```

控制台输出：

``` json
{
          "tags" => [
        [0] "beats_input_codec_plain_applied"
    ],
        "source" => "/home/yulewei/test.log",
         "input" => {
        "type" => "log"
    },
       "message" => "hello world",
    "@timestamp" => 2019-01-12T13:32:05.131Z,
      "@version" => "1",
    "prospector" => {
        "type" => "log"
    },
          "beat" => {
        "hostname" => "ubuntu109",
         "version" => "6.5.4",
            "name" => "ubuntu109"
    },
        "offset" => 0,
          "host" => {
                   "os" => {
             "version" => "16.04.4 LTS (Xenial Xerus)",
            "platform" => "ubuntu",
            "codename" => "xenial",
              "family" => "debian"
        },
         "architecture" => "x86_64",
                   "id" => "29b1bf39547d4ca9ae26c3b7656ff9e3",
        "containerized" => false,
                 "name" => "ubuntu109"
    }
}
```

# 集成 Filebeat, Logstash, Elasticsearch, Kibana

真实场景下，日志文件可能分布在多台服务器上，同一台服务器上也可能分布着不同来源类型的日志。现在我们来尝试下，使用 Filebeat 把两个日志文件各自采集到两个不同的 Elasticsearch 索引中，并用 Kibana 可视化。有两个日志文件 `test-beats1.log` 和 `test-beats2.log`，内容如下：

``` sh
$ cat test-beats1.log
hello world1
hello world1
$ cat test-beats2.log
hello world2
```

`filebeat.yml` 配置示例：

``` yml
filebeat.inputs:
- type: log
  paths:
    - /home/yulewei/test-beats1.log
  fields:
    log_type: test1
- type: log 
  paths:
    - /home/yulewei/test-beats2.log
  fields:
    log_type: test2

output.logstash:
  hosts: ["localhost:5044"]
```

示例配置文件使用了 `filebeat.inputs.fields` 选项，[fields](https://www.elastic.co/guide/en/beats/filebeat/6.5/filebeat-input-log.html#filebeat-input-log-fields) 选项用于在日志事件输出中添加字段。添加的字段名可以任意指定，示例中名为 `log_type`。因为现在在 filebeat 配置中同时导入两个日志文件，输出到同一个 logstash 中。使用这个额外字段是为了区分日志是来自 `test-beats1.log` 还是 `test-beats2.log`。示例中，第 1 个日志事件输出的 `log_type` 字段值配置为 `test1`， 第 2 个日志配置为 `test2`。

管道配置文件示例，`test-beats-elasticsearch.conf`：

``` conf
input {
  beats {
    port => 5044
  }
}
output {
  elasticsearch {
    hosts => ["http://192.168.2.109:9200"]
    index => "filebeat-%{[fields][log_type]}-%{+YYYY.MM.dd}"
  }
  stdout {
    codec => rubydebug
  }
}
```

输出插件同时使用了 `elasticsearch` 和 `stdout`。配置文件中的 `elasticsearch` 输出插件的 `index` 指令被设置为 `filebeat-%{[fields][log_type]}-%{+YYYY.MM.dd}`。`[fields][log_type]` 引用的是在 `filebeat.yml` 的 `filebeat.inputs.fields` 选项添加的 `log_type` 字段（关于在配置文件引用字段的语法，可以参考[官方文档](https://www.elastic.co/guide/en/logstash/6.5/event-dependent-configuration.html#logstash-config-field-references)）。根据 `log_type` 字段不同，把日志将输出到不同的索引中。因为 `filebeat.yml` 配置文件中设置的 `log_type` 字段是 `test1` 或者 `test2`，所以最终生成的索引名是 `filebeat-test1-*` 或者 `filebeat-test1-*`。`filebeat-test1-*` 索引中全部日志数据来自 `test-beats1.log` 日志文件，`filebeat-test2-*` 索引数据来自 `test-beats2.log`。

启动 filebeat 和 logstash：

``` sh
$ sudo systemctl restart filebeat.service
$ sudo /usr/share/logstash/bin/logstash -r -f ~/test-beats-elasticsearch.conf
```

控制台输出：

``` json
...
{
      "@version" => "1",
          "host" => {
        "name" => "ubuntu109"
    },
       "message" => "hello world2",
    "prospector" => {
        "type" => "log"
    },
        "fields" => {
        "log_type" => "test2"
    },
        "offset" => 0,
          "tags" => [
        [0] "beats_input_codec_plain_applied"
    ],
          "beat" => {
            "name" => "ubuntu109",
         "version" => "6.5.4",
        "hostname" => "ubuntu109"
    },
    "@timestamp" => 2019-01-13T09:32:11.845Z,
        "source" => "/home/yulewei/test-beats2.log",
         "input" => {
        "type" => "log"
    }
}
...
```

查看在 elasticsearch 上新创建的 `filebeat-test1-*` 和 `filebeat-test1-*` 索引：

``` sh
$ curl http://192.168.2.109:9200/_cat/indices/filebeat-*
yellow open filebeat-test1-2019.01.13 NLVfFJl5TQ-7I1r9KoVLaQ 5 1 5 0 31.8kb 31.8kb
yellow open filebeat-test2-2019.01.13 NnsBp3P9Q3-mLc8chE-Tiw 5 1 3 0 24.1kb 24.1kb
```

在 kibana 上查看收集的日志：

<img width="800" alt="Kibana" title="Kibana" src="https://static.nullwy.me/filebeat-kibana.png">

整体架构上，如下图所示 [ [doc](https://www.elastic.co/guide/en/logstash/6.8/deploying-and-scaling.html) ]：

<img width="800" alt="Deploying Logstash" title="Deploying Logstash" src="https://static.nullwy.me/logstash-deploy2.png">

---

**附注**：本文中提到的配置文件，可以在 github 上访问得到，[elastic-stack-conf](https://github.com/yulewei/elastic-stack-conf)。

# 参考资料

- 发展历程| Elastic <https://www.elastic.co/cn/about/history-of-elasticsearch>
- 2013-01 Welcome Drew & Rashid (Kibana) <https://www.elastic.co/blog/welcome-drew-rashid>
- 2013-08 Welcome Jordan & Logstash <https://www.elastic.co/blog/welcome-jordan-logstash>
- 2015-05 Welcome Packetbeat, Tudor & Monica <https://www.elastic.co/blog/welcome-packetbeat-tudor-monica>
- 2015-11 The Beats 1.0.0 <https://www.elastic.co/blog/beats-1-0-0>
- 2016-02 Heya, Elastic Stack and X-Pack <https://www.elastic.co/blog/heya-elastic-stack-and-x-pack>
- 2016-10 Elastic Stack 5.0 正式发布 <https://www.elastic.co/cn/blog/elastic-stack-5-0-0-released>
- 2018-05 官方：Logstash 实用介绍 <https://www.elastic.co/cn/blog/a-practical-introduction-to-logstash>
- Filebeat vs. Logstash — The Evolution of a Log Shipper <https://logz.io/blog/filebeat-vs-logstash/>
- 精通Elastic Stack，Gupta，2017 <https://book.douban.com/subject/30326542/>
- logstash - open source log management <https://web.archive.org/web/20150512135526/http://logstash.net/>
- Logstash Config Language <https://web.archive.org/web/20150907161920/http://logstash.net/docs/1.4.2/configuration>
- Accessing Event Data and Fields in the Configuration <https://www.elastic.co/guide/en/logstash/6.5/event-dependent-configuration.html>
- Logstash Configuration Examples <https://www.elastic.co/guide/en/logstash/6.5/config-examples.html>
- Logstash Reference: Deploying and Scaling Logstash <https://www.elastic.co/guide/en/logstash/6.8/deploying-and-scaling.html>

