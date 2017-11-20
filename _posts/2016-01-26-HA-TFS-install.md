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

<!-- more -->

　　淘宝的TFS可以选择使用HA搭建，增强集群的高可用性。在这里HA主要体现在NameServer节点的主备搭建。HA还支持其他模式的高可用架构，本文通过TFS高可用集群的搭建阐述主备模式。
　　TFS的编译、安装和配置详情请参考[官方网站](http://code.taobao.org/p/tfs/wiki/index)。官方使用的Heartbeat是v3版本，在机器级别可以对节点进行监控，如果其中一个NS机器宕机（或网络中断）后可以做出响应；如果需要进一步对NS节点的资源（Nameserver服务）进行监控，要使用CRM模式实现资源管理。  

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
* 本次使用的主要软件版本为：
	* CentOS release 5.10 (Final) 2.6.18-371.11.1.el5 x86_64
	* tfs-stable-2.0.1
	* heartbeat-3.0.3-2.3.el5
	* heartbeat-libs-3.0.3-2.3.el5
	* cluster-glue-1.0.6-1.6.el5
	* cluster-glue-libs-1.0.6-1.6.el5
	* resource-agents-1.0.4-1.1.el5

　　在[这里(PART1)](http://download.csdn.net/detail/smartmz/8460685)和[这里(PART2)](http://download.csdn.net/detail/smartmz/8460703)下载全部RPM包。在TFS安装目录下包含了配置HA的配置文件和脚本，具体目录在$TFS_HOME/scripts/ha/。
####配置ha.cf
```sh
# cat /etc/ha.d/ha.cf
debugfile /data1/tfs/nameServer/logs/ha-debug
debug 1
keepalive 2
warntime 5
deadtime 10
initdead 30
auto_failback off
autojoin none
bcast eth3			# 心跳线连接的设备，用于广播心跳信息
udpport 694
node tc28
node tc02402
compression bz2
logfile /data1/tfs/nameServer/logs/ha-log
logfacility     local0
crm respawn			# 采用2.x style的CRM(Pacemaker)
```
> 有关ha.cf配置文件各项参数的含义参见附录1。

####拷贝ha.cf到目标目录

```sh
# cp -f ha.cf /etc/ha.d/
```
####生成authkeys
```sh
# (echo -ne "auth 1\n1 sha1 "dd if=/dev/urandom bs=512 count=1 | openssl md5) > authkeys.tmp
```
####配置authkeys
```sh
# cp -f authkeys.tmp /etc/ha.d/authkeys
# chmod 600 /etc/ha.d/authkeys
# chown root.root /etc/ha.d/authkeys
```
> $TFS_HOME/scripts/ha/deploy脚本集成了以上工作，可以直接运行完成。

####配置资源脚本（服务脚本）
```sh
# cp -f $TFS_HOME/scripts/ha/NameServer /usr/lib/ocf/resource.d/heartbeat/
# chmod u+x /usr/lib/ocf/resource.d/heartbeat/NameServer
```
> OCF资源的编写 参考[《[HA]Pacemaker的资源执行(PA)》](http://smartmz.github.io/2016/01/21/ha-pacemaker-ocf)

####启动heartbeat并配置资源
```sh
# service heartbeat start
# crm_attribute --type crm_config --attr-name symmetric-cluster --attr-value true
# crm_attribute --type crm_config --attr-name stonith-enabled --attr-value false
# crm_attribute --type rsc_defaults --name resource-stickiness --update 100
```
　　主要的设置为：

1. 配置所有的节点为对等关系，即所有的节点都能接管服务；
2. 禁用stonish

　　**注意**：ha.cf和authkeys在主备节点上需要完全相同，建议手工拷贝的另一个节点上。尤其注意authkeys，是随机生成一段认证码，必须拷贝到另一个节点上。
####配置资源CIB
　　CIB可以在heartbeat启动之前配置好（配置文件为：/var/lib/heartbeat/crm/cib.xml），也可以在启动后动态加入。本次采用动态加入的办法。$TFS_HOME/scripts/ha/ns.xml是要动态加入的资源。

配置ns.xml（主要注意加了注释的地方）：

```xml
<resources>
  <group id="ns-group">
    <primitive class="ocf" id="ip-alias" provider="heartbeat" type="IPaddr2">
      <instance_attributes id="ip-alias-instance_attributes">
        <!--设置为虚拟ip -->
        <nvpair id="ip-alias-instance_attributes-ip" name="ip" value="123.125.106.30"/>
        <!-- 设置绑定的外网的设备(不是心跳线的设备，虚拟IP会落到这个设备上)-->
        <nvpair id="ip-alias-instance_attributes-nic" name="nic" value="eth0:0"/>
      </instance_attributes>
      <operations>
        <!-- 监控的时间间隔 -->
        <op id="ip-alias-monitor-2s" interval="2s" name="monitor"/>
      </operations>
      <meta_attributes id="ip-alias-meta_attributes">
        <nvpair id="ip-alias-meta_attributes-target-role" name="target-role" value="Started"/>
      </meta_attributes>
    </primitive>
    <!-- 配置监控的脚本 -->
    <primitive class="ocf" id="tfs-name-server" provider="heartbeat" type="NameServer">
      <instance_attribute：s id="tfs-name-server-instance_attributes">
        <!-- 配置为$TFS_HOME -->
        <nvpair id="tfs-name-server-instance_attributes-basedir" name="basedir" value="/data1/tfs/nameServer"/>
        <!-- NameServer运行的主机 -->
        <nvpair id="tfs-name-server-instance_attributes-nsip" name="nsip" value="localhost"/>
        <!-- NameServer运行的端口，和NS配置文件中配置的端口一致 -->
        <nvpair id="tfs-name-server-instance_attributes-nsport" name="nsport" value="60000"/>
        <!-- NameServer运行的用户 -->
        <nvpair id="tfs-name-server-instance_attributes-user" name="user" value="root"/>
      </instance_attributes>
      <operations>
        <!-- 定时检测NameSererver是否存活的间隔时间 -->
        <op id="tfs-name-nameserver-monitor-2s" interval="2s" name="monitor"/>
        <!-- NameServer 启动超时间-->
        <op id="tfs-name-nameserver-start" interval="0s" name="start" timeout="30s"/>
        <!-- NameServer 关闭超时间-->
        <op id="tfs-name-nameserver-stop" interval="0s" name="stop" timeout="30s"/>
      </operations>
      <meta_attributes id="tfs-name-server-meta_attributes">
        <nvpair id="tfs-name-server-meta_attributes-target-role" name="target-role" value="Started"/>
        <nvpair id="tfs-name-server-meta_attributes-resource-stickiness" name="resource-stickiness" value="INFINITY"/>
        <nvpair id="tfs-name-server-meta_attributes-resource-failure-stickiness" name="resource-failure-stickiness" value="-INFINITY"/>
      </meta_attributes>
    </primitive>
  </group>
</resources>
```
####动态添加CIB资源

　　**注意**：添加资源后heartbeat将主动拉起主节点上的Nameserver服务，因此在添加资源之前应该完成Nameserver的配置。

```sh
# cibadmin --replace --obj_type=resources --xml-file $TFS_HOME/scripts/ha/ns.xml
```
　　通过以下命令查看HA主备节点的状态

```sh
# crm status
============
Last updated: Fri Feb 27 17:34:48 2015
Stack: Heartbeat
Current DC: tc28 (4ac4335a-1d77-47fc-8354-cd9a06887d6e) - partition with quorum
Version: 1.0.12-unknown
2 Nodes configured, unknown expected votes
1 Resources configured.
============

Online: [ tc02402 tc28 ]

 Resource Group: ns-group
     ip-alias   (ocf::heartbeat:IPaddr2):       Started tc28
     tfs-name-server    (ocf::heartbeat:NameServer):    Started tc28
```

####主备切换
　　该HA模式下，如果主节点上的NameServer服务宕掉，HA会自动拉起另一个节点上的NameServer服务，并且将虚拟IP漂移到另一个节点上。这样就实现了资源级别的HA。

　　这里需要**注意**的是，服务已经迁移到备节点上后，如果备节点也宕掉，HA不会主动拉起主节点上的NameServer服务。要想HA能够自动重新拉起主节点上的服务，需要运行以下命令将主节点的错误标记清0：

```sh
# crm resource failcount tfs-name-server set tc28 0
```
　　可以在crontab里添加一个脚本，定时清0节点上的错误标记。

###NameServer×2 + CMAN + Pacemaker + DataServer×N

　　该模式实现的目标和上一模式是一样的，记录的原因是之前遇到过在高版本的CentOS环境中上一模式无法成功运行heartbeat，因此可以将这一方案作为备选方案。

* 本次使用的软件均为YUM安装，版本为：
	* CentOS CentOS release 6.6 (Final) 2.6.32-431.11.2.el6.toa.2.x86_64
	* tfs-stable-2.0.1
	* pacemaker-1.1.12-4.el6.x86_64
	* cman-3.0.12.1-68.el6.x86_64
	* pcs-0.9.123-9.0.1.el6.centos.x86_64（非必须，方便集群配置管理）
	* ccs-0.16.2-75.el6.x86_64（非必须，方便集群配置管理）
	* resource-agents-3.9.5-12.el6_6.2.x86_64

####CMAN配置和启动
#####配置集群信息
　　主要是对/etc/cluster/cluster.conf进行配置，ccs提供了相关工具对该文件进行配置。

　　先加入两个NS节点：

```sh
# ccs -f /etc/cluster/cluster.conf --addnode tc28
# ccs -f /etc/cluster/cluster.conf --addnode tc02402
```
#####配置属性：

```sh
# ccs -f /etc/cluster/cluster.conf --addfencedev pcmk agent=fence_pcmk
# ccs -f /etc/cluster/cluster.conf --addmethod pcmk-redirect tc28
# ccs -f /etc/cluster/cluster.conf --addmethod pcmk-redirect tc02402
# ccs -f /etc/cluster/cluster.conf --addfenceinst pcmk tc28 pcmk-redirect port=tc28
# ccs -f /etc/cluster/cluster.conf --addfenceinst pcmk tc02402 pcmk-redirect port=tc02402
# echo "CMAN_QUORUM_TIMEOUT=0" >> /etc/sysconfig/cman
```
#####启动CMAN：

```sh
# service cman start
```
> 如果这里不启动cman，也可以在下面直接启动Pacemaker。配置属性也在后面进行。

#####设置属性：

```sh
# crm_attribute --type crm_config --attr-name symmetric-cluster --attr-value true  
# crm_attribute --type crm_config --attr-name stonith-enabled --attr-value false 
# crm_attribute --type rsc_defaults --name resource-stickiness --update 100
```
####Pacemaker配置
#####建立组和用户（YUM安装无需建立）

```sh
# groupadd haclient
# useradd -g haclient hacluster -M -s /sbin/nologin
```
#####启动Pacemaker

```sh
# chkconfig pacemaker on
# service pacemaker start
```
#####配置属性

```sh
# pcs property set stonith-enabled=false
# pcs property set no-quorum-policy=ignore
# pcs resource defaults migration-threshold=1
```
#####配置cib.xml和添加资源
　　添加OCF_ROOT环境变量：

```sh
# echo "export OCF_ROOT=/usr/lib/ocf" >> /etc/profile
# source /etc/profile
```
　　采用动态添加资源的方法加入NameServer资源，cib.xml配置参考上一模式中的“配置资源CIB”小节。

　　**注意**：在添加CIB资源时，cibadmin命令与上一模式的版本不同，参数有变，运行以下命令即可：

```sh
# cibadmin --replace –scope resources --xml-file $TFS_HOME/scripts/ha/ns.xml
```
　　同样**注意**，在添加CIB资源之前需要先完成NS的配置。
#####查看集群状态：

```sh
# crm_mon -1
Last updated: Fri Feb 27 19:02:40 2015
Last change: Wed Feb  4 13:43:26 2015
Stack: cman
Current DC: tc28 (4ac4335a-1d77-47fc-8354-cd9a06887d6e) - partition with quorum
Version: 1.1.11-97629de
2 Nodes configured
2 Resources configured

Online: [ tc02402 tc28 ]

 Resource Group: ns-group
     ip-alias   (ocf::heartbeat:IPaddr2):       Started tc28
     tfs-name-server    (ocf::heartbeat:NameServer):    Started tc28 
```
#####主备切换
　　该HA模式下，如果主节点上的NameServer服务宕掉，HA会自动拉起另一个节点上的NameServer服务，并且将虚拟IP漂移到另一个节点上。这样就实现了资源级别的HA。

　　同上一模式需要注意的一样，服务已经迁移到备节点上后，如果备节点也宕掉，HA不会主动拉起主节点上的NameServer服务。要想HA能够自动重新拉起主节点上的服务，需要运行以下命令将主节点的错误标记清0：

```sh
# crm resource failcount tfs-name-server show tc28
# crm resource failcount tfs-name-server set tc28 0
```
　　同样，可以在crontab里添加一个脚本，定时清0节点上的错误标记。
　　
<hr />

###附录1：ha.cf各项参数含义

选项|默认/可选值|含义
---|---|---debugfile|/var/log/ha-debug|debug文件路径，这个选项要开启下面的debug才生效debug|1/0|1=开启,0=关闭 debug日志
logfile|/var/log/ha-log|日志文件路径keepalive|2|在heartbeat之间连接保持多久，缺省的时间单位是秒warntime|5|how long before issuing "late heartbeat" warning?deadtime|30|主机挂了多久算死掉？initdead| |在某些机器/操作系统等中，网络在机器重启后需要花一定的时间启动并正常工作。因此我们必须分开他们初次起来的dead time，这个值应该最少设置为两倍的正常deadauto_failback|on/off|是否自动抢占资源，就是slave起来后，是否自动强制切为master,一般设置为offautojoin|none|资源自动加入，就是无需配置ha.cf可以自动加入集群，一般设置为none(不可以)bcast|eth2|心跳线接口mcast|bond0 225.0.0.10 694 1 0|多播接口，设置为连交换机的接口，后面是多播地址和端口udpport|694|这个端口与上面选项中的端口相同，如果一个交换机内有多个HA，那么每对选定的端口应该不同。
node|{hostname1} {hostname2}|节点名字，HA的两个节点主机名，可以并列空格间隔，也可以用两个node分别指定compression|bz2|压缩方法crm|respawn|利用crm管理资源（这个一定要写）


