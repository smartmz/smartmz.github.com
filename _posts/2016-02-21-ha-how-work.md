---
layout: post
title: "[HA]Linux-HA原理详解"
date: 2016-02-21 08:00
keywords: ha
description: HA基本原理
categories: [Storage]
tags: [ha]
group: archive
icon: globe
---
　　私信附件、长文在底层使用淘宝的TFS分布式存储对资源进行存储。TFS由NameServer和DataServer两类节点组成，NameServer接收资源存储请求、整体负责数据创建、删除、复制、均衡、整理等任务的管理和分配，而实际数据的读写工作由DataServer完成。一套完整的TFS集群理论是只需要 1\*NameServer+N\*DataServer，但在实际使用时为了保证TFS提供稳定的存储服务，NameServer推荐通过部署为高可用集群。我们在线上环境中使用了Linux-HA的方案。本文重点来介绍一下Linux-HA的原理和使用。   
　　高可用性HA —— High Availability，指通过尽量缩短因日常维护操作（计划）和突发的系统崩溃（非计划）所导致的停机时间，以提高系统和应用的可用性。HA系统是目前企业防止核心服务系统因故障停机的最有效手段之一。     
　　Linux-HA项目是高可用HA的一套开源实现，经过多年的应用和重构已经非常成熟。它基于事务驱动，提出一套完整的高可用HA集群事务决策模型，定义了消息传输层、资源分配层、资源层等自下而上三个层次，包括集群间通信、事务触发、资源管理、资源监控等实现。  

<hr/>

	[TOC]

<hr/>

#[HA模型]
　　本节主要介绍高可用模型的概念及其基本设计原理、模型的缺陷——资源分裂问题。

###基本概念　　　　　　
　　为了比较好的理解模型的原理，我们首先介绍HA模型中的各种概念。这里总结了一张表，将HA中的主要组件比拟为一个公司的各种部门和职务，以期从感性上对这些概念有一些认识。在阅读过程中可以在该表中查询，加深理解。

<!-- more -->

| 部件 | 比拟角色 | 含义 |
| --- | --- | --- |
| **HA** | 公司 | High Available，高可用。Linux-HA通过一组应用保证集群的高可用性。三方面内容：1 集群中各个节点之间可以相互感知；2 节点故障自动恢复；3 节点上的应用服务故障自动恢复 |
| Heartbeat | 环境 | Cluster Messaging Layer，心跳信息传递层。集群各节点相互感知和通信的基础环境。节点通过此能感知其他节点（或节点上的应用服务）是否存活。心跳系统通过hearbeat可以在节点（机器）级别对集群中各个节点进行监控 |
| HA_aware | 制度 | 集群事务决策，也叫策略引擎，PE(Policy Engine)，根据底层的心跳信息，调用API进行集群事务决策，主要包括实现剔除无效节点，上线重新设置的节点，设置主节点、辅助节点，并且能够将底层信息传递给更上层 |
| DC | 协调员 | Designated Controller，从多个节点中选举出来，用来完成集群中服务分配的所有决策。 **注意**，DC节点和资源运行节点不一定是同一个节点 |
| CRM | 董事长 | Cluster Resources Manager，v2.x增加，比v1.x的haresource具有更强大的资源管理功能，对集群的资源进行管理，实现资源分配，集群中任何资源都不应该自行启动，任何节点的资源都由CRM管理是否启动。CRM在各个节点上都有部署。通过CRM可以在资源（服务）级别对集群中各个节点上的服务进行监控 |
| LRM | 经理 | Local resources Manage，本地资源管理，获取本地某个资源的状态，并且实现本地资源监控，如当检测到资源不可用时，启动进程等 |
| RA | 员工 | Resource Agent，资源代理，对某一个资源完成管理的工具(其实就是支持启停、监控资源等功能的脚本)，接收LRM的调度管理，对具体资源执行监控管理操作，OCF类的RA是最常使用的，默认安装在`/usr/lib/ocf/resource.d/heartbeat/` |
| CIB | 数据库 | Cluster information base，集群信息库，每个节点上都保存一份集群的CIB |

###其他涉及的概念

名称 | 含义
--- | ---
haresource | 集群资源管理器，v1.x、v2.x都包含，v2.x还提供了功能更强大的CRM
ha-logd | 集群时间日志服务
CCM | Consensus Cluster Membership，集群成员一致性管理模块
PE | Cluster Policy Engine，集群策略引擎，根据CRM对资源的分配做出节点上的服务应该启动还是停止的策略，但不具体实行。只有被选为DC的节点才运行
Stonith-Daemon | 根据心跳信息使出现问题的节点（硬件级别）从集群环境中脱离或重启

###集群事务决策原理
　　直接上图。
<center>![图1 HA的集群事务决策模型](http://ww3.sinaimg.cn/large/a8484315gw1f00mmfk00lj20i10fjad5.jpg)</center><br/><center>
<font size=2>图1 HA的集群事务决策模型</font></center>

####流程描述：
* Messaging Layer层负责集群节点间探测消息传输。各节点上部署支持Heartbeat信息交换的工具，通过广播交换心跳信息，并将所有信息聚合到CRM，由CRM统筹管理。心跳信息包含了服务器服务的运行状态信息等。
* Resource Allocated Layer层负责合理分配集群资源(CIB)、检测资源状态。CRM通过Messaging Layer收集到集群节点的信息从所有节点中选出资源DC节点，DC的PE根据CIB配置分配集群资源，通知给各个节点的CRM，达成一致后，各节点通过LRM下发控制资源的命令。同时LRM还负责对本机资源状态的监控。
* Resource Layer层实现资源的具体控制，比如启停逻辑、监控逻辑等。RA接收LRM的指令对资源进行控制。

###HA集群的主要类型
* N-M型：N个节点M个服务，N>M，N-M个备用节点。
* N-N型：N个节点N个服务。
* **Active/Passive型**：2个节点，同时只有一个主动提供服务，另一个闲置，线上TFS集群使用该类型。
* Active/Active型：2个节点均为互为镜像。

　　这些类型主要通过CRM提供。前两种类型，集群中均为多个节点，通过仲裁策略推举DC工作。后两种类型集群中均只有两个节点，A/A型使用的场景一般是需要提供负载均衡。

###集群分裂和资源隔离
　　在高可用（HA）系统中，当联系节点的心跳断开时，本来为一个整体、动作协调的HA系统就会分裂为独立的个体（每个个体中可以包含一个或多个节点）。由于相互失去了联系，彼此都以为对方出了故障，节点上的HA软件像“裂脑人”一样，“本能”地争抢“共享资源”、争起“应用服务”，就会发生严重后果，或者共享资源被瓜分、多个服务都起不来了，或者多个服务都起来了，同时读写“共享存储”，导致数据损坏或者数据不完整。这种现象俗称“脑裂”。
　　脑裂正常情况下是无法见到的，现代集群都应该有保护机制来避免这种情况发生。例如RHCS引入的fence的概念，例如2个节点集群，当心跳断开导致节点之间互相无法通信的时候，每个节点会尝试fence掉对方（确保对方释放掉文件系统资源）后再继续运行服务访问资源。这样，就可以确保只有一个节点可以访问资源而不会导致数据损坏。
　　另一种防止集群分裂后防止个体之间竞争资源，可以采取办法隔离资源。分裂后的个体中的节点数有多有少，通过选举策略选择出得票数高的个体作为集群成员，其他的个体放弃成为集群成员。实现隔离的办法有：

* 节点级别的隔离：利用STONITH，断掉放弃成为集群成员的个体的电源。
* 资源级别的隔离：利用交换机，阻断放弃成为集群成员的个体的网络通信。

###理论对应实现
　　对Linux-HA主要有两类实现，Heartbeat和OpenAIS.

* heartbeat：Linux-HA开源项目发布的高可靠应用环境集群服务的核心软件，负责节点健康信息检测、可靠的节点间通讯和集群管理等工，经历了heartbeat v1（集成功能）；heartbear v2（基于进程拆分功能）；heartbeat v3多个版本（基于项目拆分功能），各个版本之间对heartbeat做了很大的重构。
	* heartbeat v1.x：单进程实现，功能包括heartbeat、ha-logd、CCM、LRM、Stonith Daemon、CRM (haresource)、PE等，通过haresources配置文件进行配置。
		* heartbeat v2.x：主要功能由独立进程完成，分为客户端crm和服务端crmd，各个节点都运行服务进程，自带crm命令工具集进行资源管理和配置，功能组件基本维持v1.x版本的所有功能。
		* **heartbeat v3.x**：大多数组件逐渐出现了多种实现，为了能够在层次上兼容其他开源项目，heartbeat分裂为多个项目：heartbeat+Pacemaker+Cluster-Glue，功能组件包括heartbeat、Cluster Glue、Resource Agent、Pacemacker等。这是本文关注的重点。
			* heartbeat用于底层信息传递，它是集群间节点联络的基础；
			* Pacemaker负责CRM资源管理，是HA集群中的核心，也是后文的重点。它不仅可以和heartbeat工作，也可以和corosync（基于OpenAIS）一起工作；实际上目前heartbeat是在Pacemaker项目中的；
			* Cluster-Glue相当于中间层，将heartbeat（或corosync）和Pacemaker联系起来，包含LRM和STONITH：
				* LRM负责管理和监控本地的资源，Resource Agents指各种的资源的LSB/OCF/STONITH脚本，必须能够接受参数{start|stop|restart|status}；LSB就是RedHat提供的/etc/rc.d/init.d/*，OCF是专门与pacemacker兼容的资源代理脚本；
				* STONITH，节点心跳系统，通过Heartbeat心跳信息层实现在节点级别对节点进行控制。
	* corosync：OpenAIS发展衍生出来的开放性集群引擎工程。OpenAIS是基于SA Forum标准的集群框架的应用程序接口规范。SA Forum（服务可用性论坛）开发并发布应用接口规范AIS，用来定义应用程序接口的开放性规范的集合。
	* cman：Cluser Manger，RedHat对OpenAIS的另一种实现。
		* RGManager：Resource Group (Service) Manager，与cman搭配使用的CRM资源管理。
	
####  _小结_
　　本节介绍了HA模型理论，提到的概念比较多，建议大家在理顺模型工作原理的基础上把这些概念串联起来，就比较容易理解了。节末列举了模型的实现方案，我们主要关注Heartbeat v3版本。在下面的小节中我们将Heartbeat v3的实现重点放在Pacemaker的运行方面，理解这些对我们实际使用Heartbeat v3搭建HA集群有非常有益的指导作用。

<hr />

##[RA]
　　在上一节中我们已经知道，HA模型的主要工作就是根据集群当前情况对资源进行及时分配调度，达到服务不中断的目的。这里涉及到两方面，一是资源本身，二是根据资源和PE的调度指令对资源实现调度。    
　　对应到具体的实现上，第一部分内容涉及Pacemaker的调度资源，要注意通常说的**HA资源**并不是指我们希望集群最后提供的服务，而是指具备调度该服务的运行逻辑，也就是RA，因此RA一般要支持服务的启停，高级的还要支持服务的监控、管理等。这一部分我们重点关注OCF类的。    
　　第二部分涉及Pacemaker如何进行资源调度，就是CIB配置。集群DC节点上的PE通过CIB配置决策资源在各个节点上的分配，然后由CRM通过节点上的LRM调用RA完成资源的启停。在后面小节中我们重点介绍CIB配置，了解PE分配资源的决策机制。  
　　Heartbeat v3版本中Pacemaker完成了CRM的全部功能，所以我们谈RA就是要谈Pacemaker支持的RA资源。   
　　Pacemaker支持的RA资源有Open Cluster Framework(OCF)、Linux Standard Base(LSB)、Systemd、Upstart、System Services、STONITH、Nagios Plugins等多种类型，但常用的主要是两类，LSB和OCF。

* LSB 指 Linux 标准服务，通常就是 /etc/init.d 目录下那些脚本。Pacemaker 可以用这些脚本来启停服务。查看LSB资源可以使用命令：  
`# crm ra list lsb`
* OCF 实际上是对 LSB 服务的扩展，增加了一些高可用集群管理的功能如故障监控等和更多的元信息，更加通用。查看OCF资源可以使用命令：  
`# crm ra list ocf`

　　如果需要pacemaker更好地对服务进行高可用保障，就得实现一个OCF类的RA资源。     
　　Pacemaker自带的资源管理程序默认都在`/usr/lib/ocf/resource.d`目录中。其中的`heartbeat`目录包含了自带的常用服务，可以作为自己实现的参考。

###自定义一个OCF
　　每个OCF资源本质上是一个可执行的脚本文件，通过**命令行参数和CIB环境变量**接收来自Pacemaker的输入。下面是一个简单的例子，创建了一个名叫 test 的资源。

``` sh
#!/bin/bash
# OCF parameters:
# OCF_RESKEY_service_path : service path
# OCF_RESKEY_probe_url    : probe url
: ${OCF_FUNCTIONS_DIR=${OCF_ROOT}/resource.d/heartbeat}
. ${OCF_FUNCTIONS_DIR}/.ocf-shellfuncs
SERVICE_PATH="${OCF_RESKEY_service_path}"
PROBE_URL="${OCF_RESKEY_probe_url}"
COMMAND=$1
SERVICE_NAME=test
metadata_test() {
    cat <<END
		<?xml version="1.0"?>
		<!DOCTYPE resource-agent SYSTEM "ra-api-1.dtd">
		<resource-agent name="test">
		<version>1.0</version>
		<longdesc lang="en">
		A test resource
		</longdesc>
		<shortdesc lang="en">test</shortdesc>
		<parameters>
		    <parameter name="service_path" required="1" unique="1">
		        <longdesc lang="en">
		        This is a required parameter. The path of service script.
		        </longdesc>
		        <shortdesc>the path of service script</shortdesc>
		        <content type="string" default=""/>
		    </parameter>
		    <parameter name="probe_url" required="1" unique="1">
		        <longdesc lang="en">
		        This is a required parameter. The url to detect the status of the service.
		        </longdesc>
		        <shortdesc>the url to detect the status of the service</shortdesc>
		        <content type="string" default=""/>
		    </parameter>
		</parameters>
		<actions>
		    <action name="start" timeout="60s" />
		    <action name="stop" timeout="30s" />
		    <action name="status" timeout="30s" />
		    <action name="monitor" depth="0" timeout="30s" interval="10s" />
		    <action name="meta-data" timeout="5s" />
		    <action name="validate-all"  timeout="5s" />
		</actions>
		</resource-agent>
	END
    return $OCF_SUCCESS
}
monitor_test() {
    return $OCF_SUCCESS
}
start_test() {
    return $OCF_SUCCESS
}
stop_test() {
    return $OCF_SUCCESS
}
status_test() {
    return $OCF_SUCCESS
}
validate_all_test() {
    ocf_log info "validate_all_test"
    if [[ ! -x "$SERVICE_PATH" ]]; then
        ocf_log err "service_path is not found"
        exit $OCF_ERR_CONFIGURED
    fi
    if [[ ! -x "$PROBE_URL" ]]; then
        ocf_log err "probe_url is required"
        exit $OCF_ERR_CONFIGURED
    fi
    return $OCF_SUCCESS
}
run_func() {
    local cmd="$1"
    ocf_log debug "Enter $SERVICE_NAME $cmd"
    $cmd
    local status=$?
    ocf_log debug  "Leave $SERVICE_NAME $cmd $status"
    exit $status
}
case "$COMMAND" in
    meta-data)
        run_func meta-data
        ;;
    start)
        run_func start
        ;;
    stop)
        run_func stop
        ;;
    status)
        run_func status
        ;;
    monitor)
        run_func monitor
        ;;
    validate-all)
        run_func validate-all
        ;;
    *)
        exit $OCF_ERR_ARGS
        ;;
esac
```

###资源的Meta
　　使用`# crm ra meta test`命令显示该OCF资源的meta信息：

``` text
ocf:heartbeat:test
A test resource
Parameters (* denotes required, [] the default):
service_path* (string): the path of service script
    This is a required parameter. The path of service script.
probe_url* (string): the url to detect the status of the service
    This is a required parameter. The url to detect the status of the service.
Operations' defaults (advisory minimum):
    start         timeout=60s
    stop          timeout=30s
    status        timeout=30s
    monitor_0     interval=10s timeout=30s
```
　　实际上就是脚本中`metadata_test()`定义的内容，定义了脚本需要接收的参数和支持的action（`meta-data`和`validate-all`不显示）。这个OCF资源相比LSB资源来说，多了`monitor_0`这样一个命令。   
　　脚本中定义参数的`parameter/content`支持的类型有 string、integer、boolean等。default属性可以定义参数的默认值。需要注意的是，定义资源未指定参数时，指定的默认值并不会出现在`OCF_RESKEY_<参数名>`环境变量中，而是会放在另外的环境变量`OCF_RESKEY_<参数名>_default`中。

###资源的Action
　　start、stop这两个命令，主要实现服务启动或停止；monitor命令实现状态检测。在实际编写脚本的时候需要**注意**，一定要保证在start、stop命令执行完成返回后，使用monitor命令检测状态。如果在未完成start或stop时就返回，比如直接将命令放后台执行，会导致通过monitor命令检测状态时，由于此时还未启动完成或stop而失败，导致被认为是故障，从而导致资源重启或切换。

###资源的参数
　　命令的参数在比较新的版本中还可以直接通过环境变量传递。比如test资源中的service\_path 和 probe\_url 会分别通过环境变量 `$OCF_RESKEY_service_path` 和 `$OCF_RESKEY_service_path` 传递。
　　更通用的做法是通过CIB配置传递的，这一部分我们在后面介绍CIB配置时详细展开。

###命令返回值
OCF资源常用的返回值有：

```text
OCF_SUCCESS=0
OCF_ERR_GENERIC=1
OCF_ERR_ARGS=2
OCF_ERR_UNIMPLEMENTED=3
OCF_ERR_PERM=4
OCF_ERR_INSTALLED=5
OCF_ERR_CONFIGURED=6
OCF_NOT_RUNNING=7
```
　　所有命令执行成功时应当返回`OCF_SUCCESS`。出错时根据具体情况返回对应错误码。如start启动失败，monitor监控到服务故障时应当返回OCF\_NOT\_RUNNING。在检查参数错误时应当返回 OCF\_ERR\_CONFIGURED。其他错误一般可以返回OCF\_ERR\_GENERIC。**所有命令完成后都需要有返回值**。

###日志
　　输出日志使用 `ocf_log {日志级别} {日志内容}`，该命令将日志输出到ha配置指定的文件中。ha的配置文件`/etc/ha.d/ha.cf`中定义了日志输出路径和日志输出级别：

```text
# cat /etc/ha.d/ha.cf
debugfile /var/log/ha-debug
debug 1
... ...
```

> 有关OCF资源更详细的内容请看[官方文档](http://www.linux-ha.org/doc/dev-guides/ra-dev-guide.html)

###实例
　　这里我们给出TFS-HA OCF资源RA实例，大家可以对照上面的介绍加深理解。

```sh
#!/bin/sh
# Initialization:
. ${OCF_ROOT}/resource.d/heartbeat/.ocf-shellfuncs

getpid() {
        grep -o '[0-9]*' $1
}

monitor_nameserver() {
      if ! test -x $montool
      then
              return $OCF_ERR_PERM
      fi
      ocf_log debug "begin to monitor $binfile service with command $cmd"
      cmd="$montool -i $nsip -p $nsport -n"
      eval $cmd
      if [ $? -ne 0 ]
      then
              return $OCF_ERR_GENERIC
      else
              return $OCF_SUCCESS
      fi
}

switch_nameserver() {
    cmd="$montool -i $nsip -p $nsport -n -s 1"
    eval $cmd
}

nameserver_status() {
      if test -f "$pidfile"
      then
              if pid=`getpid $pidfile` && [ "$pid" ] && kill -0 $pid
              then
                      #status=`check_nameserver_running_status`
                      return $OCF_SUCCESS
              else
                      # pidfile w/o process means the process died
                      return $OCF_ERR_GENERIC
              fi
      else
              return $OCF_NOT_RUNNING
      fi
}

nameserver_start() {
      if ! nameserver_status
      then
              if [ -f "$binfile" ]
              then
                      cmd="su - $user -c \"$binfile $cmdline_options\""
              fi
              ocf_log debug "Starting $process: $cmd"
              # Execute the command as created above
              eval $cmd
              sleep 1

              if nameserver_status
              then
                      ocf_log debug "$process: $cmd started successfully"
            switch_nameserver
                      return $OCF_SUCCESS
              else
                      ocf_log err "$process: $cmd could not be started"
                      return $OCF_ERR_GENERIC
              fi
      else
              # If already running, consider start successful
              ocf_log debug "$process: $cmd is already running"
        switch_nameserver
              return $OCF_SUCCESS
      fi
}

nameserver_stop() {
        if [ -n "$OCF_RESKEY_stop_timeout" ]
        then
                stop_timeout=$OCF_RESKEY_stop_timeout
        elif [ -n "$OCF_RESKEY_CRM_meta_timeout" ]; then
                # Allow 2/3 of the action timeout for the orderly shutdown
                # (The origin unit is ms, hence the conversion)
                stop_timeout=$((OCF_RESKEY_CRM_meta_timeout/1500))
        else
                stop_timeout=10
        fi

      if nameserver_status
      then
                pid=`getpid $pidfile`
                kill $pid
                i=0
                while [ $i -lt $stop_timeout ]
                do
                        if ! nameserver_status
                        then
                              rm -f $pidfile
                                return $OCF_SUCCESS
                        fi
                        sleep 1
                        let "i++"
                done

                ocf_log warn "Stop with SIGTERM failed/timed out, now sending SIGKILL."
                kill -9 $pid
                rm -f $pidfile
                if ! nameserver_status
                then
                        ocf_log warn "SIGKILL did the job."
                        return $OCF_SUCCESS
                else
                        ocf_log err "Failed to stop - even with SIGKILL."
                        return $OCF_ERR_GENERIC
                fi
      else
              # was not running, so stop can be considered successful
              ocf_log warn "nameserver not running."
              rm -f $pidfile
              return $OCF_SUCCESS
      fi
}

nameserver_monitor() {
      nameserver_status
      ret=$?
      if [ $ret -eq $OCF_SUCCESS ]
      then
              #ocf_log debug "nameserver_monitor hook func: $OCF_RESKEY_monitor_hook"
              OCF_RESKEY_monitor_hook="monitor_nameserver"
              if [ -n "$OCF_RESKEY_monitor_hook" ]; then
                      eval "$OCF_RESKEY_monitor_hook"
                        if [ $? -ne $OCF_SUCCESS ]; then
                                return ${OCF_ERR_GENERIC}
                        fi
                      return $OCF_SUCCESS
              else
                      true
              fi
      else
              return $ret
      fi
}

# FIXME: Attributes special meaning to the resource id
process="$OCF_RESOURCE_INSTANCE"
binfile="$OCF_RESKEY_basedir/bin/nameserver"
cmdline_options="-f $OCF_RESKEY_basedir/conf/ns.conf -d"
pidfile="$OCF_RESKEY_pidfile"
[ -z "$pidfile" ] && pidfile="$OCF_RESKEY_basedir/logs/nameserver.pid"

user="$OCF_RESKEY_user"
[ -z "$user" ] && user=admin

### monitor namserver parameters ########
nsip="$OCF_RESKEY_nsip"
nsport="$OCF_RESKEY_nsport"
montool="$OCF_RESKEY_montool"
[ -z "$montool" ] && montool="$OCF_RESKEY_basedir/bin/ha_monitor"

nameserver_validate() {
      if ! su - $user -c "test -x $binfile"
      then
              ocf_log err "binfile $binfile does not exist or is not executable by $user."
              exit $OCF_ERR_INSTALLED
      fi
      if ! getent passwd $user >/dev/null 2>&1
      then
              ocf_log err "user $user does not exist."
              exit $OCF_ERR_INSTALLED
      fi
      return $OCF_SUCCESS
}

nameserver_meta() {
cat <<END
	<?xml version="1.0"?>
	<!DOCTYPE resource-agent SYSTEM "ra-api-1.dtd">
	<resource-agent name="nameserver">
	<version>1.0</version>
	<longdesc lang="en">
		This is a OCF RA to manage tfsv1.4 ha nameserver.
	</longdesc>
	<shortdesc lang="en">Manages an tfs nameserver service</shortdesc>

	<parameters>
		<parameter name="basedir" required="1" unique="1">
		<longdesc lang="en">nameserver root directory </longdesc>
		<shortdesc lang="en">Full path name of the binary to be executed</shortdesc>
		<content type="string"/>
	</parameter>
	<parameter name="pidfile" required="0">
		<longdesc lang="en">
			File to read/write the PID from/to.
		</longdesc>
		<shortdesc lang="en">File to write STDOUT to</shortdesc>
		<content type="string"/>
	</parameter>
	<parameter name="user" required="0">
		<longdesc lang="en">
			User to run the command as
		</longdesc>
		<shortdesc lang="en">User to run the command as</shortdesc>
		<content type="string" default="admin"/>
	</parameter>
	<parameter name="stop_timeout">
		<longdesc lang="en">
			In the stop operation: Seconds to wait for kill -TERM to succeed
			before sending kill -SIGKILL. Defaults to 2/3 of the stop operation timeout.
		</longdesc>
		<shortdesc lang="en">Seconds to wait after having sent SIGTERM before sending SIGKILL in stop operation</shortdesc>
		<content type="string" default=""/>
	</parameter>
	<parameter name="nsip">
		<longdesc lang="en"> nameserver ipaddr or hostname </longdesc>
		<shortdesc lang="en">nameserver ipaddr or hostname </shortdesc>
		<content type="string" default="localhost"/>
	</parameter>
	<parameter name="nsport">
		<longdesc lang="en"> nameserver port </longdesc>
		<shortdesc lang="en">nameserver port </shortdesc>
		<content type="integer" default="3100"/>
	</parameter>
	<parameter name="montool">
		<longdesc lang="en"> check nameserver if working well, run as -i ip -p port </longdesc>
		<shortdesc lang="en"> check nameserver status </shortdesc>
		<content type="string"/>
	</parameter>
</parameters>
<actions>
	<action name="start"   timeout="90" />
	<action name="stop"    timeout="180" />
	<action name="monitor" depth="0"  timeout="20" interval="5" />
	<action name="meta-data"  timeout="5" />
	<action name="validate-all"  timeout="5" />
</actions>
</resource-agent>
END
exit 0
}

case "$1" in
      meta-data|metadata|meta_data)
              nameserver_meta
      ;;
      start)
              nameserver_start
      ;;
      stop)
              nameserver_stop
      ;;
      monitor)
              nameserver_monitor
      ;;
      validate-all)
              nameserver_validate
      ;;
      *)
              ocf_log err "$0 was called with unsupported arguments: $*"
              exit $OCF_ERR_UNIMPLEMENTED
      ;;
esac
```

<hr />

##[CIB原理及配置]
　　从这一节开始我们将关注点移到Resource Allocated Layer层。在Heartbeat v3中，CRM是由Pacemaker承担资源管理工作。  
　　
> Pacemaker的[官方网站](http://clusterlabs.org/wiki/Main_Page)  
 
　　目前Pacemaker的版本1.1.14，2016年1月中旬发布，已经支持了CentOS7. 包含的组件包括：

* stonithd：心跳系统，决定节点是否加入或脱离集群。【**节点级别**】
* lrmd：LRM，本地资源管理守护进程。它提供了一个通用的接口支持的资源类型。直接调用资源代理（脚本）。
* pengine：PE，策略引擎。根据当前状态和配置集群计算的下一个状态。产生一个过渡图，包含行动和依赖关系的列表。
* CIB：集群信息库。包含所有群集选项，节点，资源，他们彼此之间的关系和现状的定义。**自动同步更新**到所有群集节点。
* CRMD：集群资源管理守护进程。主要是消息代理的PEngine和LRM，还选举一个领导者（DC）统筹活动（包括启动/停止资源）的集群。【**资源级别**】
* OpenAIS：OpenAIS的消息和成员层。
* Heartbeat：心跳消息层，替代OpenAIS，系列文章中都使用Heartbeat.
* CCM：集群成员一致性管理模块。

###内部架构
　　引用官方的一张图将 _图1_ 中的Resource Allocated Layer层进行拓展：
<center>![图1 Pacemaker内部原理图](http://ww3.sinaimg.cn/bmiddle/a8484315jw1f1140i776qj20m80gowh5.jpg)</center><br/><center><font size=2>图2 Pacemaker内部原理图</font></center>
　　CIB使用xml配置集群和当前集群中的所有资源，会在节点之间自动同步。PE通过CIB决策集群如何达到最佳状态（选出master）。  
　　master选举逻辑根据PE的决策指令在所有的节点中选择一个节点（CRMd）作为master，如果该节点的CRMd出错，会再选择一个。这个选举逻辑官方叫做 DC(Designated Controller)。具体的过程是：  
　　DC将PE的决策指令按照指定的优先级顺序传递给本节点的LRMd、通过CRMd传递给其他节点的LRMd，同时确定一个希望的（expected）master. 各个节点的LRMd调用TE执行具体的决策指令选出各自认为的master，将结果回传给DC，DC根据之前希望的master和其他节点已经回传的执行结果确定是继续等待优先级高的节点的结果，还是终断本次选举并通知PE再进行一轮选举。   
　　在一些情况下，为了保护共享数据或者完成数据恢复，DC可能决策某节点需要脱离集群，这需要STONITHd协助完成。    
　　在整个过程中重点需要搞清楚的有两点：CIB怎么配置，这是本文的重点；DC如何分配资源我们将在下一节介绍。

###Pacemaker支持的集群模式
　　前文提到的几种集群形式Pacemaker都可以支持，在TFS集群的搭建过程中主要使用主备模式。    
　　Pacemaker官方介绍了其他的模式，可以参考[这里](http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/_types_of_pacemaker_clusters.html)和[这里](http://clusterlabs.org/wiki/Main_Page)。这些模式实际上是通过CIB的资源分数、节点属性设置实现的。一定要注意，HA集群不仅仅支持两个节点、一个资源，完全可以支持多节点、多资源。

###CIB配置
　　CIB描述了集群全部信息供DC使用。

####CIB内容
　　配置文件cib.xml默认在`/var/lib/heartbeat/crm/cib.xml`，最简单的CIB包括以下几项。

```xml
<cib crm_feature_set="3.0.7" validate-with="pacemaker-1.2" admin_epoch="1" epoch="0" num_updates="0">
  <configuration>
    <crm_config/>
    <nodes/>
    <resources/>
    <constraints/>
  </configuration>
  <status/>
</cib>
```

* **cib** CIB配置文件根节点 →[参考][cib-link]
	* **configuration** 集群配置信息的根节点   
		* **crm_config** 集群全局配置   
			* **cluster_property-set** 属性配置 →[参考][cluster-property-set-link]
		* **nodes** 集群各个节点的配置 →[参考][nodes-link]
			* **instance_attributes** Node参数配置   
		* **resources** 服务资源，TFS集群搭建HA使用RA资源 →[参考][resources-link]
			* **group** 可选，将多个资源合并为一组同时执行操作 →[参考][group-link]
				* **primitive** 配置资源及其属性，包括资源的分数(Score) →[参考][primitive-link]
				* **instance_attributes** OCF_RA所需参数配置，`crm ra meta OCF_RA_XXX`命令查询 →[参考][instance-attributes-link]
				* **meta_attributes** 配置资源的属性 →[参考][meta-attributes-link]
				* **operations** 操作配置，操作必须是OCF RA支持的操作 →[参考][operations-link]
		* **constraints** 资源在不同节点上的切换规则   
			* **rsc_location** 资源定位约束，确定不同节点上资源切换优先级（Score），也即针对同一资源，节点的优先级 →[参考][rsc_location-link]
				* **rule** 直接使用rsc-loaction可以实现优先级配置，使用rule可以使得配置项更加灵活易改，它能够精确配置节点的分数。rule不仅仅可以[应用于location][rule-location-link]，还可以[应用于meta_attributes][rule-meta-attributes-link] →[参考][rule-link]
			* **rsc_order** 可选，配置多资源的启动/停止顺序，多个配置之间是AND关系；可以使用[Ordered Set](http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-resource-sets-ordering.html#_ordered_set)的方式完成更多的顺序方式 →[参考][rsc-order-link]
			* **rsc_colocation** 可选，配置多资源的依赖；也可以使用[Colocation Set](http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-resource-sets-colocation.html)完成更多方式 →[参考][rsc-colocation-link]
		* **rsc_defaults** 全局资源默认配置。如果在<resources/>中对资源进行了配置，以<resources/>中的为准，否则以这里配置的为准   
	* **status** 集群每个节点的历史资源。集群根据该节点的内容构建完整的状态。注意该节点主要由LRMd在内存中操作，不会存盘，也不需要人为编辑，使用`#cibadmin --query|-Q`命令可以查看该节点的内容。   

> 配置中的score、stickiness等容易混淆，在配置时只要简单记住score用在constraints中，stickiness用在resource中即可。有关详细内容请参考下一节。

#####  _CIB例子_
　　下面通过TFS搭建HA时的CIB配置来说明一些关键的地方。

```xml
<cib epoch="37" admin_epoch="0" validate-with="pacemaker-1.0" crm_feature_set="3.0.1" have-quorum="1" dc-uuid="7508b0ea-78aa-4011-917a-fe5f5d7082cd" num_updates="184">
  <configuration>
    <crm_config>
      <cluster_property_set id="cib-bootstrap-options">
        <!--配置了4个参数-->
        <nvpair id="cib-bootstrap-options-dc-version" name="dc-version" value="1.0.12-unknown"/>
        <nvpair id="cib-bootstrap-options-cluster-infrastructure" name="cluster-infrastructure" value="Heartbeat"/>
        <nvpair id="cib-bootstrap-options-symmetric-cluster" name="symmetric-cluster" value="true"/>
        <nvpair id="cib-bootstrap-options-stonith-enabled" name="stonith-enabled" value="false"/>
      </cluster_property_set>
    </crm_config>
    <nodes>
      <!--两个节点-->
      <node id="268a3f70-2d62-4003-a323-0cd946fdc100" uname="yf04501" type="normal">
        <instance_attributes id="nodes-268a3f70-2d62-4003-a323-0cd946fdc100">
          <nvpair id="nodes-268a3f70-2d62-4003-a323-0cd946fdc100-standby" name="standby" value="off"/>
        </instance_attributes>
      </node>
      <node id="7508b0ea-78aa-4011-917a-fe5f5d7082cd" uname="yf04401" type="normal">
        <instance_attributes id="nodes-7508b0ea-78aa-4011-917a-fe5f5d7082cd">
          <nvpair id="nodes-7508b0ea-78aa-4011-917a-fe5f5d7082cd-standby" name="standby" value="off"/>
        </instance_attributes>
      </node>
    </nodes>
    <resources>
      <!--使用一个group将两个资源划分为一组，同时启动、停止、监控-->
      <group id="ns-group">
        <!--资源1：ocf类型，名字为ip-alias，RA使用IPaddr2，调度VIP-->
        <primitive class="ocf" id="ip-alias" provider="heartbeat" type="IPaddr2">
          <instance_attributes id="ip-alias-instance_attributes">
            <!--给RA传参，RA需要的参数可以通过 # crm ra meta XXX 命令查询-->
            <nvpair id="ip-alias-instance_attributes-ip" name="ip" value="172.16.138.252"/>
            <nvpair id="ip-alias-instance_attributes-nic" name="nic" value="eth1:1"/>
          </instance_attributes>
          <meta_attributes id="ip-alias-meta_attributes">
            <!--设置集群中节点资源的允许状态为 允许运行 的-->
            <nvpair id="ip-alias-meta_attributes-target-role" name="target-role" value="Started"/>
          </meta_attributes>
          <operations>
            <!--添加操作，只配置了monitor，每2s执行一次-->
            <op id="ip-alias-monitor-2s" interval="2s" name="monitor"/>
          </operations>
        </primitive>
        <!--资源2：ocf类型，名字为ip-alias，RA使用NameServer，调度TFS NS进程-->
        <primitive class="ocf" id="tfs-name-server" provider="heartbeat" type="NameServer">
          <instance_attributes id="tfs-name-server-instance_attributes">
            <nvpair id="tfs-name-server-instance_attributes-basedir" name="basedir" value="/data/tfs"/>
            <nvpair id="tfs-name-server-instance_attributes-nsip" name="nsip" value="localhost"/>
            <nvpair id="tfs-name-server-instance_attributes-nsport" name="nsport" value="10000"/>
            <nvpair id="tfs-name-server-instance_attributes-user" name="user" value="root"/>
          </instance_attributes>
          <meta_attributes id="tfs-name-server-meta_attributes">
            <!--设置集群中节点资源的允许状态为 允许运行 的；设定资源启动成功的初始分数为 正无穷；设定失败后设为 负无穷-->
            <nvpair id="tfs-name-server-meta_attributes-target-role" name="target-role" value="Started"/>
            <nvpair id="tfs-name-server-meta_attributes-resource-stickiness" name="resource-stickiness" value="INFINITY"/>
            <!--resource-failure-stickiness已经弃用，改用migration-threshold-->
            <nvpair id="tfs-name-server-meta_attributes-resource-failure-stickiness" name="resource-failure-stickiness" value="-INFINITY"/>
          </meta_attributes>
          <operations>
            <!--配置NameServer操作，monitor每2s执行一次,start和stop不会循环执行-->
            <op id="tfs-name-nameserver-monitor-2s" interval="2s" name="monitor"/>
            <op id="tfs-name-nameserver-start" interval="0s" name="start" timeout="30s"/>
            <op id="tfs-name-nameserver-stop" interval="0s" name="stop" timeout="30s"/>
          </operations>
        </primitive>
      </group>
    </resources>
    <constraints>
      <!--针对tfs-name-server这一资源设置不同节点的分数-->
      <rsc_location id="cli-prefer-tfs-name-server" rsc="tfs-name-server">
        <!--添加规则，如果#uname为yf04401，则该节点的分数设置为 正无穷-->
        <rule id="cli-prefer-rule-tfs-name-server" score="INFINITY" boolean-op="and">
          <expression id="cli-prefer-expr-tfs-name-server" attribute="#uname" operation="eq" value="yf04401" type="string"/>
        </rule>
      </rsc_location>
    </constraints>
    <!--添加全局默认资源配置，初始分数配置为100，在resource节点中，tfs-name-server资源覆盖了该配置，ip-alias资源使用该配置-->
    <rsc_defaults>
      <meta_attributes id="rsc_defaults-options">
        <nvpair id="rsc_defaults-options-resource-stickiness" name="resource-stickiness" value="100"/>
      </meta_attributes>
    </rsc_defaults>
  </configuration>
  <status>
    ......
  </status>
</cib>
```

####如何操作CIB
　　官方给了三条**规定**：
　　1. 不要手动操作cib.xml   
　　2. 遵循第1条   
　　3. 如果不遵循第1条，集群有权拒绝使用手动更改的CIB       
　　鉴于此，我们需要通过 `cibadmin` 更新CIB。所有的更新及时生效并同步到所有集群节点。

#####查询（Query）
```sh
#cibadmin --query|-Q
#cibadmin --query|-Q --obj_type {Xml_Node}
```
　　第一条命令查询内存中整个CIB配置，第二条查询CIB的一个XML节。这两条命令可以结合修改命令对内存中的CIB配置进行修改：

```sh
# cibadmin --query > tmp.xml
# vi tmp.xml
# cibadmin --replace --xml-file tmp.xml
```
　　结合第二条命令可以只对指定的XML节进行修改，降低修改出错的风险。

#####删除（Delete）
```sh
cibadmin --delete|-D --crm_xml '{XML_Tag}'
```
　　一些情况下需要删除部分CIB配置，就可以使用上面的命令，比如

```sh
# cibadmin -Q | grep stonith
<nvpair id="cib-bootstrap-options-stonith-enabled" name="stonith-enabled" value="false"/>
# cibadmin --delete --crm_xml '<nvpair id="cib-bootstrap-options-stonith-enabled">'
```
　　注意 {XML_Tag}需要指明Tag的id.

#####更新（Update）
```sh
# cibadmin --replace --xml-file tmp.xml
```
　　更新CIB配置和查询命令配合使用，参考上面查询命令的使用。大多数的CIB配置更新其实不需要直接修改xml文件，而使用其他命令代替cibadmin，比如：

```sh
# crm_attribute --name stonith-enabled --update 1/0 开启/关闭Stonith
# crm_attribute --type crm_config --attr-name symmetric-cluster --attr-value true/false 是否可以在集群的任何节点上迁移服务
# crm_attribute --type rsc_defaults --name resource-stickiness --update 100 设置全局默认资源配置
```
> 官方提供了沙箱操作来测试更新后的CIB，可以参考[Making Configuration Changes in a Sandbox](http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-config-sandboxes.html)，如果需要，请自行学习，文中不再详述。

###CRM shell
　　启动Pacemaker服务后，更新CIB配置即可让集群运转起来。这里介绍一个工具 crmsh，该工具可以查看当前集群中的资源运行情况，包括查看目前资源的master节点，配置是否正确，集群系统支持的资源类型等，当然也可以对CIB资源进行直接操作配置。   
　　从pacemaker 1.1.8开始，Crmsh发展成一个独立项目，pacemaker中不再提供，[官方网站](https://savannah.nongnu.org/forum/forum.php?forum_id=7672)提供下载。下载RPM包结合yum解决依赖安装完成。   
　　在终端直接输入`crm`就进入CRM Shell控制台。按两下`TAB`键提示当前支持的命令，`help`可以查看帮助信息，`help CMD`可以查看具体命令的帮助信息。

####查看配置
```sh
# crm
crm(live)#
crm(live)# configure
crm(live)configure# show
node $id="3b8aa51e-88c4-4e3a-8fdb-2311ca791307" tc102
node $id="9b08998a-46c9-43f8-b0a9-18c7cb3acf5f" tc101
primitive ip-alias ocf:heartbeat:IPaddr2 \
	params ip="10.73.14.105" nic="eth1:1" \
	op monitor interval="2s" \
	meta target-role="Started"
primitive tfs-name-server ocf:heartbeat:NameServer \
	params basedir="/data0/tfs" nsip="localhost" nsport="10000" user="root" \
	op monitor interval="2s" \
	op start interval="0s" timeout="30s" \
	op stop interval="0s" timeout="30s" \
	meta target-role="Started" resource-stickiness="INFINITY" resource-failure-stickiness="-INFINITY"
group ns-group ip-alias tfs-name-server
property $id="cib-bootstrap-options" \
	dc-version="1.0.12-unknown" \
	cluster-infrastructure="Heartbeat" \
	symmetric-cluster="true" \
	no-quorum-policy="ignore" \
	stonith-enabled="0"
rsc_defaults $id="rsc_defaults-options" \
	resource-stickiness="100"
``` 
　　结果展示了CIB配置，比起Xml更加简洁明了。

####检查配置
```sh
接上
crm(live)configure# verify
ERROR: tfs-name-server: attribute resource-failure-stickiness does not exist
WARNING: tfs-name-server: specified timeout 30s for start is smaller than the advised 90
WARNING: tfs-name-server: specified timeout 30s for stop is smaller than the advised 180
crm(live)configure#
```
　　提示我们tfs-name-server资源没有`resource-failure-stickiness`属性。这是因为在Pacemaker 1.0以后，`resource-failure-stickiness`属性被`migration-threshold`属性替代了。

####修改配置
```sh
接上
crm(live)configure# edit
```
　　进入vim编辑模式，编辑内容为之前crm show的内容，而不是xml内容，将 `resource-failure-stickiness="-INFINITY"` 修改为 `migration-threshold="-INFINITY"`，保存退出。crm会自动检测编辑后的配置，如果还有ERROR，会询问是否继续编辑，如果选择否，之前的编辑不会应用。再次调用`verify`检查配置，已经没有ERROR提示了。   
　　修改全部完成后提交一下：

```sh
接上
crm(live)configure# commit
``` 
　　Crm Shell其他的功能可以参考它的帮助文档。

[rule-location-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/_using_rules_to_control_resource_options.html
[rule-meta-attributes-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/_using_rules_to_control_cluster_options.html
[rule-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/ch08.html
[primitive-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/primitive-resource.html  
[instance-attributes-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/_resource_instance_attributes.html
[meta-attributes-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-resource-options.html#_resource_meta_attributes
[operations-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/_resource_operations.html
[cib-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/ch03.html#_cib_properties
[cluster-property-set-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/_cluster_options.html
[nodes-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/ch04.html
[instance_attributes-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-node-attributes.html
[resources-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/ch05.html
[group-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/ch10.html#group-resources
[rsc-location-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/_deciding_which_nodes_a_resource_can_run_on.html#_location_properties
[rsc-order-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-resource-ordering.html#_ordering_properties
[rsc-colocation-link]:http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/s-resource-colocation.html#_colocation_properties

##[资源分配决策原理]
　　Pacemaker根据CIB配置决策服务在集群节点上的分配情况，原理是怎么样的呢？这里涉及了资源级别的资源黏性（stickiness）和节点级别的节点分数（score），它们都是依靠分数机制来表征的，有人也把分数机制叫做得票机制。

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

#####　 _Pacemaker 1.0之前_
　　resource-stickiness和resource-failure-stickiness搭配，resource-stickiness表示资源成功一次资源获得多少分，一般设置正数；resource-failure-stickiness的绝对值表示资源失败一次资源扣掉多少分，一般设置负数。未配置时默认都为0.    
　　比如：   

* resource-stickiness=100 & resource-failure-stickiness=-100 资源每次成功黏性增强就加100分，每次失败黏性降低减100分
* resource-stickiness=INFINITY & resource-failure-stickiness=-INFINITY 资源一旦成功就变为集群中**最愿意**运行服务的节点，以后需要迁移服务时仍然在该节点上运行服务的倾向最高；资源一旦失败就变为集群中**最不愿意**运行服务的节点，尽量不会再在该节点上运行服务。

#####　 _Pacemaker 1.0之后_
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
2. 节点上的资源在**停止的时候出错**，可能在迁移节点后造成一个以上节点提供相同资源服务，形成资源分裂，如果[STONITH](http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Pacemaker_Explained/ch13.html#_what_is_stonith)是启用状态则触发Fencing机制（该机制防止脑裂破坏数据）停止造成集群分裂的节点，如果未启用，则不执行迁移。

####  _小结_

　　本文介绍了HA模型原理和Heartbeat v3中RA、Pacemaker的相关原理和配置，涉及到HA模型中关键的CRM、PE决策策略、RA等组件，基本可以将HA的主要原理说清楚了。对于底层的Heartbeat，在实际HA搭建时不需要了解太多，后面有时间再补充。


