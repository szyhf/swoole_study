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

第三个参数mode中我们填入的PROCESS，即表示当前Server是运行于多进程模式的。

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

这三个进程中，所有进程的根进程，也就是例子中的2829进程，就是所谓的Master进程；而2831进程，则是Manager进程；最后的2836进程，是Worker进程。

基于此，我们简单梳理一下，当执行的start方法之后，发生了什么：

1. 当前进程退出（守护进程模式下），fork出Master进程，并触发OnMasterStart事件。
2. Master进程启动成功之后，fork出Manager进程，并触发OnManagerStart事件。
3. Manager进程启动成功时候，fork出Worker进程，并触发OnWorkerStart事件。

所以，一个最基础的Swoole Server，至少需要有3个进程，分别是Master进程、Manager进程和Worker进程。

> 不要看到进程多就觉得麻烦咯，其实全赖它们各司其职，才有Swoole重新定义PHP的壮举。

事实上，一个多进程模式下的Swoole Server中，有且只有一个Master进程；有且只有一个Manager进程；却可以有n个Worker进程。

> 那么这几个进程之间是怎么协同工作的呢？我们先暂时考虑只有一个Worker的情况。

那么，我们又可以拉出之前写的最简单Server，来看看这个过程中，三种进程之间是怎么协作的。

1. Client主动Connect的时候，Client实际上是与Master进程中的某个Reactor线程发生了连接。
1. 当TCP的三次握手成功了以后，由这个Reactor线程将连接成功的消息告诉Manager进程，再由Manager进程转交给Worker进程。
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

> 实际上，一对多投递这种模式总是在并发的程序设计非常常见：1个Master进程投递n个Reactor线程；1个Manager进程投递n个Worker进程。

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

> 读书万卷不若自己亲手写一行，试验一下这个配置下，Server启动后，pstree的结构。

# 进程模型与数据共享

在以前的讨论中，我们最常接触到的回调方法如下：

1. OnConnect
1. OnReceive
1. OnClose

如上一节所说，这三个回调其实都是在Worker进程中发生的，而了解了进程模型以后，我们可以认识一下更多的回调方法了：

```php
// 以下回调发生在Master进程
$server->on("start", function (\swoole_server $server){
	echo "On master start.";
});
$server->on('shutdown', function (\swoole_server $server){
	echo "On master shutdown.";
});

// 以下回调发生在Manager进程
$server->on('ManagerStart', function (\swoole_server $server){
	echo "On manager start.";
});
$server->on('ManagerStop', function (\swoole_server $server){
	echo "On manager stop.";
});

// 以下回调也发生在Worker进程
$server->on('WorkerStart', function (\swoole_server $server, $worker_id){
	echo "Worker start";
});
$server->on('WorkerStop', function(\swoole_server $server, $worker_id){
	echo "Worker stop";
});
$server->on('WorkerError', function(\swoole_server $server, $worker_id, $worker_pid, $exit_code){
	echo "Worker error";
});
```

OK，现在我们更新一下我们的测试代码，以展示不同进程之间，数据共享的特点和关系：

```php
$server = new \swoole_server("127.0.0.1",8088,SWOOLE_PROCESS,SWOOLE_SOCK_TCP);

$server->on('connect', function ($serv, $fd){ });

$server->on('receive', function ($serv, $fd, $from_id, $data){ });

$server->on('close', function ($serv, $fd){ });

// 在交互进程中放入一个数据。
$server->BaseProcess = "I'm base process."

// 为了便于阅读，以下回调方法按照被起调的顺序组织
// 1. 首先启动Master进程
$server->on("start", function (\swoole_server $server){
    echo "On master start.".PHP_EOL;
    // 先打印在交互进程写入的数据
    echo "server->BaseProcess = ".$server->BaseProcess.PHP_EOL;
    // 修改交互进程中写入的数据
    $server->BaseProcess = "I'm changed by master.";
	// 在Master进程中写入一些数据，以传递给Manager进程。
	$server->MasterToManager = "Hello manager, I'm master.";
});

// 2. Master进程拉起Manager进程
$server->on('ManagerStart', function (\swoole_server $server){
	echo "On manager start.".PHP_EOL;
	// 打印，然后修改交互进程中写入的数据
	echo "server->BaseProcess = ".$server->BaseProcess.PHP_EOL;
	$server->BaseProcess = "I'm changed by manager.";
	// 打印，然后修改在Master进程中写入的数据
	echo "server->MasterToManager = ".$server->MasterToManager.PHP_EOL;
	$server->MasterToManager = "This value has changed in manager.";
	
	// 写入传递给Worker进程的数据
	$server->ManagerToWorker = "Hello worker, I'm manager.";
});

// 3. Manager进程拉起Worker进程
$server->on('WorkerStart', function (\swoole_server $server, $worker_id){
	echo "Worker start".PHP_EOL;
	// 打印在交互进程写入，然后在Master进程，又在Manager进程被修改的数据
	echo "server->BaseProcess = ".$server->BaseProcess.PHP_EOL;
	
	// 打印，并修改Master写入给Manager的数据
	echo "server->MasterToManager = ".$server->MasterToManager.PHP_EOL;
	$server->MasterToManager = "This value has changed in worker.";
	
	// 打印，并修改Manager传递给Worker进程的数据
	echo "server->ManagerToWorker = ".$server->ManagerToWorker.PHP_EOL;
	$server->ManagerToWorker = "This value is changed in worker.";
});

// 4. 正常结束Server的时候，首先结束Worker进程
$server->on('WorkerStop', function(\swoole_server $server, $worker_id){
	echo "Worker stop".PHP_EOL;
	// 分别打印之前的数据
	echo "server->ManagerToWorker = ".$server->ManagerToWorker.PHP_EOL;
	echo "server->MasterToManager = ".$server->MasterToManager.PHP_EOL;
	echo "server->BaseProcess = ".$server->BaseProcess.PHP_EOL;
});

// 5. 紧接着结束Manager进程
$server->on('ManagerStop', function (\swoole_server $server){
    echo "Manager stop.".PHP_EOL;
	// 分别打印之前的数据
	echo "server->ManagerToWorker = ".$server->ManagerToWorker.PHP_EOL;
	echo "server->MasterToManager = ".$server->MasterToManager.PHP_EOL;
	echo "server->BaseProcess = ".$server->BaseProcess.PHP_EOL;
});

// 6. 最后回收Master进程
$server->on('shutdown', function (\swoole_server $server){
	echo "Master shutdown.".PHP_EOL;
	// 分别打印之前的数据
	echo "server->ManagerToWorker = ".$server->ManagerToWorker.PHP_EOL;
	echo "server->MasterToManager = ".$server->MasterToManager.PHP_EOL;
	echo "server->BaseProcess = ".$server->BaseProcess.PHP_EOL;
});

$server -> start();
```

这段程序测试的时候，我们需要开两个会话，第一个会话用于执行并打印输出；第二个会话用于使用kill命令通知Server执行一些工作，然后我们看看输出的结果：

```shell
# 在会话一中
> php swoole_server_demo.php
On master start.
server->BaseProcess = I'm base process.
On manager start.
server->BaseProcess = I'm base process.
server->MasterToManager = 
Worker start
server->BaseProcess = I'm base process.
server->MasterToManager = 
server->ManagerToWorker = 
```

从Manager start和Worker start中的输出，我们发现BaseProcess、MasterToManager、ManagerToWorker并没有分别在Master、Manager中被修改，并在子进程中打印出被修改后的结果，这是为什么呢？别急，我们继续做个实验。

打开会话二，先执行pstree -ap|grep php找到刚刚启动的Server的Master进程的PID，然后向该进程发送-10信号，然后再次实行pstree命令看看：

```shell
> pstree -ap|grep php
  |   |       `-php,5512 swoole_server_demo.php
  |   |           |-php,5513 swoole_server_demo.php
  |   |           |   `-php,5515 swoole_server_demo.php
>  kill -10 5512
> pstree -ap|grep php
  |   |       `-php,5512 swoole_server_demo.php
  |   |           |-php,5513 swoole_server_demo.php
  |   |           |   `-php,5522 swoole_server_demo.php
```

-10信号的作用是，要求Swoole重启Worker服务，我们会发现原来的Worker[5515]被干掉了，而产生了一个新的Worker[5522]，此时如果我们切换回会话一，会发现增加了以下的输出：

```shell
[2016-10-03 02:00:26 $5513.0]	NOTICE	Server is reloading now.
Worker stop
server->ManagerToWorker = This value is changed in worker.
server->MasterToManager = This value has changed in worker.
server->BaseProcess = I'm base process.
Worker start
server->BaseProcess = I'm changed by manager.
server->MasterToManager = This value has changed in manager.
server->ManagerToWorker = Hello worker, I'm manager.
```

首先是Swoole自己打印的日志信息，Server正在被reloading，然后Worker[5515]被终止，执行了WorkerStop的方法，此时WorkerStop输出的值我们可以看出，在WorkerStart中的赋值都是生效了的；然后，新的Worker[5522]被启动了，重新触发WorkerStart方法，这时我们发现，BaseProcess、MasterToManager和ManagerToWorker都分别被打印了出来？这是什么原因呢？

原因在方法被执行的顺序上，我们前文中的进程起调顺序并没有问题，但有些地方我们要做一点小小的细化：

1. Master进程被启动。
2. Manager进程Master进程fork出来。
3. Worker进程被Manager进程fork出来。
4. MasterStart被回调。
5. ManangerStart被回调。
6. WorkerStart被回调。

也就是说，三种进程的OnStart方法被回调的时候都有一定的延迟，底层事实上已经完工了fork的行为，才回调的，因此，默认启动的时候，我们在OnMasterStart、OnManagerStart中写入的数据并不能按预期被fork到Manager进程或者Worker进程。

然后，我们执行了kill -10重新拉起Worker进程的时候，此时Worker进程仍然是由Mananger进程fork出来的，但此时ManangerStart已经被执行过了，所以我们会发现在OnWorkerStart的时候，输出变成了ManagerStart中修改过的内容。

> OK，现在我们回到Shell会话二，向Master进程发送kill -15命令

```shell
> kill -15 5512
```

然后回到会话一，我们发现输出增加了如下的内容：

```shell
[2016-10-03 02:17:35 #5512.0]	NOTICE	Server is shutdown now.
Worker stop
server->ManagerToWorker = This value is changed in worker.
server->MasterToManager = This value has changed in worker.
server->BaseProcess = I'm changed by manager.
Manager stop.
server->ManagerToWorker = Hello worker, I'm manager.
server->MasterToManager = This value has changed in manager.
server->BaseProcess = I'm changed by manager.
Master shutdown.
server->ManagerToWorker = 
server->MasterToManager = Hello manager, I'm master.
server->BaseProcess = I'm changed by master.
```

-15命令是通知Swoole正常终止服务，首先停止Worker进程，触发OnWorkerStop回调，此时我们输出的内容懂事我们在WorkerStart中修改过的版本。

然后停止Manager进程，这时候要留意，我们在Worker中做的所有操作并没有反应在Manager进程上，OnManagerStop的输出仍然是在OnManagerStart中赋值的内容。

最后停止Master进程，也会有相同的事情发生。

所以，通过这个实验，展示了多进程Server的两个重要特性：的两个重要特性：

1. 父进程fork出子进程的时候，子进程会拷贝一份父进程的所有数据。
2. 各个进程之间的数据一般情况下是不共享的。


> 所以，学习Swoole的进一步需求就是，要弄清楚各个回调方法分别是在哪个进程中发生的，且发生的顺序是什么。


这两个特性会引起什么问题呢？如果没有弄清楚当前的代码是在哪个进程执行的，很有可能就会引起数据的错误。

> 以上例子中，为了便于输出，没有启用守护进程模式，所以交互进程与Master进程是同一个进程，有兴趣的童鞋欢迎在守护进程下实验。

# 回顾

这个系列我已经上传到了github上，欢迎围观：github.com/szyhf/swoole_study

1. 当SWOOLE遇上PHP 【SWOOLE安装、PHP的CLI模式】
1. 当SWOOLE遇上SERVER 【TCP/IP】
1. 当SWOOLE遇上TCP【TCP】
1. 当SWOOLE遇上PROTOCOL【通信协议】

番外：

1. 守护进程二三事与Supervisor
