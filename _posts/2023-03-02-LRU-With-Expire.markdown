---
layout:     post
title:      实现带过期时间的LRU
subtitle:   Remember that not getting what you want is sometimes a wonderful stroke of luck.
date:       2023/3/2 17:46
author:     "MuZhou"
header-img:  "img/2023/bg-03-02.jpeg"
catalog: true
tags:
- 面试
---
>设计一个带有过期时间的LRU缓存。

仍旧是同事面试时遇到的题，记录一下我的解法。  
经典的LRU题目，leetcode上有初始版本，[lru-cache](https://leetcode.com/problems/lru-cache/)。
这道面试题在leetcode题目的基础上增加了过期时间，到达容量上限后，先逐出过期的数据，然后再逐出  
最久未使用的数据。   
仿照leetcode的题目，本题大致是实现这样一个数据结构。
```java
   class LRUCache {

        public LRUCache(int capacity) {

        }

        public int get(int key) {

        }

        public void put(int key, int value, long expire) {

        }
    }

/**
 * Your LRUCache object will be instantiated and called as such:
 * LRUCache obj = new LRUCache(capacity);
 * int param_1 = obj.get(key);
 * obj.put(key,value, expire);
 */ 
```

#### LinkedHashMap + 惰性删除
众所周知，双向链表 + hash map可以实现LRU，而Java里LinkedHashMap正好两者都有。可以说，用LinkedHashMap实现LRU，是众多Javaer面试必会的题目之一了。     
本题还需要处理数据过期，而逐出过期数据不由让人想起Redis的惰性删除策略。      
利用LinkedHashMap，再仿照Redis的惰性删除
```java
public class LRUCache extends LinkedHashMap<Integer, LRUCache.Node> {
    int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }

    public int get(int key) {
        Node node = super.get(key);
        if (node == null) {
            return -1;
        }
        if (node.expire < System.currentTimeMillis()) {
            remove(node);
            return -1;
        }
        return node.val;
    }

    public void put(int key, int value, long expire) {
        Node node = new Node(key, value, expire);
        super.put(key, node);
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<Integer, Node> eldest) {
        if (size() <= capacity) {
            return false;
        }
        long now = System.currentTimeMillis();
        if (eldest.getValue().expire < now) {
            return true;
        }
        Iterator<Node> iterator = values().iterator();
        while (iterator.hasNext()) {
            Node node = iterator.next();
            if (node.expire < now) {
                iterator.remove();
                return false;
            }
        }

        return true;
    }

    class Node {
        private int key;
        private int val;
        private long expire;

        public Node(int key, int val, long expire) {
            this.key = key;
            this.val = val;
            this.expire = expire;
        }
    }
}
```

再仔细看看LinkedHashMap的get和put方法，了解下具体过程。

### get过程

java.util.LinkedHashMap#get在获取数据之后，如果按访问排序（accessOrder=true)，会执行afterNodeAccess方法，将数据移到双向链表尾部。
```java
    /**
     * The iteration ordering method for this linked hash map: {@code true}
     * for access-order, {@code false} for insertion-order.
     *
     * @serial
     */
    final boolean accessOrder;    

    /**
     * Returns the value to which the specified key is mapped,
     * or {@code null} if this map contains no mapping for the key.
     *
     * <p>More formally, if this map contains a mapping from a key
     * {@code k} to a value {@code v} such that {@code (key==null ? k==null :
     * key.equals(k))}, then this method returns {@code v}; otherwise
     * it returns {@code null}.  (There can be at most one such mapping.)
     *
     * <p>A return value of {@code null} does not <i>necessarily</i>
     * indicate that the map contains no mapping for the key; it's also
     * possible that the map explicitly maps the key to {@code null}.
     * The {@link #containsKey containsKey} operation may be used to
     * distinguish these two cases.
     */
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }

    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```

### put过程
put的过程比get更复杂一些，首先是java.util.HashMap#put和putVal，这代码也是经典Java面试题目之一，不同jdk版本实现也不一样，截取jdk 17👇
```java

    /**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with {@code key}, or
     *         {@code null} if there was no mapping for {@code key}.
     *         (A {@code null} return can also indicate that the map
     *         previously associated {@code null} with {@code key}.)
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    
        /**
     * Implements Map.put and related methods.
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
跟本题相关的重点是afterNodeAccess和afterNodeInsertion方法。afterNodeAccess在get时也会调用，将数据移到双向链表尾部。
```java
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }

    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }
```
put操作的最后一步是调用afterNodeInsertion，通过removeEldestEntry判断是否需要删除元素。  
本题的解答就是重写了removeEldestEntry，来实现必要时删除过期元素。

### 其他
总的来说，过期key删除有三个策略：  
- 定时删除：创建定时器，到达过期时间时，就删除key。
  - 最简单的实现：new一个线程，死循环执行删除任务。
  - 内存友好，不会让过期数据占存储空间，但是会增加CPU压力。
- 惰性删除：到必要时候才删除key，比如get一个过期key时。
  - 内存不友好，CPU友好
- 定期删除：间隔一段时间，随机抽样一批key，删除其中过期的
  - 定时删除和惰性删除的调和版本，可以通过调参来控制对CPU和内存影响。

本文只提供了惰性删除策略的实现，网上也有很多实现定时删除的文章，可以参考。   
此外，LinkedHashMap并不是线程安全的，如果需要保证线程安全，可以换成ConcurrentLinkedDeque + ConcurrentHashMap来实现。
