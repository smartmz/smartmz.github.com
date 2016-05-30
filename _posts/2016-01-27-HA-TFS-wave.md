---
layout: post
title: "TFS-HA集群服务大幅波动问题"
date: 2016-01-27 16:11
keywords: ha,tfs
description: 从TFS集群的运维看HA的运行机制
categories: [Storage]
tags: [ha]
group: archive
icon: globe
---

###描述
　　由一个历史监控脚本引起。该脚本每隔5分钟检查一次NameServer进程，如果进程不存在，就拉起该进程。具体的，脚本逻辑如下：

<!-- more -->

<center>![图1 NS检测脚本流程](http://ww2.sinaimg.cn/bmiddle/a8484315jw1f0btx2z2owj208o05k0sw.jpg)</center><br/><center><font size=2>图1 NS检测脚本流程</font></center>

　　问题出在 **重启Heartbeat服务** 这一步。通过跟踪Heartbeat服务重启的过程可以找到原因，不过在此之前，先梳理一下当时两个NS之间的环境上下文。
###场景和问题　　
　　最开始线上TFS集群稳定的状态如下：
　　
<center>![图2 最初稳定状态](http://ww3.sinaimg.cn/bmiddle/a8484315jw1f0cz71d9amj20at06uaaf.jpg)</center><br/><center><font size=2>图2 最初稳定状态（NS-001为主NS，VIP落在该节点上）</font></center>

　　因为集群扩容4台机器共4*11个DS节点，过程中发现有一台机器一直无法被标记成迁移数据目标机，猜想可能是NS对该机器上DS的索引信息和负载信息有误，打算迁移一下NS节点重新获取DS信息(未迁移的原因实际上是TFS自动平衡策略导致的，详情可以参考[这篇文章](http://smartmz.github.io/2016/02/01/tfs-balance)。于是停止 NS进程-001 ，HA自动将服务迁移到 NS-002 节点；然后短时间内再停止 NS进程-002，HA自动将服务迁回到 NS-001 节点。这时集群的状态为：
　　
<center>![图3 手动迁移服务后的集群状态](http://ww3.sinaimg.cn/bmiddle/a8484315jw1f0cz71c3u5j20b406y3yu.jpg)</center><br/><center><font size=2>图3 手动迁移服务后的集群状态（NS-001为主NS，VIP落在该节点上）</font></center>

　　之后就出现了**问题**，NS进程-001 每5分钟自动重启一次，而服务不迁移，导致TFS数据迁移一直处于建立任务状态，大部分任务始终无法正常下发到DS节点上，整个TFS集群处于长时间数据迁移状态。
###发生了什么
　　脚本每5分钟运行一次，检测NS进程是否存在。NS-002 上的NS进程被kill触发脚本执行，重启Heartbeat服务，然后拉起NS进程。通过HA的debug日志（默认在/var/log/ha-debug）可以看到该过程具体干了哪些事情，HA的日志里包含了Heartbeat、资源管理组件等全量日志。
####Heartbeat启停行为
　　Heartbeat的版本为：

```sh
# /usr/lib64/heartbeat/heartbeat -V
3.0.2 (node: 7153d58dcb99ff4251449c5404754e26ee1af48e)
```
#####NS-002
　　NS-002比较简单，HA日志片段如下：

```sh
# HA进程退出
## 给进程组发送SIGTERM进程
Jan 26 14:50:31 NS-002 heartbeat: [17932]: debug: Process 17932 processing SIGTERM
## 停止CRM资源管理决策服务
Jan 26 14:50:31 NS-002 heartbeat: [17932]: debug: Shutting down client /usr/lib64/heartbeat/crmd
Jan 26 14:50:34 NS-002 crmd: [18037]: info: do_exit: [crmd] stopped (0)
## 停止LRM资源管理执行服务
Jan 26 14:50:34 NS-002 lrmd: [18034]: info: lrmd is shutting down
Jan 26 14:50:34 NS-002 lrmd: [18034]: debug: [lrmd] stopped
## 卸载CIB资源信息库
Jan 26 14:50:34 NS-002 heartbeat: [17932]: debug: Shutting down client /usr/lib64/heartbeat/cib
## 停止CCM集群成员管理模块
Jan 26 14:50:36 NS-002 heartbeat: [17932]: debug: Shutting down client /usr/lib64/heartbeat/ccm
Jan 26 14:50:36 NS-002 heartbeat: [17932]: debug: Final client "/usr/lib64/heartbeat/ccm" died.
Jan 26 14:50:37 NS-002 heartbeat: [17932]: info: NS-002 Heartbeat shutdown complete.
Jan 26 14:50:37 NS-002 heartbeat: [17932]: debug: Exiting from pid 17932 [rc=0]
# HA进程初始化启动
## 初始化和启动CRM、LRM、CIB、CCM等进程
Jan 26 14:50:57 NS-002 heartbeat: [18302]: debug: add_option(XXXX)
## 读取配置
Jan 26 14:50:57 NS-002 heartbeat: [18302]: info: Configuration validated. Starting heartbeat 3.0.2
Jan 26 14:50:57 NS-002 heartbeat: [18302]: debug: HA configuration OK.  Heartbeat starting.
## 联络HA中其他节点
Jan 26 14:50:58 NS-002 heartbeat: [18303]: debug: sending reqnodes msg to node tc102
Jan 26 14:50:59 NS-002 heartbeat: [18303]: debug: Get a repnodes msg from NS-001
Jan 26 14:50:59 NS-002 heartbeat: [18303]: debug: nodelist received:NS-002 NS-001 
Jan 26 14:50:59 NS-002 heartbeat: [18303]: info: Local status now set to: 'active'
## 应用资源配置
Jan 26 14:50:59 NS-002 cib: [18404]: debug: log_data_element: readCibXmlFile: [on-disk] <XXX>
Jan 26 14:50:59 NS-002 cib: [18404]: info: startCib: CIB Initialization completed successfully
## 获取HA中的节点信息
Jan 26 14:51:04 NS-002 ccm: [18403]: debug: dump current membership 0x2b1e5faec010
Jan 26 14:51:04 NS-002 ccm: [18403]: debug: 	nodename=NS-002 bornon=8
Jan 26 14:51:04 NS-002 ccm: [18403]: debug: 	nodename=NS-001 bornon=1
```
　　到这里Heartbeat本身已经启动完成。由于在TFS的CIB资源配置中没有配置resource-discovery参数，取默认值always，所以Heartbeat在启动完成后，CRM会自发检测一次两个NS节点上的资源（NS进程）运行状态，然后决定本节点上NS进程的启停。
　　根据
> When Pacemaker first starts a resource, it runs one-time monitor operations (referred to as probes) to ensure the resource is running where it’s supposed to be, and not running where it’s not supposed to be. ([官方文档](http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html-single/Pacemaker_Explained/#_cluster_options))

的描述，如果HA中其他节点的NS进程已经启动，该节点就不应该启动。NS-002节点上NS检测脚本在启动heartbeat完成后马上启动了NS进程，这样在之后CRM自发检测资源运行状态时会发现本节点和NS-001上的NS进程都已经启动，于是HA该节点将刚启动的NS进程停止了：

```sh
#检测双方资源在线
Jan 26 14:51:05 NS-002 crmd: [18408]: info: crm_update_peer_proc: NS-001.ais is now online
Jan 26 14:51:05 NS-002 crmd: [18408]: info: crm_update_peer_proc: NS-002.ais is now online
#根据配置信息决策该节点的资源是否应该运行（这个信息来自NS-001节点）
Jan 26 14:51:06 NS-002 crmd: [18408]: debug: do_cl_join_query: Querying for a DC
Jan 26 14:51:07 NS-002 crmd: [18408]: info: update_dc: Set DC to NS-001 (3.0.1)
#停止该节点运行的NS进程
Jan 26 14:51:12 NS-002 lrmd: [18405]: debug: on_msg_perform_op: add an operation operation stop[4] on ocf::NameServer::tfs-name-server for client 18408, its parameters: crm_feature_set=[3.0.1]  to the operation list.
Jan 26 14:51:12 NS-002 lrmd: [18405]: info: rsc:tfs-name-server:4: stop
```
　　到这里，这个TFS集群的状态又回到了图3所示的状态，5分钟后NS检测脚本再次被触发，NS-002节点循环上面整个过程。
#####NS-001
　　如果只是NS-002节点循环上面的过程，还不至于影响整个TFS集群的服务，因为此时服务是落在NS-001节点上的。而事实上，这个过程对NS-001节点是有影响的，同样分析HA的日志：

```sh
#探测NS-002节点失败，此时NS-001节点正在重启heartbeat
Jan 26 14:51:03 NS-001 heartbeat: [15721]: WARN: 1 lost packet(s) for [NS-002] [13:15]
Jan 26 14:51:03 NS-001 heartbeat: [15721]: info: No pkts missing from NS-002!
Jan 26 14:51:04 NS-001 heartbeat: [15721]: WARN: 1 lost packet(s) for [NS-002] [18:20]
Jan 26 14:51:04 NS-001 heartbeat: [15721]: info: No pkts missing from NS-002!
#同步NS-002节点资源，此时NS-002节点已经启动NS进程
Jan 26 14:51:05 NS-001 cib: [15729]: debug: sync_our_cib: Syncing CIB to NS-002
#像NS-002节点发送Join集群请求
Jan 26 14:51:06 NS-001 crmd: [15733]: debug: join_make_offer: join-7: Sending offer to NS-002
Jan 26 14:51:07 NS-001 crmd: [15733]: debug: do_dc_join_filter_offer: Processing req from NS-002
#检测到两个节点都运行了相同资源
Jan 26 14:51:11 NS-001 pengine: [15852]: notice: unpack_rsc_op: Operation tfs-name-server_monitor_0 found resource tfs-name-server active on NS-002
Jan 26 14:51:11 NS-001 pengine: [15852]: ERROR: native_add_running: Resource ocf::NameServer:tfs-name-server appears to be active on 2 nodes.
Jan 26 14:51:11 NS-001 pengine: [15852]: WARN: See http://clusterlabs.org/wiki/FAQ#Resource_is_Too_Active for more information.
Jan 26 14:51:11 NS-001 pengine: [15852]: info: determine_online_status: Node NS-001 is online
Jan 26 14:51:11 NS-001 pengine: [15852]: info: determine_online_status: Node NS-002 is online
#停止该节点运行的NS进程
Jan 26 14:51:11 NS-001 lrmd: [15730]: debug: on_msg_perform_op: add an operation operation stop[14] on ocf::NameServer::tfs-name-server for client 15733, its parameters: crm_feature_set=[3.0.1]  to the operation list.
Jan 26 14:51:11 NS-001 crmd: [15733]: info: te_rsc_command: Initiating action 9: stop tfs-name-server_stop_0 on NS-001 (local)
Jan 26 14:51:11 NS-001 lrmd: [15730]: info: rsc:tfs-name-server:14: stop
#停止NS-002节点运行的NS进程
Jan 26 14:51:11 NS-001 crmd: [15733]: info: te_rsc_command: Initiating action 10: stop tfs-name-server_stop_0 on NS-002
#启动该节点的NS进程
Jan 26 14:51:13 NS-001 crmd: [15733]: info: te_rsc_command: Initiating action 11: start tfs-name-server_start_0 on NS-001 (local)
```
　　NS-001节点的NS进程被重启了。为什么会这样…… 根据日志提示的文档跟踪到[官方文档](http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-resource-options.html)：

> Field | Default | Description
> --- | --- | ---
> multiple-active|stop_start|What should the cluster do if it ever finds the resource active on more than one node? Allowed values: **block**: mark the resource as unmanaged;**stop_only**: stop all active instances and leave them that way;**stop_start**: stop all active instances and start the resource in one location only

　　然后查看当前TFS集群NS两个节点的multiple-active参数配置：

```sh
# crm_resource -m -r tfs-name-server -g multiple-active
Error performing operation: The object/attribute does not exist
```
　　果然未设置，取了默认值stop_start，HA发现两个节点都运行了NS进程，把主节点上的NS进程重启了。

#####综述发生了什么

<center>![图4 整个触发过程](http://ww1.sinaimg.cn/bmiddle/a8484315jw1f0e47eg6pvj20bm0awjsb.jpg)</center><br/><center><font size=2>图4 整个触发过程（NS-001为主NS，VIP落在该节点上）</font></center>

###解决方案

1. NS检测脚本中检测到NS进程不存在后，直接拉起即可，不需要重启Heartbeat，这样就不会触发一系列LRM检测资源运行状态的机制，不会造成整个过程循环。
2. 设置multiple-active参数为block，NS进程-001 就不会重启，不会影响TFS集群对外服务，但是整个过程会循环。


