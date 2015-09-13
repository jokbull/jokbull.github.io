---
layout: post
title:  "kdb websocket"
date:   2015-09-04 20:13:08
categories: kdb+
---

A very good [example][link] about websocket.

First of all, [kdb supports websocket][kdbwebsocket]! A q session as server, and web browser as client. 
[.z.ws][dotzdotws] handle the received message, and use ` {neg[.z.w]x}` echo the message back to client(web).

My ideal schema is :

TP/Chained publish the real-time data to a kdb client(named after 'Visualization') which is the websocket server, 
a javascript chart using echarts([dynamic line chart][echart1],[dynamic K chart][echart2]) as a client of websocket, the Visualization upd function is `{neg[.z.w] data}` to web,
and in javascript,  

```{javascript}
ws.onopen    = function(x){
	// create the link
	...
	// initialize the chart
	...
}
ws.onmessage = function(x){
	// myChart.addData([x]}
	...
}

```
Not sure how to use shiny/htmlwidgets to generate a html like this. Guess JS() function is helpful...

Hope the final real-time chart like [this][highchart], highcharts addPoint is the same as echart addData...


At least, I have an example to dynamically fill the updated data in the chart
1. websocket server is a client of tp, -p 5099
2. use htmlwidgets to generate the index.html
3. change the renderValue function, adding ws part
4. in q: 

```q
.z.ws:{$[x~"connection";h::neg[.z.w];h string 1]}
upd:h string x
```

```javascript
ws = new WebSocket("ws://localhost:5099/"); 
ws.onopen=function(e){ws.send("connection")} 
ws.onclose=function(e){}
ws.onmessage=function(e){
  instance.addData([
      [
          0,        // 系列索引
          e.data,   // 新增数据
          false,     // 新增数据是否从队列头部插入
          true    // 是否增加队列长度，false则自定删除原有数据，队头插入删队尾，队尾插入删队头
      ]
    ]);
 instance.setOption(x);
}
ws.onerror=function(e){}


instance.setOption(x);
```

5. The only problem is failed to add additionData item for category x-value.
6. In future, a shiny web app would be 

```text
subscribe/change the viewer 
	
	-> recreate a shiny widget 
	
	-> create ws -> ws.open -> .u.sub 

	-> .u.upd push the data through websocket

	-> re-plot the chart embed in widget
```


[link]: 	http://www.ibm.com/developerworks/cn/web/1112_huangxa_websocket/
[dotzdotws]:  http://code.kx.com/wiki/Reference/dotzdotws
[kdbwebsocket]: http://code.kx.com/wiki/Cookbook/Websocket
[echart1]:	http://echarts.baidu.com/doc/example/dynamicLineBar.html
[echart2]:  http://echarts.baidu.com/doc/example/dynamicScatterK.html
[highchart]: http://www.highcharts.com/stock/demo/dynamic-update
