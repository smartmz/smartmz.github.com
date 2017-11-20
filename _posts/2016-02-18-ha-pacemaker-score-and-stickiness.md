---
layout: post
title: "Pacemaker的分数机制和资源黏性"
date: 2016-02-18 10:42
keywords: tfs,ha
description: Heartbeat v3资源管理层的重点
categories: [HA]
tags: [ha, pacemaker]
group: archive
icon: globe
---

<!-- more -->

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
　　**注意**，该性质只是表征资源对节点的倾向，在多节点的集群中一般不能完全约束资源运行在哪个节点上。在集群工作过程中，资源黏性分数不会动态变化，但是会影响节点分数，从而对HA进行资源迁移产生影响。

####规则
　　Stickiness可以取分数值允许的所有值，不同值的节点上资源黏性的强弱遵循以下规则：   

* **等于0** 表示资源愿意运行在集群中HA认为最合适的节点位置上，如果有更好的节点或者是负载低的节点进入集群，服务就会切换，这样就等同于集群故障自动恢复。
* **大于0** 表示资源更愿意迁移到（或留在）当前位置，分数值越高表示资源越愿意迁移到（或留在）该节点上。如果集群中出现正的更高分数值的节点，资源会迁移到新的节点。
* **小于0** 表示资源更愿意从当前位置迁离，分数值越小表示资源越愿意从该节点上迁离。如果集群中出现负的更高分数值的节点，资源会迁移到新的节点。
* **INFINITY** 表示资源总是愿意迁移到（或留在）该节点，如果节点服务失败，HA认为服务还是倾向于该节点，大致上等同禁用了集群故障自动恢复。**例外**情况：1，节点因为离线 2，待机 3，达到migration-threshold配置的迁移阈值、配置被更改。
* **-INFINITY** 表示资源总是希望从该节点迁离，除非只能在它上面运行该资源。

####配置示例
　　需要根据版本分情况。

#####　Pacemaker 1.0之前
　　resource-stickiness和resource-failure-stickiness搭配，resource-stickiness表示资源成功一次资源获得多少分，一般设置正数；resource-failure-stickiness的绝对值表示资源失败一次资源扣掉多少分，一般设置负数。未配置时默认都为0.    
　　比如：   

* resource-stickiness=100 & resource-failure-stickiness=-100 资源每次成功黏性增强就加100分，每次失败黏性降低减100分
* resource-stickiness=INFINITY & resource-failure-stickiness=-INFINITY 资源一旦成功就变为集群中**最愿意**运行服务的节点，以后需要迁移服务时仍然在该节点上运行服务的倾向最高；资源一旦失败就变为集群中**最不愿意**运行服务的节点，尽量不会再在该节点上运行服务。

#####　Pacemaker 1.0之后
　　resource-stickiness和migration-threshold搭配，resource-stickiness表示资源成功一次资源获得多少分，一般设置正数，未配置默认为0；migration-threshold表示资源失败多少次以后资源就会从该节点迁离，一般也设置正数，未配置默认为INFINITY。   
　　比如：

* resource-stickiness=100 & migration-threshold=10 资源每次成功加100分，失败10次后就要将服务从该节点迁离
* resource-stickiness=100 & migration-threshold=INFINITY 资源每次成功加100分，失败无影响

###节点分数(Score)
　　表述某资源应该在哪个节点上运行。在CIB配置中需要在cib→configuration→**constraints**→**rsc_location**中配置，约束某个节点对某一资源运行的倾向，参数为score.    
　　**注意**，该性质约束了可以运行某个资源的节点有哪些，需要和资源黏性区分开。节点分数在多节点集群中如果针对同一资源约束了多个节点，那么一般也不能完全确定该资源会运行在哪个特定节点上，除非使用INFINITY/-INFINITY。在集群工作过程中，节点分数会动态变化。

####规则
　　可以取分数值允许的所有值，不同的取值遵循以下规则：

* **大于0** 表示某资源应该运行在该节点上，分数值大的优先。
* **小于0** 表示某资源不应该运行在该节点上。
* **INFINITY** 表示某资源必须运行在该节点上，除非**例外**情况：1，节点因为离线 2，待机 3，达到migration-threshold配置的迁移阈值、配置被更改。
* **-INFINITY** 表示资源不能运行在该节点上，除非只能在它上面运行。

　　对于INFINITY和-INFINITY这两个值，实际上主要用途就是为了控制永远不切换和只要失败必须切换用的。

####配置示例
　　以下情况不考虑资源黏性，只是用来体会以下节点分数的作用。假设有三个节点N{1,2,3}，有两个资源S{1,2}。   
　　配置示例之前需要介绍一个CIB配置的资源全局参数symmetric-cluster，在cib→configuration→cluster\_property\_set中配置，默认为true表示资源可以在集群的任何节点上运行，置为false表示资源需要根据约束配置来决定在哪个节点上运行。也可以使用命令来设置该值：
　　`# crm_attribute --name symmetric-cluster --update false`

* 配置1

	| | | | | | | |
	|---|---|---|---|---|---|---|
	|资源|S1|S2|S1|S2|S1|S2|
	|节点|N1|N1|N2|N2|N3|N3|
	|分数|200|未设置|未设置|200|0|0|

　　将symmetric-cluster=false，禁止资源在任意节点上运行，这样资源S1不能在N2上运行，资源S2不能在N1上运行。集群启动后，N1-S1-score > N3-S1-score， S1在N1，同理S2在N2上运行；如果N1、N2宕机，S1、S2会迁移到N3上运行。   

* 配置2

	| | | | | | | |
	|---|---|---|---|---|---|---|
	|资源|S1|S2|S1|S2|S1|S2|
	|节点|N1|N1|N2|N2|N3|N3|
	|分数|200|-INFINITY|-INFINITY|200|未设置|未设置|

　　将symmetric-cluster=true，资源可以在任意节点上运行，该配置和配置1就等同了。

###基于资源黏性和节点分数的决策原理
　　由于资源黏性在Pacemaker1.0版本前后设置属性不同，决策原理上也有所改变，需要分开讨论。
####　Pacemaker 1.0之前
　　节点决策得分的计算公式为：`Node_Score+Resource_Stickiness+Fail-Count*Resource_failure_Stickiness`，其中Fail-Count为资源在该节点运行失败的次数，每个资源在节点上都会对应记录Fail-Count.    
　　现在给出一个集群中某个资源S1在节点N{1,2}的配置来说明决策过程。

```xml
<!--设置资源黏性为 成功100，失败-100-->
<primitive class="ocf" id="S1">
  <meta_attributes>
    <nvpair id="fail-stickiness" name="resource-failure-stickiness" value="-100"/>
    <nvpair id="succ-stickiness" name="resource-stickiness" value="100"/>
  </meta_attributes>
</primitive>
<!--设置节点分数 S1-N1 200， S2-N2 150-->
<constraints>
  <rsc_location id="s1_loc" rsc="S1">
    <rule id="s1_n1_loc_rule" score="200">
      <expression id="s1_loc_rule_expr" attribute="#uname" operation="eq" value="N1"/> 
    </rule>
    <rule id="s1_n2_loc_rule" score="150">
      <expression id="s1_loc_rule_expr" attribute="#uname" operation="eq" value="N2"/>
    </rule>
  </rsc_location>
</constraints> 
```

1. 初始，启动N{1,2}上的Pacemaker，资源未运行算0分，只算节点分数：

	> N1 = 200+0+0\*(-100) = 200   
	> N2 = 150+0+0\*(-100) = 150   
	> 决策：N1>N2，S1在N1上运行

2. N1的S1运行，引起节点分数变化：
 
	> N1 = 200+100+0\*(-100) = 300   
	> N2 = 150+0+0\*(-100) = 150   
	> 决策：N1>N2，S1仍在N1上运行，不发生迁移

3. N1的S1运行一段时间无法被检测到，引起节点分数变化：

	> N1 = 200+100+1\*(-100) = 200   
	> N2 = 150+0+0\*(-100) = 150   
	> 决策：N1>N2，S1仍在N1上运行，不发生迁移

4. N1的S1运行一段时间再次无法被检测到，引起节点分数变化：
  
	> N1 = 200+100+2\*(-100) = 100   
	> N2 = 150+0+0\*(-100) = 150   
	> 决策：N2>N1，S1从N1迁移到N2

5. N2的S1运行，引起节点分数变化：

	> N1 = 200+0+2\*(-100) = 0 //资源未运行，第二项为0   
	> N2 = 150+100+0\*(-100) = 250   
	> 决策：N2>N1，S1仍在N2上运行，不发生迁移

6. N2的S1运行一段时间无法被检测到，引起节点分数变化：   

	> N1 = 200+0+2\*(-100) = 0   
	> N2 = 150+100+1\*(-100) = 150   
	> 决策：N2>N1，S1仍在N2上运行，不发生迁移

7. N2的S1运行一段时间再次无法被检测到，引起节点分数变化：

	> N1 = 200+0+2\*(-100) = 0   
	> N2 = 150+100+2\*(-100) = 50   
	> 决策：N2>N1，S1仍在N2上运行，不发生迁移

8. N2的S1运行一段时间再次无法被检测到，引起节点分数变化：

	> N1 = 200+0+2\*(-100) = 0   
	> N2 = 150+100+3\*(-100) = -50   
	> 决策：N1>N2 && N2<0，S1从N2迁移到N1

9. N1的S1运行，引起节点分数变化：

	> N1 = 200+100+2\*(-100) = 100   
	> N2 = 150+0+3\*(-100) = -150   
	> 决策：N1>N2 && N2<0，S1仍在N1上运行，不发生迁移；由于N2为负数，S1会一直在N1上无法被检测到、重启，不会迁移到N2

　　重启节点的Pacemaker或Heartbeat服务，节点分数会被重置；也可以通过以下相关命令设置Fail-Count值来打破当前状态。

```sh
# crm resource failcount 资源ID show 节点Uname
# crm resource failcount 资源ID set 节点Uname N

# crm_failcount -G -U 节点Uname -r 资源ID
# crm_failcount -D -U 节点Uname -r 资源ID //重置为0
```
　　如果是资源属于某个资源组的话，

1. 集群中有一个资源组，包含多个资源，可以认为没有资源组，只有资源，只是资源需要迁移的时候迁移对象是整个资源组。
2. 集群中有多个资源组，每个组包含一个资源，可以把资源组认为就是资源。
3. 集群中有多个资源组，每个组包含多个资源，节点分数是资源组内各个资源对节点分数的贡献的总和，资源需要迁移的时候迁移对象是整个资源组。

####　Pacemaker 1.0之后
　　节点决策得分的计算公式为：`Node_Score+Resource_Stickiness`，同时还需要比较Fail-Count和migration-threshold的大小，其中Fail-Count为资源在该节点运行失败的次数。   
　　现在给出一个集群中某个资源S1在节点N{1,2}的配置来说明决策过程。

```xml
<!--设置资源黏性为 成功100，失败迁移阈值2-->
<primitive class="ocf" id="S1">
  <meta_attributes>
    <nvpair id="fail-stickiness" name="migration-threshold" value="2"/>
    <nvpair id="succ-stickiness" name="resource-stickiness" value="100"/>
  </meta_attributes>
</primitive>
<!--设置节点分数 S1-N1 200， S2-N2 150-->
<constraints>
  <rsc_location id="s1_loc" rsc="S1">
    <rule id="s1_n1_loc_rule" score="200">
      <expression id="s1_loc_rule_expr" attribute="#uname" operation="eq" value="N1"/> 
    </rule>
    <rule id="s1_n2_loc_rule" score="150">
      <expression id="s1_loc_rule_expr" attribute="#uname" operation="eq" value="N2"/>
    </rule>
  </rsc_location>
</constraints> 
```
　　注： _Fail-Count_:FC  _migration-threshold_:MT

1. 初始，启动N{1,2}上的Pacemaker，资源未运行算0分，只算节点分数：

	> N1 = 200+0 = 200  &  FC=0   
	> N2 = 150+0 = 150  &  FC=0   
	> 决策：N1>N2 & N1-FC<N1-MT，S1在N1上运行

2. N1的S1运行，引起节点分数变化：

	> N1 = 200+100 = 300  &  FC=0   
	> N2 = 150+0 = 150  &  FC=0   
	> 决策：N1>N2 & N1-FC<N1-MT，S1仍在N1上运行，不发生迁移

3. N1的S1运行一段时间无法被检测到，引起节点FC变化：

	> N1 = 200+100 = 300  &  FC=1   
	> N2 = 150+0 = 150  &  FC=0   
	> 决策：N1>N2 & N1-FC<N1-MT，S1仍在N1上运行，不发生迁移

4. N1的S1运行一段时间再次无法被检测到，引起节点分数变化：

	> N1 = 200+100 = 300  &  FC=2   
	> N2 = 150+0 = 150  &  FC=0   
	> 决策：N1<N2 & N1-FC=N1-MT，S1从N1迁移到N2

5. N2的S1运行，引起节点分数变化：

	> N1 = 200+ 0 = 200  &  FC=2   
	> N2 = 150+100 = 250  &  FC=0   
	> 决策：N2>N1 & N2-FC<N2-MT，S1仍在N2上运行，不发生迁移

6. N2的S1运行一段时间无法被检测到，引起节点FC变化：

	> N1 = 200+0 = 200  &  FC=2   
	> N2 = 150+100 = 250  &  FC=1   
	> 决策：N2>N1 & N2-FC<N2-MT，S1仍在N2上运行，不发生迁移

7. N2的S1运行一段时间再次无法被检测到，引起节点分数变化：

	> N1 = 200+0 = 200   &  FC=2   
	> N2 = 150+100 = 250  &  FC=2   
	> 决策：N1-FC=N1-MT & N2-FC=N2-MT，S1停止运行，集群无法提供该资源服务

　　如果重启节点的Pacemaker或Heartbeat服务，节点分数和FC会被重置；也可以通过以下相关命令设置FC值来恢复该资源服务。

```sh
# crm resource failcount 资源ID show 节点Uname
# crm resource failcount 资源ID set 节点Uname N

# crm_failcount -G -U 节点Uname -r 资源ID
# crm_failcount -D -U 节点Uname -r 资源ID //重置为0
```
　　针对migration-threshold参数，还有一个**参数failure-timeout**与它有关，如果一个资源在某个节点上FC都达到MT，如果设置了failure-timeout=30s，则意味着从FC=MT开始过30s后，该节点又变得允许服务在该节点上运行，依赖于该节点的资源黏性和节点分数。这样，如果达到上面第6步的状态时，过30s后N{1,2}都变得允许运行服务。    
　　这里还需要注意**集群选项cluster-recheck-interval参数**，它的含义是每隔多长时间HA检查一次节点分数，进行资源重新分配。我们知道HA检查节点分数是事件驱动的，就像上面的例子一样，但是有些特殊情况下，比如更新了配置但是没有发生事件、比如上面第6步，就需要HA主动去发起检查工作。再来看上面第7步，如果我们设置failure-timeout=60s，但是cluster-recheck-interval=15min，则有可能最长要等15min后，HA检查节点状态时才能重新启动服务。   

　　最后，我们再来说两种**特殊情况**：

1. 节点上的资源在**启动的时候出错**，认为该资源在该节点上不可用，FC会直接置为INFINITY，表示该节点上无法再运行该资源了，马上迁移到其他节点。
2. 节点上的资源在**停止的时候出错**，可能在迁移节点后造成一个以上节点提供相同资源服务，形成脑裂，如果[STONITH](http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/ch13.html#_what_is_stonith)是启用状态则触发Fencing机制（该机制防止脑裂破坏数据）停止造成集群分裂的节点，如果未启用，则不执行迁移。

