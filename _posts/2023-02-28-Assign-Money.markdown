---
layout:     post
title:      分配红包算法
subtitle:   If you're not making mistakes, then you're not making decisions.
date:       2023/2/28 16:58
author:     "MuZhou"
header-img:  "img/2023/bg-02-28.jpeg"
catalog: true
tags:
- 面试
---

> 请编写一个红包随机算法。需求为：给定一定的金额，一定的人数，保证每个人都能随机获得一定的金额。
比如100元的红包，10个人抢，每人分得一些金额。约束条件为，最佳手气金额不能超过红包总额的90%

最近同事面试某公司，被问到了这个题，记录一下我的解法。

（为了简化问题，钱的单位统一为分）   
首先，借鉴微信红包的分配算法，在 **[1, 2倍剩余平均值)** 之间随机（注意是左闭右开区间）。  
比如2000，分给2个人，那么第一个人最少得1，最多得1999。

其次，优化一下上下限边界，实现**最佳手气金额不能超过红包总额的90%**。  
假设有n个人，总金额是money，那么前n-1个人合计至少拿走money * 10%，且每个人不得超过总额money * 90%。   
叠加之后，区间边界就变成了
```java
int lowerBound = Math.max(1, money / 10 / (count - 1));
int upperBound = Math.min(money * 9 / 10, average * 2);
```
某些场景下，average * 2 会小于lowerBound。    
比如6个人分200，lowerBound = 4。假设前4个人手气很好，一共取走了198，剩下的2人此时average * 2 = 2。    
简单优化一下，如果累计取走金额已到达10%，就使用微信红包的区间[1, average * 2)

```java

import com.google.common.base.Joiner;
import com.netflix.discovery.shared.Pair;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ThreadLocalRandom;

public class Test {

    static List<Integer> assignMoney(int money, int count) {
        List<Integer> res = new ArrayList<>();
        if (count < 2) {
            res.add(money);
            return res;
        }

        int remainMoney = money;
        int lowerBound = (int) Math.ceil(money * 0.1 / (count - 1));
        int upperBound = (int) Math.ceil(money * 0.9);
        while (count > 1) {
            int average = remainMoney / count;
            if (remainMoney <= money * 0.9) {
                lowerBound = 1;
                upperBound = average * 2;
            }
            int curMoney = ThreadLocalRandom.current().nextInt(lowerBound, Math.min(average * 2, upperBound));
            res.add(curMoney);
            remainMoney -= curMoney;
            count--;
        }
        res.add(remainMoney);
        return res;
    }

    public static void main(String[] args) throws Exception {
        List<Pair<Integer, Integer>> testCase = new ArrayList<>();
//        testCase.add(new Pair<>(1, 1));
        testCase.add(new Pair<>(2, 2));
        testCase.add(new Pair<>(10, 10));
        testCase.add(new Pair<>(100, 2));
        testCase.add(new Pair<>(100, 10));

        for (int i = 0; i < 10000; i++) {
            int count = ThreadLocalRandom.current().nextInt(2, 1000);
            int money = count * ThreadLocalRandom.current().nextInt(1, 1000);
            testCase.add(new Pair<>(money, count));
        }
        for (Pair<Integer, Integer> pair : testCase) {
            List<Integer> res = assignMoney(pair.first(), pair.second());
            check(res, pair.first(), pair.second());
        }
    }

    static void check(List<Integer> res, int money, int count) throws Exception {
        int max = money * 9 / 10; //不超过90%
        int min = 1;//每人至少一分
        int sum = 0;
        String resStr = Joiner.on(",").join(res);
        for (Integer each : res) {
            if (each > max || each < min) {
                String msg = String.format("非法金额：%s, money: %s, count: %s, res: %s", each, money, count, resStr);
                throw new Exception(msg);
            }
            sum += each;
        }
        if (sum != money) {
            String msg = String.format("非法分配： money: %s, count: %s, res: %s", money, count, resStr);
            throw new Exception(msg);
        }
    }
}

```