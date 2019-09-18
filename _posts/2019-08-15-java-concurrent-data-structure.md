---
layout:     post
title:      "复习04 Java中的并发容器"
subtitle:   ""
date:       2019-08-15 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Java
---


## CopyOnWriteArrayList

读完全无锁，使用 volatile 修饰 array，保证读可见性，因此读是不加锁的。写的时候是使用了 ReentrantLock，创建一个新拷贝做修改，然后再把引用指向新拷贝。

读写性能较高，因为写的过程也不会造成阻塞，缺点是不是强一致性的，特别是做耗时遍历的时候。CopyOnWriteArrayList 内部的迭代器会保留一份单独的拷贝，不受写操作的影响。另外每次修改就重新创建数组，写操作多的时候会产生很多垃圾，因此使用时要考虑内容规模和读写比例。

## ConcurrentLinkedQueue

内部类 Node，封装一个储存内容 + 下一个节点的引用，形成链表。线程安全的实现是 Node 在替换自己的 2 个属性的时候用的都是 CAS 操作，因此锁是独立在各节点的，是性能比较高的非阻塞性线程安全队列。

## BlockingQueue

阻塞的线程安全队列，阻塞在某些场景下反而是好处，因为阻塞后减少了很多异常出现的概率，不容易出错，比如消息队列场景，有就取出来传，没有就阻塞，简单明了。

阻塞和唤醒均使用了 Condition 类，在非空、非满等不同条件下分别唤醒。

### ArrayBlockingQueue

用数组来实现队列，固定容量。由于底层直接操作数组，因此读写用的是同一个锁。

### LinkedBlockingQueue

封装 Node 组成链表实现队列，默认 Integer.MAX_VALUE 作为数组容量，也可以指定大小。读写操作的不是同一个 Node，因此读写锁分开。比较常用，用的时候注意控制好边界。

### PriorityBlockingQueue

可以按指定顺序排序的 BlockingQueue，要么初始化时候传入一个 `Comparator<? super E> comparator`，要么内容实现了 `Comparable`，必须是可比较的。底层也是用了数组，默认大小 11，区别是超过大小会自动扩容，扩容时使用了 CAS 机制加锁保证不会重复申请新数组，

## ConcurrentSkipListMap

跳表结构，使用 ` 双层 for 循环 + CAS 方式 ` 做并发处理，操作失败则跳出当前循环到上层循环里重试操作，里面的实现就比较复杂了，封装了好几层。

1. Node 封装储存的内容，key、value、next
2. Index 封装节点的层级关系，向下向右查询，node、down、right
3. HeadIndex 作为每一层头节点的类，继承 Index ，封装了这一层的层级，level

### 插入逻辑

1. 插入到底层链表的位置上
2. 计算 level，根据 level 插入到上层链表上，如果没有这个 level，则新建并插入
3. 关联各链表的右和下指针
4. 假设头节点变化，重新关联头节点

### 移除逻辑

1. 根据 key 判断是否有对应节点需要删除
2. 判断前驱后置节点是否存在，赋值要删除的节点为 null
3. 插入虚假节点 Marker 分散并发冲突，将前驱节点连接到原来的后置节点
4. 该层判断是否还有节点，没有则删除整层
