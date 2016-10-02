# 前言

前文再续，就书接上一回，随着与Server、TCP、Protocol的邂逅，Swoole终于迎来了自己的故事，今天，我们来聊聊Swoole的进程模。

> 前边几篇东西虽然标题是Swoole，其主要讲的是操作系统、计算机网络方面的知识，包括一点点笔者自己的私货，今天终于放假了，咱可以讨论一下公的了=。=

# 并发之始

之前我们已经初步讨论的一个WebServer是怎样工作的，但之前的例子中，我们看到的服务都是一个客户端与一个服务端一问一答的场景，但事实上，绝大部分时候我们预期的服务并不是只向一个客户端提供服务，所以，作为一个成熟的Server，并发\并行问题是必须解决的。

> 其实，“并发”和“并行”两个概念在计算机中是相关但不同的，有兴趣的童鞋可以自己搜索一下，笔者今天仅讨论并发咯。

而软件开发中，最常见的并发问题解决方案，莫过于多线程/多进程两种模式了。

> 微软的体系中，除了线程，还有“纤程”；而最近非常火爆的“协程”，则又是另一个解决方案了。

在《计算机组成原理》中我们都学过，并发中最迫切需要解决的问题之一，就是数据的可靠性问题，而不同的并发模型，其并发数据可靠性的机制往往各有特点，因此，在使用Swoole Server\Client的过程中，其并发解决方案的模型是必须要了解的，否则使用上很容易出现不符合预期的结果。

> 简单说，就是防止脏读脏写

Swoole目前总共有三种运行模式，其中Base模式基本没有生产应用价值；协程模式暂时还处于预览阶段；因此，笔者在此想和大家讨论的，就是Swoole的多进程模式，也是官方目前最推荐用于生产环境的模式。

> 事实上，Swoole曾经还有多线程模式，但由于Zend在多线程模式本身的缺陷，在1.6版本后，多线程模式已经被关闭。

# 进程模型

首先，我们还是来简单回顾一下Swoole Server的构造函数，之前我们已经解决了Host、Port、Protocol的问题，这期我们来看最后一个参数的：

```php
<?php
$server = new \swoole_server("127.0.0.1",8088,SWOOLE_PROCESS,SWOOLE_SOCK_TCP);
```

第三个参数mode中我们填入的PROCESS，即表示当前Server是运行与多进程模式的。

> 其他mode的可选参数可以参考手册

然后，我们简单实现一个没有任何内容的Server，并运行在守护进程模式下：

```php
<?php
$server = new \swoole_server("127.0.0.1",8088,SWOOLE_PROCESS,SWOOLE_SOCK_TCP);

$server->on('connect', function ($serv, $fd){ });

$server->on('receive', function ($serv, $fd, $from_id, $data){ });

$server->on('close', function ($serv, $fd){ });

$server->set([
	"daemonize"=>true
]);

$server -> start();
```

相比以前，这里我们在启动Server之前，通过set方法对Server进行了配置，daemonize表示当前服务在启动后会进入守护进程。

> 关于守护进程问题，可以参考拙作《守护进程二三事与Supervisor》

在启动服务之后，我们继续在Shell中输入以下命令：

```shell
> php swoole_server_demo.php
> pstree -ap|grep swoole_server_demo
  |-php,2829 swoole_server_demo.php
  |   |-php,2831 swoole_server_demo.php
  |   |   `-php,2836 swoole_server_demo.php
```

> pstree命令可以查看进程的树模型

从系统的输出中，我们可以很容看出server其实有3个进程，进程的pid分别是2829、2831、2836，其中2829是2831的父进程，而2831又是2836的父进程。

> 所以，其实我们虽然看起来只是启动了一个Server，其实最后产生的是三个进程。

这三个进程中，所有进程的根进程，在例子中的2829进程，就是所谓的Master进程；而2831进程，则是Manager进程；最后的2836进程，是Worker进程。

基于此，我们简单梳理一下，当执行的start方法之后，发生了什么：

1. 当前进程退出（守护进程模式下），fork出Master进程，并触发OnMasterStart事件。
2. Master进程启动成功之后，fork出Manager进程，并触发OnManagerStart事件。
3. Manager进程启动成功时候，fork出Worker进程，并触发OnWorkerStart事件。

所以，一个最基础的Swoole Server，至少需要有3个进程，分别是Master进程、Manager进程和Worker进程。

> 不要看到进程多就觉得麻烦咯，其实全赖它们各司其职，才有Swoole重新定义PHP的壮举。

事实上，一个多进程模式下的Swoole Server中，有且只有一个Master进程；有且只有一个Manager进程；却可以有n个Worker进程。

> 那么这几个进程之间是怎么协同工作的呢？

那么，我们又可以拉出之前写的最简单Server，来看看这个过程中，三种进程之间是怎么协作的。

1. Client主动Connect的时候，Client实际上是与Master进程中的某个Reactor线程发生了连接。
1. 当TCP的三次握手成功了以后，由这个Reactor线程将连接成功的消息告诉Manager进程，再由Manager进程转交给某个Worker进程。
1. 在这个Worker进程中触发了OnConnect的方法。
1. 当Client向Server发送了一个数据包的时候，首先收到数据包的是Reactor线程，同时Reactor线程会完成组包，再将组好的包交给Manager进程，由Manager进程转交给Worker。
1. 此时Worker进程触发OnReceive事件。
1. 如果在Worker进程中做了什么处理，然后再用Send方法将数据发回给客户端时，数据则会沿着这个路径逆流而上。

> 同样的故事，随着认识的加深，会发现不一样的精彩

首先，Master进程是一个多线程进程，其中有一组非常重要的线程，叫做Reactor线程（组），每当一个客户端连接上服务器的时候，都会由Master进程从已有的Reactor线程中，根据一定规则挑选一个，专门负责向这个客户端提供维持链接、处理网络IO与收发数据等服务。

> 以前我们提到的分包拆包等功能也是在这里完成的哦。

而Manager进程，某种意义上可以看做一个代理层，它本身并不直接处理业务，其主要工作是将Master进程中收到的数据转交给Worker进程，或者将Worker进程中希望发给客户端的数据转交给Master进程进行发送。

另外，Manager进程还负责监控Worker进程，如果Worker进程因为某些意外挂了，Manager进程会重新拉起新的Worker进程，有点像Supervisor的工作

> 而这个特性，也是最终实现热重载的核心机制。

最后就是Worker进程了，顾名思义，Worker进程其实就是处理各种业务工作的进程，Manager将数据包转交给Worker进程，然后Worker进程进行具体的处理，并根据实际情况将结果反馈给客户端。

如果要打个比方的话，Master进程就像业务窗口的，Reactor就是前台接待员，用户很多的时候，后边的用户就需要排队等待服务；Reactor负责与客户直接沟通，对客户的请求进行初步的整理（传输层级别的整理——组包）；然后，Manager进程就是类似项目经理的角色，要负责将业务分配给合适的Worker（例如空闲的Worker）；而Worker进程就是工人，负责实现具体的业务。

> 实际上，一对多投递这种模式总是在并发的程序设计非常常见：1个Master进程投递n个Reactor线程；1个Manager进程投递n个Worker进程。这其实是并发开发中常见协同机制：消息队列。而Swoole中的并发工作几乎都是通过消息队列实现的。

而多进程的模型会影响什么呢？首先，多进程模型的两个重要特性：

1. 父进程fork出子进程的时候，子进程会拷贝一份父进程的所有数据。
2. 各个进程之间的数据一般情况下是不共享的。

这两个特性会引起什么问题呢？如果没有弄清楚当前的代码是在哪个进程执行的，很有可能就会引起数据的错误。

> 所以，学习Swoole的进阶需求就是，要弄清楚各个回调方法分别是在哪个进程中发生的，且发生的顺序是什么。

现在，我们来看看一个简单的多进程Swoole Server的几个基本配置：

```php
<?php
$server->set([
	"daemonize"=>true,
	"reactor_num"=>2,
	"worker_num"=>4,
]);

$server -> start();
```

reactor_num：表示Master进程中，Reactor线程总共开多少个，注意，这个可不是越多越好，因为计算机的CPU是有限的，所以一般设置为与CPU核心数量相同，或者两倍即可。

worker_num：表示启动多少个Worker进程，同样，Worker进程数量不是越多越好，仍然设置为与CPU核心数量相同，或者两倍即可。

> 读书万卷不若自己亲手写一行，可以试验一下这个配置下，启动后的pstree的样式。




# 回调方法于进程模型

在前面的学习中，我们最常接触到的回调方法如下：

1. OnConnect
1. OnReceive
1. OnClose

这三个回调其实都是在Worker进程中发生的。



# 热重载与进程

# 数据库连接与常驻进程
