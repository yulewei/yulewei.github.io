---
title: I/O 多路复用与网络服务器并发策略
date: 2023-07-26 00:44:00
categories: 架构
tags: [IO, 服务器, 网络, 架构, 并发, Nginx]
---

目前主流的网络服务器，网络 I/O 相关的底层最核心的技术都是 I/O 多路复用（I/O Multiplexing），比如 Apache HTTP Server、Nginx、Redis 等。本文尝试解释各种 I/O 模型，包括解释什么是 I/O 多路复用，同时也总结 I/O 多路复用底层的系统调用 select、poll、kqueue 和 epoll 的演进和区别，并编写了使用这些函数的示例代码。另外，本文还总结了各种基于 I/O 多路复用实现的网络服务器的并发策略的三种模式，包括对 Apache HTTP Server、Nginx 和 Redis 等网络服务器的并发策略的具体案例的解析。

<!--more-->

# I/O 模型与多路复用

类 Unix 系统下的 I/O 操作，默认是**阻塞 I/O（Blocking I/O，缩写为 BIO）**。比如，当一个进程发出了读操作请求，但没有可访问的数据时，该进程通常会阻塞在内核中，直到出现可以访问的数据为止。然而，进程有时要处理对多个描述符的 I/O 操作，需要在多个文件描述符上阻塞，典型的场景是终端 I/O 和网络 I/O。

例如，有一个远程登录程序，它要从键盘读入数据然后把这些数据通过套接字发送到一个远程的计算机上。这个程序还需要从和远程终端相连接的套接字上读取数据，并将数据显示于屏幕上。如果进程在读键盘数据时阻塞，它就不能读那些从远程终端发送到屏幕上的数据。这样一来，在来自远程终端的更多数据到达之前，用户就不知道该通过键盘输入些什么，于是，死锁便产生了。相反的，如果进程在读从远程终端送来的数据时阻塞，它将不能读来自键盘终端的数据。

历史上，**Unix 系统通过使用多个进程让应用能同时处理多个文件描述符**，这些进程间可以通过管道或者是其他的进程间通信方法进行通信。然而，如果处理上下文切换的代价比处理输入的代价更大，那么这种方法就会导致巨大的系统开销，因为它要求在进程间进行频繁的上下文切换。并且，在一个进程内实现这种应用会显得比较直观。由于上述原因，BSD 提供了三种机制，允许对描述符进行多路 I/O 访问，非阻塞式 I/O、 I/O 多路复用和信号驱动 I/O[^1]：

- **非阻塞 I/O（Nonblocking I/O，缩写为 NIO）**：进程对描述符尝试 I/O 操作，如果描述符未就绪（ready），系统不把本进程投入睡眠，而是返回一个错误（`EAGAIN` 或 `EWOULDBLOCK`）。进程收到错误后，要么放弃，要么不停地轮询（polling），直至发现有描述符可以进行 I/O 操作为止。这种轮询的方法的问题在于，进程必须连续不断地运行，检查描述符是否就绪，很浪费 CPU 时间。
- **I/O 多路复用（I/O Multiplexing）**：让系统提供一种方法在多个感兴趣的描述符中查找哪个描述符可以进行 I/O 操作，如果没有这样的描述符，系统会让进程处于睡眠状态，直到出现这样的描述符为止。这种方法的缺点是对于每个操作，进程要完成两个系统调用，一个用来查找就绪描述符，另一个是 I/O 操作本身。**非阻塞 I/O 是在用户空间轮询查找就绪描述符，而 I/O 多路复用是执行特殊的系统调用在内核空间查找就绪描述符。**
- **信号驱动 I/O（Signal-Driven I/O）**：当可以进行 I/O 操作时，使所有描述符发信号。进程只用等待这些信号就可以知道是否可以进行 I/O 操作。这种方法的缺点在于捕获这些信号的开销是巨大的，所以此方法对于那些涉及大量 I/O 操作的应用并不实用。

类 Unix 系统下，默认的 I/O 操作都是阻塞 I/O。有两种方法可以将描述符设置非阻塞 I/O：(1) 如果是调用 [open](https://man7.org/linux/man-pages/man2/open.2.html)() 获得描述符，则可以在调用时设置 `O_NONBLOCK` 标志；(2) 对于已经打开的一个描述符，则可调用 [fcntl](https://man7.org/linux/man-pages/man2/fcntl.2.html)()，由该函数为描述符设置 `O_NONBLOCK` 标志。另外，对于网络套接字的描述符，如果想在获得描述符时直接指定为非阻塞 I/O，可以在调用 [socket](https://man7.org/linux/man-pages/man2/socket.2.html)() 或 [accept](https://man7.org/linux/man-pages/man2/accept.2.html)() 时传入 `SOCK_NONBLOCK` 标志，当然也可以在获得描述符后，再调用 `fcntl()`修改。

I/O 多路复用，最早是在 4.2BSD（1983.08）中由 [select](https://man.freebsd.org/cgi/man.cgi?query=select&apropos=0&sektion=2&manpath=4.3BSD+Reno&arch=default&format=html)() 系统调用提供的。虽然该系统调用主要用于终端 I/O 和网络 I/O，但它对其他描述符同样是起作用的。`poll()` 是另外一个实现 I/O 多路复用的系统调用，和 `select()` 功能几乎相同。SVR3（1987）在增加 STREAMS 机制时增加了 [poll](https://man.freebsd.org/cgi/man.cgi?query=poll&apropos=0&sektion=2&manpath=NetBSD+1.3&arch=default&format=html)() 系统调用。但在 SVR4 （1988）之前，`poll()` 只对 STREAMS 设备起作用。SVR4 开始支持对任意描述符起作用的 `poll()`。**`select()` 和 `poll()` 系统调用，都是在 POSIX.1-2001 开始标准化定义，然而从可移植性角度考虑，支持 `select()` 的系统比支持 `poll()` 的系统要多，所以在应用的实现上，相比于 `poll()` 基于 `select()` 实现更多**。另外 POSIX 还定义了 `pselect()`，它是能够处理信号阻塞并提供了更高时间分辨率的 `select()` 的增强版本。

在 Linux 系统下，`poll()` 系统调用从 2.1.23 版本（1997.01）开始提供，而 `poll()` 库函数由 libc 5.4.28（1997.05）开始提供。早期 Linux 内核未提供 `poll()` 系统调用，glibc 使用 `select()` 来模拟实现 `poll()`。另外，Linux 还提供特有的 I/O 多路复用解决方案，即 `epoll`，详细介绍参见下文。

为了能持续不断的监听 I/O 操作就绪事件，应用实现上需要循环调用 `select()`或 `poll()`。为了方便使用，封装各个不同的 I/O 多路复用函数的第三方库，通常会把这样的循环调用被抽象为**事件循环（event loop）**，然后把 I/O 就绪事件的处理抽象成回调函数（callback）。最早的提供事件循环（event loop）抽象的典型的第三方库是 [libevent](https://libevent.org/) 库（最早在 2002.04 发布）。

信号驱动 I/O，在描述符就绪时内核会发送 `SIGIO` 信号。但是信号驱动 I/O 对于 TCP 套接字近乎无用，问题在于 `SIGIO` 信号产生得过于频繁，并且它的出现并没有告诉我们发生了什么事件，无法区分触发信号的各种情况。在 UDP 上使用信号驱动式 I/O 没有上述问题。关于信号驱动 I/O 的详细阐述，可以参阅《UNIX网络编程 卷1》的第 25 章[^2]。

## 描述符就绪条件

`select()` 和 `poll()` 系统调用是在多个文件描述符中查找就绪（ready）的描述符。就绪条件具体指是什么呢？`select()` 的 [man](https://man7.org/linux/man-pages/man2/select.2.html) 文档，有如下描述（`poll()` 的就绪条件类似，不展开讨论）：

> A file descriptor is considered ready **if it is possible to perform a corresponding I/O operation (e.g., read(2), or a sufficiently small write(2)) without blocking**.
> ...
> A file descriptor is ready for **reading** if a read operation will **not block**; in particular, a file descriptor is also ready on **end-of-file**.
> A file descriptor is ready for **writing** if a write operation will **not block**. However, even if a file descriptor indicates as writable, a large write may still block.

针对网络套接字描述符的就绪条件，《UNIX网络编程 卷1》如下总结：

<img width="600" alt="套接字描述的就绪条件小结" title="套接字描述的就绪条件小结" src="https://static.nullwy.me/unix-socket-fd-ready.png">

表中的“有数据可读”含义是，该套接字接收缓冲区中的数据字节数大于等于套接字接收缓冲区低水位标记的当前大小。对这样的套接字执行读操作不会阻塞并将返回一个大于 0 的值（也就是返回准备好读入的数据）。接收低水位标记，可以通过调用 [setsockopt](https://man.freebsd.org/cgi/man.cgi?query=setsockopt&apropos=0&sektion=2&manpath=FreeBSD+14.0-CURRENT&arch=default&format=html)() 的 `SO_RCVLOWAT` 选项来设置，默认值为 1。

表中的“有可用于写的空间”含义是，该套接字发送缓冲区中的可用空间字节数大于等于套接字发送缓冲区低水位标记的当前大小，并且或者该套接字已连接，或者该套接字不需要连接（如UDP套接字）。发送低水位标记，可以通过调用 [setsockopt](https://man.freebsd.org/cgi/man.cgi?query=setsockopt&apropos=0&sektion=2&manpath=FreeBSD+14.0-CURRENT&arch=default&format=html)() 的 `SO_SNDLOWAT` 选项来设置，默认值为 1024。

表中的“关闭连接的读一半”和“关闭连接的写一半”含义是，套接字的 TCP 连接接收了关闭 FIN，此时会收到读就绪事件和写就绪事件。对这样的套接字做读操作将不阻塞并返回 0（也就是返回 EOF）；对这样的套接字做写操作将产生 `SIGPIPE` 信号（Broken pipe: write to pipe with no readers）。

## I/O 模型的比较

上文阐述的就是 Unix 系统的 4 种 I/O 模型，**阻塞 I/O、非阻塞 I/O、I/O 多路复用和信号驱动式 I/O**。另外，还有一种 I/O 模型是，**异步 I/O（Asynchronous I/O，缩写为 AIO）**。异步 I/O，由 POSIX 规范定义，工作机制是，告知内核启动某个操作，并让内核在整个操作（包括将数据从内核复制到我们自己的缓冲区）完成后通知进程。这种模型与信号驱动模型的主要区别在于：信号驱动 I/O 是由内核通知我们何时可以启动一个 I/O 操作，而异步 I/O 模型是由内核通知我们 I/O 操作何时完成。POSIX 定义的异步 I/O 的函数为，`aio_write()`、`aio_read()` 等。

关于 POSIX 异步 IO，Linux 的 aio 的 [man](https://man7.org/linux/man-pages/man7/aio.7.html) 文档，有如下说明：

> The current Linux POSIX AIO implementation is provided in user space by glibc. This has a number of limitations, most notably that maintaining multiple threads to perform I/O operations is expensive and scales poorly. Work has been in progress for some time on a kernel state-machine-based implementation of asynchronous I/O (see io_submit(2), io_setup(2), io_cancel(2), io_destroy(2), io_getevents(2)), but this implementation hasn't yet matured to the point where the POSIX AIO implementation can be completely reimplemented using the kernel system calls.

本质上，Linux 下的 POSIX AIO 是在用户空间下用线程模拟实现的 AIO，并非真正的 AIO，性能很差，所以很少被使用。

Linux 内核实现的 AIO 是 [io_submit](https://man7.org/linux/man-pages/man2/io_submit.2.html)、[io_setup](https://man7.org/linux/man-pages/man2/io_setup.2.html)、[io_getevents](https://man7.org/linux/man-pages/man2/io_getevents.2.html) 等系统调用，也被成为“Linux Native AIO”，或者缩写为 KAIO（kernel AIO），从 Linux 2.5 开始支持（2001.11），这些系统调用对应的库函数由 `libaio` 库提供。但是 Linux Native AIO 几乎不可用，只适合以 `O_DIRECT` 方式做直接 IO（无缓存的 I/O）。如果真的实现了异步 AIO，`io_submit` 系统调用不应该阻塞，但是对缓存 I/O、网络访问、管道等，`io_submit` 会发生阻塞，整个操作将在 `io_submit` 系统调用期间执行，并且通过调用 `io_getevents`，I/O 操作完成结果可以立即访问，这样也就破坏了异步 I/O 的目的[^3] 。

最新的内核实现的 AIO 是 [io_uring](https://en.wikipedia.org/wiki/Io_uring)，已经被 Linux 5.1（2019.05）采纳。很多开源项目，比如 [libevent](https://github.com/libevent/libevent/issues/1019)、[libuv](https://github.com/libuv/libuv/pull/3952)、[Nginx](https://mailman.nginx.org/pipermail/nginx-devel/2020-November/013632.html)、[Redis](https://github.com/redis/redis/issues/10881) 等，都有打算支持或甚至已经支持 io_uring。io_uring 的杂类资料整理，可以参考“Awesome io_uring”[^4]。本文主要关注 I/O 多路复用，io_uring 不再展开讨论。

《UNIX网络编程 卷1》对这 5 种 I/O 模型做了对比[^2]：

<img width="800" alt="5 种 I/O 模型的比较" title="5 种 I/O 模型的比较" src="https://static.nullwy.me/unix-io-models.png">

可以看出，前 4 种模型的主要区别在于第一阶段（等待描述符就绪），因为它们的第二阶段是一样的：在数据从内核复制到调用者的缓冲区期间，进程阻塞于 [recvfrom](https://man7.org/linux/man-pages/man2/recv.2.html) 调用。相反，异步 I/O 模型在这两个阶段都要处理，从而不同于其他 4 种模型。

各个 I/O 模型，用户空间的应用与内核空间的交互过程如下图所示（信号驱动 I/O 实际场景较少使用，所以忽略）[^5]：

| <img width="350" alt="阻塞式 I/O" title="阻塞式 I/O" src="https://static.nullwy.me/unix-io-models-bio.gif"> | <img width="350" alt="非阻塞式 I/O" title="非阻塞式 I/O" src="https://static.nullwy.me/unix-io-models-nio.gif"> |
|---|---|
| <img width="350" alt="非阻塞式 I/O + I/O 多路复用" title="非阻塞式 I/O + I/O 多路复用" src="https://static.nullwy.me/unix-io-models-multiplexing.gif"> | <img width="350" alt="异步 I/O" title="异步 I/O" src="https://static.nullwy.me/unix-io-models-aio.gif"> |

通常对“I/O 多路复用”术语的理解，其实就是特指，**由 select() 、poll() 或类似的系统调用实现的在多个文件描述符中查找就绪状态描述符的技术**。不过，根据 McKusick 书籍的描述[^1]，**I/O 多路复用也可以泛指为，单个进程同时处理多个文件描述符的技术，与之相对立的技术是早期的由多个进程同时处理多个描述符的解决方案。广义理解的话，I/O 多路复用包括非阻塞 I/O、狭义的 I/O 多路复用、信号驱动式 I/O、异步 IO 等技术**。

单独的“多路复用（[multiplexing](https://en.wikipedia.org/wiki/Multiplexing)）”术语，维基百科的解释是，一个通信和计算机网络领域的专业术语，多路复用通常表示在一个信道上传输多路信号或数据流的过程和技术。

## 阻塞、非阻塞与同步、异步的区别

在概念上，阻塞 I/O 和非阻塞 I/O，是根据系统**是否会阻塞进程的执行**而区分的：

- 阻塞 I/O，在执行 I/O 操作后，如果 I/O 操作的描述符未就绪，系统会让进程进入睡眠状态，直到描述符就绪为止。
- 非阻塞 I/O，在执行 I/O 操作后，不会阻塞当前进程，可以继续执行其他的任务。

另外，POSIX 定义了同步 I/O（Synchronous I/O）和异步 I/O（Asynchronous I/O）两个术语[^2]：
- 同步 I/O 操作，导致请求进程阻塞，直到 I/O 操作完成
- 异步 I/O 操作，不导致请求进程阻塞。

> A synchronous I/O operation causes the requesting process to be blocked until that I/O operation completes.
> An asynchronous I/O operation does not cause the requesting process to be blocked.

按照这个定义，阻塞 I/O、非阻塞 I/O、I/O 多路复用和信号驱动 I/O 都是同步 I/O，因为其中真正的 I/O 操作将阻塞进程（第二阶段，将数据从内核复制到用户空间的缓冲区阶段）。只有异步 I/O 模型与 POSIX 定义的异步 I/O 相匹配。

不过，对于 I/O 多路复用是属于同步 I/O 还是异步 I/O 存在争议，不同视角下存在不同的理解。I/O 多路复用，单看等待 I/O 就绪阶段，其实是异步的。所以很多时候，**I/O 多路复用，虽然没有实现真正的 POSIX 定义的异步 I/O，但也被归类为异步 I/O**。比如，Jones 的文章[^5]，**将 I/O 多路复用归类为阻塞的异步 I/O，将 POSIX AIO 归类为非阻塞的异步 I/O**。类似的，封装各个不同的 I/O 多路复用函数的 `libevent` 库的官方文档[^6]自称是“Asynchronous I/O”。`Node.js` 底层是基于 I/O 多路复用封装的 [libuv](https://libuv.org/) 库，`libuv` 库也自称是“Asynchronous I/O”。另外，如果跳出 I/O 视角，从整体应用的执行流程角度看，基于 I/O 多路复用实现的应用，相对于在多个描述符列表上主轮询流程，在单个描述符上 I/O 事件的应用处理流程相对独立，可以认为是异步的。Node.js 文档对“异步”的解释如下[^7]：

> Asynchronous means that things can happen independently of the main program flow.

严格意义上，典型的 I/O 多路复用的应用是单进程单线程执行的，本质上都是串行执行的，是假异步。

# 服务器并发策略与 C10K 问题

世界上第一个 HTTP 服务器，[CERN httpd](https://en.wikipedia.org/wiki/CERN_httpd)，早期实现采用的 I/O 模型是阻塞 I/O，然后为了能同时处理多个客户端连接，会为每个客户端连接创建一个新的处理请求的子进程（process-per-connection），这种并发模式被称为 **fork 模式**，这也是传统的 Unix 服务器采用的并发模式。之后的版本，改为基于 `select()` 的 I/O 多路复用 + fork 模式（参见源码 [HTDaemon.c](https://github.com/hackervera/cern-httpd/blob/master/Daemon/Implementation/HTDaemon.c#L2118)）。

[NCSA HTTPd](https://en.wikipedia.org/wiki/NCSA_HTTPd)，是早期的第一个流行的 HTTP 服务器，在版本 1.3 以及之前版本，也是采用阻塞 I/O + fork 模式实现，会为每个客户端连接创建一个新的子进程（参见源码 [httpd.c](https://github.com/TooDumbForAName/ncsa-httpd/blob/1.3/src/httpd.c#L160)）。之后 NCSA HTTPd 的 1.4 版本，I/O 模型改造为基于 select() 的 I/O 多路复用，进程模型改为“**pre-forking**”模式，但这种模式本质上还是每个客户端连接对应单个子进程（process-per-connection），“pre-forking”的优点是在创建新客户端连接时有预先创建的子进程直接处理请求（类似进程池），避免在创建新连接的同时执行创建子进程这样的重型操作，具体可以参见 1996.03 的官方文档的对“pre-forking”模型的性能测试[^8]（相关的实现源码参见 [httpd.c](https://github.com/TooDumbForAName/ncsa-httpd/blob/1.4.2/src/httpd.c#L508)）。

Apache HTTP Server，最早在 1995.04 对外公开发布首个版本 0.6.2，这个版本的代码基于 NCSA httpd 1.3[^9]。 之后在 5 月和 6 月，Apache 也开始实现“pre-forking”特性，在版本 0.8.0 开始正式支持[^9][^10]。一直到 Apache HTTP Server 2.2，[prefork](https://httpd.apache.org/docs/2.4/en/mod/prefork.html) 模式依然是 Unix 系统下的默认模式[^11]。**在 Apache 的 prefork 模式下，每个客户端连接由单独的子进程处理，这样的子进程被 Apache 称为 worker 进程，worker 进程数就是同时处理的客户端连接数**。因为线程相对进程更加轻量，理论上每个客户端连接对应单个线程（thread-per-connection）更有优势。所以，在 2002.04，Apache 发布 2.0 的首个 GA 版本时，新增了 [worker](https://httpd.apache.org/docs/2.4/mod/worker.html) 模式，一种多进程和多线程混合的模式。**在 Apache 的 worker 模式下，每个客户端连接由单独的子线程处理，这样的子线程也就是 worker 线程，worker 线程数就是同时处理的客户端连接数。**

相对单线程，多进程或多线程的问题是，占用更多内存（每个线程都需要维护自己的线程栈），以及频繁的上下文切换。所以，有些 HTTP 服务器倾向于基于 `select()` 实现单进程单线程的网络服务器。这种模式实现的服务器一般会把循环调用 `select()` 的过程抽象为事件循环（event loop），把 I/O 操作就绪事件的处理抽象成回调函数（callback），所以也被称为**事件驱动服务器（event-driven server）**。多核 CPU 时，为了能充分使用 CPU 多核资源，事件驱动服务器的进程数（或线程数）通常为 CPU 核数。事件驱动模式，在软件架构中被也称为 [Reactor](https://en.wikipedia.org/wiki/Reactor_pattern) 模式。基于事件并发和基于线程的并发的比较，可以阅读 [John Ousterhout](https://en.wikipedia.org/wiki/John_Ousterhout) 的 1995 年的经典 slides：“Why Threads Are A Bad Idea (for most purposes)”[^12]。

早期的典型的基于 `select()` 实现的单线程 HTTP 服务器的例子，是由 ACME 实验室开发并开源的 [thttpd](https://en.wikipedia.org/wiki/Thttpd)（1995.11 对外发布 [1.0](https://web.archive.org/web/19970124214241if_/http://www.acme.com:80/software/thttpd/thttpd_100.tar.Z) 版）。thttpd 服务器作者 Jef Poskanzer 在文章“Web Server Comparisons”（1998.07）[^13]中对比了各个 Web 服务器。根据文章的对比，容易发现基于 `select()` 实现的单线程服务器，在响应性能和最大并发连接数上都占优，thttpd 支持的最大的每秒请求数 QPS 是 720，thttpd 支持的最大并发连接数是 1000+。

虽然在实验条件下表现良好，但是在真实场景下，基于 `select()` 实现的 HTTP 单线程服务器，性能并没有优于传统的基于 fork 模式的服务器[^14]。Banga 等人经过分析后得出的主要原因是，当服务器同时处理的客户端连接数超过几千后，系统调用 `select()` 或 `poll()` 的性能很差，不具备可伸缩性。

最早的 HTTP 1.0 协议，在服务响应完成后连接会立即关闭，连接无法保持。HTTP 底层是 TCP 协议，建立 TCP 连接需要经过三次握手的过程。**如果能复用 TCP 连接，同一个 HTTP 连接上的后续的 HTTP 请求就不用重新建立 TCP 连接，也就是能在同一个 HTTP 连接上支持多次 HTTP 请求和响应，这样 HTTP 性能也就得到了提高**。于是，HTTP 1.1 协议（RFC 2068，1997.01）开始[持久连接](https://en.wikipedia.org/wiki/HTTP_persistent_connection)（persistent connection），默认让 HTTP 连接“keep-alive”。关于 HTTP 持久连接的详细介绍，可以参考 RFC 2068 的“[8.1 Persistent Connections](https://datatracker.ietf.org/doc/html/rfc2068#section-8.1)”。

HTTP 协议支持持久连接后，也带来了另外一个问题，就是出现大量的冷链接（cold connection）。浏览器如果未主动关闭连接，停留在网页上，并且如果连接未超时，此时的连接虽然不活跃但会保持一段时间，这样的连接就是冷链接。同时，随着互联网的快速发展，访问网站的用户量不断上升，Web 服务器需要维持的链接数也不断上升，如何让服务支持更多客户端连接问题也愈发尖锐。当时 Web 服务器支持的最大并发连接数大致是 1K，于是 Dan Kegel 在 1999 年提出了 [C10k](https://en.wikipedia.org/wiki/C10k_problem) 问题，如何能让服务器支持 10K 的客户端连接，字母“C”代表的是“client connection”。在文章“The C10K problem”[^15]中，Dan Kegel 对 C10K 问题的描述如下：

> It's time for web servers to **handle ten thousand clients simultaneously**, don't you think? After all, the web is a big place now.

另外，值得一提的是，HTTP 1.0 和 HTTP 1.1 协议存在队头阻塞问题（HOL 阻塞，[head of line blocking](https://en.wikipedia.org/wiki/Head-of-line_blocking#In_HTTP)），为了避免队头阻塞，从而使网页能更快响应，大多数浏览器会为每个域名同时开启多个 HTTP 连接，通常是 6 个并发连接[^16]，结果导致 Web 服务器需要维持的链接数增加数倍。在发布 HTTP 1.1 协议的十几年后的 2012 年，HTTP 2.0 的首个草稿发布，而制定 HTTP/2 协议的最大的目标之一就是解决队头阻塞问题。

# select 和 poll 性能问题

当 Web 服务器需要同时处理大量客户端连接时，服务器的性能表现差，原因就出在系统调用 `select()` 和 `poll()` 的性能上，具体的问题有如下三点[^17]：

- 每次调用 `select()` 或 `poll()`，内核都必须检查所有被指定的文件描述符，看它们是否处于就绪状态。随着待检查的文件描述符数量的增加，调用耗时也随之线性增加。**若待检查的文件描述符数为 n，`select()` 或 `poll()` 的时间复杂度为 O(n)。**
- 每次调用 `select()` 或 `poll()`，程序都必须传递一个表示所有需要被检查的文件描述符的数据结构到内核，内核检查过描述符后，修改这个数据结构并返回给程序。（此外，对于 `select()` 来说，我们还必须在每次调用前初始化这个数据结构。）随着待检查的文件描述符数量的增加，传递给内核的数据结构大小也会随之增加。当检查大量文件描述符时，从用户空间到内核空间来回拷贝这个数据结构将占用大量的 CPU 时间。
- `select()` 或 `poll()` 调用完成后，程序必须检查返回的数据结构中的每个元素，以此查明哪个文件描述符处于就绪状态。

解决系统调用 `select()` 和 `poll()` 的性能问题，让服务器能同时处理大量连接，比较典型的解决方案是，FreeBSD 4.1（2000.07 发布）开始支持的 `kqueue` 系统调用 ，以及 Linux 2.5.44（2002.10 发布）开始支持的 `epoll` 系统调用。

# kqueue 和 epoll 系统调用

[kqueue](https://man.freebsd.org/cgi/man.cgi?query=kqueue&apropos=0&sektion=2&manpath=FreeBSD+14.0-CURRENT&arch=default&format=html) 相关的 API 主要涉及两个系统调用 `kqueue()` 和 `kevent()`：

- `kqueue()`：用于在内核空间创建 `kqueue` 数据结构
- `kevent()`：
   - 当传入其中的 `changelist` 等参数时，用于将感兴趣的 `kevent` 事件对象注册到 `kqueue`，`kevent` 对象上记录感兴趣文件描述符和事件类型
   - 当传入其中的 `eventlist` 等参数时，用于查询就绪的 `kevent` 事件对象列表

**kqueue 的实现原理**[^18]：调用 `kqueue()` 创建由内核空间维护 `kqueue` 实例，`kqueue` 实例内包含链表，链表上保存全部监听的 `kevent` 事件对象，`kevent` 对象上感兴趣的记录文件描述符和事件类型。通过 `kevent()` 系统调用，可以在链表上注册、删除某 `kevent` 对象。当设备 I/O 事件触发时，设备与 `kqueue` 实例关联的钩子函数（hook）会被执行，钩子函数会判断事件是否与监听的事件相符合，如果符合就把事件添加到 `kqueue` 实例内下链表 `active list` 的末尾。查询就绪事件列表时，调用 `kevent()`，内核只需要检查链表 `active list` 是否有元素，若有就把就绪事件列表拷贝到用户空间。

[epoll](https://man7.org/linux/man-pages/man7/epoll.7.html) 相关的 API 主要涉及三个系统调用 `epoll_create()`、`epoll_ctl()` 和 `epoll_wait()`：

- `epoll_create()`：用于在内核空间创建 `epoll` 实例
- `epoll_ctl()`：用于添加感兴趣的 `epoll_event` 事件对象到 `epoll` 实例，`epoll_event` 对象上记录感兴趣文件描述符和事件类型
- `epoll_wait()`：用于查询就绪的 `epoll_event` 事件对象列表

**epoll 的实现原理**：调用 `epoll_create()` 创建由内核空间维护的 `epoll` 实例，epoll 实例内包含红黑树，红黑树上保存全部监听的 `epoll_event` 事件对象，`epoll_event` 对象上记录感兴趣的文件描述符和事件类型。通过 `epoll_ctl()` 系统调用，可以在红黑树上注册、删除某 `epoll_event` 对象。所有添加到红黑树中的事件都会与设备驱动程序建立回调关系，当 I/O 就绪事件触发时，会把事件添加到 `epoll` 实例内的链表 `rdllist`。查询就绪事件列表时，调用 `epoll_wait()`，内核只需要检查链表 `rdllist` 是否有元素，若有就把就绪事件列表拷贝到用户空间。

## 水平触发和边缘触发

`epoll` 系统调用的事件通知模式，区分水平触发（level-triggered，LT）和边缘触发（edge-triggered，ET），默认通知模式是水平触发 LT，`EPOLLET` 标志可以将通知模式改为边缘触发。`poll()` 和 `select()` 所提供的通知模式是水平触发，不支持边缘触发。水平触发和边缘触发的通知模式的含义如下[^17]：

- **水平触发通知模式**：如果文件描述符上可以非阻塞地执行 I/O 系统调用，此时认为它已经就绪。水平触发模式下，应用程序可以不立即处理该事件，当应用程序下一次调用 epoll_wait() 时，epoll_wait() 还会再次向应用程序通告此事件，直到该事件被处理。这种模式下 epoll 相当于一个效率较高的 poll。
- **边缘触发通知模式**：如果文件描述符自上次状态检查以来有了新的 I/O 事件，此时需要触发通知。也就是说，当 epoll_wait() 检测到某 I/O 事件发生并将此事件通知应用程序后，后续的 epoll_wait() 调用将不再向应用程序通知这一事件，只通知新的 I/O 事件。边缘触发模式在很大程度上降低了同一个 epoll 事件被重复触发的次数，因此效率要比水平触发模式高。但是相对水平触发，边缘触发模式下开发难度更大。

“水平触发”和“边缘触发”术语源于电子工程领域。水平触发是只要有状态发生就触发。边缘触发是只有在状态改变的时候才会发生。条件触发关心的是事件状态，边缘触发关心的是事件本身。

**采用边缘触发通知的程序通常要按照如下规则来设计：**

- 在接收到一个 I/O 事件通知后，程序在某个时刻应该在相应的文件描述符上尽可能多地执行 I/O（比如尽可能多地读取字节）。如果程序没这么做，那么就可能失去执行 I/O 的机会。
- 如果尽可能多地执行 I/O，而文件描述符被设置为阻塞模式，那么最终当没有更多的 I/O 可执行时，I/O 系统调用就会阻塞。所以，每个被检查的文件描述符通常都应该设置为非阻塞模式。

另外，在边缘触发 ET 模式下，如果多个线程同时监听相同的描述符，只会有一个线程被唤醒用来处理 I/O 事件。epoll 的 [man](https://man7.org/linux/man-pages/man7/epoll.7.html) 文档对这个特性有如下描述，这个特性也避免了“惊群问题”（[thundering herd problem](https://en.wikipedia.org/wiki/Thundering_herd_problem)）。

> If multiple threads (or processes, if child processes have inherited the epoll file descriptor across fork(2)) are blocked in epoll_wait(2) waiting on the same epoll file descriptor and a file descriptor in the interest list that is marked for edge-triggered (EPOLLET) notification becomes ready, just one of the threads (or processes) is awoken from epoll_wait(2). This provides a useful optimization for avoiding "thundering herd" wake-ups in some scenarios.

因此，边缘触发通知模式其中一个适用的场景是，多核 CPU 上的多线程服务器，每个 CPU 核上运行一个线程，这些线程同时监听相同的描述符[^19]。

`kqueue` 文档没有使用水平触发和边缘触发术语。但接口效果上，默认是水平触发。开启`EV_CLEAR`标志可以达到类似边缘触发的效果。`EV_CLEAR` 标志的 [man](https://man.freebsd.org/cgi/man.cgi?query=kqueue&apropos=0&sektion=2&manpath=FreeBSD+14.0-CURRENT&arch=default&format=html) 文档描述：

> After the event is retrieved by the user, its state is reset.

因为边缘触发通知模式效率更高，Nginx 服务器采用的就是边缘触发，参见源码 [ngx_epoll_module.c](https://github.com/nginx/nginx/blob/release-1.25.0/src/event/modules/ngx_epoll_module.c#L292) 和 [ngx_kqueue_module.c](https://github.com/nginx/nginx/blob/release-1.25.0/src/event/modules/ngx_kqueue_module.c#L238)。

# select、poll、kqueue 和 epoll 的比较

`select`、`poll`、`kqueue` 和 `epoll` 系统调用的多个维度的对比总结，如下表：

| **系统调用** | **select** | **poll** | **kqueue** | **epoll** |
| --- | --- | --- | --- | --- |
| **类 Unix 系统的支持情况** | POSIX 标准。最早 4.2BSD 提供（1983) | POSIX 标准。最早 SVR3 提供（1987) | BSD 专有。最早 FreeBSD 4.1 提供 （2000.08） | Linux 专有。最早 Linux 2.5.44 提供（2002.10） |
| **查询就绪描述符的时间复杂度** | O(n) | O(n) | O(1) | O(1) |
| **感兴趣描述符列表传递** | 每次 select() 都全量拷贝到内核空间 | 每次 poll() 都全量拷贝到内核空间 | 由内核空间维护 | 由内核空间维护 |
| **就绪描述符列表的返回** | 只返回就绪描述符数量，需要检查感兴趣描述符列表来判断哪些是就绪描述符 | 只返回就绪描述符数量，需要检查感兴趣描述符列表来判断哪些是就绪描述符 | 返回就绪事件数量，并同时返回就绪事件列表 | 返回就绪事件数量，并同时返回就绪事件列表 |
| **最大描述符数** | 被常数 `FD_SETSIZE` 限制（值为 1024） | 无限制 | 无限制 | 无限制 |
| **触发通知模式** | 水平 | 水平 | 水平和边缘 | 水平和边缘 |

**附注**：查询就绪描述符的时间复杂度，`select` 和 `poll` 都是 O(n)，n 为感兴趣描述符的总数，因为内核实现上需要轮询全部感兴趣的描述符列表。`kqueue` 和 `epoll` 都是 O(1)，实际的查询耗时与就绪描述符总数线性有关，但真实场景下就绪描述符数量相对描述符总数很小，可以认为是常数，所以复杂度是 O(1)。

总体上，`select` 和 `poll` 之间大同小异，而 `kqueue` 和 `epoll` 之间也是大同小异。

[libevent](https://libevent.org/) 库，对系统调用 `select`、`poll`、`kqueue` 和 `epoll` 做了性能基准测试，如下图所示（[图片来源](https://monkey.org/~provos/libevent/libevent-benchmark2.jpg)）。基准测试声明了大量连接（文件描述符），大多数连接是冷的，只有少数是活跃的。测试衡量的是，在不同的总连接数下，为 100 个活动连接提供服务所需的时间。可以看到，系统调用 `select` 和 `poll`，随着文件描述符的增加，耗时也随之线性增加，2500 个文件描述符时耗时大约 20ms，5000 个文件描述符时耗时大约 40ms，10000 个文件描述符时耗时大约 80ms。而系统调用 `kqueue` 和 `epoll`，耗时始终在 3ms ~ 5ms 之间。

<img width="700" alt="性能基准测试：select vs poll vs kqueue vs epoll" title="性能基准测试：select vs poll vs kqueue vs epoll" src="https://static.nullwy.me/libevent-benchmark2.png">

# echo 服务的简单示例代码

上文总结了 `select`、`poll`、`kqueue` 和 `epoll` 的接口特性和实现原理，但对具体应该如何使用这些函数没有切身感受。**笔者使用 I/O 多路复用函数 select、poll、epoll 和 kqueue 以及 libevent 库，各自编写了 echo 服务的简单示例代码**。所谓 echo 服务，即服务端接收到客户端的字符串输入，然后响应相同的字符串（为了方便区分响应字符串加了 `> ` 前缀）。比如，如果客户端输入字符串 `hello`，服务端将响应字符串 `> hello`；如果客户端输入字符串 `world`，服务端将响应字符串 `> world`。完整的示例代码参见 [io-multiplexing-demo](https://github.com/yulewei/io-multiplexing-demo)。

# Nginx 服务器的并发策略解析

上文介绍了 NCSA HTTPd、Apache HTTP Server 和 thttpd 等 Web 服务器的并发策略。主要的并发策略有三种模式：单连接单进程模式、单连接单线程模式和单线程的事件驱动模式。

[Nginx](https://en.wikipedia.org/wiki/Nginx) 最早是 2002 年开始开发的，2004.08 采用 BSD 协议对外开源首个版本 [0.1.0](https://github.com/nginx/nginx/tree/release-0.1.0)，开发 Nginx 的目的是为了解决 C10k 问题[^20]。2002 年，当时 FreeBSD 已经提供 `kqueue` 系统调用，而 Linux 的 `epoll` 即将正式发布，新的 `kqueue` 和 `epoll` 系统调用让 Nginx 解决 C10k 问题成为可能。根据 w3techs 的统计，在 2013.07 Nginx 超越 Apache 成为 top 1000 网站使用最多的 Web 服务器[^21]。

Nginx 采用的是**事件驱动架构**，在单线程的进程上执行事件循环，以异步非阻塞的方式处理 I/O 操作事件，事件循环底层基于高效的 `epoll` 或 `kqueue` 实现的 I/O 多路复用[^22]。

Nginx 服务器，区分 Master 进程和 Worker 进程。Master 进程，用于加载配置文件、启动 Worker 进程和平滑升级等。Worker 进程，是单线程的进程，用于执行事件循环，并以非阻塞方式处理 I/O 操作，因此单个 Worker 进程就能并发处理大量连接。一个完整的请求完全由 Worker 进程来处理，而且只在一个 Worker 进程中处理。为了能充分利用多核 CPU 资源，通常生产环境配置的 Worker 进程数量等于 CPU 核心数。Nginx 的架构图如下[^22]：

<img width="700" alt="Nginx 架构图" title="Nginx 架构图" src="https://static.nullwy.me/nginx-architecture.png">

# Redis 服务器的并发策略解析

Redis 是内存数据库，处理网络请求也是采用**单线程的事件驱动模式**，底层基于高效的 `epoll` 或 `kqueue` 实现的 I/O 多路复用。事件循环处理的事件主要有，建立客户端新连接事件、客户端连接的缓冲区可读事件、客户端连接的缓冲区可写事件。Redis 的命令处理过程如下：

- 在收到建立客户端新连接事件后，会在新建立的客户端套接字上监听可读事件，用于等待客户端发起命令请求。客户端连接可能会一直保持，处理之后的多个客户端命令请求。
- 如果监听到客户端连接的缓冲区可读事件，也就是收到客户端的命令请求，服务器会读取命令、解析命令，然后执行命令，最后把命令响应结果输出到内存缓冲区。值得注意的是，命令响应结果输出到内存缓冲区，但并未输出客户端连接的缓冲区。
- 在等到开启新的事件循环时，Redis 会在等待接收新的 I/O 事件之前，统一将全部内存缓冲区的命令响应结果输出到各个客户端。当命令响应结果数据量非常大时，无法一次性将所有数据都发送给某客户端，这时就会监听该客户端缓冲区可写事件。
- 如果监听到客户端连接的缓冲区可写事件，Redis 就会发送剩余部分的数据给客户端。

上述的命令处理过程，在源码层面上涉及的核心代码都在 `networking.c` 中：处理客户端新连接建立的事件的回调函数是 [acceptTcpHandler](https://github.com/redis/redis/blob/4.0.14/src/networking.c#L695)，处理客户端命令请求事件的回调函数 [readQueryFromClient](https://github.com/redis/redis/blob/4.0.14/src/networking.c#L1391)，命令响应结果输出到各个客户端对应的函数是 [handleClientsWithPendingWrites](https://github.com/redis/redis/blob/4.0.14/src/networking.c#L1011)，处理客户端的可写事件的回调函数是 [sendReplyToClient](https://github.com/redis/redis/blob/4.0.14/src/networking.c#L1004)。更详细的实现原理解析，本文不再展开，可以自行深入阅读相关源代码或书籍资料。

Redis 与 Nginx 在并发策略上有不同的选择，Nginx 有多个 Worker 进程，每个 Worker 进程都运行自己的事件循环，而 **Redis 整体上只有一个事件循环，采用的是单线程架构**。这样的架构设计带来的问题就是 Redis 无法多核 CPU 并发。针对无法多核 CPU 并发问题，Redis 官方 FAQ 的推荐的解决方案是[^23]：在多核 CPU 的单台机器上启动多个 Redis 实例。Redis 作者 antirez，解释了选择单线程而不选择多线程的原因，主要是：在 Redis 的数据结构上实现并发控制太复杂，多线程编程降低开发速度并且导致 bug 修复困难[^24][^25]。采用单线程的原因，概况成一句话就是[^25]：

> There is less to gain, and a lot of complexity to add. 

不过，随着 Redis 版本的演进，部分逻辑已经改成了多线程实现，Redis 新增的多线程特性有三处，Redis 2.4 新增的异步磁盘 IO、Redis 4.0 新增的“[Lazy Freeing](https://github.com/redis/redis/blob/4.0.0/redis.conf#L603)”和 Redis 6.0 新增的“[Threaded I/O](https://github.com/redis/redis/blob/6.0.0/redis.conf#L972)”。**但整体设计上，还是可以认为 Redis 主要使用单线程设计，依然是单线程的事件循环，并以单线程的方式执行命令**（绝大多数命令，“Lazy Freeing”相关的命令除外）[^26][^27]。

# 参考资料

[^1]: FreeBSD操作系统设计与实现，McKusick，2004：6.4.5 描述符上的多路I/O操作
[^2]: Unix网络编程 卷1：套接字联网API，Stevens，第3版2003
[^3]: 2014-04 AIO User Guide: A description of how to use AIO <https://web.archive.org/web/0/http://code.google.com/p/kernel/wiki/AIOUserGuide>
[^4]: Awesome io_uring <https://github.com/espoal/awesome-iouring>
[^5]: 2006-08 M. Jones: Boost application performance using asynchronous I/O <https://developer.ibm.com/articles/l-async/>
[^6]: Learning Libevent: A tiny introduction to asynchronous IO <https://libevent.org/libevent-book/01_intro.html>
[^7]: Node.js: JavaScript Asynchronous Programming and Callbacks <https://nodejs.dev/en/learn/javascript-asynchronous-programming-and-callbacks/>
[^8]: 1995-04 NCSA httpd: Performance of Several HTTP Demons on an HP 735 Workstation <https://web.archive.org/web/0/http://www.ncsa.uiuc.edu/InformationServers/Performance/V1.4/report.html>
[^9]: About the Apache HTTP Server Project <https://httpd.apache.org/ABOUT_APACHE.html>
[^10]: Changes with Apache (12 Jun 1995: This release included modified versions of a lot of code from the Apache 0.6.4 public release, plus an early pre-forking patch codeveloped by Robert Thau and Rob Hartill.) <https://github.com/apache/httpd/blob/1.3.x/src/CHANGES#L9427>
[^11]: Apache HTTP Server Version 2.2: Multi-Processing Modules (MPMs) <https://httpd.apache.org/docs/2.2/en/mpm.html#defaults>
[^12]: 1995 John Ousterhout: Why Threads Are A Bad Idea (for most purposes) (slides) <http://www.cc.gatech.edu/classes/AY2010/cs4210_fall/papers/ousterhout-threads.pdf>
[^13]: 1998-07 Jef Poskanzer: Web Server Comparisons（thttpd 服务器作者） <http://www.acme.com/software/thttpd/benchmarks.html>
[^14]: 1998 Gaurav Banga, Jeffrey C. Mogul: **Scalable Kernel Performance for Internet Servers Under Realistic Loads**. USENIX Annual Technical Conference 1998 [dblp](https://dblp.uni-trier.de/rec/conf/usenix/BangaM98.html) [usenix.org](https://www.usenix.org/conference/1998-usenix-annual-technical-conference/scalable-kernel-performance-internet-servers)
[^15]: 1999-05 Dan Kegel: The C10K problem（最后更新时间 2011.07） <http://www.kegel.com/c10k.html>
[^16]: 2014-02 详解浏览器最大并发连接数 <https://web.archive.org/web/0/http://www.iefans.net/liulanqi-zuida-bingfa-lianjieshu>
[^17]: Linux Unix系统编程手册，Kerrisk 下册：第63章 其他备选的I/O模型
[^18]: 2001 Jonathan Lemon: **Kqueue - A Generic and Scalable Event Notification Facility**. USENIX Annual Technical Conference 2001 [dblp](https://dblp.org/rec/conf/usenix/Lemon01.html) [usenix.org](https://www.usenix.org/conference/2001-usenix-annual-technical-conference/kqueue%E2%80%93-generic-and-scalable-event-notification)
[^19]: What is the purpose of epoll's edge triggered option? <https://stackoverflow.com/a/73540436/689699>
[^20]: 2012-01 Interview with Igor Sysoev, author of Apache's competitor NGINX <https://web.archive.org/web/0/http://www.freesoftwaremagazine.com/articles/interview_igor_sysoev_author_apaches_competitor_nginx>
[^21]: 2013-07 Nginx just became the most used web server among the top 1000 websites <https://w3techs.com/blog/entry/nginx_just_became_the_most_used_web_server_among_the_top_1000_websites>
[^22]: 2012-03 AOSA Volume 2 - nginx (Andrew Alexeev) <https://aosabook.org/en/v2/nginx.html>
[^23]: Redis FAQ: How can Redis use multiple CPUs or cores? <https://redis.io/docs/getting-started/faq/#how-can-redis-use-multiple-cpus-or-cores>
[^24]: 2010-09 antirez: An update on the Memcached/Redis benchmark <http://oldblog.antirez.com/post/update-on-memcached-redis-benchmark.html>
[^25]: 2019-02 antirez: An update about Redis developments in 2019 <http://antirez.com/news/126>
[^26]: Redis Doc: Diagnosing latency issues: Single threaded nature of Redis <https://redis.io/docs/management/optimization/latency/#single-threaded-nature-of-redis>
[^27]: 2019-08 林添毅：正式支持多线程！Redis 6.0与老版性能对比评测 <https://mp.weixin.qq.com/s/6WQNq5dNk-GuEhZXtVCo-A>