---
layout:     post
title:      LRU Cache with expiry time
subtitle:   Implementation of LRU cache with an expiry time for each entry
date:       2023/3/6 14:58
author:     "MuZhou"
header-img:  "img/2023/bg-03-06.jpeg"
catalog: true
tags:
- English
- interview
---
> Design an LRU cache that expires entries base on their individual expiry times, evicting expired entries before the least recently used entries.

It's still a question that my colleague was asked during his interview. Let me share my solution.        

LRU cache is a classic question, and there is an original version of it on LeetCode called [lru-cache](https://leetcode.com/problems/lru-cache/).     
In this question, an expiry time is added to each entry. Once the max capacity of the cache is reached, expired entries are evicted first, followed by the least recently used entries.      

Like the question on the LeetCode, this question requires implementing such a data structure:
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

#### LinkedHashMap + passive deletion
As we know, the LRU cache can be implemented by a doubly linked list and a hash map. Coincidentally, the LinkedHashMap in Java contains both of them. Implementing an LRU cache by LinkedHashMap is therefore a common interview question for Java developers.      

In this question, we have to deal with expired entries, and expelling expired entries here is similar to Redis's passive expiration.

We can solve the problem using a LinkedHashMap and imitating Redis as follows
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

Let's dive into the details of the 'get' and 'put' methods
### get operation
When using the 'access-order' iteration order,  invoking the 'java.util.LinkedHashMap#get' method triggers the 'afterNodeAccess' method, which moves the accessed node to the end of the doubly linked list.
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

### put operation
The 'put' operation is more complicated than 'get' and relies on the 'java.util.HashMap#put' method. This code block is a classic Java interview question, and its implementation varies from different JDK versions.     
The code snippet below is from JDK 17*
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
The 'afterNodeAccess' method and the 'afterNodeInsertion' method are the emphases related to this problem.    
We are already familiar with 'afterNodeAccess', which is mentioned in the previous 'get' operation and moves the accessed node to the end of the doubly linked list
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
The 'put' operation's final step involves calling the 'afterNodeInsertion' method, which may trigger the removal of the eldest node depending on the return value of the 'removeEldestEntry' method.    

In the solution we presented above, we override the 'removeEldestEntry' method to remove expired nodes when necessary.

### Other
There are generally three policies for expelling expired keys:
- Scheduled deletion: create a scheduler that deletes expired keys once their expiry time is reached.
    - the simplist implementation: create a new thread and delete expired keys in an infinite loop.
    - it's memory-friendly since the expired data is deleted immediately, freeing up storage space. But it will increase the load on the CPU.
- Passive Deletion: Only delete keys when necessary, such as when accessing an expired key.
    - it's CPU-friendly but can be memory-intensive.
- Periodic deletion: Sample a bacth of key periodically and delete outdated ones.
    - it's a compromise between scheduled and passive deletion, allowing for great control over the impact on CPU and memory by tuning parameters

This article provides an implementation of the passive deletion policy. However, there are many other articles avaible online that provide implementations of the scheduled deletion policy, which may be useful for reference.

Additionally, it should be noted that 'LinkedHashMap' is not thread-safe. To ensure thread-safety, it is recommended to use 'ConcurrentLinkedDeque' and 'ConcurrentHashMap'.