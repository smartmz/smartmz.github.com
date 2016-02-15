---
layout: post
title: "TFS自动均衡机制"
date: 2016-02-01 18:00
keywords: ha,tfs
description: TFS自动均衡的策略
categories: [Storage]
tags: [tfs]
group: archive
icon: globe
---
###描述
　　TFS集群支持平滑扩容，当加入新的节点后，NameServer会对DataServer上的Blocks进行均衡，使所有DataServers的容量尽早达到均衡。进行均衡计划时，首先通过计算将机器划分为两堆，一堆是超过平均数量的，作为移动源；一类是低于平均数量的，作为移动目标。  
　　移动目标的选择基于以下策略：首先一个block的移动的源和目的，应该保持在同一网段内；在作为目标的一定机器内，优先选择同机器的源到目标之间移动。

<!-- more -->

###代码

```c++
bool LayoutManager::build_balance_plan(const int64_t plan_seqno, const time_t now, int64_t& need)
{
  ......
  //统计集群存储信息
  statistic_all_server_info(need, total_capacity, total_use_capacity, total_load, alive_server_size);
  ......
  //计算集群总的可用容量占比
  double percent = calc_capacity_percentage(total_use_capacity, total_capacity);
  std::set<ServerCollect*> target; //存放均衡目标DS
  std::multimap<int64_t, ServerCollect*> source; //存放均衡源DS
  ......
  //计算集群平均负载
  int64_t average_load = total_load / alive_server_size;
  //将集群中的DS分为两大类-均衡目标DS 和 均衡源DS
  split_servers(need, average_load, percent, source, target);
  ......
  //从后向前遍历源DS，已用容量占比大的排在后面，需要尽早迁移
  std::multimap<int64_t, ServerCollect*>::const_reverse_iterator it = source.rbegin();
  for (; it != source.rend() && !(interrupt_ & INTERRUPT_ALL) && need > 0 && !target.empty(); ++it)
  {
    it->second->rdlock();
    std::set<BlockCollect*, ServerCollect::BlockIdComp> blocks(it->second->hold_);
    force = it->second->is_full();//如果ds容量已满,强制迁移
    it->second->unlock();
    //一般情况下，从第一个block开始迁移
    std::set<BlockCollect*, ServerCollect::BlockIdComp>::const_iterator cn_iter = blocks.begin();
    //强制迁移的情况下，随机从某个block开始迁移
    if (force && !blocks.empty())
    {
      random_value = random() % blocks.size();
      while (index < random_value && cn_iter != blocks.end())
      {
        ++index;
        ++cn_iter;
      }
    }
    //遍历需要迁移的block
    for (; cn_iter != blocks.end() && !(interrupt_ & INTERRUPT_ALL) && need > 0; ++cn_iter, ++index)
    {
        ......
        //选择一个目标ds
        bool bret = elect_move_dest_ds(target, servers, it->second, &target_ds);
        ......
        std::vector<ServerCollect*> runer;
        runer.push_back(it->second);
        runer.push_back(target_ds);
        //创建迁移任务
        MoveTaskPtr task = new MoveTask(this, PLAN_PRIORITY_NORMAL,  block_id, now , now, runer, plan_seqno);
        add_task(task)
        ......
        --need;
        std::set<ServerCollect*>::iterator tmp = target.find(it->second);
        if (tmp != target.end()) target.erase(tmp);
        tmp = target.find(target_ds);
        if (tmp != target.end()) target.erase(tmp);
        break;
    }
  }
  return true;
}

void LayoutManager::statistic_all_server_info(
    const int64_t need,           //任务队列中剩余未处理的任务
    uint64_t& total_capacity,     //total_capacity-集群总容量
    uint64_t& total_use_capacity, //total_use_capacity-集群已使用容量
    int64_t& total_load,          //total_load-集群总负债
    int64_t& alive_server_size)   //alive_server_size-存活DS的数量
{
  RWLock::Lock lock(server_mutex_, READ_LOCKER);
  SERVER_MAP::const_iterator iter = servers_.begin();
  //遍历所有DS
  for (; iter != servers_.end() && !(interrupt_ & INTERRUPT_ALL) && need > 0; ++iter)
  {
    if (iter->second->is_alive())
    {
      //总容量=实际容量*最大可用容量比，最大可用容量比对应TFS配置的use_capacity_ratio
      total_capacity += (iter->second->total_capacity() * SYSPARAM_NAMESERVER.max_use_capacity_ratio_) / 100;
      total_use_capacity += iter->second->use_capacity();
      total_load     += iter->second->load();
      ++alive_server_size;
    }
  }
}

void LayoutManager::split_servers(const int64_t need,
    const int64_t average_load, const double percent, 
    std::multimap<int64_t,ServerCollect*>& source, 
    std::set<ServerCollect*>& target)
{
  RWLock::Lock lock(server_mutex_, READ_LOCKER);
  //遍历所有DS
  SERVER_MAP::const_iterator iter = servers_.begin();
  for (; iter != servers_.end() && !(interrupt_ & INTERRUPT_ALL) && need > 0; ++iter)
  {
    ......
    //根据存储占比将servers分为迁移源servers和迁移目标servers两类
    split_servers_helper(average_load, percent, iter->second, source, target);
  }
}

void LayoutManager::split_servers_helper(const int64_t average_load,
    const double percent,
    ServerCollect* server,
    std::multimap<int64_t, ServerCollect*>& source,
    std::set<ServerCollect*>& target)
{
  UNUSED(average_load); //和平均负载没关系?!
  ......
  const uint64_t current_total_capacity = server->total_capacity() * SYSPARAM_NAMESERVER.max_use_capacity_ratio_ / 100; //该ds的总容量
  double current_percent = calc_capacity_percentage(server->use_capacity(), current_total_capacity);//ds当前已用容量占比
  if ((current_percent > percent + SYSPARAM_NAMESERVER.balance_percent_)//ds当前已用容量占比 大于 集群已用容量平均占比+平衡因子，平衡因子对应TFS配置的balance_percent参数
      || (current_percent >= 1.0))//ds当前已用容量占比 大于等于 1(存储已满)
  {//加入迁移源ds，multimap类型，按照已用容量占比从小到大插入
    source.insert(std::multimap<int64_t, ServerCollect*>::value_type(
          static_cast<int64_t>(current_percent * PERCENTAGE_MAGIC), server));
  }
  if ((current_percent < percent - SYSPARAM_NAMESERVER.balance_percent_))//ds当前已用容量占比 小于 集群容量平均占比-平衡因子
  {//加入迁移目标ds
    target.insert(server);
  }
  ......
}

bool elect_move_dest_ds(const std::set<ServerCollect*>& targets, const vector<ServerCollect*> & source, const ServerCollect* mover, ServerCollect** result)
{
  ......
  //计算所有的目标ds的weight，从小到大放入weights，weights是一个multimap
  DS_WEIGHT weights;
  StoreWeight < ReplicateSourceStrategy > store(strategy, weights);
  std::for_each(targets.begin(), targets.end(), store); 
  //将所有的源ds的网段放入existlan
  std::set < uint32_t > existlan;
  std::vector<ServerCollect*>::const_iterator s_iter = source.begin();
  for (; s_iter != source.end(); ++s_iter)
  {
    lan = Func::get_lan((*s_iter)->id(), SYSPARAM_NAMESERVER.group_mask_);
    existlan.insert(lan);
    ......
  }
  ServerCollect* first_result = NULL;
  DS_WEIGHT::const_iterator iter = weights.begin();
  while (iter != weights.end())
  {
    id = iter->second->id();
    lan = Func::get_lan(id, SYSPARAM_NAMESERVER.group_mask_);
    //des-ds和src-ds在同一网段
    if ((first_result == NULL) && (existlan.find(lan) == existlan.end())) 
      first_result = iter->second;
    //des-ds和src-ds在同一台机器上
    if ((*result == NULL)
        && (existlan.find(lan) == existlan.end())
        && (get_ip(id) == get_ip(mover->id()))) 
      *result = iter->second;
    //des-ds和src-ds在同网段并且在同台机器上,最优先使用
    if ((first_result != NULL) && (*result != NULL)) break;
    ++iter;
  }
  //des-ds和src-ds在同网段,优先使用
  if (*result == NULL)  *result = first_result;
  return (*result != NULL);
}
```
###问题
　　从代码的逻辑来看，如果扩容时，上了不同网段的DS，新的DS是无法正常被选为目标迁移DS的。但是向集群写数据、复制或紧急复制任务的情况下，新的DS会被优先选为目标DS接收数据。  
　　计算网段的逻辑如下：

```c++
uint32_t Func::get_lan(const uint64_t ipport, const uint32_t ipmask)
{
   IpAddr* adr = (IpAddr *) (&ipport);
   return (adr->ip_ & ipmask);
}
```
　　其中的ipmask来自`SYSPARAM_NAMESERVER.group_mask_`，对应TFS配置的group_mask，因此通过修改该配置可以将网段放宽。

