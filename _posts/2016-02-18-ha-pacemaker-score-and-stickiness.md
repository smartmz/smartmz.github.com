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

　　Pacemaker根据CIB配置决策服务在集群节点上的分配情况，原理是怎么样的呢？这里涉及了得分和资源黏性的相关概念。在[《[HA]Pacemaker的CIB配置》](http://smartmz.github.io/2016/02/16/ha-pacemaker-cib)一文中，提到CIB配置中的stickiness参数和score参数就是针对分数和资源黏性的设置，本文详述。
###分数值(Value)
　　分数值是资源黏性、节点分数等的基础概念，分数值的高低体现资源的黏性程度和节点优先使用级别，是HA做出资源分配、资源迁移决策的直接影响因素。   
　　Pacemaker支持的分数值有 整数(Integer)、正无穷(+INFINITY)、负无穷(-INFINITY)，无穷在Pacemaker中实现为1,000,000. 分数值之间可以做加减运算：

* Integer 正常运算
* Any Value + INFINITY = INFINITY
* Any Value - INFINITY = -INFINITY
* INFINITY - INFINITY = -INFINITY
###资源黏性(Stickiness)
　　表述资源更倾向于运行在哪个节点。在CIB配置中的cib→configuration→resources→primitive→**meta_attributes**中配置，具体的配置参数是resource-stickiness和migration-threshold (表示失败多少次就要将节点迁离，Pacemaker 1.0之前用resource-failure-stickiness，绝对值表示资源失败一次黏性降低多少)。   
　　**注意**，该性质只是表征资源对节点的倾向，不能完全约束资源运行在哪个节点上。在集群工作过程中，资源的启停会使分数值动态变化，从而对HA进行资源迁移产生影响。
####规则
　　Stickiness可以取分数值允许的所有值，不同值的节点上资源黏性的强弱遵循以下规则：
* **等于0** 表示资源愿意运行在集群中HA认为最合适的节点位置上，如果有更好的节点或者是负载低的节点进入集群，服务就会切换，这样就等同于集群故障自动恢复。
* **大于0** 表示资源更愿意迁移到（或留在）当前位置，分数值越高表示资源越愿意迁移到（或留在）该节点上。如果集群中出现正的更高分数值的节点，资源会迁移到新的节点。
* **小于0** 表示资源更愿意从当前位置迁离，分数值越小表示资源越愿意从该节点上迁离。如果集群中出现负的更高分数值的节点，资源会迁移到新的节点。
* **INFINITY** 表示资源总是愿意迁移到（或留在）该节点，如果节点服务失败，HA认为服务还是倾向于该节点，大致上等同禁用了集群故障自动恢复。**例外**情况：1，节点因为离线 2，待机 3，达到migration-threshold配置的迁移阈值、配置被更改。
* **-INFINITY** 表示资源总是希望从该节点迁离，除非只能在它上面运行该资源。
####几种配置的情况
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
　　表述该

