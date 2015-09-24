---
layout: 
title: replay quote data
categories: kdb
tags: kdb and R
published: True

---

背景是假定已经有了TP，用C/JAVA写的client基本思路都是开一个thread然后while(1)监听是否有数据推过来。见[here](http://www.timestored.com/kdb-guides/kdb-c-api-example)。

用Python也是完全类似的思路，用threading库开一个线程，在这个线程里不断receive从q推过来的data. 见[qpython](https://github.com/exxeleron/qPython)。 用qPython的问题是数据是通过IPC传输的,不确定是不是共享内存。而[PyQ](http://code.kx.com/wiki/Contrib/PyQ)是share-memory, 可能速度更快，但是坑是PyQ是for linux, not for windows. 用了os.uname, windows版python不支持(已经报了[bug](http://bugs.python.org/issue8080))。

用R有先天的劣势：没有办法像C#和Python一样简单的开Thread, 所以只能开一个R程序，或者用[Rserve](http://www.rforge.net/Rserve/). Rserve的简单介绍文章可以看[这里](http://blog.fens.me/r-rserve-server/)。 简单的说，就是单独开了一个R进程，把R嵌入其他语言的主体中。所以在kdb+tick的系统里，在某个client里，在初始化client时候直接开一个Rserve，在upd里把数据推到Rserve里由R来处理数据。也就是嵌入了R。劣势是通信用的TCP/IP, 不确认到底慢多少了。

结合我之前的backtest的框架，先在R里面开发一个package, 然后在research阶段，用一个sample dataset在R里进行研发，然后把train出来的StrategyClass save出来。在q里面启动整个TP框架，在client里订阅相应的数据及启动Rserve及load StrategyClass/TradeModule, 在client接到数据后，push数据进StrategyClass,用model得到预测结果, 把单子发到TradeModule(可以是一个单独的进程或者和StrategyClass在同一个R进程里），然后把单子拆到其他的Account里。Account根据单子订阅行情，并在得到行情后进行撮合。所以Account也应该是一个单独的进程Rserve。（坑是两个Rserve之间是否可以通信？） 当成交回报或者不成交的回报回到account的时候，被推回TradeModule进行处理（是否追单、撤单、轧差？）

通信的路径是 
1. 产生交易单的路径：DataFeed -> TP -> client -> RServe_Strategy (-> TradeModule) -> Account 
2. 交易执行的路径：Account -> client -> TP -> client -> Account -> TradeModule
这个过程当然是越快越好，但是到底有多慢，尤其是通信中浪费了多久，需要有了第一个版本的demo以后再测试。

在DataFeed的部分需要一个replay rdb的功能，大约应该是这样的

```
.z.ps:h(".u.upd";`quote;x)
-11!`file
```
以上是logfile格式的数据，或者如果是rdb的话，就是类似
```
feed:{
d+:1;
h(".u.upd";`quote;select from quote where date = d);
}
.z.ts:feed
\t 5
```

当回测完成时，直接把datafeed接入实盘，就能产生交易信号了。Account从虚拟账户变成实盘账户，就可以了。另外，不同券商的接口只在Account中暴露，这也是设计的初衷之一。

拿R写很痛苦啊。。。不行就直接上C#或者python. 如果是qpython，因为python可以搞thread,而且可以subscribe tp，大约设计是
DataFeed -> TP -> Strategy(Thread1) -> TradeModule(Thread2) -> Account(Thread3) 
Account相对与TradeModule是作为daemon或者server存在，在接受到client的sendorder的命令后，如果是虚拟账户，则和TP通信进行订阅、upd，并进行模拟撮合， 如果是实盘账户，则和券商的API进行通信，得到回报。





