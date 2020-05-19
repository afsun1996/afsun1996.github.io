---
title: "「算法」LeetCode-LRU最少使用缓存机制"
subtitle: ""
layout: post
author: "afsun"
header-style: text
tags:
  - 算法
  - LeetCode
---

# LRU算法

> LRU算法在Redis中就有用到，当redis中的内存占用量到达一定阈值的时候，会启动淘汰机制。其中一项淘汰机制叫做【ttl-lru】淘汰机制，就是淘汰内存中最近最少使用的数据。

[Leetcode地址](<https://leetcode-cn.com/problems/lru-cache/>)

![1589783808774](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/18/143649-106406.png)

## 采用LinkedList双端队列实现

> 通过双端队列来存放每次操作后的数据，然后每次过期队列中的头。
>
> get的时间复杂度为O(n)
>
> put的时间复杂度为O(n)

```java
class LRUCache {
    int capacity;
    LinkedList<Node> list = new LinkedList();
    int size = 0;

    class Node {
        int value;
        int key;

        @Override
        public boolean equals(Object o) {
            Node node = (Node) o;
            return node.key == this.key;
        }

        @Override
        public int hashCode() {
            return Objects.hash(key);
        }

        @Override
        public String toString() {
            return "key:"+key;
        }
    }

    public int get(int key) {
        Node temp = new Node();
        temp.key = key;
        int index = list.indexOf(temp);
        if (index < 0) {
            return -1;
        }
        Node removeNode = list.remove(index);
        list.addLast(removeNode);
        return removeNode.value;
    }

    public void put(int key, int value) {
        Node temp = new Node();
        temp.key = key;
        temp.value = value;
        int index = list.indexOf(temp);
        if (index >= 0) {
            list.remove(index);
            list.addLast(temp);
            return;
        }
        if (this.size >= this.capacity) {
            list.removeFirst();
            list.addLast(temp);
            return;
        }
        this.list.addLast(temp);
        this.size++;
    }

    public LRUCache(int capacity) {
        this.capacity = capacity;
    }
}
```

![1589786776599](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/18/152617-758763.png)

## 双端列表+HASH表实现

> 双端列表快速移动，HAHS表可以以O(1)的时间复杂度获取到节点值。这样一来优化后的时间复杂度只有O(1)

```java
public class LRUCache {

    class DNode {
        int key;
        int value;
        DNode next;
        DNode pre;

        public DNode(int key, int value) {
            this.key = key;
            this.value = value;
        }

        public DNode() {

        }
    }

    class DLinkedList {
        DNode head;
        DNode tail;
        public DLinkedList() {
            head = new DNode();
            tail = new DNode();
            head.next = tail;
            tail.pre = head;
        }

        public void addLast(DNode node) {
            node.pre = tail.pre;
            tail.pre.next = node;
            node.next = tail;
            tail.pre = node;
        }

        public void removeNode(DNode dNode) {
            DNode pre = dNode.pre;
            DNode next = dNode.next;
            pre.next = next;
            next.pre = pre;
        }

        public void move2Last(DNode dNode) {
            // 删除
           this.removeNode(dNode);
            // 添加
            addLast(dNode);
        }

        public int removeFirst(){
            DNode firstOne = head.next;
            int key = firstOne.key;
            head.next = firstOne.next;
            firstOne.next.pre = head;
            return key;
        }
    }

    HashMap<Integer,DNode> cache = new HashMap();
    DLinkedList dLinkedList = new DLinkedList();
    int capacity;
    int size = 0;
    public LRUCache(int capacity) {
        this.capacity = capacity;
    }

    public void put(int key,int value){
        DNode dNode = new DNode(key,value);
        if (!cache.containsKey(key)) {
           if (size >= capacity) {
               int oleKey = this.dLinkedList.removeFirst();
               this.dLinkedList.addLast(dNode);
               this.cache.put(key,dNode);
               this.cache.remove(oleKey);
               return;
           }
            this.dLinkedList.addLast(dNode);
            this.cache.put(key,dNode);
            size++;
        }else {
            DNode cacheNode = cache.get(key);
            this.dLinkedList.removeNode(cacheNode);
            this.dLinkedList.addLast(dNode);
            this.cache.put(key,dNode);
        }
    }

    public int get(int key) {
        DNode dNode = this.cache.get(key);
        if (dNode != null) {
            this.dLinkedList.move2Last(dNode);
            return dNode.value;
        }
        return -1;
    }
}
```



## 采用LinkedHashMap来实现

> LinkedHahMap中维护了插入map的顺序，所有利用这个容器就能很便捷的实现LRU的算法

```java
class LRUCache extends LinkedHashMap<Integer, Integer>{
    private int capacity;
    
    public LRUCache(int capacity) {
        super(capacity, 0.75F, true);
        this.capacity = capacity;
    }

    public int get(int key) {
        return super.getOrDefault(key, -1);
    }

    public void put(int key, int value) {
        super.put(key, value);
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
        return size() > capacity; 
    }
}
```



![1589787010436](http://tuchuansun.oss-cn-hangzhou.aliyuncs.com/typora/202005/18/153010-285053.png)