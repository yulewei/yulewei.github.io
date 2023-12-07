---
title: 微服务 API 网关 Kong 实践 
date: 2020-05-30 15:22:41
categories: 架构
tags: [Kong, API, 微服务, Nginx, OpenResty, 网关, 架构]
---

# Kong 简介

Kong 是云原生、高效、可扩展、分布式的微服务抽象层，被称为 API 网关，或者 API 中间件。Kong 在 2015 年 4 月由 Mashape 公司开源，基于 OpenResty 和 Apache Cassandra/PostgreSQL 构建，提供易于使用的 RESTful API 来操作和配置 API 系统[^1][^2]。

<!--more-->

Mashape 是 API 集市，是为应用开发者与 API 提供者服务的 API 交易市场，Mashape 让发者能够方便地查找与购买 API，而 API 提供商则能轻松地销售与管理 API[^3]。随着 Mashape 市场上的 API 越来越多，原先基于 Node.js 实现的 API 代理不再适用，不能处理大流量尖峰，无法快速扩容。于是，寻找处理大流量的方案，同时需要保证可靠性和安全性，成为 Mashape 亟待解决的问题。2013 年，在 CloudFlare（当时 OpenResty 背后的公司）的工程师的建议下，Mashape 开始在 OpenResty 基础上开发 Kong 项目[^4]。Mashape 公司的名字，MashAPE，有人猿猩猩的含义，公司 logo 也是相应动物。类似的，Kong，对应的是，[King Kong](https://en.wikipedia.org/wiki/King_Kong)，就是电影里的金刚[^4]。在 Mashape 开启 Kong 项目两年后，2015 年 4 月，Mashape 公司开源了 Kong[^2]。

2017 年 5月，Mashape 和 RapidAPI 合并，组成全球最大的 API 集市[^5][^6]。5 个月后，Mashape, Inc. 改名为 [Kong Inc.](https://en.wikipedia.org/wiki/Kong_Inc%2E)，新的公司以 Kong 项目为聚焦，把全部工程师投入到 Kong 开发中，并且于此同时他们发布了 [Kong 企业版](https://konghq.com/products/kong-enterprise/)[^7]。

Kong，作为微服务的请求的网关，能通过插件提供负载均衡、日志记录、鉴权、限流、转换以及其他等功能。相对与旧的、没有使用网关的方式，Kong 把这些通用功能中心化，让微服务更加专注于业务本身。

<img width="800" alt="The Old way vs. The Kong Way" title="The Old way vs. The Kong Way" src="https://static.nullwy.me/kong-old-way-vs-kong-way.png">

Kong 的整体架构，如下图所示[^1]：

  - 管理 API：通过 RESTful API 管理 Kong；能自动化集成；管理 API 能通过插件扩展
  - 插件：使用 Lua 脚本创建 Plugins；实现强力的定制化；与第三方服务集成
  - 集群和数据存储：数据存储可选择 PostgreSQL 或 Cassandra；能从单节点扩展为集群；使用内存缓存提高性能
  - OpenResty：拦截请求/响应生命周期；基于 NGINX 扩展；Lua 脚本化
  - NGINX：验证过的高性能基础组件；HTTP 和反向代理服务器；处理底层操作
 
<img width="300" alt="Kong Architecture" title="Kong Architecture" src="https://static.nullwy.me/kong-architecture.png">

 
# Kong 安装

目前最新的 Kong 版本是 2.0.x，2.0 发布时间是 2020 年 1 月，而 1.0 发布时间是 2018 年 12 月[^8][^9]。笔者公司使用的 Kong 版本是 0.14.1，暂时未升级自最新版，所以下文阐述的 Kong 版本主要以 0.14.1 为准，并同时会提及其他版本的特性。

安装 Kong 很简单，参见[官方文档](https://konghq.com/install/)即可。在 [Ubuntu](https://docs.konghq.com/install/ubuntu/) 18.04 下安装 Kong 0.14.1，可以执行下面的命令：

``` bash
# kong 安装
$ sudo apt update
$ sudo apt install openssl libpcre3 procps perl
$ wget -O kong-community-edition-0.14.1.trusty.all.deb https://bintray.com/kong/kong-community-edition-deb/download_file?file_path=dists/kong-community-edition-0.14.1.trusty.all.deb
$ sudo dpkg -i kong-community-edition-0.14.1.trusty.all.deb
$ kong version
0.14.1
```

Kong 依赖数据库，Postgres 或者 Cassandra，默认依赖 Postgres（kong 1.1 开始支持无数据库声明式配置[^10]）。我们预先安装 Postgres：

``` bash
# 安装 postgresql
$ sudo apt install postgresql
$ sudo service postgresql start
$ psql --version
psql (PostgreSQL) 10.12 (Ubuntu 10.12-0ubuntu0.18.04.1)
```

在 Postgres 下添加 Kong 需要的的数据库实例和用户。下面的示例，创建数据库 `kong`，用户名 `kong`，密码为 `kong`：

``` bash
$ sudo -u postgres psql
postgres=# CREATE USER kong; CREATE DATABASE kong OWNER kong;
postgres=# ALTER USER kong WITH PASSWORD 'kong';
postgres=# \q
```

执行完成后，即可使用用户名为 `kong` 的用户连接 Postgres，`psql -h localhost -U kong -d kong`。

Kong 安装完成后，默认会创建配置文件 `/etc/kong/kong.conf.default`，这份配置文件在 [GitHub](https://github.com/Kong/kong/blob/0.14.1/kong.conf.default) 上也能找到，被注释掉的配置项，就是默认设置。

在启动 Kong 网关服务器前，我们参考 `kong.conf.default`，创建自己的 `kong.conf` 配置文件。我们把配置文件放在 `/home/yulewei/kong` 目录下，同时也把这目录当作为 Kong 的 `prefix` 目录。修改这配置 `kong.conf`，文件末尾添加：

```
prefix = /home/yulewei/kong/
pg_user=kong
pg_password=kong
pg_database = kong
```

使用 [`kong`](https://docs.konghq.com/0.14.x/cli/) 命令，启动 Kong 网关服务器：

``` bash
# 初始化或迁移数据库数据
$ kong migrations up -c /home/yulewei/kong/kong.conf
# 启动 kong
$ kong start -c /home/yulewei/kong/kong.conf
```

Kong 默认绑定 4 个端口：

  - `:8000` 用来接收来自客户端的 HTTP 流量的请求，并转发到上游服务
  - `:8443` 用来接收来自客户端的 HTTPS 流量的请求，并转发到上游服务
  - `:8001` 用来接收访问 [Admin API](https://docs.konghq.com/0.14.x/admin-api/) 的 HTTP 流量的请求
  - `:8444` 用来接收访问 Admin API 的 HTTPS 流量的请求

所以，可以执行下面的命令，来确认 Kong 是否正常运行：

``` bash
# 确认 Kong 是否正常运行
$ curl -i http://localhost:8000/
$ curl -i http://localhost:8001/
```

Kong 底层依赖 OpenResty，启动 Kong 后，可以看到 nginx 进程：

``` bash
# 查看 nginx 进程
$ ps -ef | grep nginx
yulewei  19090     1  0 16:05 ?        00:00:00 nginx: master process /usr/local/openresty/nginx/sbin/nginx -p /home/yulewei/kong -c nginx.conf
yulewei  19091 19090  0 16:05 ?        00:00:00 nginx: worker process
yulewei  19092 19090  0 16:05 ?        00:00:00 nginx: worker process
```

安装 Kong 0.14.1，自动安装的 OpenResty 版本是 1.13.6.2，OpenResty 捆绑的安装了 LuaJIT。

``` bash
$ /usr/local/openresty/nginx/sbin/nginx -v
nginx version: openresty/1.13.6.2

$ /usr/local/openresty/bin/resty -V
resty 0.21
nginx version: openresty/1.13.6.2
built by gcc 4.8.4 (Ubuntu 4.8.4-2ubuntu1~14.04.4)
built with OpenSSL 1.0.2n  7 Dec 2017
TLS SNI support enabled
configure arguments: ... 省略 ...

$ /usr/local/openresty/luajit/bin/luajit -v
LuaJIT 2.1.0-beta3 -- Copyright (C) 2005-2017 Mike Pall. http://luajit.org/
```

另外，同时也安装了 LuaRocks，LuaRocks 关联的是 OpenResty 捆绑的 LuaJIT。事实上，Kong 就是一个 LuaRocks 的 [rock](https://luarocks.org/modules/kong/kong) 包，在Kong 项目的 GitHub 上可以看到 [rockspec](https://github.com/Kong/kong/blob/0.14.1/kong-0.14.1-0.rockspec) 文件。Kong 的安装，底层实现上，是通过 `luarocks` 命令完成的，类似这样的命令，`luarocks install kong 0.14.1-0`[^11][^12]。可以使用 `luarocks show` 命令查看这个 Kong 的 rcok 包：

``` bash
$ luarocks show kong
kong 0.14.1-0 - Kong is a scalable and customizable API Management Layer built on top of Nginx.

License: 	MIT
Homepage: 	http://getkong.org
Installed in: 	/usr/local

Modules:
	kong (/usr/local/share/lua/5.1/kong/init.lua)
	kong.api (/usr/local/share/lua/5.1/kong/api/init.lua)
... 省略 ...
```

有个小细节值得注意，通过 `luarocks` 看到，Kong 采用的协议是 MIT。但事实上，Kong 0.5.0 开始协议从 MIT [改成了](https://github.com/Kong/kong/commit/9faa7f0e0648a037bb6a465bad08d5928a9ca31f) Apache 2.0。此处是一个小 bug，Kong 的 rockspec 文件没有及时更新，这个问题后来修复了，参见 [#4125](https://github.com/Kong/kong/pull/4125)。

## GUI 管理工具

管理 Kong 可以直接使用 Admin API，当然也有基于 Admin API 实现 GUI 管理工具。

Kong 官方的企业版提供了 GUI 管理工具，[`Kong Manager`](https://konghq.com/products/kong-enterprise/kong-manager/)（Kong EE 0.34 之前称为 [`Admin GUI`](https://docs.konghq.com/enterprise/0.33-x/admin-gui/overview/)），Kong 社区版没有提供 GUI 管理工具。

第三方的开源 GUI 工具，比较活跃的就是 [`Konga`](https://github.com/pantsel/konga)，值得推荐，如下图。

<img width="800" alt="konga" title="konga" src="https://static.nullwy.me/konga.png">

另外，还有其他的 GUI 工具，比如 [`Kong Dashboard`](https://github.com/PGBI/kong-dashboard)，也可以了解下。

# Kong 使用

Kong 核心概念：

- `Service`：对应位于 Kong 后方的自身的 `Upstream` API 或微服务。
- `Route`：Kong 的入口点，定义了如何把请求发送到特定 `Service` 的规则。一个 `Service` 可以有多个`Route`。
- `Plugin`：插件提供了模块化系统，用来修改或控制 Kong。插件提供了大量功能，比如访问控制、缓存、限流、日志记录等。
- `Consumer`：消费者，表示使用 API 的用户，能用来对用户进行访问控制、跟踪等。

Kong 网关的请求响应工作流，如下图所示：

<img width="700" alt="Kong Overview" title="Kong Overview" src="https://static.nullwy.me/kong-overview.png">


## 反向代理

Kong 的核心功能就是对现有的上游服务的 API 作反向代理。反向代理，官方的完整的文档参见[^13]。现在我们来试验下 Kong 的反向代理功能，执行下面的命令：

``` bash
# 添加 service
$ curl -XPOST -H 'Content-Type: application/json' \
     -d '{"name":"example.service","url":"http://httpbin.org"}' \
     http://localhost:8001/services/

# 在 service 上添加 route
$ curl -XPOST -H 'Content-Type: application/json' \
     -d '{"paths":["/base64"],"strip_path":false}' \
     http://localhost:8001/services/example.service/routes
```

上面的第一条命令，通过调用 Kong 提供的 Admin API，让 Kong 创建了名为 `example.service` 的 `service`，`service` 指向的上游服务是 `http://httpbin.org`。第二条命令，在 `example.service` 上添加 `route` 规则，规则是让请求路径前缀为 `/base64` 的请求转发到这个 `service`。来验证下，刚刚的 Kong 的配置：

``` bash
# 验证 Kong 配置结果
$ curl http://httpbin.org/base64/aGVsbG8ga29uZw==
hello kong
$ curl http://localhost:8000/base64/aGVsbG8ga29uZw==
hello kong
```

上文的 Kong 配置，等价的 `nginx.conf` 配置文件的写法是：

```
server {
  listen 8000;
  location /base64 {
    proxy_pass http://httpbin.org/base64;
  }
}
```

除了前缀外，`route` 规则的 `paths` 字段也支持 PCRE [正则表达式](https://docs.konghq.com/0.14.x/proxy/#using-regexes-in-paths)，来看下示例：

``` bash
curl -XPOST -H 'Content-Type: application/json' \
     -d '{"paths":["/status/\\d+"],"strip_path":false}' \
     http://localhost:8001/services/example.service/routes
```

上面的命令，添加 `route` 规则，设置的 `paths` 字段值为 `/status/\d+`，让只有请求路径的中包含数字才能匹配。

``` bash
# 验证 Kong 配置结果
$ curl -sI http://httpbin.org/status/418 | head -n1
HTTP/1.1 418 I'M A TEAPOT
$ curl -sI http://localhost:8000/status/418 | head -n1
HTTP/1.1 418 I'M A TEAPOT
$ curl -sI http://localhost:8000/status/200 | head -n1
HTTP/1.1 200 OK
$ curl http://localhost:8000/status/abc
{"message":"no route and no API found with those values"}
```

## 负载均衡

上文的反向代理指向的是单台的上游服务器，如果要指向多台上游服务器，实现负载均衡，要如何配置呢？负载均衡，Nginx 可以通过 `upstream` 指令实现，而类似的，Kong 通过创建 `upstream` 对象实现。

假设在服务器 `192.168.2.100:80` 和 `192.168.2.101:80` 上运行着本地版的 `httpbin.org` 的 REST API 服务（通过 `docker run -p 80:80 kennethreitz/httpbin`）。执行下面的命令：

``` bash
# 添加 upstream
curl -XPOST -H 'Content-Type: application/json' \
     -d '{"name":"example.upstream"}' \
     http://localhost:8001/upstreams/

# 在 upstream 上添加 target
curl -XPOST -H 'Content-Type: application/json' \
     -d '{"target":"192.168.2.100:80"}' \
     http://localhost:8001/upstreams/example.upstream/targets

# 在 upstream 上添加 target
curl -XPOST -H 'Content-Type: application/json' \
     -d '{"target":"192.168.2.101:80"}' \
     http://localhost:8001/upstreams/example.upstream/targets

# 添加 service
curl -XPOST -H 'Content-Type: application/json' \
     -d '{"name":"example.service","host":"example.upstream"}' \
     http://localhost:8001/services/

# 在 service 上添加 route
curl -XPOST -H 'Content-Type: application/json' \
     -d '{"paths":["/base64"],"strip_path":false}' \
     http://localhost:8001/services/example.service/routes
```

上面的命令，先创建了 `upstream` 对象，虚拟主机名（virtual hostname）为 `example.upstream` 。然后在这个 `upstream` 上添加 `target`，`192.168.2.100:80` 和 `192.168.2.101:80`。再然后把 `service` 对象的 `host` 字段值设置为  `example.upstream`。这样全部发送到这个 `service` 的请求都会被转发到 `example.upstream` 这个 `upstream`，`upstream` 再执行负载均衡算法，把请求转发到最终的上游服务器。和 Nginx 一样，默认的负载均衡算法为加权轮询算法（weighted-round-robin）。

``` bash
# 验证 Kong 配置结果
$ curl http://localhost:8000/base64/aGVsbG8ga29uZw==
hello kong
```

上文的 Kong 配置，等价的 `nginx.conf` 配置文件的写法是：

```
upstream example.upstream {
  server 192.168.2.100:80;
  server 192.168.2.101:80;
}
server {
  listen 8000;
  location /base64 {
    proxy_pass http://example.upstream/base64;
  }
}
```

关于 Kong 负载均衡的更多介绍，可以阅读官方文档[^14]，本文不再展开。

## 开启插件

Kong 提供了很多插件，官方整理维护的全部插件列表，可以在[官网](https://docs.konghq.com/hub/)上看到。全部插件分 8 大类：身份认证类插件（Authentication）、安全控制类插件（Security）、流量控制类插件（Traffic Control）、无服务器计算类插件（Serverless）、分析与监控类插件（Analytics & Monitoring）、协议转换类插件（Transformations）、日志记录类插件（Logging）、部署类插件（Deployment）。Kong 0.14.1 社区版默认绑定的预定义插件，全部 31 个，调用下面的 [Admin API](https://docs.konghq.com/0.14.x/admin-api/#plugin-object) 可以查看：

``` bash
# Kong 社区版全部默认绑定的插件，共 31 个
$ curl http://localhost:8001/plugins/enabled
{"enabled_plugins":["response-transformer","oauth2","acl","correlation-id","pre-function","jwt","cors","ip-restriction","basic-auth","key-auth","rate-limiting","request-transformer","http-log","file-log","hmac-auth","ldap-auth","datadog","tcp-log","zipkin","post-function","request-size-limiting","bot-detection","syslog","loggly","azure-functions","udp-log","response-ratelimiting","aws-lambda","statsd","prometheus","request-termination"]}
```

现在我们来试下 Kong 的 [basic-auth](https://docs.konghq.com/hub/kong-inc/basic-auth/) 插件，用来实现 [HTTP Basic 认证](https://en.wikipedia.org/wiki/Basic_access_authentication)（RFC 7617）。执行下面的命令，在上文的 `example.service` 的 `service` 上开启 `basic-auth` 插件：

``` bash
# 在 service 上开启 basic-auth 插件
$ curl -XPOST --data "name=basic-auth" \
       http://localhost:8001/services/example.service/plugins
```

这样全部到 `example.service` 的请求都需要进行 Basic 认证。再次请求之前的 `/base64` 接口，返回状态码 `401 Unauthorized`：

``` bash
# 接口 HTTP 状态码返回 401
$ curl -i http://localhost:8000/base64/aGVsbG8ga29uZw==
HTTP/1.1 401 Unauthorized
... 省略 ...

{"message":"Unauthorized"}
```

添加身份认证的凭证，添加 username/password：

``` bash
# 添加 consumer
$ curl -XPOST --data "username=Jason" \
       http://localhost:8001/consumers/
# 在 consumer 上添加 basic-auth 插件的凭证 username/password
$ curl -XPOST --data "username=test&password=123456" \
       http://localhost:8001/consumers/Jason/basic-auth
```

现在请求头上带上凭证，重新请求 `/base64` 接口，响应正常：

``` bash
$ curl -u 'test:123456' http://localhost:8000/base64/aGVsbG8ga29uZw==
hello kong
```

Kong 插件，除了绑定到 `service` 上外，也可以绑定在 `route` 和 `consumer` 上。如果开启插件时，`service`、`route` 或 `consumer` 全部都不关联，就是全局范围开启插件，插件会在全部请求上运行。全局范围上开启 `basic-auth` 插件，命令如下：

``` bash
# 全局范围上开启 basic-auth 插件
$ curl -XPOST --data "name=basic-auth" \
       http://localhost:8001/plugins
```

关于 Kong 插件的更多介绍，可以阅读[官方文档](https://docs.konghq.com/1.5.x/admin-api/#plugin-object)，本文不再展开。

# Kong 插件开发

Kong 基于 OpenResty，OpenResty 通过 [`ngx_http_lua_module`](https://github.com/openresty/lua-nginx-module) 模块实现了在 Nginx 中内嵌 Lua 脚本的能力。Kong 插件，使用 Lua 脚本实现，全部默认加载的预定义插件对应的 Lua 源码（包括上文提到的 basic-auth 插件），可以在 Kong 项目仓库的 `kong/plugins` 目录下看到。

除了能使用 Kong 预定义插件，我们可以根据 Kong 插件开发文档[^15]，开发自定义插件。

如何开发插件，文本不展开。Kong 官方提供了自定义插件的模板代码，源码参见项目 kong-plugin[^16]。另外，有兴趣也可以参考笔者提供的 Kong 自定义插件示例，源码参见 kong-plugin-demo[^17]。

值得注意的是，Kong 使用 Lua 的 [`rxi/classic`](https://github.com/rxi/classic) 模块来模拟 Lua 中的类，自定义 Kong 的插件时，实现 handler 需要继承 `BasePlugin` class，目前最新的文档还是采用这种写法。不过，Kong 1.2 开始，Kong 内部的预定义实现的插件，废弃了继承 `BasePlugin` class 的写法，参见 Pull Request [#4590](https://github.com/Kong/kong/pull/4590)，“plugins handlers do not have to inherit from BasePlugin anymore #4590”。去掉对 `BasePlugin` class 的继承后，在开启单个插件（key-auth）的场景下，压测 Kong 性能提升 6%，开启多个插件的场景，性能提升更高。

# 参考资料

[^1]: What is Kong? <https://konghq.com/about-kong/> 
[^2]: 2015-04 Mashape 开源 API 网关——Kong <https://www.infoq.cn/article/2015/04/kong/>
[^3]: 2012-07 打造大集市：API交易网站Mashape正式推出 <https://www.csdn.net/article/2012-07-31/2807936>
[^4]: 2015-10 How Mashape Manages Over 15,000 APIs & Microservices <https://stackshare.io/kong/how-mashape-manages-over-15000-apis-and-microservices>
[^5]: 2017-05 Mashape 和 RapidAPI 合并，组成全球最大的应用编程接口（API）集市！ <https://www.sohu.com/a/144114294_465914>
[^6]: 2017-05 The API Marketplace Joins RapidAPI <https://konghq.com/blog/the-api-marketplace-joins-rapidapi/>
[^7]: 2017-10 Welcome Kong Inc. A New Name, a New Product, a New Era. <https://konghq.com/blog/introducing-kong-inc/>
[^8]: 2018-12 Kong 1.0 GA <https://konghq.com/blog/kong-1-0-ga/>
[^9]: 2020-01 Kong Gateway 2.0 GA <https://konghq.com/blog/kong-gateway-2-0-0-released/>
[^10]: Documentation for Kong: DB-less and Declarative Configuration <https://docs.konghq.com/1.1.x/db-less-and-declarative-config/>
[^11]: Kong Installation: Compile Source <https://docs.konghq.com/install/source/>
[^12]: Build tools to package and release Kong <https://github.com/Kong/kong-build-tools>
[^13]: Documentation for Kong: Proxy Reference <https://docs.konghq.com/0.14.x/proxy/>
[^14]: Documentation for Kong: Load Balancing Reference <https://docs.konghq.com/0.14.x/loadbalancing/>
[^15]: Documentation for Kong: Plugin Development <https://docs.konghq.com/0.14.x/plugin-development/>
[^16]: Kong 官方自定义插件的模板代码 <https://github.com/Kong/kong-plugin>
[^17]: Kong 自定义插件示例 <https://github.com/yulewei/kong-plugin-demo>
