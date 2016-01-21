---
layout: post
title: "[HA]Linux-HA基本原理"
date: 2016-01-14 00:18
keywords: redis,command
description: Redis命令
categories: [Storage]
tags: [ha]
group: archive
icon: globe
---
###集群事务决策

先来看一下HA中出现的各种概念。

| 部件 | 角色 | 含义 |
| --- | --- | --- |
| HA | 公司 | High Available，高可用。Linux-HA通过一组应用保证集群的高可用性。三方面内容：1 集群中各个节点之间可以相互感知；2 节点故障自动恢复；3 节点上的应用服务故障自动恢复 |
| Heartbeat | 环境 | Cluster Messaging Layer，心跳信息传递层。集群各节点相互感知和通信的基础环境。节点通过此能感知其他节点（或节点上的应用服务）是否存活。通过hearbeat可以在节点（机器）级别对集群中各个节点进行监控 |
| HA_aware | 制度 | 集群事务决策，也叫策略引擎(Policy Engine)，根据底层的心跳信息，调用API进行集群事务决策，主要包括实现剔除无效节点，上线重新设置的节点，设置主节点、辅助节点，并且能够将底层信息传递给更上层。节点之间通过XML传递数据 |
| DC | 协调员 | Designated Coordinator，根据ha_aware从多个节点中选举出Leader。DC上运行了两个进程，决策引擎(PE)和事务引擎(TE) |
| CRM | 董事长 | Cluster Resources Manager，v2.x增加，比v1.x的haresource具有更强大的资源管理功能，对集群的资源进行管理，任何节点的资源都由CRM管理是否启动。CRM在各个节点上都有部署，针对集群的资源统筹管理。通过CRM可以在资源（服务）级别对集群中各个节点上的服务进行监控 |
| LRM | 经理 | Local resources Manage，本地资源管理，落实CRM的决策，真正管理本地资源的启停和状态监控 |
| RA | 员工 | Resource Agent，资源管理客户端，对某一个资源进行管理的工具，接收LRM的调度管理，对具体资源执行管理操作 |
| CIB | 数据库 | Cluster information base，集群信息库，每个节点上都保存一份集群的CIB |
###其他涉及的组件
名称 | 含义
--- | ---
haresource | 集群资源管理器，v1.x、v2.x都包含，v2.x还提供了功能更强大的CRM
ha-logd | 集群时间日志服务
CCM | Consensus Cluster Membership，集群成员一致性管理模块
PE | Cluster Policy Engine，集群策略引擎
TE | Cluster Transition Engine，策略执行引擎
Stonith-Daemon | 使出现问题的节点从集群环境中脱离或重启
###组件对应软件
* Messaging Layer
	* heartbeat：Linux-HA开源项目发布的高可靠应用环境集群服务的核心软件，负责节点健康信息检测、可靠的节点间通讯和集群管理等工，经历了heartbeat v1（集成功能）；heartbear v2（基于进程拆分功能）；heartbeat v3多个版本（基于项目拆分功能）。
	* heartbeat v1.x/v2.x组件：heartbeat、ha-logd、CCM、LRM、Stonith Daemon、CRM (haresource)、PE、TE
	* heartbeat v3.x组件：heartbeat、Cluster Glue、Resource Agent、Pacemacker (CRM)
	* corosync：OpenAIS发展衍生出来的开放性集群引擎工程。OpenAIS是基于SA Forum标准的集群框架的应用程序接口规范。SA Forum（服务可用性论坛）开发并发布应用接口规范AIS，用来定义应用程序接口的开放性规范的集合。
	* cman：Cluser Manger，RedHat对OpenAIS的另一种实现。
* Allocated Layer 
	* heartbeat v1：包含该功能，通过haresources配置文件进行配置。 
	* heartbeat v2：独立进程，分为客户端crm和服务端crmd，各个节点都运行服务进程，默认端口为5566，使用crm命令进行配置。
	* heartbeat v3：多个项目，heartbear+Pacemaker+Cluster-Glue，其中heartbeat用于底层信息传递；Pacemaker负责CRM资源管理，不仅可以和heartbeat工作，也可以和corosync一起工作；Cluster-Glue相当于中间层，将heartbeat（或corosync）和Pacemaker联系起来，包含LRM和STONITH；LRM调用Resource Agents实现各种资源启动、停止、监控等，Resource Agents指各种的资源的LSB/OCF/STONITH脚本，必须能够接受参数{start|stop|restart|status}；LSB就是RedHat提供的/etc/rc.d/init.d/*，OCF是专门与pacemacker兼容的资源代理脚本，STONITH使用电源交换机连接到健康服务器的电源设备自动重启失效服务器。
	* RGManager：Resource Group (Service) Manager，与cman搭配使用的CRM资源管理。
	* RHCS：红帽集群套件，通过LVS（Linux Virtual Server）提供负载均衡集群，通过GFS文件系统提供存储集群功能。

###原理图
<center>![图1 HA的集群事务决策模型](http://ww3.sinaimg.cn/large/a8484315gw1f00mmfk00lj20i10fjad5.jpg)</center><br/><center>
<font size=2>图1 HA的集群事务决策模型</font></center>

#####流程描述：
* 在Messaging Layer层，各节点上部署支持Heartbeat信息交换的工具，通过广播交换心跳信息，并将所有信息聚合到CRM，由CRM统筹管理。心跳信息包含了服务器服务的运行状态信息等。
* CRM通过Messaging Layer收集到集群节点的信息，从节点中推举出一个DC节点。可以使用任意一个节点的CRM配置更新CIB。
* DC节点运行两个进程TE和PE，其中PE根据CIB进行决策，决定资源的分配和行为策略。如果CIB配置更新，DC负责同步更新到所有节点。
* CRM根据DC的决策控制LRM调用RA来完成具体的资源行为。

###集群分裂和资源隔离
　　在高可用（HA）系统中，当联系节点的心跳断开时，本来为一个整体、动作协调的HA系统就会分裂为独立的个体（每个个体中可以包含一个或多个节点）。由于相互失去了联系，彼此都以为对方出了故障，节点上的HA软件像“裂脑人”一样，“本能”地争抢“共享资源”、争起“应用服务”，就会发生严重后果，或者共享资源被瓜分、多个服务都起不来了，或者多个服务都起来了，同时读写“共享存储”，导致数据损坏或者数据不完整。这种现象俗称“脑裂”。
　　脑裂正常情况下是无法见到的，现代集群都应该有保护机制来避免这种情况发生。例如RHCS引入的fence的概念，例如2个节点集群，当心跳断开导致节点之间互相无法通信的时候，每个节点会尝试fence掉对方（确保对方释放掉文件系统资源）后再继续运行服务访问资源。这样，就可以确保只有一个节点可以访问资源而不会导致数据损坏。
　　另一种防止集群分裂后防止个体之间竞争资源，可以采取办法隔离资源。分裂后的个体中的节点数有多有少，通过选举策略选择出得票数高的个体作为集群成员，其他的个体放弃成为集群成员。实现隔
离的办法有：

* 节点级别的隔离：利用STONITH，断掉放弃成为集群成员的个体的电源。
* 资源级别的隔离：利用交换机，阻断放弃成为集群成员的个体的网络通信。

###HA集群的类型
* N-M型：N个节点M个服务，N>M，N-M个备用节点。
* N-N型：N个节点N个服务。
* Active/Passive型：2个节点，同时只有一个主动提供服务，另一个闲置。
* Active/Active型：2个节点均为互为镜像。
　　前两种类型，集群中均为多个节点，通过仲裁策略推举DC工作。后两种类型集群中均只有两个节点，A/A型使用的场景一般是需要提供负载均衡。

###HA集群的负载均衡
　　比较流行的NGINX、HA-PROXY、LVS等都可以支持Linux平台上的HA集群负责均衡方案，但在实现性质上各不相同。

* HA-PROXY，实际是运行在HA集群之上，基于第三方应用提供负载均衡。
* Nginx，属于web服务器，同HA-PROXY一样，运行在HA集群之上提供负载均衡。
* LVS，基于Linux操作系统，采用IP负载均衡技术和基于内容请求分发技术，支持NAT、TUNNEL、DR三种模式。

> Keepalived使用的vrrp协议方式，虚拟路由冗余协议 (Virtual Router Redundancy Protocol，简称VRRP)，Heartbeat是基于主机或网络的服务的高可用方式；keepalived的目的是模拟路由器的双机，heartbeat的目的是用户service的双机；LVS的高可用建议用keepavlived，业务的高可用用heartbeat.

