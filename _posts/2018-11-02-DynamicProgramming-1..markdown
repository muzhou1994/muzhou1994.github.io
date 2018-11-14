---
layout:     post
title:      动态规划（一）
subtitle:   The mind is everything. What you think you become.
date:       2018/11/2 下午4:03
author:     "MuZhou"
header-img:  "img/2018/bg-dynamic-programming-1-header.jpg"
catalog: true
tags:
    - Java
    - leetcode
    - 面向工资编程
---

十一月是动态规划的一个月，这个月目标做3道easy，四道Medium，一道Hard。

先学习一下动态规划，[维基百科](https://zh.wikipedia.org/wiki/%E5%8A%A8%E6%80%81%E8%A7%84%E5%88%92)上对动态规划对定义是：
```$xslt
一种在数学、管理科学、计算机科学、经济学和生物信息学中使用的，通过把原问题分解为相对简单的子问题的方式求解复杂问题的方法。
常常适用于有重叠子问题和最优子结构性质的问题。
```
计算机领域里很多算法都是把一个复杂问题分解成子问题去求解，比如分治、比如递归，动态规划的特点在于分解的子问题有重叠，通过存储子问题的解来减少计算量，
所以实际上有些动态规划的题目也可以用递归来求解，只不过计算量会大一些，比如求解斐波那契数列。

递归方式
```java
/**
 * @author nidaren
 */
public class Test {
    //n >= 0
    int fib(int n) {
        if (n < 2) {
            return n;
        }
        return fib(n - 1) + fib(n - 2);
    }
}
```
动态规划方式
```java
/**
 * @author nidaren
 */
public class Test {
    //n >= 0
    int fib(int n) {
        if (n < 2) {
            return n;
        }
        int[] fibArr = new int[n + 1];
        fibArr[0] = 0;
        fibArr[1] = 1;
        for (int i = 2; i <= n; i ++) {
            fibArr[i] = fibArr[i - 1] + fibArr[i - 2];
        }
        return fibArr[n];
    }
}
```
除了重叠子问题之外，动态规划还用来解决最优子结构问题。
所谓最优子结构问题就是一个问题的最优解包含其子问题的最优解，局部最优解能决定全局最优解，比如爬楼梯问题、背包问题、最长公共子序列问题（LCS)。



今天是动态规划第一次题目，先是一个简单的爬楼梯，题目链接在[这里](https://leetcode.com/problems/min-cost-climbing-stairs/description/)。
这是动态规划里经典题目之一，题目大意是：
```text
爬楼梯，每一层楼梯有不同的cost，耗费该层的cost后可以向上爬一层或者两层，求爬到楼顶（n层）的最小cost
```
先来刻画最优解的结构特征，这也是解决动态规划题目最难的一步。
我们将cost视为停留cost，即cost[i]表示在i层停留的成本，用dp[i]来表示爬到第i层并停留的最优cost，那么所求解就是Min(dp[n - 1], dp[n - 2]).
最优解的结构特征：
```
dp[i] = Min(dp[i - 1], dp[i - 2]) + cost[i]
```
有了结构特征后我们来确认一下初始值，很显然我们需要先求出前两项dp[0]和dp[1]，不难得出：
```text
dp[0] = cost[0]
dp[1] = cost[1]
```
初始值求出后我们再来考虑一下corner case:
当n小于2时，也就是到楼顶到阶梯小于两层（只有1层或更少），那么显然我们中间可以不用停留，直接跨到楼顶，cost=0.

接下来可以开始写代码了：
```java
class Solution {
    public int minCostClimbingStairs(int[] cost) {
        if (cost.length < 2) {
            return 0;
        }
        int[] dp = new int[cost.length];
        dp[0] = cost[0];
        dp[1] = cost[1];
        for(int i = 2; i < cost.length; i ++) {
            dp[i] = Math.min(dp[i - 1], dp[i - 2]) + cost[i];
        }
        return Math.min(dp[cost.length - 1], dp[cost.length - 2]);
    }
}
```

上述解法时间复杂度O(n),空间复杂度也是O(n),可以更进一步优化，不引入dp而使用cost，可以将空间复杂度将为O(1),不过这会改变输入数据，需要视情况而定。
```java
class Solution {
    public int minCostClimbingStairs(int[] cost) {
        if (cost.length < 2) {
            return 0;
        }
        for(int i = 2; i < cost.length; i ++) {
            cost[i] += Math.min(cost[i - 1], cost[i - 2]);
        }
        return Math.min(cost[cost.length - 1], cost[cost.length - 2]);
    }
}
```