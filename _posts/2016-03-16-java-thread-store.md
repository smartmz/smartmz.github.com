---
layout: post
title: "JAVA线程转储"
date: 2016-03-16 15:00
keywords: java,thread
description: Java线程转储的相关知识
categories: [JAVA]
tags: [java, thread]
group: archive
icon: globe
---

<!-- more -->

　　Java的web服务在线上运行过程中会遇到的一些常见问题现象有系统负载过高、系统吞吐率低、服务响应慢等。如果要定位这些现象的产生原因，由于复现难度大，只能在生产环境中进行Java虚拟机的线程转储，才能快速定位问题。
　　在讨论转储手段之前，先来回顾一下Java线程的基础知识。

##JAVA虚拟机线程状态
java.lang.Thread.State枚举了线程的状态：

```java
/**
 * A thread can be in only one state at a given point in time.
 * These states are virtual machine states which do not reflect
 * any operating system thread states.
 */
public enum State {
        /**
         * Thread state for a thread which has not yet started.
         */
        NEW,

        /**
         * Thread state for a runnable thread.  A thread in the runnable
         * state is executing in the Java virtual machine but it may
         * be waiting for other resources from the operating system
         * such as processor.
         */
        RUNNABLE,

        /**
         * Thread state for a thread blocked waiting for a monitor lock.
         * A thread in the blocked state is waiting for a monitor lock
         * to enter a synchronized block/method or reenter a synchronized
         * block/method after calling Object.wait.
         */
        BLOCKED,

        /**
         * Thread state for a waiting thread.
         * A thread is in the waiting state due to calling one of the
         * following methods:
         *   Object.wait() with no timeout
         *   Thread.join() with no timeout
         *   LockSupport.park()
         *
         * A thread in the waiting state is waiting for another thread to
         * perform a particular action.
         *
         * For example, a thread that has called <tt>Object.wait() on an
         * object is waiting for another thread to call Object.notify()
         * or Object.notifyAll() on that object. A thread that has called
         * Thread.join() is waiting for a specified thread to terminate.
         */
        WAITING,

        /**
         * Thread state for a waiting thread with a specified waiting time.
         * A thread is in the timed waiting state due to calling one of
         * the following methods with a specified positive waiting time:
         *   sleep Thread.sleep
         *   Object.wait() with timeout
         *   Thread.join() with timeout
         *   LockSupport.parkNanos()
         *   LockSupport.parkUntil()
         */
        TIMED_WAITING,

        /**
         * Thread state for a terminated thread.
         * The thread has completed execution.
         */
        TERMINATED;
    }
```
　　代码注释很详细，可以对照线程状态转移图来理解。

<center>![JAVA线程状态转移](http://ww1.sinaimg.cn/bmiddle/a8484315jw1f1yrr002srj20j60co0tw.jpg)</center><br/><center>
<font size=2>图1 JAVA线程状态转移</font></center>

