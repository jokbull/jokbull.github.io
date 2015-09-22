---
layout: 
title: uqer backtest platform
categories: [quant]
tags: [python]
published: True

---


通联数据搞的[量化实验室](https://uqer.io/labs/)，基于Python，估计背后是和IPython一模一样的架构，一遍写代码，一遍写文档。说起来RMarkdown, knit也都是这个趋势，科技写作。

他画图的部分使用的highcharts, 看上去只有净值图，这个对于research来说是远远不够的。相信他后期也会加入新的类型。我最看好的两个js library是echarts和amcharts，echarts是baidu出品，速度快，画单个图非常实用。另外不得不提的是，对数据流的动态图也是perfect。但是这个library对我来说目前的瓶颈是r api不够好(画最基本的单图都已经封装)，对复合图支持也不够好（应该需要先手写css之类，然后用htmlwidgets::JS往里面加JS function之类的方法。。。还没有研究清楚）。等待大牛们的github更新。。。
而amCharts中amStockChart足够完美~~， rAmCharts封装的赏心悦目。暂时的第一选择。

回到uqer量化实验室，他有 Strategy回测、CAL金融定价库、DATAAPI三个核心的部分。
其中DATAAPI应该是通联的强项，但也是没有太多可以借鉴的地方。略过。CAL金融定价库，从期权的example来看，和QuantLib的思路是一模一样的。完美但是用不上。回测的部分可以好好研究借鉴：


#### [深入浅出量化实验室](/material/深入浅出量化实验室.pdf)
链接中的pdf即帮助手册。从35页到46页详细介绍了quantz的回测系统架构。

###### account class
1. sim_params: 回测参数
2. universe_all: 证券池（也许应该并入回测参数）
3. universe: 根据每个交易日的数据，剔除了数据缺失和数据异常的证券的证券池\
4. current_date/current_minute
5. days_counter: 计数
6. trading_days：回测期间的所有交易日列表
7. position： 现金及证券头寸 (Cash也作为一种特殊的证券？)
8. cash：  现金头寸
9. secpos: 证券头寸，字典，键为证券代码，值为头寸
10. valid_secpos: 有效证券头寸
11. avail_secpos: 可卖，即：上一交易日结束时有效证券头寸
12. referencePrice：参考价，一般使用的是上一日收盘价或者上一分钟收盘价，PrevClosePrice
13. referenceReturn: 参考收益率，一般使用的是上一日收益率
14. referencePortfolioValue：参考投资策略价值，使用参考价计算
15. blotter：下单指令列表
16. commission：手续费标准
17. slippage：滑点标准
18. pending_blotter：所有未成交指令
19. initial(): 策略初始
20. handle_data(): 每日调仓。



从设计上，account着眼的是模拟一个股票账户，记录下股票账户的状态（持仓、交易、交易成本设定、行情）。考虑到 单策略对多账户、多策略对多账户、多策略对单账户的可能，account类是一个非常必要的。
但是可以考虑把参数、模型等先打包，然后在account里只是加入对这些类的reference即可，不必要在每个account类里实例化各个东西。








