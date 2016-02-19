---
layout: post
title: "[HA]Pacemaker的分数机制和资源黏性"
date: 2016-02-18 10:42
keywords: tfs,ha
description: Heartbeat v3资源管理层的重点
categories: [Storage]
tags: [ha, pacemaker]
group: archive
icon: globe
---

　　Pacemaker根据CIB配置决策服务在集群节点上的分配情况，原理是怎么样的呢？这里涉及了级别的资源黏性和节点级别的节点分数，它们都是依靠分数机制来表征的，有人也把分数机制叫做得票机制。在[《[HA]Pacemaker的CIB配置》](http://smartmz.github.io/2016/02/16/ha-pacemaker-cib)一文中，提到CIB配置中的stickiness参数和score参数就是针对分数和资源黏性的设置，本文详述。

###分数值(Value)
　　分数值是资源黏性、节点分数等的基础，分数值的高低体现资源的黏性程度和节点优先使用级别，是HA做出资源分配、资源迁移决策的直接影响因素。   
　　Pacemaker支持的分数值有 整数(Integer)、正无穷(+INFINITY)、负无穷(-INFINITY)，无穷在Pacemaker中实现为1,000,000. 分数值之间可以做加减运算：   

* Integer 正常运算
* Any Value + INFINITY = INFINITY
* Any Value - INFINITY = -INFINITY
* INFINITY - INFINITY = -INFINITY

###资源黏性(Stickiness)
　　表述资源更倾向于运行在哪个节点。在CIB配置中的cib→configuration→resources→primitive→**meta_attributes**中配置，具体的配置参数是resource-stickiness和migration-threshold (表示失败多少次就要将节点迁离，Pacemaker 1.0之前用resource-failure-stickiness，绝对值表示资源失败一次黏性降低多少)。   
　　**注意**，该性质只是表征资源对节点的倾向，在多节点的集群中一般不能完全约束资源运行在哪个节点上。在集群工作过程中，资源的启停会使分数值动态变化，从而对HA进行资源迁移产生影响。

####规则
　　Stickiness可以取分数值允许的所有值，不同值的节点上资源黏性的强弱遵循以下规则：   

* **等于0** 表示资源愿意运行在集群中HA认为最合适的节点位置上，如果有更好的节点或者是负载低的节点进入集群，服务就会切换，这样就等同于集群故障自动恢复。
* **大于0** 表示资源更愿意迁移到（或留在）当前位置，分数值越高表示资源越愿意迁移到（或留在）该节点上。如果集群中出现正的更高分数值的节点，资源会迁移到新的节点。
* **小于0** 表示资源更愿意从当前位置迁离，分数值越小表示资源越愿意从该节点上迁离。如果集群中出现负的更高分数值的节点，资源会迁移到新的节点。
* **INFINITY** 表示资源总是愿意迁移到（或留在）该节点，如果节点服务失败，HA认为服务还是倾向于该节点，大致上等同禁用了集群故障自动恢复。**例外**情况：1，节点因为离线 2，待机 3，达到migration-threshold配置的迁移阈值、配置被更改。
* **-INFINITY** 表示资源总是希望从该节点迁离，除非只能在它上面运行该资源。

####配置示例
　　需要根据版本分情况。

#####Pacemaker 1.0之前
　　resource-stickiness和resource-failure-stickiness搭配，resource-stickiness表示资源成功一次资源获得多少分，一般设置正数；resource-failure-stickiness的绝对值表示资源失败一次资源扣掉多少分，一般设置负数。未配置时默认都为0.
　　比如：   

* resource-stickiness=100 & resource-failure-stickiness=-100 资源每次成功黏性增强，加100分，每次失败黏性降低，减100分
* resource-stickiness=INFINITY & resource-failure-stickiness=-INFINITY 资源一旦成功就变为集群中**最愿意**运行服务的节点，以后需要迁移服务时仍然在该节点上运行服务的倾向最高；资源一旦失败就变为集群中**最不愿意**运行服务的节点，尽量不会再在该节点上运行服务。

#####Pacemaker 1.0之后
　　resource-stickiness和migration-threshold搭配，resource-stickiness表示资源成功一次资源获得多少分，一般设置正数，未配置默认为0；migration-threshold表示资源失败多少次以后资源就会从该节点迁离，一般也设置正数，未配置默认为INFINITY。
　　比如：

* resource-stickiness=100 & migration-threshold=10 资源每次成功加100分，失败10次后就要将服务从该节点迁离
* resource-stickiness=100 & migration-threshold=INFINITY 资源每次成功加100分，失败无影响

###节点分数(Score)
　　表述某资源应该在哪个节点上运行。在CIB配置中需要在cib→configuration→**constraints**→**rsc_location**中配置，约束某个节点对某一资源运行的倾向，参数为score.    
　　**注意**，该性质约束了可以运行某个资源的节点有哪些，需要和资源黏性区分开。节点分数在多节点集群中如果针对同一资源约束了多个节点，那么一般也不能完全确定该资源会运行在哪个特定节点上，除非使用了INFINITY/-INFINITY。在集群工作过程中，节点的分数**不会**动态变化。

####规则
　　可以取分数值允许的所有值，不同的取值遵循以下规则：

* **大于0** 表示某资源应该运行在该节点上，分数值大的优先。
* **小于0** 表示某资源不应该运行在该节点上。
* **INFINITY** 表示某资源必须运行在该节点上，除非**例外**情况：1，节点因为离线 2，待机 3，达到migration-threshold配置的迁移阈值、配置被更改。
* **-INFINITY** 表示资源不能运行在该节点上，除非只能在它上面运行。
####配置示例
　　以下情况不考虑资源黏性，只是用来体会以下节点分数的作用。假设有三个节点N{1,2,3}，有两个资源S{1,2}。   
　　配置示例之前需要介绍一个CIB配置的资源全局参数symmetric-cluster，在cib→configuration→cluster_property_set中配置，默认为true表示资源可以在集群的任何节点上运行，置为false表示资源需要根据约束配置来决定在哪个节点上运行。

* 配置1
<center>

 | | | | | | | 
---|---|---|---|---|---|---
节点|N1|N1|N2|N2|N3|N3
资源|S1|S2|S1|S2|S1|S2
分数值|200|未设置|未设置|200|0|0

</center>
symmetric-cluster=false


