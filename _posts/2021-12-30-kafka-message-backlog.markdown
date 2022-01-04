---
layout:     post
title:      糟了，线上kafka消息积压了
subtitle:   Luck is what happens when preparation meets opportunity.
date:       2021/12/30 10:59
author:     "MuZhou"
header-img:  "img/2021/bg-kafka-message-backlog-header.jpg"
catalog: true
tags:
    - kafka
---

某天，深夜突然收到线上报警，某个业务的kafka消息消费发生堆积了。
>【阿里云】尊敬的用户xxx , 22:34 您的消息队列 Kafka实例instanceId=xxxxx，consumerGroup=xxxxx   消息堆积量当前值超过10000Count, 请登录云监控关注

由于发生在深夜，很快确定对应时间点没有进行任何线上变更，流量也并没有突增，consumer业务服务也正常，那是怎么回事呢？     

之后在查看kafka的监控信息时发现了异常，rebalance 次数明显增多，大概一分钟一次。        

#### rebalance是什么
kafka有几个关键术语，
- topic
- partition
- producer
- consumer
- consumer group
- broker         

这些术语都是kafka的基础知识，略过不再解释。    
我们都知道，一个partition只能被分配给一个consumer消费。    
比如现在有1、2、3、4四个partition，有ab两个consumer，那么分配关系可以是a消费1和2，b消费3和4。

| consumer | partition 1 | partition 2 | partition 3| partition 4|
| :--: | :--: | :--: |  :--: | :--: |  
| consumer a| ✅ |✅|||
|consumer b|||✅|✅| 

当然也可以是a消费1和3，b消费2和4。    

这个分配关系不是一成不变的，某些情况下会被打破。    
比如b挂了，那b消费的partition 3和partition 4就需要分配给consumer a。     
比如新加入了consumer c，那就需要从四个partition里分些给c消费。     

当之前的分配规则被打破，所有consumer要重新确认自己需要消费哪些partition，这个重新达成共识的过程就是rebalance。  

#### 哪些情况会发生rebalance
分配关系里有两方，一方是consumer，一方是需要被消费的partition。任意一方的数量发生变化时，都会导致rebalance。

一个consumer group可以订阅多个topic，订阅时支持模糊匹配。topic数量发生变化时，需要被消费的partition也会发生变化，所以也会发生rebalance。

即rebalance会发生在：
- consumer数量变化时
- consumer group订阅的topic数量变化时
- topic内partition数量变化时

#### consumer是怎么被判定为活跃/不活跃的？
这里引入一个新角色，consumer group coordinator。      
coordinator其实就是一个broker，consumer group里的每个consumer会向它发心跳。如果超过一定时间没发心跳，就会被判定为不活跃，coordinator会触发rebalance。        

不通版本的kafka心跳行为有点不一样。在0.10.1之前，consumer是单线程的，拉取消息(poll)或者提交偏移量时发送心跳。

```java

    /**
     * Do one round of polling. In addition to checking for new data, this does any needed
     * heart-beating, auto-commits, and offset updates.
     * @param timeout The maximum time to block in the underlying poll
     * @return The fetched records (may be empty)
     */
    private Map<TopicPartition, List<ConsumerRecord<K, V>>> pollOnce(long timeout) {
        //......省略.......

        long now = time.milliseconds();

        // execute delayed tasks (e.g. autocommits and heartbeats) prior to fetching records
        client.executeDelayedTasks(now);

        //......省略.......
    }
```
```java
    public void commitOffsetsAsync(final Map<TopicPartition, OffsetAndMetadata> offsets, OffsetCommitCallback callback) {
        //......省略.......

        // ensure the commit has a chance to be transmitted (without blocking on its completion).
        // Note that commits are treated as heartbeats by the coordinator, so there is no need to
        // explicitly allow heartbeats through delayed task execution.
        client.pollNoWakeup();
    }
```
```java
    /**
     * Initialize the coordination manager.
     */
    public AbstractCoordinator(ConsumerNetworkClient client,
                               String groupId,
                               int sessionTimeoutMs,
                               int heartbeatIntervalMs,
                               Metrics metrics,
                               String metricGrpPrefix,
                               Time time,
                               long retryBackoffMs) {
        //......省略.......
        this.heartbeat = new Heartbeat(this.sessionTimeoutMs, heartbeatIntervalMs, time.milliseconds());
        this.heartbeatTask = new HeartbeatTask();
        //......省略.......
    }

    public Heartbeat(long timeout,
                 long interval,
                 long now) {
        if (interval >= timeout)
            throw new IllegalArgumentException("Heartbeat must be set lower than the session timeout");

        this.timeout = timeout;
        this.interval = interval;
        this.lastSessionReset = now;
    }
```
以上代码为kafka-client 0.10.0.0版，可以看到heartbeat主要有两个参数：session.timeout.ms、heartbeat.interval.ms。    

在0.10.1版本后，心跳由单独的HeartbeatThread线程执行。
```java
    private synchronized void startHeartbeatThreadIfNeeded() {
        if (heartbeatThread == null) {
            heartbeatThread = new HeartbeatThread();
            heartbeatThread.start();
        }
    }
```
```java
    public Heartbeat(long sessionTimeout,
                     long heartbeatInterval,
                     long maxPollInterval,
                     long retryBackoffMs) {
        if (heartbeatInterval >= sessionTimeout)
            throw new IllegalArgumentException("Heartbeat must be set lower than the session timeout");

        this.sessionTimeout = sessionTimeout;
        this.heartbeatInterval = heartbeatInterval;
        this.maxPollInterval = maxPollInterval;
        this.retryBackoffMs = retryBackoffMs;
    }
```

以上代码为kafka-client 0.10.1.1版，可以看到heartbeat新增了一个参数：max.poll.interval.ms。
>The new Java Consumer now supports heartbeating from a background thread. There is a new configuration max.poll.interval.ms which controls the maximum time between poll invocations before the consumer will proactively leave the group (5 minutes by default).

根据官方说明，如果两次poll间隔超过了max.poll.interval.ms，consumer就会主动离开组，coordinator会发起rebalance。

#### 本次故障到底是怎么发生的
我们使用的kafka-client版本是0.10.0.0，producer由于某些原因提交了一些格外大的数据，consumer消费的时长陡增，大于session.timeout.ms，发生了频繁rebalance。

解决这个问题，可以调大session.timeout.ms，可以升级kafka-client到0.10.1以上。

#### 其他遗留问题
- rebalance期间会丢消息吗，会重复消费吗


参考：
1. [一篇文章带你快速搞定Kafka术语](https://time.geekbang.org/column/article/99318)
2. [消费者组重平衡能避免吗？](https://time.geekbang.org/column/article/105737)
3. [What is the difference in Kafka between a Consumer Group Coordinator and a Consumer Group Leader?](https://stackoverflow.com/questions/42015158/what-is-the-difference-in-kafka-between-a-consumer-group-coordinator-and-a-consu)
4. [使用消息队列Kafka版时消费客户端频繁出现Rebalance](https://help.aliyun.com/document_detail/154454.html?spm=a2c4g.11186623.0.0.631c6e91m4hwqs)
5. [Notable changes in 0.10.1.0](https://kafka.apache.org/0101/documentation.html#upgrade_1010_notable)