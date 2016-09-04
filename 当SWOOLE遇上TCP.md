# 前言

前文再续，就书接上一回（拍一下惊堂木，然后喝口茶install一下B），话说笔者当初最早接触Swoole的时候，正迫切的期望能找到一个使用PHP作为主要开发语言的TCP Server的解决方案，因为公司业务中积累了大量的PHP代码，而新增的业务又迫切需要实现与客户端的主动通信，最终在盆友的推荐下，找到了Swoole。

# 轮询与长连接

一般情况下，我们接触PHP都是作为一个Web网站的开发语言而接触的，例如一个最简单的HelloWorld.php，往往是这么写的：

``` php
<?php
echo "Hello PHP";
```

> LAMP的配置这里就不多说了

不自觉的，蓦然间会让我们产生一种错觉，PHP只能用来处理这种场景的工作，其他事情并不合适。

> 亦或者说，很多盆友并没有意识到，PHP其实还隐藏了洪荒之力

当时，笔者需要开发一个实时的消息服务APP，消息的实时性要求较高，也就是说，服务端需要可以主动向客户端推送消息，而这个时候，如果再采用传统的http api的方式，势必陷入轮询的困局。

> 客户端每隔1s向服务器请求，检查是否有新的数据，这种场景可能会产生大量无用的请求，也会极大的增加服务端的负荷。

传统的Web服务，采用http/htpps作为应用层协议，并且通过“请求->响应”的机制实现客户端和服务端的通讯，也就是说，服务端总是“被动”的提供服务，服务端“难以”主动的将消息告知客户端。

> 这其实也是是websocket的产生背景。

这个时候，我们可以考虑实现自己的TCP Server，以解决这个问题。

> 显然，这里讨论的问题并不局限于开发语言，.Net、Java、Go、NodeJS等都有对应的解决方案。

通过TCP协议构建的Server，是可以实现服务端和客户端保持一个持久的链接，链接一旦建立，就像电话打通了一样，通话的双方都可以主动向对方发送消息。

> 其实http/https协议的传输层也是tcp协议，但为啥http/https协议变成了一次性的服务呢？有缘的话，下回分解。

因此，双方的链接会呈现出“持久在线”的状态，也就是长连接这一说法的由来。

> 有兴趣的盆友可以自行查找TCP是怎么实现“在线”这个状态的，还记得笔者上学时，计算机网络的老师的一句话，网络通信上绝对的可靠是不存在的。

# TCP Server在干啥？

回到我们的应用场景，客户端需要先与服务端建立TCP长连接，并维持这个链接，当服务端产生了新的消息时，服务端主动将新消息发送给客户端，客户端接收消息并解析，然后将结果展示给客户端。

> 以下例子，改编自拙作《当SWOOLE遇上SERVER》

``` php
<?php 
    //vi swoole_tcp_server_demo.php

    $server = new \swoole_server("127.0.0.1",8088,SWOOLE_PROCESS,SWOOLE_SOCK_TCP);

    $server->on('connect', function ($serv, $fd){
            echo "Client:Connect.\n";
            
            //启动一个循环，定时向客户端发一个消息
            
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

> 如果你是在远程服务器上运行的，请将127.0.0.1替换为你的远程服务器公网IP（或者你能访问的内网IP）。

上一章的例子中，我们每次receive了一个客户端的消息以后，就关闭了与这个客户端的链接，并没有向客户端发出响应，但事实上，服务端完全可以在收到消息以后，向客户端发出一个回复，就像“请求->响应”的工作机制一样：

``` php
<?php 
//我们修改一下on reveive的回调，然后启动服务
$server->on('receive', function ($serv, $fd, $from_id, $data) 
{
    //根据收到的消息做出不同的响应
    switch($data)
    {
        case 1:
        {
            $serv->send($fd,"1 for apple\n");
            break;
        }
        case 2:
        {
            $serv->send($fd,"2 for boy\n");
            break;
        }
        default:
        {
            $serv->send($fd,"Others is default\n");
        }
    }
});
```

用telnet作为客户端访问一下我们刚刚启动Server

``` shell
> telnet 127.0.0.1 8088
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
```

然后分别输入“1"、“2”、“hello”并回车


> 以下是telnet的输出

``` shell
> telnet 127.0.0.1 8088
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
1
1 for apple
2
2 for boy
3
others is default
5
others is default
hello
others is default
```

这段代码很简单，如果receive了客户端的消息，对消息做一个switch，根据switch的结果，向客户端返回不同的消息。

> 这个场景是一个很典型的“请求->响应”的场景

那有些盆友也许会问了，这样做的话，跟我使用URL访问网站获取响应有什么区别？

> 这个问题很好，思考是不断进步的阶梯

那么我们来做些不一样的，继续修改on receive的回调：

``` php
<?php
//我们修改一下on reveive的回调，然后启动服务
$server->on('receive', function ($serv, $fd, $from_id, $data) 
{
    //根据收到的消息做出不同的响应
    switch($data)
    {
        case 1:
        {
            foreach($serv->connections as $tempFD)
            {
                 $serv->send($tempFD,"1 for apple\n");
            }
            break;
        }
        case 2:
        {
            $serv->send($fd,"2 for boy\n");
            break;
        }
        default:
        {
            $serv->send($fd,"Others is default\n");
        }
    }
});
```

当case 1的时候，我们遍历了$serv的connections成员，获得了与当前服务器连接的所有客户端，并且向所有的客户端都发送了“1 for apple\n”这个字符串。继续用telnet作为客户端，我们这次需要打开两个telnet，当两个telnet都成功连接了Server之后，用第一个telnet发送1：

> 第一个telnet客户端的输出

``` shell
> telnet 127.0.0.1 8088
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
1
1 for apple
```

> 第二个telnet客户端的输出

``` shell
> telnet 127.0.0.1 8088
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
1 for apple
```

第二个telnet客户端虽然并没有向server端发送“1”作为消息，server端仍然向第二个客户端发送了消息“1 for apple\n”，这可以做什么？如果我们要做一个聊天室的话，就可以简单的实现发送公共聊天消息的功能。

> 如果打开一下脑洞，在Server的业务中将用户分类存储，发送的时候有选择的向不同的用户发送消息，就可以实现私聊，亦或者是分组消息。

如果只是这样，可能又有童鞋问了，仅仅这样做，还是一个“请求->响应”的工作模式吖，只不过是将一对一的请求响应，变成了一对多的请求响应？

> 确实有点这个感觉，那我们来做点不一样，这次，server会不断向客户端发送消息，不管有没有请求。

``` php
<?php
//这次我们要修改的是on connect回调哦！
$server->on('connect', function ($serv, $fd)
{
    $serv->tick(1000, function() use ($serv, $fd) {
            $serv->send($fd, "这是一条定时消息\n");
        });
});
```

以上代码中的tick方法，表示启动一个定时器，该定时器每1000毫秒触发一次，并执行回调方法。

> Swoole Tick是Swoole工具箱中的一个强大工具，它比PHP原生的pcntl_alarm更加精确，也支持异步调用，非常方便，更多介绍可以[参考手册](http://wiki.swoole.com/wiki/page/244.html)。

这次仍然是打开telnet

``` shell 
> telnet 127.0.0.1 8088
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
这是一条定时消息
这是一条定时消息
这是一条定时消息
这是一条定时消息
这是一条定时消息
这是一条定时消息
这是一条定时消息
2
这是一条定时消息
2 for boy
这是一条定时消息
这是一条定时消息
这是一条定时消息
这是一条定时消息
这是一条定时消息
这是一条定时消息
...
```

这次，只要连接上服务器，不管客户端说没说话，会一直收到“这是一条定时消息”的消息，并且，如果我们见缝插针地写个2并发送，就会收到on receive中的反馈“2 for boy”，并不会与“这是一条定时消息”冲突。

> 这里，前者就是服务端主动发出，客户端被动接受的消息；而后者，却又是“请求->响应”的工作模式，两者并不冲突，仅取决于具体的代码实现。

# 小结

好滴，今天的三分热度就到这了，再多就得超时了，这篇的内容主要列举了TCP Server的几个基本工作场景，及这些场景通过Swoole Server的简单实现。其实TCP Server的核心应用特征就在于，一旦连接建立，双方都可以平等地自由选择什么时候向对方发出消息，并选择是否对收到的消息做出响应。

> 想象一下，你跟你的基友在电话两旁自说自话 QAQ
