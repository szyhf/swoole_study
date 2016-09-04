一直想写点Swoole的东西，毕竟它重新定义了php，却一直不知道怎么下手写

> Swoole涉及的知识点非常多，互为表里，每次想写都发现根本理不出一个头绪

Swoole是一个php的扩展，它的核心目的就是解决php在实现server服务中可能遇到的一系列问题，这些问题用源生的php往往并不能很高效（执行效率）的解决，一般也不会使用php来解决，所以会有说swolle重新定义的php的说法。

> 毕竟php作为一门成熟的脚本语言，开发效率是先天优势。

扩展的英文名称是Extension，php扩展是用C语言作为开发语言，基于Zend引擎提供的API，编译成的一个动态库。

> 如果曾经做过类似动态库调用开发的童鞋可能会更好理解一些，例如Android中的NDK开发

在php的配置文件中配置好extension的属性后，就可以引用这个动态库了。

> 也就是说，swoole本身是用C语言编写的，它可以让php获得一些额外的function。

然后是运行方式，swoole的许多功能都只能运行在cli模式下，而cli模式往往是很多刚接触swoole的phper遇到的第一个问题。

> 我当初学习swoole的时候也在cli这里栽了个大跟头

我们现在整理一下最常见的php代码执行方式：

1. 安装apache、php
2. 配置apache对那个目录进行php解析
3. 用浏览器访问那个目录的php文件

> 更多的细节这里就不提了，毕竟我相信每个phper对这个都是很熟悉的。

但这里就开始出现了第一个问题，我们知道，php是一个脚本语言，脚本语言的核心特点在于不用编译，随时执行，而执行脚本的工具就是解析器，而php的解析器就是zend引擎。

> 严格来说，zend并不是唯一的选择，不过，zend是最官方的。另外，Zend Studio和Zend Engine不是同一个东西，本文中的Zend全部指Zend Engine。

换个角度讲，只要有解析器，写好的php脚本就是可以执行的，而zend引擎与apache之间并没有绝对的关系

> 实际上，apahce是调用了zend对php脚本进行执行，然后将执行结果输出给了浏览器

所以所谓cli模式（CommandLine，命令行模式），其实就是在命令行下直接调用zend引擎对php脚本进行解析并执行，并获得程序输出结果的php脚本执行方式。

> 其实php也可以作为shell脚本来使用哦，就像bash shell一样

既然问题讲清楚了，在一个系统中具体怎么操作呢？

> 本文以CentOS 7.5作为系统环境，swoole是针对linux系统开发的，windows下并不适用。学习swoole的一个前题是懂得基本的linux系统使用。

当安装好php的时候，找到php的安装目录，如果是默认安装的话，可以试试whereis命令

``` shell
# 某种简单的方法
whereis php
> /usr/local/bin/php;
```

> locate whereis find这些命令都可以试试，目的是找到php

然后我们来写一个最经典的php脚本：

``` php
//vi hello_cli.php
<?php
    echo 'Hello PHP Cli';
```

> 编写纯php脚本时，php标签不要封口

然后我们在shell里执行它：

``` shell
/usr/local/bin/php hello_cli.php
> Hello PHP Cli
```

> 这段代码中的第一个php，是一个可执行文件，它接受一个php脚本文件作为输入参数，并解析执行这个php脚本文件（通过zend）。

没有错，第一个cli模式下的php程序就被你执行成功了！

> 默认情况下，php都会被安装在了$PATH的目录下，那就可以直接省略路径前缀了，下文中调用php的时候，全都省略了路径前缀。

因为swoole是pecl的项目，所以使用pecl安装是最简单的方法，强烈推荐第一次接触的童鞋先使用pecl安装，在熟悉了swoole之后，再考虑使用编译安装的方式以获取更多进阶功能。

> pecl这个工具基本都会被安装在与php相同的目录下（往往也都是$PATH目录）

``` shell
pecl install swoole
```

执行以下命令查看是否安装成功：

``` shell
php -m | grep swoole
> swoole
```

如果正确的输出了swoole，那么恭喜你，这次安装很成功

> 另一个常见的比较麻烦的问题是，有些童鞋的电脑里安装了多个php，而安装的时候没有正确的安装到预期的php的扩展目录中，就会导致无法正常工作，解决方案就是弄清楚各个php安装目录及配置关系，选择正确的目录进行安装。

其实本文还没正式开始介绍swoole，都是在学习swoole之前的准备工作，swoole的上手门槛比一般的php应用要高的多，如果没有网络开发和操作系统方面的一些知识，学习它并不是一件容易的事情，学习曲线很陡峭。

> 这句话我在群里说了无数次

---

很多新手会诟病swoole的手册写的太模糊，其实是前置知识不足，而手册也给出了需要的前置知识列表，以下引用至[官网的手册-学习swoole需要哪些知识？](http://wiki.swoole.com/wiki/page/487.html)

## 多进程/多线程

+ 了解Linux操作系统进程和线程的概念
+ 了解Linux进程/线程切换调度的基本知识
+ 了解进程间通信的基本知识，如管道、UnixSocket、消息队列、共享内存

## socket

+ 了解SOCKET的基本操作如accept/connect、send/recv、close、listen、bind
+ 了解SOCKET的接收缓存区、发送缓存区、阻塞/非阻塞、超时等概念

## IO复用

+ 了解select/poll/epoll
+ 了解基于select/epoll实现的事件循环，Reactor模型
+ 了解可读事件、可写事件

## TCP/IP网络协议

+ 了解TCP/IP协议
+ 了解TCP、UDP传输协议

## 调试工具

+ 使用gdb调试Linux程序
+ 使用strace跟踪进程的系统调用
+ 使用tcpdump跟踪网络通信过程
+ 其他Linux系统工具，如ps、lsof、top、vmstat、netstat、sar、ss等

---

学习并理解一个新事务并不是一个容易的事情，特别对于swoole这种具备一定颠覆性的工具，要有耐心和实践。

> 淡定的把手册看完，遇到不理解的名字学会使用搜索引擎学习，swoole的手册其实是个大宝库，网络开发常见的问题其实里边都涉及到了。
