---
layout: post
title: "[HA]TFS紧急复制引起NameServer切换问题"
date: 2016-01-28 10:45
keywords: ha,tfs
description: HA基本原理
categories: [Storage]
tags: [ha]
group: archive
icon: globe
---
###描述
　　TFS给HA提供的OCF资源中，监控NS服务是否可用逻辑不完善，导致在NS节点网络、负载较大时将NS服务标记为不可用，引起NameServer切换。
　　有关HA-OCF资源的知识可以参考[这篇文章](http://smartmz.github.io/2016/01/21/ha-ocf/)，文章最后“附2 - TFS-HA OCF资源脚本实例”给出的例子就是本文主要涉及的TFS-OCF资源脚本。
###场景和问题
　　TFS在复制或者迁移（下面统称“任务”）时，集群内部的网络压力和负载都会升高：NS节点拥有所有DS节点与DS节点上所有block的对应关系，主要用来建立和下发任务；DS节点完成具体的任务，两个过程分别来看一下。
#####NS建立任务
　　NS建立任务（以迁移任务为例）这个过程大致分为这样几个过程：

```c++
bool LayoutManager::build_replicate_plan(const int64_t plan_seqno,
    const time_t now,
    int64_t& need,
    int64_t& adjust,
    int64_t& emergency_replicate_count)
{
  ......
  # 1. 获取需要复制的所有blocks
  # middle中存放这些blocks
  # emergency_replicate_count中存放迁移的数量
  find_need_replicate_blocks(need, now, emergency_replicate_count, middle);
  ......
  # 2. 遍历middle中的所有blocks，建立并下发任务
  std::multimap<PlanPriority, BlockCollect*>::const_reverse_iterator iter = middle.rbegin();
  for (; iter != middle.rend() && !(interrupt_ & INTERRUPT_ALL) && need > 0; ++iter)
  {
    ......
    {
      ......
      # 2.1. 给block加锁
      BlockChunkPtr ptr = get_chunk(iter->second->id());
      RWLock::Lock lock(*ptr, READ_LOCKER);
      ......
      {
        ......
        # 2.2. 从存在block的所有DS节点中选取一个节点作为src-DS
        # 正常情况下count=1
        # result中存放选取出来的src-DS
        count = elect_replicate_source_ds(*this, source, except, 1, result);
      }
      ......
      # 2.3. runer是个vector，临时存放block的src-DS和des-DS
      runer.push_back(result.back());
      ......
      {
        ......
        # 2.4. 从剩下的DS节点中选取一个节点作为des-DS
        # 正常情况下count=1
        # target中存放选取出来的des-DS
        count = elect_replicate_dest_ds(*this, except, 1, target);
      }
      ......
      runer.push_back(target.back());
      # 2.5. 形成一个复制任务，添加到任务队列，异步执行
      ReplicateTaskPtr task = new ReplicateTask(this, iter->first, block_id,now, now, runer, plan_seqno);
      ......
      add_task(task)；
      ......
    }
  }
  return true;
}

##### 紧接着，看一下ReplicateTask中的handle过程：

int LayoutManager::ReplicateTask::handle()
{
  ......
  {
    ......
    # 将任务载入msg（REPLICATE_BLOCK_MESSAGE），发送到src_DS
    iret = send_msg_to_server(block.source_id_, &msg, status);
    ......
  }
  return iret;
}
```
　　这个过程中有两点需要注意，一是时间复杂度较高，循环套循环的方式，二是给block加读锁，意味着该block在任务执行过程中不能提供写操作，变成了不可写block，造成可写block数下降。集群为了保证能够顺利写入数据，必须保证一定数量的可写block，在可写block数下降量大时会临时新建可写block，所以如果出现大量复制任务时，在监控上可以看到集群的可写block数猛增：
<center>![图1 集群的可写block数猛增](http://ww2.sinaimg.cn/bmiddle/a8484315jw1f0f3bhv822j20eo06nt95.jpg)</center><br/><center><font size=2>图1 集群的可写block数猛增</font></center>
　　这两点在DS节点多、需要复制blocks数量大的情况下都能加重NS节点的负载，而第二点还会引发其他的集群行为，增加集群的内压。
#####DS执行任务
　　NS的任务发送到src-DS节点，src-DS执行任务。直接看代码：

```c++
int DataService::replicate_block_cmd(ReplicateBlockMessage* message)
{
  ......
  # 从msg中获取到复制任务，添加到任务列表
  ReplBlock* b = new ReplBlock();
  memcpy(b, message->get_repl_block(), sizeof(ReplBlock));
  ......
  repl_block_->add_repl_task(b);
  ......
  return TFS_SUCCESS;
}

int ReplicateBlock::add_repl_task(ReplBlock* tmp_rep_blk)
{
  ......
  #将要迁移的block添加到队列
  repl_block_queue_.push_back(tmp_rep_blk);
  ......
}

##### 接下来看一下执行任务的具体过程

int ReplicateBlock::run_replicate_block()
{
  ......
  ReplBlock *b = repl_block_queue_.front();
  repl_block_queue_.pop_front();
  ......
  # 将block数据和元信息发送到des-DS节点
  int ret = replicate_block_to_server(b);
  ......    
  return TFS_SUCCESS;
}

int ReplicateBlock::replicate_block_to_server(const ReplBlock* b)
{
  ......
  # 1.1. 获取逻辑block，这个包含了block的元信息。
  uint32_t block_id = b->block_id_;
  LogicBlock* logic_block = BlockFileManager::get_instance()->get_logic_block(block_id);
  ......
  int32_t total_len = logic_block->get_data_file_size();
  # 1.2. 循环发送block数据到des-DS节点，每次发送MAX_READ_SIZE
  # 发送msg: WRITE_RAW_DATA_MESSAGE
  while (offset < total_len || 0 == total_len)
  {
    int32_t read_len = MAX_READ_SIZE;
    ret = logic_block->read_raw_data(tmp_data_buf, read_len, offset);
    ......
    WriteRawDataMessage req_wrd_msg;
    req_wrd_msg.set_XXX(XXX);
    ......
    send_msg_to_server(ds_ip, client, &req_wrd_msg, rsp_msg);
    offset += len;
    ......
  }
  ......
  # 2.1. 获取block元数据
  ret = logic_block->get_meta_infos(raw_meta_vec);
  ......
  # 2.2. 发送block元数据到des-DS节点
  # 发送msg：WRITE_INFO_BATCH_MESSAGE
  WriteInfoBatchMessage req_wib_msg;
  req_wib_msg.set_XXX(XXX);
  ......
  send_msg_to_server(ds_ip, client, &req_wib_msg, rsp_msg);
  return ret;
}
```
　　src-DS节点的处理过程需要说明两点：一是刚才在NS上这个blockId对应的block被上了锁，整个过程为不可写状态，可能引发的其他任务在上面已经说明过了；二是发送block数据的过程没有考虑对网络的压力，在复制任务量大的情况下，TFS集群内压会大幅度升高：
<center>![图2 集群内压增大（某一个DS的状态）](http://ww2.sinaimg.cn/bmiddle/a8484315jw1f0g8f3k8kjj20qv0j6n0l.jpg)</center><br/><center><font size=2>图2 集群内压增大（某一个DS的状态）</font></center>
　　在新的TFS版本中可以在线调整任务数上限，可是这个版本没有……
　　了解任务的执行过程，简单说明一下导致当前TFS集群复制任务量大的原因。一般在两种情况会触发复制机制：
　　1. 正常情况下，向TFS集群存文件，一个DS节点收到文件后，会根据配置中的min_replication和max_replication参数决定副本数，触发复制任务达到要求；
　　2. 如果有DS节点的blocks丢失、损坏，或min_replication或max_replication参数调整，如果blocks的副本数不达要求，就会触发复制任务，为了区分1，叫做“紧急复制”（emergency replication）。
　　当前TFS集群是因为调整参数引起的大量紧急复制任务：

 |min_replication|max_replication
---|---|---
修改前|1|2
修改后|3|3
　　任务开始后，整个集群的压力陡增，NS节点每隔一段时间就会迁移一次，导致之前NS节点建立的复制任务不能正常进行，新NS节点重建任务，陷入恶性循环。
###问题原因
　　问题实际上不是出在TFS集群本身，而是出在HA对NS进程可用性的检测上，通过NS主节点上的HA的日志可以清楚的看到：

```sh
Jan 12 13:21:00 yf04401 lrmd: [17616]: debug: rsc:tfs-name-server:38: monitor
Jan 12 13:21:00 yf04401 lrmd: [17616]: info: RA output: (tfs-name-server:monitor:stderr) [2016-01-12 13:21:00] ERROR get_server_running_status (ha_monitor.cpp:60) [46930444118096] send packet error
......
Jan 12 13:29:28 yf04401 lrmd: [17616]: debug: rsc:ip-alias:36: monitor
Jan 12 13:29:28 yf04401 lrmd: [17616]: info: RA output: (tfs-name-server:monitor:stderr) [2016-01-12 13:29:28] ERROR get_server_running_status (ha_monitor.cpp:60) [46938235480144] send packet error
Jan 12 13:29:28 yf04401 lrmd: [17616]: info: RA output: (tfs-name-server:monitor:stderr) [2016-01-12 13:29:28] ERROR get_server_running_status (ha_monitor.cpp:60) [46938235480144] send packet error
......
Jan 12 13:30:46 yf04401 lrmd: [17616]: debug: rsc:tfs-name-server:38: monitor
Jan 12 13:30:47 yf04401 lrmd: [17616]: info: RA output: (tfs-name-server:monitor:stderr) [2016-01-12 13:30:47] ERROR get_server_running_status (ha_monitor.cpp:60) [47584978045008] send packet error
Jan 12 13:30:48 yf04401 lrmd: [17616]: info: RA output: (tfs-name-server:monitor:stderr) [2016-01-12 13:30:48] ERROR get_server_running_status (ha_monitor.cpp:60) [47584978045008] send packet error
Jan 12 13:30:48 yf04401 lrmd: [17616]: info: RA output: (tfs-name-server:monitor:stderr) [2016-01-12 13:30:48] ERROR get_server_running_status (ha_monitor.cpp:60) [47584978045008] send packet error
[2016-01-12 13:30:48] ERROR main (ha_monitor.cpp:130) [47584978045008] ping server failed, get status: -1
......
```
　　从13:21:00开始，HA探测集群服务报“send packet error”异常，并且逐渐增多到3次后，HA任务服务不可用，启动NS主备切换。HA的探测机制又是怎么样的呢？它使用了TFS提供的OCF资源，其中monitor部分的逻辑是这样：

```sh
#OCF_RESKEY_basedir即为TFS的Home目录
binfile="$OCF_RESKEY_basedir/bin/nameserver"
montool="$OCF_RESKEY_montool"
[ -z "$montool" ] && montool="$OCF_RESKEY_basedir/bin/ha_monitor"

monitor_nameserver() {
        if ! test -x $montool
        then
                return $OCF_ERR_PERM
        fi
        # Bug,这个应该在下一句后面，$cmd现在为空
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

nameserver_monitor() {
        nameserver_status
        ret=$?
        if [ $ret -eq $OCF_SUCCESS ]
        then
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

case "$1" in
        ......
        monitor)
                nameserver_monitor
        ;;
        ......
esac
```
　　OCF资源调用了ha_monitor，这是TFS集群自带的一个检测工具。看一下关键的代码片段：

```c++
static int8_t get_server_running_status(const uint64_t ip_port, const int32_t switch_flag)
{
  ......
  do
  {
    ++count;
    client = NewClientManager::get_instance().create_client();
    if (NULL != client)
    {
      #向目标Server发送心跳包，msg HEARTBEAT_AND_NS_HEART_MESSAGE
      HeartBeatAndNSHeartMessage heart_msg;
      heart_msg.set_ns_switch_flag_and_status(switch_flag, 0);
      tbnet::Packet* rsp_msg = NULL;
      send_msg_to_server(ip_port, client, &heart_msg, rsp_msg, TIMEOUT);
      ......
      NewClientManager::get_instance().destroy_client(client);
    }
  }
  while (count < 3 && -1 == status);
  return status;
}
```
　　monitor每次检测循环最多发送3次心跳包，如果有响应，就认为NS进程可用，否则就认为不可用。从HA的日志来看，大量复制任务开始执行后，网络负载逐步加大，NS节点处理任务逐步增加，最后无法及时处理任务队列中monior的心跳包，monitor无法正常收到NS进程的回包，HA认为服务不可用，触发主备切换。
###解决问题
　　TFS提供的ha_monitor工具相当于从服务的层面对NS进程进行监控，但是在负载重的时候可能没有办法及时响应monitor. 目前采取的解决方案是直接监控NS进程是否还存在，这样不能防止NS进程僵死的情况，但是从日常经验来看，还没有遇到过这种情况。

```sh
monitor_nameserver() {
        NSEXSIT=$(/bin/ps x|/bin/grep $TFS_HOME/bin/nameserver|/bin/grep -v grep|/bin/grep -v python|/bin/awk '{print $1}')
        echo "NSEXSIT:$NSEXSIT"
        if [ -z "$NSEXSIT" ]; then
                ocf_log debug "ERRORRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRRR!"
                return $OCF_ERR_GENERIC
        else
                return $OCF_SUCCESS
        fi
}
```


