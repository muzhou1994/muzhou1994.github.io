---
layout:     post
title:      真实业务场景：计算用户击败了多少人
subtitle:   What you get by achieving your goals is not as important as what you become by achieving your goals.
date:       2024/1/18 17:50
author:     "MuZhou"
header-img:  "img/2024/bg-01-18.jpeg"
catalog: true
hide_in_home: false
tags:
- 面试
- 分享
---

最近遇到一个业务需求，简单的crud解决不了，感觉有点意思，纪录一下。     
具体业务场景是这样的：
> 每个用户有个分数，根据分数算一下用户击败了多少人，比如 “击败了99%的同路人”。   
> 分数每周日0点统一更新，且分布在[0, 70_0000]之间，近似正态分布。

这问题乍一看挺简单的，根据分数排个序，自然就能算出答案。    

但这个功能至少有3000w用户，并且随着时间流逝，用户数会越来越多。  
那么问题来了，如果用排序，这计算的工作交给MySQL还是Java服务呢？    

凭多年老后端的经验，千万级的数据量下，交给谁都不太行。  
不过还是测试一下，拿一下定量的数据。
本文会用到：
- MySQL 5.6
- MacBook Pro 2021，16G
- 一张user_score表，随机写入3000w条数据
```sql
CREATE TABLE `user_score` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `user_id` bigint(20) NOT NULL,
  `score` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `user_id` (`user_id`),
  KEY `score` (`score`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### MySQL计算
MySQL计算的方案很简单，比如用户分数是1000，那就执行以下SQL，用cnt1 / cnt2就能算出答案。
```sql
mysql> select count(*) as cnt1 from user_score where score < 1000;
+-------+
| cnt1  |
+-------+
| 43008 |
+-------+
1 row in set (0.01 sec)

mysql> select count(*) as cnt2 from user_score;
+----------+
| cnt2     |
+----------+
| 30000000 |
+----------+
1 row in set (3.88 sec)
```
肉眼可见的，cnt2耗时3.88s，这对线上业务来说无法接受。   
细心的朋友会发现，cnt2可以一周只算一次，加个缓存就能解决问题。但即便cnt2加上缓存，这个方案就可行了吗？cnt1会不会有问题？   

可以看到上面的cnt1计算非常快，0.01 sec，但这只是命中4.3w行的情况。    
如果命中100w行、500w行、1000w行、3000w行，速度还会这么快吗？
```sql
mysql> select count(*) as cnt1 from user_score where score < 25000;
+---------+
| cnt1    |
+---------+
| 1071142 |
+---------+
1 row in set (0.19 sec)

mysql> select count(*) as cnt1 from user_score where score < 115000;
+---------+
| cnt1    |
+---------+
| 4928825 |
+---------+
1 row in set (0.78 sec)

mysql> select count(*) as cnt1 from user_score where score < 240000;
+----------+
| cnt1     |
+----------+
| 10287614 |
+----------+
1 row in set (1.70 sec)

mysql> select count(*) as cnt1 from user_score where score < 700000;
+----------+
| cnt1     |
+----------+
| 30000000 |
+----------+
1 row in set (5.68 sec)
```
可以看到，随着分数越来越高，cnt1的计算耗时越来越大，3000w行时甚至需要5秒以上。活跃的用户一般分数会更高，他们的计算也就更耗时，实时去计算这个值不太可行。  

有人可能会想，那给cnt1也加个缓存，每个用户只需要在当周第一次请求的时计算一次，这样可行吗？   
那当然是不行。就算只计算一次，但每次计算耗时都是秒级，QPS稍微高一点（都不需要多高，可能个位数），这种慢SQL就会急速拖垮数据库。  
好奇的朋友可以点这里，[慢SQL是如何拖垮数据库的](https://zhuanlan.zhihu.com/p/626602259)。


至此，这个方案彻底宣告失败。

### Java计算
Java计算比MySQL计算更不可行，首先，把数据读取到Java服务里就是一个很大的开销。  
3000w数据不可能一次性读出来，分批读取，每批1w条也要读3000次，1次算1ms，就需要3s了。  
虽然这个方案十分不靠谱，但我们也测试一下耗时。
```sql
select * from user_score where id > #{prevId} order by id asc limit 10000
```

```java
    public void test() {
        long start = System.currentTimeMillis();
        int cnt1 = 0, cnt2 = 0, prevId = 0;
        int targetScore = 1000;//随便写的值
        List<UserScore> list = mapper.queryUserScore(prevId);
        while (!CollectionUtils.isEmpty(list)) {
            for (UserScore r : list) {
                cnt2++;
                if (r.score < targetScore) {
                    cnt1++;
                }
            }
            prevId = list.get(list.size() - 1).id;
            list = mapper.queryUserScore(prevId);
        }
        System.out.println("cnt1: " + cnt1 + ", cnt2: " + cnt2);
        System.out.println("cost: " + (System.currentTimeMillis() - start));
    }
```
```
cnt1: 43008, cnt2: 30000000
cost: 38153
```
看得出来确实完全不可行。  
这个方案唯一的好处是，无论用户分数多少，这个计算耗时都很稳定，不会像MySQL计算那样，分数越小算得越快🐶🐶🐶

### 离线计算
简单的实时计算方案都不可行，只能试试提前离线算好了。  
借鉴Java计算的方案和[计数排序](https://zh.wikipedia.org/wiki/%E8%AE%A1%E6%95%B0%E6%8E%92%E5%BA%8F)的思想，先把每个分数有多少人统计出来，这样查询用户打败了多少人时，直接从统计表就能得到结果。
新增一个统计表：
```sql
CREATE TABLE `user_score_aggregation` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `score` int(11) NOT NULL,
  `user_cnt` int(11) NOT NULL DEFAULT '0' COMMENT '分数等于score的用户数',
  `exceed_user_cnt` int(11) NOT NULL DEFAULT '0' COMMENT '分数小于score的用户数',
  PRIMARY KEY (`id`),
  UNIQUE KEY `score` (`score`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
#### 离线方案一
- 循环遍历user_score表，每次取1w条数据，计算(score,user_cnt) map，3000w条数据需要循环3000次
- 将分数排序，依次计算每个分数的exceed_user_cnt
- 将计算的结果1000一批插入聚合表user_score_aggregation

```java
    public void offlineCalc1() {
        long start = System.currentTimeMillis();
        int prevId = 0;
        Map<Integer, Integer> scoreCntMap = new HashMap<>();
        List<UserScore> list = mapper.queryUserScore(prevId);
        while (!CollectionUtils.isEmpty(list)) {
            for (UserScore r : list) {
                scoreCntMap.put(r.score, scoreCntMap.getOrDefault(r.score, 0) + 1);
            }
            prevId = list.get(list.size() - 1).id;
            list = mapper.queryUserScore(prevId);
        }
        List<UserScoreAggregation> data = new ArrayList<>();
        int exceedUserCnt = 0;
        List<Integer> sortedScoreList = scoreCntMap.keySet().stream().sorted().collect(Collectors.toList());
        for (int score : sortedScoreList) {
            int userCnt = scoreCntMap.get(score);
            UserScoreAggregation aggregation = new UserScoreAggregation();
            aggregation.score = score;
            aggregation.userCnt = userCnt;
            aggregation.exceedUserCnt = exceedUserCnt;
            data.add(aggregation);
            if (data.size() >= 1000) {
                mapper.batchInsertAggregation(data);
                data.clear();
            }
            exceedUserCnt += userCnt;
        }
        System.out.println("cost: " + (System.currentTimeMillis() - start));
    }
```
```
cost: 59474
```
大概要花1分钟，【击败了x%的同路人】这个数据需要延迟1分钟更新，听上去还可以接受，但是能不能更快一点呢？

当然是可以的。我们其实可以把每个分数有多少人交给MySQL计算，这样就不用从MySQL读取3000w行数据到Java里了。

#### 离线方案二
- 从0分到70w分，每1000一个区间，执行下面的SQL查询对应分数有多少人
- 依次计算exceed_user_cnt
- 将计算的结果插入聚合表user_score_aggregation   

```sql
select score, count(*) as user_cnt from user_score 
where score >= #{start} and score < {end} 
group by score 
order by score asc
```

```java
    public void offlineCalc2() {
        long start = System.currentTimeMillis();
        int step = 1000, maxScore = 70_0000;
        int exceedUserCnt = 0;
        for (int i = 0; i <= maxScore; i += step) {
            List<ScoreCnt> list = mapper.aggregateScoreCnt(i, i + step);
            if (CollectionUtils.isEmpty(list)) {
                continue;
            }
            List<UserScoreAggregation> data = new ArrayList<>();
            for (ScoreCnt x : list) {
                UserScoreAggregation aggregation = new UserScoreAggregation();
                aggregation.score = x.score;
                aggregation.userCnt = x.userCnt;
                aggregation.exceedUserCnt = exceedUserCnt;
                exceedUserCnt += x.userCnt;
                data.add(aggregation);
            }
            mapper.batchInsertAggregation(data);
        }
        System.out.println("cost: " + (System.currentTimeMillis() - start));
    }
```
```
cost: 20594
```
耗时提升很明显，而且真实环境里可以通过最低分和最高分进行剪枝，可以算得更快。      
这个方案的好处在于，只要分数区间[0, 70_0000]不变，它的计算开销也变化不大。    
离线方案一由于需要全表遍历&取回数据到Java服务，如果用户数增加到6kw，那么耗时基本上也会翻倍。    

但离线方案2也有缺点，如果分数分布不均匀，就会造成慢SQL，拉低数据库性能。     
假设有1000w人都在2000-3000分之间，那么算这个区间每个分数的人数时，SQL的执行速度将会在秒级。
```sql
mysql> update user_score set score = 2001 where id < 10000000;

mysql> select score, count(*) as user_cnt from user_score
    -> where score >= 2000 and score < 3000
    -> group by score
    -> order by score asc;
    1000 rows in set (3.32 sec)
```

### 复杂实时计算
上面的聚合表也可以实时计算，这样就不需要等20s/1分钟才能出结果。  
实时计算的方案要复杂一些，核心思想是当分数变动时，维护聚合表数据同步变动，将计算成本摊薄到每一次分数变化时。 

比如用户分数从2000涨到3000时，聚合表里2000分对应的user_cnt - 1, 3000分对应的user_cnt + 1。
算打败x%人时：
```sql
mysql> select sum(user_cnt) as cnt1 from user_score_aggregation where score < 600000;
+----------+
| cnt1     |
+----------+
| 25714374 |
+----------+
1 row in set (0.14 sec)

mysql> select sum(user_cnt) as cnt2 from user_score_aggregation;
+----------+
| cnt2     |
+----------+
| 30000000 |
+----------+
1 row in set (0.12 sec)
```
从上面的数据看得出这是个可行的方案。    
然而在我们真实的业务场景里，分数是在周天0点统一变化，所以这个方案跟离线方案的差距不大。







































