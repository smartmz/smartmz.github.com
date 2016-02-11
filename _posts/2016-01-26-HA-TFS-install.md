---
layout: post
title: "TFS高可用集群的搭建"
date: 2016-01-21 11:44
keywords: tfs,ha
description: 利用HA搭建TFS高可用集群
categories: [Storage]
tags: [tfs]
group: archive
icon: globe
---
　　淘宝的TFS可以选择使用HA搭建，增强集群的高可用性。在这里HA主要体现在NameServer节点的主备搭建。HA还支持其他模式的高可用架构，本文通过TFS高可用集群的搭建阐述主备模式。
　　TFS的编译、安装和配置详情请参考[官方网站](http://code.taobao.org/p/tfs/wiki/index)。官方使用的Heartbeat是v3版本，在机器级别可以对节点进行监控，如果其中一个NS机器宕机（或网络中断）后可以做出响应；如果需要进一步对NS节点的资源（Nameserver服务）进行监控，要使用CRM模式实现资源管理。  

<!-- more -->

　　Heartbeat v3已经将heartbeat、CRM拆分，heartbeat只负责心跳信息的通信，CRM部分主要有Pacemaker等实现资源管理，如果选择使用Pacemaker的话，还可以用CMAN来代替heartbeat. 因此，搭建TFS-HA集群时NS的主备部署有三种方式：

* **NameServer×2 + heartbeat + DataServer×N，节点（机器）级别的HA**
* **NameServer×2 + heartbeat + Pacemaker + DataServer×N，资源级别的HA，TFS官方**
* **NameServer×2 + CMAN + Pacemaker + DataServer×N，资源级别的HA**

　　HA需要两台独立的主机互为主备，并且要求这两台机器采用直接互联的独立连接方式，有如下两种可选：

1. 串口直连。
2. 网络直连线，要求HA主机有两个以上的网络接口，其中一个连接交换机，另外一个用于直连。

　　本文使用方案2。在实际生产环境中，推荐是两块网卡做BOND后进行直连，减小出错的几率。

　　本次HA相关配置如下：
#####环境

角色|IP/Hostname/Device|作用
---|---|---|
NameServer|123.125.106.28 / tc28 / eth0外部 / eth3心跳|主NS
NameServer|123.125.106.24 / tc02402 / eth0外部 / eth3心跳|备NS
虚拟IP（VIP）|123.125.106.30 / eth0:0|集群接收和响应请求
#####系统hosts
```sh
# cat /etc/hosts
123.125.106.28 tc28
123.125.106.29 tc02402
```
#####其他服务关闭
```sh
# service iptables stop
# chkconfig iptables stop
# service NetworkManager stop
# chkconfig NetworkManager off
```
###NameServer×2 + heartbeat + DataServer×N

* 本次使用的软件版本为：
	* CentOS release 5.10 (Final) 2.6.18-371.11.1.el5 x86_64
	* tfs-stable-2.0.1
	* heartbeat-3.0.3-2.3.el5
	* heartbeat-libs-3.0.3-2.3.el5
	* cluster-glue-1.0.6-1.6.el5
	* cluster-glue-libs-1.0.6-1.6.el5
	* resource-agents-1.0.4-1.1.el5
* heartbeat  
　　可以直接通过YUM源安装，也可以下载RPM或编译安装。安装完成后需要配置ha.cf，路径默认在`/etc/ha/ha.cf`，主备节点除路径外配置相同。
　　ha.cf的配置实例：

	```sh
　　# cat /etc/ha.d/ha.cf      
　　debugfile /var/log/ha-debug
　　debug 1
　　keepalive 2
　　warntime 5
　　deadtime 10
　　initdead 30
　　auto_failback off
　　autojoin none
　　# 心跳线连接的设备，用于广播心跳信息
　　bcast eth3
　　udpport 694
　　node tc28
　　node tc02402
　　compression bz2
　　logfile /data1/tfs/nameServer/logs/ha-log
　　logfacility     local0
　　# 不采用2.x style的CRM(Pacemaker)，采用1.x style
　　crm off
	```
　　配置好后，运行以下命令启动Heartbeat，就可以ping通虚拟IP 123.125.106.30了。

	```sh
　　# service heartbeat start
	```
	
* 两个NameServer节点的配置
参考官方文档：http://code.taobao.org/p/tfs/wiki/deploy/ns.conf

* DataServer节点的配置
参考官方文档：http://code.taobao.org/p/tfs/wiki/deploy/ds.conf

　　启动NS和DS之后，TFS集群就可以使用了。通过Heartbeart，当其中主NS宕机以后，虚拟IP会自动落在备用的NS节点上，继续提供服务。在两个节点上的NS进程推荐**同时启动**，主NS提供服务的同时DS的变化信息会同步给备NS进程，这样在服务迁移时可以达到瞬移，并且保证NS上的DS信息不回丢失。
　　NameServer×2 + heartbeat + DataServer×N的模式在节点级别对NS主备节点进行监控。但是如果其中一台机器没有宕机或下线，只是NS进程停止运行了，这种模式就无法检测到了。
###NameServer×2 + heartbeat + Pacemaker + DataServer×N
	


