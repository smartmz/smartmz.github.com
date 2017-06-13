---
layout: post
title: "京东追踪系统-Hydra"
date: 2016-08-23 15:45
keywords: trace, hydra
description: JD的Hydra
categories: [Trace]
tags: [trace]
group: Trace
icon: globe
---

# 京东-Hydra

对SOA服务提供对基础组件的Trace功能

## 功能

实现在基础组件上收集在组件上产生的行为的时间消耗，并且提供跟踪查询页面，对跟踪到的数据进行查询和展示。

<!-- more -->

![领域模型](http://ww4.sinaimg.cn/mw690/a8484315jw1f73j8gfa9tj213l0oo78k.jpg)

* Trace:一次服务调用追踪链路，TraceId是一次追踪链路的唯一标识。
* Span:追踪服务调基本结构，多span形成树形结构组合成一次Trace追踪记录，SpanId是个Span的唯一标识，一条追踪链路上的SpanId按序列自增，通过parentId形成树。

	> 	![Span示例](http://ww2.sinaimg.cn/mw690/a8484315jw1f73o6myrmhj20i00dht9c.jpg)    
	> 	在领域模型中，服务A被调用到调用完成的过程，就是一次trace。而每一个服务被调用并返回的过程（一去一回的箭头）为一个Span。可以看到这个示例中包含5个span，client-A，A-B，A-C，C-D，C-E。span本身以树形结构展开，A-C是C-D和C-E的父 span，而client-A是整个树形结构的root span。之后要提到的一个概念就是annotation，annotation代表在服务调用过程中发生的一些我们感兴趣的事情，如图所示C-E上标出 来的那四个点，就是四个annotation，来记录事件时间戳，分别是C服务的cs（client send），E服务的sr（server receive）,E服务的ss（server send）, C服务的cr（client receive）。如果有一些自定义的annotation，会作为BinaryAnnotation，其实就是一个k-v对，记录任何跟踪系统想记录的信息，比如服务调用中的异常信息，重要的业务信息等。

* Annotation:在Span中的标注点，记录整个Span时间段内发生的事件
* BinaryAnnotation:属于Annotation一种类型，和普通Annotation区别是自定义标注在Span中发生的事件，自定义一些信息

## 架构机制

### 完整架构

**高并发，大数据量：Hydra-client | Queue | hbase**

![完整架构](http://ww3.sinaimg.cn/mw690/a8484315jw1f73j8hl3y5j212k0qvae9.jpg)

* Hydra-dubbo: dubbo过滤器（埋点）
* Hydra-client: 封装跟踪采集trace_data的通用API，通过API调用获取Hydra-dubbo输出的trace_data，通过Queue（ArrayBlockingQueue）异步推送(jar，直接由dubbo依赖)
* Hydra-collector: 接收Hydra-client输出的trace_data，推队列（web服务，java）
* Queue(metaQ): 非阻塞队列，异步传输trace_data
* Hydra-collector-service: 读队列，trace_data落地（web服务，java）
* HBase: trace_data存储
* Hydra-web: 获取trace_data，展示（web）
* Mysql: Hydra信息存储
* Hydra-manager: 服务（接口）注册、采样率调整、发送seed供生成全局唯一TraceId（web服务，java）
* + zookeeper: 各个服务点依赖zk读取Hydra-manager和Hydra-collector获取数据交互路由点，来完成跟踪数据的推送和跟踪的控制。

### 精简架构

**高并发，小数据量：Hydra-client | Queue | mysql**
**低并发，小数据量：Hydra-client | mysql** <下例>

![精简架构](http://ww3.sinaimg.cn/mw690/a8484315jw1f73j8hnb72j20yi0qvdi9.jpg)

* Mysql: Hydra信息存储 + trace_data存储，异步

	> 监控数据，时效性较高，长期的监控数据保留意义不大，可以在主表上设计明确的时间戳字段，使用者自行决定何时对保存的历史数据进行迁移。

## 可视化

基于angularJS，D3.js实现。

![查询界面](http://ww3.sinaimg.cn/mw690/a8484315jw1f73o6mzq2ij20rb0b90vz.jpg)

* 提供查询功能，通过应用名（池子）--服务（接口）查询，可以指定查询时间段、跟踪次数。
* 提供筛选条件，调用响应时间、是否发生异常。调用响应时间指的是这一次服务调用从调用开始到调用结束的时间，是否发生异常则包括一次服务调用中所有历经的服务抛出的异常都会捕获到。
* 提供排序功能，对后台查询到的数按照时间或耗时排序。

每一次跟踪展示服务调用层级与响应时间的时序图。

![追踪界面](http://ww2.sinaimg.cn/mw690/a8484315jw1f73o6mw3e2j20q006mabe.jpg)

* 绿色代表服务调用时间，浅蓝色代表网络耗时，如果服务调用抛出异常被Hydra捕捉到，用红色表示。
* 鼠标移动到时序图中的每一个对象上，会Tip展现详细信息，包括服务名、方法名、调用时长、Endpoint、异常信息等。
* 左侧的树形结构图可以收起和展开，同时右侧的时序图产生联动。

## 存储

trace_data: hbase/mysql
Hydra_data: mysql

## 其他

* 入侵程度：Hydra-client作为jar直接由dubbo依赖，不入侵业务代码。
* 检测粒度：接口调用级别
* 插件：不支持第三方插件

## 参考资料

https://github.com/MarsYoung/hydra （fork）    
http://blog.csdn.net/wilsonke/article/details/39935097


