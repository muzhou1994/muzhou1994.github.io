---
layout:     post
title:      三个线程交替打印ABC
subtitle:   Never let your fear decide your fate.
date:       2023/2/27 15:28
author:     "MuZhou"
header-img:  "img/2023/bg01.jpeg"
catalog: true
hide_in_home: false
tags:
    - 面试
---

> 编写一个程序，开启三个线程， 这三个线程的 ID 分别是 A、B 和 C
， 每个线程把自己的 ID 在屏幕上打印 10 遍， 要求输出结果必须按 ABC 的顺序显示，如 ABCABCABC... 依次递推

最近同事面试阿里某部门，被问到了这个题，遥想18年的时候就被问过，记录一下。

### 信号量版本

```java
import java.util.concurrent.Semaphore;

public class Test {

    static class CustomerRunnable implements Runnable {
        String name;
        Semaphore waitSemaphore;
        Semaphore nextSemaphore;

        public CustomerRunnable(String name, Semaphore waitSemaphore, Semaphore nextSemaphore) {
            this.name = name;
            this.waitSemaphore = waitSemaphore;
            this.nextSemaphore = nextSemaphore;
        }

        public void run() {
            for (int i = 0; i < 10; i++) {
                try {
                    waitSemaphore.acquire();
                    System.out.println(name);
                    nextSemaphore.release();
                } catch (InterruptedException e) {

                }
            }
        }
    }

    public static void main(String[] args) {
        Semaphore A = new Semaphore(0), B = new Semaphore(0), C = new Semaphore(0);
        Thread threadA = new Thread(new CustomerRunnable("A", A, B));
        Thread threadB = new Thread(new CustomerRunnable("B", B, C));
        Thread threadC = new Thread(new CustomerRunnable("C", C, A));
        threadA.start();
        threadB.start();
        threadC.start();
        A.release();
    }
}

```

### volatile版本
同事的方案
```java
public class Test {

    static class CustomerRunnable implements Runnable {
        private static volatile int flag;
        int remain;
        String name;

        public CustomerRunnable(String name, Integer remain) {
            this.name = name;
            this.remain = remain;
        }

        public void run() {
            for (int i = 0; i < 10; ) {
                if (flag % 3 == remain) {
                    i++;
                    System.out.println(name);
                    flag++;
                }
            }
        }
    }

    public static void main(String[] args) {
        Thread threadA = new Thread(new CustomerRunnable("A", 0));
        Thread threadB = new Thread(new CustomerRunnable("B", 1));
        Thread threadC = new Thread(new CustomerRunnable("C", 2));
        threadA.start();
        threadB.start();
        threadC.start();
    }
}

```