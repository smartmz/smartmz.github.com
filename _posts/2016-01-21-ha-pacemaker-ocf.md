---
layout: post
title: "Pacemaker定制OCF类RA资源"
date: 2016-01-21 12:18
keywords: ha,ra,ocf
description: Pacemaker的OCF类RA资源
categories: [HA]
tags: [ha, pacemaker]
group: archive
icon: globe
--- 

<!-- more -->
 
　　HA的主要工作就是根据集群当前情况对资源进行分配调度，达到服务不中断的目的。这里涉及到两方面，一是资源本身，二是根据资源和PE的调度指令对资源实现调度。对应到具体的实现上，第一部分内容涉及Pacemaker的调度资源，就是RA，RA重点关注OCF类的；第二部分涉及Pacemaker如何进行资源调度，就是CIB配置。集群DC节点上的PE通过CIB配置决策资源在各个节点上的分配，然后由CRM通过节点上的LRM调用RA完成资源的启停。第二部分参见[《[HA]Pacemaker的CIB配置》](http://smartmz.github.io/2016/02/16/ha-pacemaker-cib)，本文重点来说第一部分。   
　　Pacemaker 支持的RA资源有Open Cluster Framework(OCF)、Linux Standard Base(LSB)、Systemd、Upstart、System Services、STONITH、Nagios Plugins等多种类型，但主要是两类，LSB和OCF。

* LSB 指 Linux 标准服务，通常就是 /etc/init.d 目录下那些脚本。Pacemaker 可以用这些脚本来启停服务。查看LSB资源可以使用命令：  
`# crm ra list lsb`
* OCF 实际上是对 LSB 服务的扩展，增加了一些高可用集群管理的功能如故障监控等和更多的元信息。查看OCF资源可以使用命令：  
`# crm ra list ocf`

　　如果需要pacemaker更好地对服务进行高可用保障，就得实现一个OCF类的RA资源。这是我们在本文重点要说的。   
　　Pacemaker自带的资源管理程序都在`/usr/lib/ocf/resource.d`目录中。其中的`heartbeat`目录包含了那些自带的常用服务，这些脚本可以作为自己实现的参考。

　　每个OCF资源是一个可执行的脚本文件，通过**命令行参数和环境变量**接收来自 Pacemaker 的输入。下面是一个简单的例子，创建了一个名叫 test 的资源。

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
###资源的meta
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

##资源的action
　　start、stop这两个命令，主要实现服务启动或停止；monitor命令实现状态检测。在实际编写脚本的时候需要**注意**，一定要保证在start、stop命令执行完成返回后，使用monitor命令检测状态。如果在未完成start或stop时就返回，比如直接将命令放后台执行，会导致通过monitor命令检测状态时，由于此时还未启动完成或stop而失败，导致被认为是故障，从而导致资源重启或切换。

####参数
　　命令的参数是通过环境变量传递的。   
　　比如test资源中的service\_path 和 probe\_url 会分别通过环境变量 `$OCF_RESKEY_service_path` 和 `$OCF_RESKEY_service_path` 传递。
####命令返回值
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
####日志
　　输出日志使用 `ocf_log {日志级别} {日志内容}`，该命令将日志输出到ha配置指定的文件中。ha的配置文件`/etc/ha.d/ha.cf`中定义了日志输出路径和日志输出级别：

```text
# cat /etc/ha.d/ha.cf
debugfile /var/log/ha-debug
debug 1
... ...
```
> 附1 - 参考文档
> http://www.linux-ha.org/doc/dev-guides/ra-dev-guide.html

> 附2 - TFS-HA OCF资源脚本实例
> 
> ``` sh
> #!/bin/sh
> # Initialization:
> . ${OCF_ROOT}/resource.d/heartbeat/.ocf-shellfuncs
>
> getpid() {
>         grep -o '[0-9]*' $1
> }
>
> monitor_nameserver() {
>       if ! test -x $montool
>       then
>               return $OCF_ERR_PERM
>       fi
>       ocf_log debug "begin to monitor $binfile service with command $cmd"
>       cmd="$montool -i $nsip -p $nsport -n"
>       eval $cmd
>       if [ $? -ne 0 ]
>       then
>               return $OCF_ERR_GENERIC
>       else
>               return $OCF_SUCCESS
>       fi
> }
>
> switch_nameserver() {
>     cmd="$montool -i $nsip -p $nsport -n -s 1"
>     eval $cmd
> }
>
> nameserver_status() {
>       if test -f "$pidfile"
>       then
>               if pid=`getpid $pidfile` && [ "$pid" ] && kill -0 $pid
>               then
>                       #status=`check_nameserver_running_status`
>                       return $OCF_SUCCESS
>               else
>                       # pidfile w/o process means the process died
>                       return $OCF_ERR_GENERIC
>               fi
>       else
>               return $OCF_NOT_RUNNING
>       fi
> }
>
> nameserver_start() {
>       if ! nameserver_status
>       then
>               if [ -f "$binfile" ]
>               then
>                       cmd="su - $user -c \"$binfile $cmdline_options\""
>               fi
>               ocf_log debug "Starting $process: $cmd"
>               # Execute the command as created above
>               eval $cmd
>               sleep 1
>
>               if nameserver_status
>               then
>                       ocf_log debug "$process: $cmd started successfully"
>             switch_nameserver
>                       return $OCF_SUCCESS
>               else
>                       ocf_log err "$process: $cmd could not be started"
>                       return $OCF_ERR_GENERIC
>               fi
>       else
>               # If already running, consider start successful
>               ocf_log debug "$process: $cmd is already running"
>         switch_nameserver
>               return $OCF_SUCCESS
>       fi
> }
>
> nameserver_stop() {
>         if [ -n "$OCF_RESKEY_stop_timeout" ]
>         then
>                 stop_timeout=$OCF_RESKEY_stop_timeout
>         elif [ -n "$OCF_RESKEY_CRM_meta_timeout" ]; then
>                 # Allow 2/3 of the action timeout for the orderly shutdown
>                 # (The origin unit is ms, hence the conversion)
>                 stop_timeout=$((OCF_RESKEY_CRM_meta_timeout/1500))
>         else
>                 stop_timeout=10
>         fi
>
>       if nameserver_status
>       then
>                 pid=`getpid $pidfile`
>                 kill $pid
>                 i=0
>                 while [ $i -lt $stop_timeout ]
>                 do
>                         if ! nameserver_status
>                         then
>                               rm -f $pidfile
>                                 return $OCF_SUCCESS
>                         fi
>                         sleep 1
>                         let "i++"
>                 done
>
>                 ocf_log warn "Stop with SIGTERM failed/timed out, now sending SIGKILL."
>                 kill -9 $pid
>                 rm -f $pidfile
>                 if ! nameserver_status
>                 then
>                         ocf_log warn "SIGKILL did the job."
>                         return $OCF_SUCCESS
>                 else
>                         ocf_log err "Failed to stop - even with SIGKILL."
>                         return $OCF_ERR_GENERIC
>                 fi
>       else
>               # was not running, so stop can be considered successful
>               ocf_log warn "nameserver not running."
>               rm -f $pidfile
>               return $OCF_SUCCESS
>       fi
> }
>
> nameserver_monitor() {
>       nameserver_status
>       ret=$?
>       if [ $ret -eq $OCF_SUCCESS ]
>       then
>               #ocf_log debug "nameserver_monitor hook func: $OCF_RESKEY_monitor_hook"
>               OCF_RESKEY_monitor_hook="monitor_nameserver"
>               if [ -n "$OCF_RESKEY_monitor_hook" ]; then
>                       eval "$OCF_RESKEY_monitor_hook"
>                         if [ $? -ne $OCF_SUCCESS ]; then
>                                 return ${OCF_ERR_GENERIC}
>                         fi
>                       return $OCF_SUCCESS
>               else
>                       true
>               fi
>       else
>               return $ret
>       fi
> }
>
> # FIXME: Attributes special meaning to the resource id
> process="$OCF_RESOURCE_INSTANCE"
> binfile="$OCF_RESKEY_basedir/bin/nameserver"
> cmdline_options="-f $OCF_RESKEY_basedir/conf/ns.conf -d"
> pidfile="$OCF_RESKEY_pidfile"
> [ -z "$pidfile" ] && pidfile="$OCF_RESKEY_basedir/logs/nameserver.pid"
>
> user="$OCF_RESKEY_user"
> [ -z "$user" ] && user=admin
>
> ### monitor namserver parameters ########
> nsip="$OCF_RESKEY_nsip"
> nsport="$OCF_RESKEY_nsport"
> montool="$OCF_RESKEY_montool"
> [ -z "$montool" ] && montool="$OCF_RESKEY_basedir/bin/ha_monitor"
>
> nameserver_validate() {
>       if ! su - $user -c "test -x $binfile"
>       then
>               ocf_log err "binfile $binfile does not exist or is not executable by $user."
>               exit $OCF_ERR_INSTALLED
>       fi
>       if ! getent passwd $user >/dev/null 2>&1
>       then
>               ocf_log err "user $user does not exist."
>               exit $OCF_ERR_INSTALLED
>       fi
>       return $OCF_SUCCESS
> }
>
> nameserver_meta() {
> cat <<END
> 	<?xml version="1.0"?>
> 	<!DOCTYPE resource-agent SYSTEM "ra-api-1.dtd">
> 	<resource-agent name="nameserver">
> 	<version>1.0</version>
> 	<longdesc lang="en">
> 		This is a OCF RA to manage tfsv1.4 ha nameserver.
> 	</longdesc>
> 	<shortdesc lang="en">Manages an tfs nameserver service</shortdesc>
>
> 	<parameters>
> 		<parameter name="basedir" required="1" unique="1">
> 		<longdesc lang="en">nameserver root directory </longdesc>
> 		<shortdesc lang="en">Full path name of the binary to be executed</shortdesc>
> 		<content type="string"/>
> 	</parameter>
> 	<parameter name="pidfile" required="0">
> 		<longdesc lang="en">
> 			File to read/write the PID from/to.
> 		</longdesc>
> 		<shortdesc lang="en">File to write STDOUT to</shortdesc>
> 		<content type="string"/>
> 	</parameter>
> 	<parameter name="user" required="0">
> 		<longdesc lang="en">
> 			User to run the command as
> 		</longdesc>
> 		<shortdesc lang="en">User to run the command as</shortdesc>
> 		<content type="string" default="admin"/>
> 	</parameter>
> 	<parameter name="stop_timeout">
> 		<longdesc lang="en">
> 			In the stop operation: Seconds to wait for kill -TERM to succeed
> 			before sending kill -SIGKILL. Defaults to 2/3 of the stop operation timeout.
> 		</longdesc>
> 		<shortdesc lang="en">Seconds to wait after having sent SIGTERM before sending SIGKILL in stop operation</shortdesc>
> 		<content type="string" default=""/>
> 	</parameter>
> 	<parameter name="nsip">
> 		<longdesc lang="en"> nameserver ipaddr or hostname </longdesc>
> 		<shortdesc lang="en">nameserver ipaddr or hostname </shortdesc>
> 		<content type="string" default="localhost"/>
> 	</parameter>
> 	<parameter name="nsport">
> 		<longdesc lang="en"> nameserver port </longdesc>
> 		<shortdesc lang="en">nameserver port </shortdesc>
> 		<content type="integer" default="3100"/>
> 	</parameter>
> 	<parameter name="montool">
> 		<longdesc lang="en"> check nameserver if working well, run as -i ip -p port </longdesc>
> 		<shortdesc lang="en"> check nameserver status </shortdesc>
> 		<content type="string"/>
> 	</parameter>
> </parameters>
> <actions>
> 	<action name="start"   timeout="90" />
> 	<action name="stop"    timeout="180" />
> 	<action name="monitor" depth="0"  timeout="20" interval="5" />
> 	<action name="meta-data"  timeout="5" />
> 	<action name="validate-all"  timeout="5" />
> </actions>
> </resource-agent>
> END
> exit 0
> }
>
> case "$1" in
>       meta-data|metadata|meta_data)
>               nameserver_meta
>       ;;
>       start)
>               nameserver_start
>       ;;
>       stop)
>               nameserver_stop
>       ;;
>       monitor)
>               nameserver_monitor
>       ;;
>       validate-all)
>               nameserver_validate
>       ;;
>       *)
>               ocf_log err "$0 was called with unsupported arguments: $*"
>               exit $OCF_ERR_UNIMPLEMENTED
>       ;;
> esac
> ```


