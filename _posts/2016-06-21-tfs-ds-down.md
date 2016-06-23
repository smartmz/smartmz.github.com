---
layout: post
title: "TFS的DataServer宕机一例排查"
date: 2016-06-21 15:45
keywords: tfs
description: TFS基本原理
categories: [Storage]
tags: [tfs]
group: archive
icon: globe
---
### 表现

线上TFS的DS突然宕机，DS日志看不到有用信息。在查询/var/log/messages的系统日志时，发现以下日志：

```
Nov 16 08:55:23 yf13401 kernel: dataserver[18232]: segfault at 00002aaae4b40020 rip 0000000000416c24 rsp 000000004a7beb10 error 4
```

<!-- more -->

### 解释日志

|日志|含义|
| --- | --- |
|Nov 16 08:55:23|日期|
|yfxxx|机器host|
|kernel: dataserver[18232]|进程和进程号|
|segfault at 00002aaae4b40020|访问的非法内存地址|
|rip 0000000000416c24|coredump时代码段指针|
|rsp 000000004a7beb10|当前栈指针|
|error 4|错误码，4就是100，bit2=1, bit1=0, bit0=0|

其中，

* bit2: 值为1表示是用户态程序内存访问越界，值为0表示是内核态程序内存访问越界
* bit1: 值为1表示是写操作导致内存访问越界，值为0表示是读操作导致内存访问越界
* bit0: 值为1表示没有足够的权限访问非法地址的内容，值为0表示访问的非法地址根本没有对应的页面，也就是无效地址

error 4就是说用户态程序内存访问越界，也就是DS的程序\$TFS_HOME/bin/dataserver使用了NULL指针，对应汇编代码的行数为416c24.

### 看源码

将dataserver程序反汇编`objdump -d dataserver  > dump`，在dump中查找地址416c24，得到下面片段：

```c
0000000000416bf6
<_ZNK3tfs10dataserver11IndexHandle11bucket_sizeEv>:
;;……
411462    <_ZNK3tfs10dataserver17MMapFileOperation12get_map_dataEv>
416c15:    48 89 45 f8    mov %rax, 0xfffffffffffffff8(%rbp)
;;与NULL比较
416c19:    48 83 7d f8 00    cmpq $0x0,0xfffffffffffffff8(%rbp)
;;如果相等，跳转到416c2a
416c1e:    74 0a    je 416c2a 
;;不相等，执行下面三句）
416c20:    48 8b 45 f8    mov 0xfffffffffffffff8(%rbp),%rax ;;注意
416c24:    8b 40 20    mov 0x20(%rax),%eax ;;注意
;;赋值
416c27:    89 45 f4    mov %eax,0xfffffffffffffff4(%rbp) ;;注意
416c2a:    8b 45 f4    mov 0xfffffffffffffff4(%rbp),%eax
416c2d:    c9    leaveq
416c2e:    c3    retq  
416c2f:    90    nop   
```
这里就是ds停止进程前coredump的地方。从这里可以跟踪到tfs源码的这里，IndexHandle类的一个方法：

```c
int32_t bucket_size() const
{
    int32_t size = -1;
    void* data = file_op_->get_map_data();
    if (NULL != data)
    {
        size = reinterpret_cast<IndexHeader*> (data)->bucket_size_;
    }    // 对应汇编中的三句需要注意的逻辑
    return size;
}
```
结合DS中断时的日志：

```
[2014-11-16 08:55:23] INFO  mmap_file.cpp:126 [1427966272] map file need remap. size: 40960, fd: 9088
[2014-11-16 08:55:23] INFO  mmap_file.cpp:149 [1427966272] mremap start. fd: 9088, now size: 40960, new size: 45056, old data: 0x2aaae4b40000
[2014-11-16 08:55:23] INFO  mmap_file.cpp:159 [1427966272] mremap success. fd: 9088, now size: 40960, new size: 45056, old data: 0x2aaae4b40000, new data: 0x2aaae5db0000
```

分析应该是mmap_file 的mremap调用之后发生了coredump.
通过查看MMapFile类的remap_file()方法，发现其中有这样的逻辑：

```c
int32_t new_size = size_ + mmap_file_option_.per_mmap_size_;
void* new_map_data = mremap(data_, size_, new_size, MREMAP_MAYMOVE);
TBSYS_LOG(INFO, "mremap success. fd: %d, now size: %d, new size: %d, old data: %p, new data: %p", fd_, size_, new_size,
      data_, new_map_data);
 
data_ = new_map_data;
size_ = new_size;
```

这里使用mremap将内存里的old data重新映射到new data中，而这个data在上面的bucket_size中由get_map_data()获取，在强制转换为IndexHeader*时coredump了。也就是说，remap后的data指针为空了。

### 为什么

什么情况下才能出现这种情况呢？再仔细查看一下coredump信息和DS的最后日志:
```
Coredump：segfault at 00002aaae4b40020
DS日志：old data: 0x2aaae4b40000, new data: 0x2aaae5db0000
```

看起来程序在new data地址上加了0x20作为new data的地址进行了访问，而new data在这里为NULL，实际上new data的地址实在0x2aaae5db0000。到这里，开始怀疑是不是DS的remap操作和读data操作在这里出现了并发问题。

之后，在[这里](http://blog.chinaunix.net/uid-20196318-id-3395833.html)找到了相关的论述：
 
> 如果在读写index时coredump（每个block的索引通过mmap直接映射到内存），则多是由于访问到还没有map或是已经remap的内存，出现过的场景包括：在启动加载block的过程中，block信息已加载但index还没有映射，这时直接使用映射内存地址就会出问题；修复的方法是：访问映射内存前要检查内存是否已经映射，同时在ds加载的过程中，会拒绝客户端的访问。由于某些读取index的调用没有严格加读锁保护，导致在读与写（需要更改index的数据，可能会remap index数据）并发时，读获取到remap前的内存指针，从而在访问时coredump。当发生这种情况时，通过info threads可以查看到哪些线程在并发读写。

发帖人zyd_cu是TFS团队的成员之一，该结论可靠，并且和自己的推测是一样的。

进一步分析，引起index改变的情况主要是index数据的更改，前面分析的迁移策略就会改变index数据的更改，频繁的迁移动作增加了并发问题的出现几率。



