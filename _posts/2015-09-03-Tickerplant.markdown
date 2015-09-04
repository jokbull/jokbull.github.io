---
layout: post
title:  "Tickerplant"
date:   2015-09-03 20:13:08
categories: kdb+
---

***KDB最强大的功能就是关于Realtime database, DataFeed和Client之间精巧的解决办法***

一个Hello World Level的[Demo][demo]程序, 令人惊叹的是它其实也就是一个完整的product version. 

其基本的结构如下：

{% highlight ruby linenos %}
DataFeed 
	-> tickerplant 
		-> rdb, hlcv, last etc. 
			(-> clients)
		(-> chained tickerplant) 
			(-> clients)
{% endhighlight %}


## 1.  **DataFeed** 
	
	数据接口. 可以接Realtime data也可以接Historical data. 当Realtime data的数据源发生变化时，只需要改相应的datafeed文件即可; 当需要Backtest时，把datafeed改成hdb，
用\t .z.ts等q自带的功能，实现历史回放。另外，DataFeed程序可以灵活的用**各种**语言实现。以下是[Demo][demo]程序中给出的feed.q, 随机生成数据塞给tickerplant, 也就是5010端口
{% highlight c linenos %}
	/ generate data for rdb demo

	sn:2 cut (
	 `AMD;"ADVANCED MICRO DEVICES";
	 `AIG;"AMERICAN INTL GROUP INC";
	 `AAPL;"APPLE INC COM STK";
	 `DELL;"DELL INC";
	 `DOW;"DOW CHEMICAL CO";
	 `GOOG;"GOOGLE INC CLASS A";
	 `HPQ;"HEWLETT-PACKARD CO";
	 `INTC;"INTEL CORP";
	 `IBM;"INTL BUSINESS MACHINES CORP";
	 `MSFT;"MICROSOFT CORP")

	s:first each sn
	n:last each sn
	p:33 27 84 12 20 72 36 51 42 29 / price
	m:" ABHILNORYZ" / mode
	c:" 89ABCEGJKLNOPRTWZ" / cond
	e:"NONNONONNN" / ex

	/ init.q

	cnt:count s
	pi:acos -1
	gen:{exp 0.001 * normalrand x}
	normalrand:{(cos 2 * pi * x ? 1f) * sqrt neg 2 * log x ? 1f}
	randomize:{value "\\S ",string "i"$0.8*.z.p%1000000000}
	rnd:{0.01*floor 0.5+x*100}
	vol:{10+`int$x?90}

	/ randomize[]
	\S 235721

	/ =========================================================
	/ generate a batch of prices
	/ qx index, qb/qa margins, qp price, qn position
	batch:{
	 d:gen x;
	 qx::x?cnt;
	 qb::rnd x?1.0;
	 qa::rnd x?1.0;
	 n:where each qx=/:til cnt;
	 s:p*prds each d n;
	 qp::x#0.0;
	 (qp raze n):rnd raze s;
	 p::last each s;
	 qn::0}
	/ gen feed for ticker plant

	len:10000
	batch len

	maxn:15 / max trades per tick
	qpt:5   / avg quotes per trade

	/ =========================================================
	t:{
	 if[not (qn+x)<count qx;batch len];
	 i:qx n:qn+til x;qn+:x;
	 (s i;qp n;`int$x?99;1=x?20;x?c;e i)}

	q:{
	 if[not (qn+x)<count qx;batch len];
	 i:qx n:qn+til x;p:qp n;qn+:x;
	 (s i;p-qb n;p+qa n;vol x;vol x;x?m;e i)}

	feed:{h$[rand 2;
	 (".u.upd";`trade;t 1+rand maxn);
	 (".u.upd";`quote;q 1+rand qpt*maxn)];}

	feedm:{h$[rand 2;
	 (".u.upd";`trade;(enlist a#x),t a:1+rand maxn);
	 (".u.upd";`quote;(enlist a#x),q a:1+rand qpt*maxn)];}

	init:{
	 o:"t"$9e5*floor (.z.T-3600000)%9e5;
	 d:.z.T-o;
	 len:floor d%113;
	 feedm each `timespan$o+asc len?d;}

	h:neg hopen `::5010
	/ h(".u.upd";`quote;q 15);
	/ h(".u.upd";`trade;t 5);

	init 0
	.z.ts:feed
{% endhighlight %}


这个程序的核心逻辑是用.z.ts每隔一定的时间调用feed函数。feed函数就是用[Process Communication][IPC]方式，把random生成的数据发给tickerplant，
在tickerplant的程序里用.u.upd函数整理数据. 注意，这里h就用了neg hopen，也就是异步的方法。这是必须的，当数据来的太快，而tickerplant处理不过来的时候,
同步会造成feed这个程序线程堵塞，有可能丢数据，而异步的方法就安全很多。

## 2. **tickerplant**

先贴出tick.q
{% highlight c linenos %}
	/q tick.q SRC [DST] [-p 5010] [-o h]
	system"l tick/",(src:first .z.x,enlist"sym"),".q"

	if[not system"p";system"p 5010"]

	\l tick/u.q
	\d .u
	ld:{if[not type key L::`$(-10_string L),string x;.[L;();:;()]];i::j::-11!(-2;L);if[0<=type i;-2 (string L)," is a corrupt log. Truncate to length ",(string last i)," and restart";exit 1];hopen L};
	tick:{init[];if[not min(`time`sym~2#key flip value@)each t;'`timesym];@[;`sym;`g#]each t;d::.z.D;if[l::count y;L::`$":",y,"/",x,10#".";l::ld d]};

	endofday:{end d;d+:1;if[l;hclose l;l::0(`.u.ld;d)]};
	ts:{if[d<x;if[d<x-1;system"t 0";'"more than one day?"];endofday[]]};

	if[system"t";
	 .z.ts:{pub'[t;value each t];@[`.;t;@[;`sym;`g#]0#];i::j;ts .z.D};
	 upd:{[t;x]
	 if[not -16=type first first x;if[d<"d"$a:.z.P;.z.ts[]];a:"n"$a;x:$[0>type first x;a,x;(enlist(count first x)#a),x]];
	 t insert x;if[l;l enlist (`upd;t;x);j+:1];}];

	if[not system"t";system"t 1000";
	 .z.ts:{ts .z.D};
	 upd:{[t;x]ts"d"$a:.z.P;
	 if[not -16=type first first x;a:"n"$a;x:$[0>type first x;a,x;(enlist(count first x)#a),x]];
	 f:key flip value t;pub[t;$[0>type first x;enlist f!x;flip f!x]];if[l;l enlist (`upd;t;x);i+:1];}];

	\d .
	.u.tick[src;.z.x 1];
{% endhighlight %}

### 2.1 sym.q 
 在tick.q其中第二行load了tick/sym.q,其内容如下：
{% highlight c linenos %}
quote:([]time:`timespan$(); sym:`g#`symbol$(); bid:`float$(); ask:`float$(); bsize:`long$(); asize:`long$(); mode:`char$(); ex:`char$())
trade:([]time:`timespan$(); sym:`g#`symbol$(); price:`float$(); size:`int$(); stop:`boolean$(); cond:`char$(); ex:`char$())
{% endhighlight %}
就是建立了quote和trade两个table，但是暂时不明白的事，rdb其实不在这里，为什么要建立这两张表，占位么？

### 2.2 u.q 
  在tick.q第六行load了tick/u.q, 就是utils, 包括init/del/select/publish/subscribe/add/end 这几个基本功能
{% highlight c linenos %}
\d .u
init:{w::t!(count t::tables`.)#()}

del:{w[x]_:w[x;;0]?y};.z.pc:{del[;x]each t};

sel:{$[`~y;x;select from x where sym in y]}

pub:{[t;x]{[t;x;w]if[count x:sel[x]w 1;(neg first w)(`upd;t;x)]}[t;x]each w t}

add:{$[(count w x)>i:w[x;;0]?.z.w;.[`.u.w;(x;i;1);union;y];w[x],:enlist(.z.w;y)];(x;$[99=type v:value x;sel[v]y;0#v])}

sub:{if[x~`;:sub[;y]each t];if[not x in t;'x];del[x].z.w;add[x;y]}

end:{(neg union/[w[;;0]])@\:(`.u.end;x)}
{% endhighlight %}
注意，::就是全局变量

#### .u.init

 建立了两个全局变量.u.w和.u.t, 其中.u.t是tables, .u.w是tables的行数统计

#### .u.del

 在.u.w中删除关于y table的关于x的数据?
 [.z.pc][.z.pc] 当handle close的时候，在每一个表里删去记录

#### .u.sel

 from x where y

#### .u.pub

 publish 核心函数！当.u.upd被调用时，执行了.u.pub, 在各个client端执行client下面的upd函数

```

 	(neg first w)(`upd;t;x)

```

#### .u.add

 subscribe record list

#### .u.sub
	
 subscribe 核心函数！在x表里订阅sym是y的股票

#### .u.end
 
 在各个clients远端执行`.u.end


## tick.q
	
	tick.q本身在建立了quote和trade两张空表之后，增加了.u中的函数

#### .u.ld
 
 关于log的

#### .u.tick
 
 主函数，在最后一行.u.tick[src;.z.x 1]被调用。
 调用了.u.init, 检查tables是否都包含`sym`time列，`g#化每张表，
 定义了全局变量.u.d为当时日期，初始化日志

#### .u.endofday
  
  执行.u.end d, 日期+1， 重置log

#### .u.ts
  
  执行endofday的入口?

#### .u.upd

 核心函数！两种模式，当在启动的命令行中指定了-t参数的时候，
 是执行第14-18行的部分，当没有指定-t参数的时候对应20-24行


 主要是20-24行的模式，补充定义了每1000ms执行一次.u.ts, 什么时候执行.u.upd? .u.upd的逻辑核心就是check then pub then log.

 ```

 pub[t;$[0>type first x;enlist f!x;flip f!x]]

 ```

 pub即.u.pub, 在client的客户端完成计算


## 3. rdb 

 r.q 即rdb, Realtime database. rdb也是一种特殊的client，link to tickerplant. 其upd函数就是insert


	/q tick/r.q [host]:port[:usr:pwd] [host]:port[:usr:pwd]
	/2008.09.09 .k ->.q

	if[not "w"=first string .z.o;system "sleep 1"];

	upd:insert;

	/ get the ticker plant and history ports, defaults are 5010,5012
	.u.x:.z.x,(count .z.x)_(":5010";":5012");

	/ end of day: save, clear, hdb reload
	.u.end:{t:tables`.;t@:where `g=attr each t@\:`sym;.Q.hdpf[`$":",.u.x 1;`:.;x;`sym];@[;`sym;`g#] each t;};

	/ init schema and sync up from log file;cd to hdb(so client save can run)
	.u.rep:{
		(.[;();:;].)each x;
		if[null first y;:()];
		-11!y;
		system "cd ",1_-10_string first reverse y};
	/ HARDCODE \cd if other than logdir/db

	/ connect to ticker plant for (schema;(logcount;log))
	.u.rep .(hopen `$":",.u.x 0)"(.u.sub[`;`];`.u `i`L)";


这个线程里的.u和tickerplant里面的.u不是一样的。.u.rep[应该是replay的意思][u_rep]，在最后一行中，按从右到左的结合顺序，应该是同步的执行了
执行了.u.sub[`;`]并返回.u.i和.u.L，返回值作为.u.rep的输入参数，也就是x是.u.i, y是.u.L。
[−11!y][-11]是streaming execute over file y, 基本就是读log文件. 这里.u.rep只会被执行一遍，也就是在初始化rdb的时候，而如果rdb在开始的时候
log文件有内容，然后那么就会-11!y,同步一遍logfile, 也就是把logfile记录的内容读进了rdb

## 4. Other clients

clients的基本原理就是和rdb一样，让tickerplant把数据publish过来的时候，被调用自身的upd函数。如果这个client仅仅是某个终端的中间结果，那么给
它分配一个独立的端口即可，让其他client连接此端口即可。


[demo]:      http://code.kx.com/wiki/Startingkdbplus/tick
[IPC]:       http://code.kx.com/wiki/Startingkdbplus/ip
[.z.pc]:     http://code.kx.com/wiki/Reference/dotzdotpc
[Dot]:		 http://code.kx.com/wiki/JB:QforMortals2/functions#Functional_Forms_of_Amend
[-11]:       http://code.kx.com/wiki/Reference/BangSymbolInternalFunction
[u_rep]:    http://www.firstderivatives.com/downloads/q_for_Gods_July_2014.pdf
