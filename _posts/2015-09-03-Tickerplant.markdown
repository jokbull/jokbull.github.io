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

 * 其中第二行load了tick/sym.q,其内容如下：
{% highlight c linenos %}
quote:([]time:`timespan$(); sym:`g#`symbol$(); bid:`float$(); ask:`float$(); bsize:`long$(); asize:`long$(); mode:`char$(); ex:`char$())
trade:([]time:`timespan$(); sym:`g#`symbol$(); price:`float$(); size:`int$(); stop:`boolean$(); cond:`char$(); ex:`char$())
{% endhighlight %}
就是建立了quote和trade两个table，但是暂时不明白的事，rdb其实不在这里，为什么要建立这两张表，占位么？

 * 第六行load了tick/u.q, 就是utils, 包括init/del/select/publish/subscribe/add/end 这几个基本功能
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






[demo]:      http://code.kx.com/wiki/Startingkdbplus/tick
[IPC]:       http://code.kx.com/wiki/Startingkdbplus/ip