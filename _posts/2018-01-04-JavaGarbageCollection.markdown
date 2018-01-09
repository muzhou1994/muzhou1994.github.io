---
layout:     post
title:      "Java垃圾回收（一）"
subtitle:   "落落者难合亦难分。欣欣者易亲亦易散。"
date:       18/1/4 下午12:45
author:     "MuZhou"
header-img:  "img/2018/bg-JavaGarbageCollection-header.jpg"
catalog: true
tags:
    - Java
    - gc
---

我们知道，在Java中内存管理是委托给虚拟机的（JVM），不像C或C++，需要手动申请及释放内存。
内存管理就涉及到内存分配和垃圾回收两个主题，前者可以参阅Java内存模型及内存分配原则，后者是本文的主要内容。
<br>
JVM在进行垃圾回收时，要解决以下三个问题：

- 哪些对象需要回收，即判断对象存活还是死亡。
- 什么时候进行回收，即垃圾回收的触发条件。
- 如何进行回收，即垃圾回收算法及垃圾收集器。

## 哪些对象需要回收
确定哪些对象还存活，也就是确定对象是否还被引用，最简单的做法就是为每个对象加一个引用计数器，被引用时计数器加一，
引用失效时计数器减一，为零时即认定对象死亡,这就是**[引用计数](https://zh.wikipedia
.org/wiki/%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0)**。

引用计数算法实现简单，判定效率高，但无法解决对象间循环引用的问题，因而在JVM中不使用引用计数来判定对象是否存活，
而是使用**可达性分析法**进行判断。

可达性分析法是选定一批对象作为GC Roots，以此为起点向下搜索，如果一个对象与GC Roots之间
不可达，则该对象可以被回收,有效的解决了循环引用的问题。
<br>

Java中GC Roots般包含以下几种对象：

- 虚拟机栈中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中JNI引用的对象

## 什么时候进行回收
垃圾回收按照回收区域一般分为Partial GC和Full GC。

- Partial GC：不回收整个GC堆
    - Minor GC: 当年轻代eden区满时触发。
    - Major GC: 触发Major GC的情况有两种：
        1. CMS中老年代使用量超过设置的值（-XX:CMSInitiatingOccupancyFraction=<N>)
        2. CMS中基于近期历史，维护一个预估值，预估离下一次老年代将要耗尽所剩的时间，在耗尽前进行一次Major GC。
    - Mixed GC: G1收集器独有，回收新生代和部分老年代。
- Full GC:
1. 永久代空间不足。
2. System.gc()
3. CMS中发生Concurrent Mode Failure后会使用serial collector进行一次Full GC。
4. 老年代空间不足。

（老年代空间不足的情况如图4种）
![内存分配过程](/img/2018/JavaGarbageCollection-memory-allocation.png)

## 如何进行回收

### 垃圾回收算法
- 标记清除: 标记对象"已死"后，清除"已死"对象。
     - 弊端: 产生不连续碎片，标记及清除两个过程效率都不高。
- 复制：内存划分为大小相等两块，每次使用一块，复制还存活的对象到另一块，再一次性清除已使用的内存块。
     - 弊端: 可用内存缩小为一半，对象存活率高时复制操作很多。
- 标记整理: 类同标记清除，标记后将存活对象移向一端，然后清理端边界之外的内存。
     - 弊端: 需要移动对象。
- 分代算法: 根据对象存活周期将n内存划分为几块，根据各代特点选择最优算法。

Java中垃圾收集器基本都使用了分代算法，将内存分为新生代和老年代。对于新生代，一般采用复制算法，将新生代内存分为一块eden区，
两块同等大小Survivor区，gc时将存活对象复制到闲置的一块Survivor区。对于老年代，一般采用标记整理或标记清除算法。

### 垃圾收集器
在HotSpot JVM中实现了以下四种垃圾收集器：

- Serial Collector：
- Parallel Collector
- CMS (Concurrent Mark Sweep) Collector
- G1 (Garbage-First) Collector

如果按回收区域划分，可以将垃圾收集器分为以下七种：
![Our Collectors](/img/2018/JavaGarbageCollection-OurCollectors.png)

<table>
  <thead>
    <tr>
      <th>垃圾收集器</th>
      <th>区域</th>
      <th>算法</th>
      <th>并行</th>
      <th>是否新特性</th>
      <th>STW</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Serial</td>
      <td>新生代</td>
      <td>复制算法</td>
      <td>单线程</td>
      <td>旧</td>
      <td>是</td>
    </tr>
    <tr>
      <td>ParNew</td>
      <td>新生代</td>
      <td>复制算法</td>
      <td>多线程</td>
      <td>旧</td>
      <td>是</td>
    </tr>
    <tr>
      <td>Parallel Scavenge</td>
      <td>新生代</td>
      <td>复制算法</td>
      <td>多线程</td>
      <td>旧</td>
      <td>是</td>
    </tr>
    <tr>
      <td>Serial Old</td>
      <td>老年代</td>
      <td>标记整理</td>
      <td>单线程</td>
      <td>旧</td>
      <td>是</td>
    </tr>
    <tr>
      <td>Parallel Old</td>
      <td>老年代</td>
      <td>标记整理</td>
      <td>多线程</td>
      <td>JDK1.6</td>
      <td>是</td>
    </tr>
    <tr>
      <td>CMS</td>
      <td>老年代</td>
      <td>标记清除</td>
      <td>多线程</td>
      <td>JDK1.5</td>
      <td>是</td>
    </tr>
    <tr>
      <td>G1</td>
      <td>整体</td>
      <td><li>整体：标记整理</li> <li>局部：复制算法</li></td>
      <td>多线程</td>
      <td>JDK1.7</td>
      <td>是</td>
    </tr>
  </tbody>
</table>


![关系](/img/2018/JavaGarbageCollection-CollectorsWithCommandLine.png)

参考资料<br>

1. [引用计数](https://zh.wikipedia.org/wiki/%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0)
2. [垃圾回收](https://zh.wikipedia.org/wiki/%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6_
(%E8%A8%88%E7%AE%97%E6%A9%9F%E7%A7%91%E5%AD%B8))
3. 《深入理解Java虚拟机》
4. [Java Garbage Collection Basics](http://www.oracle
.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html)
5. [Concurrent Mark Sweep (CMS) Collector](https://docs.oracle
.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html)
6. [Our Collectors](https://blogs.oracle.com/jonthecollector/our-collectors)