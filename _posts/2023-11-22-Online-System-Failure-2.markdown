---
layout:     post
title:      记录一次线上故障2
subtitle:   You never lose. You either win or learn.
date:       2023/11/22 15:25
author:     "MuZhou"
header-img:  "img/2023/bg-11-22.jpeg"
catalog: true
show_in_home: true
tags:
- 分享
- 线上问题
---
之前遗留下了两个问题还没答案：

Q3：为什么其他接口的RT也暴涨了？  
Q5：为什么ECS的内存会飙升到100%？

这俩问题有点复杂，写了个[demo](https://github.com/muzhou1994/online-failure-demo)，经过各种实验之后，终于搞明白了原因。

#### 从JDK21和虚拟线程说起
由于各种原因，这个项目使用了最新版的JDK21和虚拟线程的特性，并使用G1垃圾回收器。     
本文会用到以下这些相关的术语，如果不太熟悉，可以参阅网上其他文章。
- 虚拟线程
- 平台线程(Platform Thread)
- 载体线程(carrier)
- 阻塞、卸载
- 挂起        

有一些阻塞操作会让虚拟线程无法从载体线程上卸载，这个现象叫挂起（Pinned）。目前有以下两种明确的情况会发生挂起：
> - The virtual thread runs code inside a synchronized block or method
> - The virtual thread runs a native method or a foreign function (see Foreign Function and Memory API)

之前提到，故障最直接的原因是接入的阿里云验证码服务并发数不够，而这个SDK里使用了ApacheHttpClient。
查看源码发现，在获取连接的时候果然使用了synchronized。
![synchronized锁](/img/2023/synchronized.png)
到此，把整个事件串一串，就可以回答"为什么其他接口的RT也暴涨了？"。

活动开始时，大量请求涌入，调用验证服务的记为请求A，其他接口计为请求B、C、D......   
使用虚拟线程后，每个请求都由单独的虚拟线程处理，如下图：
![虚拟线程处理](/img/2023/虚拟线程处理.png)
A等待获取连接，最多会等15s，对应到图上，也就是虚拟线程A1、A2、A3无法从载体线程1、2、n卸载，会持续卡住载体线程15s。    
B遇到组塞操作会让出载体线程，比如虚拟线程B遇到redis读写、MySQL读写等情况时，会让出载体线程2。    
载体线程2会再取一个虚拟线程装载、执行，比如图中虚拟线程A4。     
虚拟线程B的处理并没有完成，但是让出载体线程后，B需要等待下一次被某个空闲载体线程装载、恢复执行，而基本所有载体线程都被请求A卡住15s，B的等待时间将会变得完全不可控。  

这就是其他接口RT也暴涨的原因。

#### 再说内存暴涨
对于"为什么ECS的内存会飙升到100%？"这个问题，之前推测跟日志打印有关，其实并不是。  

这个问题隐含了两个子问题：
- 为什么内存会涨？
  - JVM的启动参数设置了-Xms和-Xmx，并且值相同，理论上内存使用应该是一条平稳的直线，不会上涨。
- 为什么会涨到100%？
  - JVM的-Xmx值比ECS的内存小4G，其实预留了4G内存没用，不应该涨到100%。

之前以为-Xms 4G就会直接使用系统4G的内存，-Xmx和-Xms设置成一样的值内存使用率就不会上升。   
一般情况下确实是这样，因为大多数服务流量和内存匹配，用户的请求在很短时间内就能让Java服务的内存涨到-Xms值。  
而这个项目日常的流量是活动期间流量的十分之一，甚至更低。为了支持活动期间的流量，所以给Java分配的内存一直比较大，而日常流量用到的内存其实远没到-Xms。
![Java重启&ECS内存缓慢增长](/img/2023/Java重启&ECS内存缓慢增长.png)
上图是给一个小流量的服务分配了比较大的-Xms（接近机器内存的90%），重启服务之后，可以看到内存使用率随时间慢慢增长，最终稳定在60%左右，并没有到达90%。 

看gc日志，此时会以比较低的频率执行YoungGC，推测是因为G1MaxNewSizePercent默认值为60%，而新生代内存不足时会触发YoungGC，GC完成后内存就又充足了，所以内存使用率不会再涨。
![有效YoungGC](/img/2023/有效YoungGC.png)

活动开始后，由于大量请求卡住无法处理完成，RT也因此暴涨。而请求所携带的上下文（比如ThreadLoacal数据）、处理请求需要的临时变量等等，都无法释放，随着请求堆积，内存占用越来越大，但GC却无法回收。
![无效YoungGC](/img/2023/无效YoungGC.png)
可以看到，GC的频率变高了，两次GC只间隔了十多秒，回收的内存大大减少，而GC耗费的时间大大增加。

到此，内存暴涨的原因已经清晰明了。

至于为何-Xmx预留了4G内存，内存使用率却到了100%，首先是系统本身会占用部分内存，比如32G内存的规格，实际可用内存大概只有30G出头。 
![ECS实际内存小](/img/2023/ECS实际内存小.png)

其次机器上还运行了一些别的服务，比如日志收集等。
如下图，线上服务32G内存的机器，停掉Java服务后，free的内存只有27G多。   
![30G内存](/img/2023/30G内存.png)

一般Xmx设置为系统内存的50-75%，这次预留4G内存相对来说太少了。

至此，这个故障的各个细节都算理清楚了，期间做了很多实验，尝试回答这些问题：
- 怎么复现问题？
    - jmeter开启500线程，运行demo项目，使用虚拟线程，10个载体线程，控制最多5个并发，最大等待wait 15s，就可以复现。
- 相同配置，如果不用虚拟线程，使用10个tomcat的普通线程，还会发生故障吗？
- 相同配置，不用虚拟线程，500个tomcat普通线程，故障还会发生吗？
- 只减少maxWait时间，使用虚拟线程，载体线程10个，5个并发，最大等待1s，故障会发生吗？
- 只调大并发，使用虚拟线程，载体线程10个，500个并发，最大等待15s，故障会发生吗？

权当留下的思考题了，之后有空再仔细记录一下。

参考：
1. [两万字的Java 虚拟线程终极指南](https://juejin.cn/post/7282666367236276224)
2. [Virtual Threads](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html#GUID-DC4306FC-D6C1-4BCC-AECE-48C32C1A8DAA)
3. [why-does-the-jvm-consume-less-memory-than-xms-specified](https://stackoverflow.com/questions/12108706/why-does-the-jvm-consume-less-memory-than-xms-specified)