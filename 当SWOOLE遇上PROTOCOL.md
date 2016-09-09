# 前言

上回我们简单介绍了一下TCP Server的工作方式以及如何用Swoole实现一个简单的TCP Server，这次我们来聊聊信息流动中，非常重要基石之一——协议（PROTOCOL）。

# 协议，通信的基石

每次讲到协议，都会想起小时候学习语文时，有段时间特别痴迷各种文字游戏。

> 那青葱的岁月吖~

其中有一种游戏，相信各位也应该接触过，就是断句游戏——一句话，如果加上不同的标点符号，则可能会产生截然相反的歧义。

> 后来高中的文言文断句练习彻底把这种愉快的游戏毁了=。=

可能最经典但其实也不怎么好笑的就是“下雨天留客天留我不留”，短短的一句话，断做“下雨天留客，天留我不留”，抑或是是断做“下雨天，留客天，留我不？留!”

> 其实个人呢，觉得这个断的有点牵强，可能因为我还不是古人，吾还年轻~

那么断句和我说的协议又有什么关系呢？关系大了，如果断错了句，传递的信息就会发生误解，网络通信中也一样，我们都知道计算机底层的数据本质上都可以看作0和1，也就是说再复杂的消息，承载的时候也只是0和1，如果不能正确的断句，那肯定是会出问题的。

> 啥？光量子计算机有十六个状态位？好吧......

通信的双方约定一种理解的规则，以便对理解对方想表达的信息，这种解析信息的规则，就是今天的主题，协议。

> 在语文上，我们用的是标点符号；数学上，我们有各种的加减乘除......

# 奔流不息的TCP与毫无记忆的HTTP

相信TCP协议（Transmission Control Protocol）应该是想学习SWOOLE的童鞋最容易遇到的拦路虎之一，因为一般我们使用PHP做网站开发的时候，并不需要处理涉及TCP协议的东西，只要了解一部分HTTP协议（HyperText Transfer Protocol）就可以做很多事情。

> 甚至只是知道Get和Post就可以了，更细致的工作，巨人们已经帮我们完成了。

在故事继续之前，请允许我先简单引入一下传说中的4层协议，TCP就是传输层的协议，而HTTP是应用层，这两个协议有什么关系呢？我们做个有趣的实验看看：

> 以下代码改编至拙作《当SWOOLE遇上TCP》

```php
<?php

$server = new \swoole_server("127.0.0.1",8088,SWOOLE_PROCESS,SWOOLE_SOCK_TCP);

$server->on('connect', function ($serv, $fd)
{
  
});

$server->on('receive', function ($serv, $fd, $from_id, $data)
{
    // 这次，我们只需要简单的把收到的数据打印出来即可
    // 但是，我们会在一头一尾各打印一行邪恶的分隔线
    // 以便清楚的划分收到的数据内容
    
    echo "====================邪恶的开头分隔线====================".PHP_EOL;
    echo $data;//打印收到的数据正文
    echo "====================邪恶的结尾分隔线====================".PHP_EOL;
    
}

$server->on('close', function ($serv, $fd)
{
    echo "client: close.\n";
});

$server -> start();
```

> 远程主机\IP\端口的问题，本文就掠过啦，有需要看本系列的前作。

好，我们之前是通过telnet，实现与SWOOLE的TCP Server之间的简单通信的，这次我们玩点不一样的，首先仍然是启动SWOOLE Server，然后，打开浏览器，没错，在地址栏中输入：“http://127.0.0.1:8088” ————

> 喂，我运行的是TCP Server，开浏览器干什么啦？

显然，浏览器什么都没有输出，又或者爆出一个错误，但这个时候返回我们的终端看看：

```bash
> php swoole_server_demo.php
====================邪恶的开头分隔线====================
GET / HTTP/1.1
Host: 127.0.0.1:8088
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.116 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8

====================邪恶的结尾分隔线====================
```

没错，虽然我们运行的是TCP Server，虽然我们是使用浏览器，而不是telnet访问的，我们的Server仍然打印出了显然非常有规律的信息，相信很多童鞋已经发现了，我们使用Chrome开发网页时，经常使用的调试工具箱里，就会在Network工具中的Header中看到类似的东西。

> 这就是根据HTTP协议编写的一段信息。

