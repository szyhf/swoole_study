# 前言

上一回讲到，Swoole终于成功邂逅了PHP，现在要开始它们的奇妙路程了。

# Server之初

通常，我们会把网络系统分为B/S架构和C/S架构，而这里笔者想聊的就是这里提到的S，也就是服务（Server）在干什么？

> 风靡各大高校宿舍的Dota和LOL，主体上可以算是典型的C/S架构的网络系统/软件/应用/程序/What ever

这里笔者打算从比较常见的基于PHP的Web网站开始聊起。

> 这里举例采用的是最基础的Linux + Apache + PHP的开发环境。

当我们打开 http://127.0.0.1:80 并看到Apache的欢迎界面时，我们知道，我们已经成功的完成了一个简单的B/S结构的程序。

> 虽然这里输出的不是Hello World！虽然目前为止一句PHP也没写。

那么，这个时候，这里我们说B/S中的Server具体指代的是什么呢？以下两个选项哪个是你的想法？

1. 运行并保存着我们网站的服务器主机。
2. Apache正在运行的进程。

其实从笔者的角度而言，上述两个选项都是对的，因为Server这个词本身的含义就很丰富，根据特定的语境，它既可以指服务器，也可以指服务程序。

> 本文中提到的Server如无特别说明，都是指提供服务的应用程序，在当前的场景中，就是Apache。

我们来简单扒一下，在打开这个网页的过程中，Apache作为一个Server，最少要做到哪些工作？

> 最基础的工作，更深入的问题我们可以一点点讨论。

首先，Apache需要先运行起来，如果Apache没有运行，显然没法向浏览器提供服务（例如，输出Apache的欢迎页面）

> 传统的Web网站场景中，Server是被动地提供服务的，也就是客户端不请求，Server就不会提供服务，就像一般民事诉讼中的不告不理原则。

再者，浏览器需要有一个可靠的方法找到我们刚刚运行起来的Apache（就像寄快递，要有收件方的地址）

> 想象一下市民中心的办证大厅，各种各样的窗口，不同的窗口可以办理不同的证件，市民提交的材料就是“输入”，服务台提供的证件就是“输出”，找到正确的窗口，是享受服务的前题。

例如我们在浏览器输入 www.baidu.com ，我们知道会打开百度的首页。

> 输入 www.google.com ，会出现404无法访问的错误？

当Apache做到了上述两项工作时，我们可以简单认为它具备了作为一个Server的基础。

> 用物流管理的话说，就是在“正确的时间、正确的地点、正确的货物”。

## 监听：正确的地点

那么，Apache是怎么做到这两点的呢？相信怎么运行Apache这个问题不必笔者啰嗦，我们主要开始看看第二个问题。

> 关于服务要稳定常驻运行的问题，可参见拙作《守护进程二三事与Supervisor》

我们怎么去定义这个“正确的地点”呢？最常见的方案，就是TCP/IP协议中的IP协议。

> 严谨地说，TCP协议是传输层协议；IP是网络层协议。因为两者常常搭配出现，就像LAMP一样，有了TCP/IP协议这个说法。

IP协议帮助客户端在浩瀚的网络中找到正确的主机，例如上文中的 127.0.0.1 主机。

> 127.0.0.1 是IP协议中定义的一个特殊地址，表示本机，概念上有点像PHP中的$this。

如果我们的主机在局域网中被分配的IP是192.168.1.233，则其他主机也可以通过192.168.1.233这个地址找到我们的主机

> IP是“IP协议”给每一台联网设备规定一个地址，便于互相通信和发现。

但一个主机如果只能运行一个服务，就太浪费了，因此如果说IP是用来区分不同的联网设备的，Port在这就是用来区分同一个设备的不同服务的。

> 最基本的LAMP环境中，SSH需要一个端口（默认22），Apahce需要一个端口（默认80），Mysql需要一个端口（默认3306）。

一般情况下，端口的编号取值范围是 [1, 65535]，一般1000以下的端口都会被一些常用服务默认调用，所以尽可能不要使用。

> 就像大公司电话系统中的主机与分机一样，IP是主机，全世界通用（没有区号这些东西啦），Port就是分机，仅在自己的主机内通用。

写到这里，前文我们提到的 127.0.0.1:80 的含义就更清晰了，前面的 127.0.0.1 是IP地址，后边的80是端口。而因为80是HTTP服务的默认端口，所以访问一般的常见网站时我们并不需要写成 www.baidu.com:80 。

> 所以如果是自建HTTP服务的话，默认情况下还是最好提供80端口作为服务端口。

我们把某个服务通过某个端口对外提供服务的行为称为“监听”，形象的说，前文的Apache监听着本机的80端口，如果有客户通过这个端口发来请求，操作系统就会把请求交给Apache，Apache就可以根据请求的具体内容进行处理，并给出响应（例如，欢迎使用Apache！）。

> 常用“Listen”或者“Bind”这两个动词。

最后，我们可以整理出简单LAMP环境中，基于HTTP协议的Web服务的交互逻辑：

1. 客户端（浏览器）将请求提交给指定IP的主机。
2. 操作系统根据请求中的PORT，转交给监听了这个PORT的Apache。
3. Apache根据配置找到合适的目录，并获取目录中的PHP脚本。
4. 调用Zend对该PHP脚本进行解析，并获得输出结果。
5. 将输出结果返回给客户端（浏览器）.

而一般情况下，PHP脚本的工作就在第四步中处理具体业务，至于怎么与浏览器保持通信，PHPer一般是不关心的，直到，Swoole重新定义了PHP。

> 其实还有别的方案，但，俺们的主题是Swoole（写了快一百行才提到Swoole的笔者表示说这句话的时候有点心虚...）


# Swoole Server做了什么？

前文我们以Apache作为例子，简单梳理了一下Server在做什么的问题，再回来看Swoole Server，就好理解了，Swoole允许通过PHP构造一个新的Server，提供跟Apache类似的功能，监听请求，作出响应。

> 其实也回答了群里很常见的一个问题，为什么我运行Swoole Server的时候会跟Apache\Nginx冲突？因为这里用PHP写的不再只是网页的业务逻辑，同时也包括Server的部分。

那么，我们现在开始第一个简单的Swoole TCP Server的Demo。

``` php
<?php
//vi swoole_server_demo.php
$server = new \swoole_server("127.0.0.1", 8088, SWOOLE_PROCESS, SWOOLE_SOCK_TCP);
$server -> start();
```

> 后两个参数涉及到了Swoole的运行方式和传输层使用的协议问题，我们以后再聊。

这里我们新建了一个Swoole Server对象，这个对象监听了IP 127.0.0.1，和端口 8088 。然后我们打开浏览器，访问http://127.0.0.1/swoole_server_demo.php，然后我们会惊讶地发现：报错，swoole_server只能运行在cli模式下。

> 这是很多童鞋第一次接触Swoole会遇到的另一个问题，还记得我前边说的么？SwooleServer已经是一个独立的服务了，它不再依赖于Apache，它本身就是一个完整的网络Server，所以它需要运行在cli模式下，同时也不需要通过浏览器访问的方式执行了。

所以正确的启动方式应该是在shell中执行：

``` shell
php swoole_server_demo.php
> PHP Fatal error:  swoole_server::start(): require onReceive/onPacket callback in swoole_server_demo.php
```

> 关于cli的问题，可以参考拙作《当Swoole遇上PHP》

这里的报错又是什么原因呢？因为例子中使用的是TCP Server，我们用打电话的例子来梳理一下，客户给服务打电话，对于服务来说，有这么几个关键事件：

1. OnConnect，建立连接，也就是电话被拨通的时候发生。
2. OnReceive，收到消息，也就是服务听到客户说的话。
3. OnClose，关闭连接，也就是客户\服务其中一方挂掉了电话时发生。

一个完整的TCP服务需要处理这三个过程，而我们访问网页的时候，这些过程都被浏览器\服务程序内部处理掉了，平时写网页的时候只需要考虑对请求作出响应即可。

> 这种抽象进一步降低了PHP的上手门槛，也让很多人低估了PHP的能力。

而使用Swoole Server的时候，我们需要自己来管理这个过程，同时，也可以在这个基础上做到更多的事情。

> 所以说学习使用Swoole需要了解更多的是操作系统、计算机网络方面的知识，有相关背景的童鞋学习使用Swoole其实并不困难，了解一下做不同事情需要使用的接口是什么，再了解一下运作机制，就差不多可以上手了。

那我们把这几个回调函数补上，这样就能运行一个完整的Server了：

``` php
<?php 
//vi swoole_server_demo.php
    $server = new \swoole_server("127.0.0.1",8088,SWOOLE_PROCESS,SWOOLE_SOCK_TCP);

    $server->on('connect', function ($serv, $fd){
            echo "Client:Connect.\n";
});
    $server->on('receive', function ($serv, $fd, $from_id, $data) {
            //打印收到的消息
            echo "Receive message: $data";
            //关闭连接（当然，也可以不关闭）
            $serv->close($fd);
});
    $server->on('close', function ($serv, $fd) {
            echo "Client: Close.\n";
});

$server -> start();
```

然后，重新在shell里边执行：

``` shell
php swoole_server_demo.php
> 
```

如果一切顺利的话，我们会看到整个命令行好像卡住了，一个光标在闪动，但没有任何输出？这是什么情况？

> 笔者第一次遇到的时候思路也没转过弯来，一直以为故障了。

这个时候，其实Server已经启动了，并且正在运行，监听了本机的8088端口，此时Server处于等待的状态，所以没有任何输出。

> 这个时候如果有客户端访问本机的8088端口，就会触发OnConnect事件了。

我们打开另一个交互窗口（注意别关了Swoole Server正在运行的窗口），用telnet来试试：

``` shell
# 在第二个Shell窗口
telnet 127.0.0.1 8088
> Trying 127.0.0.1...
> Connected to 127.0.0.1.
> Escape character is '^]'. 
```

此时，我们再返回第一个窗口，就会看到刚才卡住的光标有输出了：

``` shell
php swoole_server_demo.php
> Client:Connect.
```

输出的正是我们在OnConnect回调中设置的内容。

> 想想贝尔第一个打通电话的瞬间。

这时我们可以随便输入一些字符看看：

``` shell
# 在第二个Shell窗口
telnet 127.0.0.1 8088
> Trying 127.0.0.1...
> Connected to 127.0.0.1.
> Escape character is '^]'. 
Hello SwooleServer!
> Connection closed by foreign host.
```

此时再切换回第一个Shell，我们看到输出增加了：

``` shell
php swoole_server_demo.php
> Client:Connect.
> Receive message: Hello SwooleServer!
> Client: Close.
```

OK，我们来看看整个过程发生了什么事：

1. 运行了一个TCP Server，它监听了 127.0.0.1 主机的 8088 端口。
2. 用telnet工具作为客户端，它尝试连接 127.0.0.1 主机的 8088 端口 的服务。
3. Server收到了连接请求，根据TCP的握手机制完成了连接过程，并触发了OnConnect回调，此时Server端输出了字符串“Client:Connect.\n”
4. telnet也获得了连接成功的消息，输出了“Connected to 127.0.0.1.”、“Escape character is '^]'.”等消息，此时，相当于电话已经打通了。
5. telnet向Server发送了一个字符串“Hello SwooleServer!”。
6. Server收到了telnet发来的字符串，并触发了OnReceive回调，在该回调中，Server打印了字符串“Receive message: Hello SwooleServer!”，然后将与telnet的连接关闭了。
7. 连接关闭后，Server触发OnClose回调，输出了字符串“Client: Close.”。
8. 连接关闭后，telnet也输出了字符串“Connection closed by foreign host.”

如果把这个过程弄清楚，那么就朝着Swoole的应用又迈出了一大步。

> PPPPPPS：本来一直是两周一更的节奏但今天女排夺冠了啊啊啊啊啊啊啊所以爬起来撸了这篇。

学习Swoole有时候很难，有时候又并不难，难点不在于Swoole的接口有多复杂，机制有多麻烦，而更多在于不知道它到底解决了什么问题，本文笔者对Swoole Server解决的问题做了简单的梳理和介绍，希望能给刚刚接触Swoole的童鞋一点启发和借鉴。

# 彪悍的Swoole工具箱

Swoole具备一系列强大的工具，允许我们借助PHP高效开发的特性，写出高性能的Web服务，那么这个工具箱里除了Swoole Server以外还有什么呢？以下引用自[官网的手册](http://wiki.swoole.com/wiki/page/1.html)：

## Swoole Server

强大的TCP/UDP Server框架，多线程，EventLoop，事件驱动，异步，Worker进程组，Task异步任务，毫秒定时器，SSL/TLS隧道加密。

+ swoole_http_server是swoole_server的子类，内置了Http的支持
+ swoole_websocket_server是swoole_http_server的子类，内置了WebSocket的支持

## Swoole Client

TCP/UDP客户端，支持同步并发调用，也支持异步事件驱动。

## Swoole Event

EventLoop API，让用户可以直接操作底层的事件循环，将socket，stream，管道等Linux文件加入到事件循环中。

> eventloop接口仅可用于socket类型的文件描述符，不能用于磁盘文件读写

## Swoole Async

异步IO接口，提供了 异步文件系统IO，异步DNS查询，异步MySQL等API。包括2个重要的子模块：

+ swoole_timer，异步毫秒定时器，可以实现间隔时间或一次性的定时任务
+ file，文件系统操作的异步接口

## Swoole Process

进程管理模块，可以方便的创建子进程，进程间通信，进程管理。

## Swoole Buffer

强大的内存区管理工具，像C一样进行指针计算，又无需关心内存的申请和释放，而且不用担心内存越界，底层全部做好了。

## Swoole Table

基于共享内存和自旋锁实现的超高性能内存表。彻底解决线程，进程间数据共享，加锁同步等问题。

> swoole_table的性能可以达到单线程每秒读写50W次


# Swoole入门系列的其他

1. 当Swoole遇上PHP
