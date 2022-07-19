---
title: Java Collection
date: 2022-07-19 21:16:37
tags:
- JDK
---
# Java Collection
## 容器底层数据结构

### List

- `ArrayList`: `Object[]` 数组
- `Vector`: `Object[]` 数组
- `LinkedList`: JDK 1.7 之前，双向循环链表；JDK 1.7 之后，双向链表


### Map 

- `HashMap`：JDK 1.8 之前，数组 + 链表；JDK 1.8 之后，数组 + 链表/红黑树

- `LinkedHashMap`：双向链表 + 数组 + 链表

- `Hashtable`: 数组 + 链表

- `TreeMap`：红黑树


### Set

`HashSet`: 基于 `HashMap` 实现

`LinkedHashSet`: 基于 `LinkedHashMap` 实现

`TreeSet`: 红黑树

> **思考**：为什么 `HashSet` 底层 `HashMap` 的 value 不存储 null，反而是一个 Object?


### Queue

`PriorityQueue`: `Object[]` 数组二叉堆

`ArrayQueue`: `Object[]` 数组 + 双指针


## 集合选择场景

### Map

- 一些签名计算要求传入的参数按照 ascii 排序可以使用 `TreeMap`

- 多线程场合 `ConcurrentHashMap`

- 其余场合 `HashMap`

### Set

保证元素唯一 `TreeSet`、`HashSet`，做一些统计


### List

没有特别要求就选择 `ArrayList`、`LinkedList`