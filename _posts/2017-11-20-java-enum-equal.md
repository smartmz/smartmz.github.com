
---
layout: post
title: "enum怎么判等不踩坑"
date: 2017-11-20 15:00
keywords: java,thread
description: enum-java基础
categories: [JAVA]
tags: [java, enum]
group: archive
icon: globe
---

<!-- more -->

之前写需求，在enum判等时踩坑，一方面是开发没有上手，功能也简单没太在意，另一方面也发现老代码在这个地方的处理上不规范（或者说习惯不好），容易出错，不容易review出来，所以大概总结一下。

## 一句话总结

enum使用equals判等，左右类型不一致导致结果恒为false.

## 现场描述

代码中enum判等的用法都是enum和Integer混用，比如

```java
ReleaseBatchStatusEnum.WAITING.getValue().equals(batch.getReleaseStatus()))
```

然后掉坑的代码：

```java
ReleaseBatchStatusEnum.SUCCESS.equals(batch.getReleaseStatus()))
```

getValue()丢了，主要是主观大意，然而，，，，简单的一段错误代码竟然闯过了IDE提示、编译检查、运行时异常。

## 为什么层层破防

原理简单，enum的equals()实现是Enum类的equals()实现（语法糖嘛~~）：

```java
 public final boolean equals(Object var1) {
        return this == var1;
 }
```

jdk的槽点出现了：Object做参数, 编译检查、运行时检查统统掩盖了真相。。。那么IDE的提示呢？

其实IDEA默认配置的时候是这样的：

![](http://wx2.sinaimg.cn/mw690/98c50222gy1fgm87vmnuoj20ui0a0gnt.jpg)

给了一个WARN的弱提醒，但是自己手欠把这个提醒关了。

## 如何避免

避免的问题需要探讨一下enum的equals()方法和\=\=运算。这里只探讨equals()方法和\=\=运算的编译检查和运行时异常问题。

### PART 1

来看一组测试：

```java
public enum ExSimple {
    ONE(1),
    TWO(2);
    private int value;
    ExSimple(int value){
        this.value = value;
    }
    public int getValue(){
        return this.value;
    }	
}

@Test
public void enumTest() throws Exception {
    ExSimple exSimple = ExSimple.ONE;

    Assert.assertFalse(exSimple.equals(1));     //编译检查通过，运行时无异常，最怕这种！！！
    //Assert.assertTrue(exSimple1 == 1);          编译检查不通过
    //ExSimple exSimple2 = 1;                     编译检查不通过
}
```

看到了吧，equals()助我踩坑到底；而\=\=运算可以在编译期间对左右进行类型检查，确保不会出现 不同类型判等 的情况。实际上用==运算还有一个小小的好处是 性能会更好（\=\=看地址equals看内容），不过与本文没什么关系。

### PART 2

就想用equals()怎么办？没办法直接在enum中重载equals方法，更没办法通过继承重写equals方法，原因看一眼上面ExSimple反编译以后的样子就知道了：

```java
>>> ./jad -p ExSimple.class
public final class ExSimple extends Enum {
    public static ExSimple[] values() {
        return (ExSimple[])$VALUES.clone();
    }
    public static ExSimple valueOf(String s) {
        return (ExSimple)Enum.valueOf(ExSimple, s);
    }
    private ExSimple(String s, int i, int j) {
        super(s, i);
        value = j;
    }
    public int getValue() {
        return value;
    }
    public static final ExSimple ONE;
    public static final ExSimple TWO;
    private int value;
    private static final ExSimple $VALUES[];
    static {
        ONE = new ExSimple("ONE", 0, 1);
        TWO = new ExSimple("TWO", 1, 2);
        $VALUES = (new ExSimple[] { ONE, TWO });
    }
}
```
ExSimple是继承了Enum的final类。 （关心去的话详细说明可以看下[这里](http://docs.oracle.com/javase/specs/jls/se8/html/jls-8.html#jls-8.9)，和本文关系不大。）

虽然不能把enum的equals()怎么样，但是我们可以这样：

```java
public enum ExWithEquals {
    ONE(1);
    TWO(2);
    int n;
    ExWithEquals(int n) {
        this.n = n;
    }
    public final boolean equalsSupportCheck(ExWithEquals exWithEquals) {
        return this == exWithEquals;
    }
}
```

这是以前工作中看到的一个办法，添加了一个支持编译期间可以帮我进行类型检查报错的equals方法（习惯性的说我们重写了equals()方法，据说凌乱了很多人），也算是曲线救国，唯一需要的是 enum的使用者多一点点好奇心：

![](http://wx3.sinaimg.cn/mw690/98c50222gy1fgmbj7l134j214a066ac4.jpg)

这样的话，就好了：

```java
//Assert.assertTrue(exWithEquals == 1);                 编译检查不通过
//Assert.assertTrue(exWithEquals.equalsWithCheck(1));   编译检查不通过
Assert.assertTrue(exWithEquals.equalsWithCheck(ExWithEquals.ONE));
```

## 结论

1. 请不要混合使用int/Integer和enum，至少把这种情况控制在Dao以内，少用getValue()之类的方法
1. 在enum的判等中，绝大多数情况下，请**使用 \=\=**，它编译期和运行期都是安全的
1. 确实想用enum equals判等，尤其是 int/Integer和enum混用的情况下，
    1. 一定要保持清醒，别忘了getValue()
    1. 按照 PART 2 来做，形成习惯，新人摸代码的时候一般会注意到的
    1. 管好自己的IDE，建议把equals()这个提示从WARN改成ERROR，这个在这里：
        ![](http://wx2.sinaimg.cn/mw690/98c50222gy1fgm87x0zptj210i042jsw.jpg)

### 题外话

enum的equals方法真的一无是处了么？其实：

```java
enum ExNull { ONE };

ExNull nothing = null;
if (nothing == ExNull.ONE) {};      // 编译检查通过，运行无异常！！！==运算不是万能的
if (nothing.equals(ExNull.ONE)) {}; // 运行抛出空指针
```



