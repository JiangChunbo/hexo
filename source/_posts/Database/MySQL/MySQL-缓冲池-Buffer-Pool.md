---
title: MySQL InnoDB Buffer Pool
date: 2022-07-20 22:12:49
tags:
- MySQL
---

# MySQL InnoDB Buffer Pool
​
> 本文主要内容源自官网：[MySQL :: MySQL 8.0 Reference Manual :: 15.5.1 Buffer Pool](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html)
> 感兴趣的可以直接阅读

​缓冲池是主内存的一块区域，其中 InnoDB 访问表和索引数据时会在其中进行缓存。Buffer Pool 允许从内存中访问频繁使用的数据，这加快了处理速度。在专用服务器上，通常将多达 80% 的物理内存分配给 Buffer Pool。

为了大容量的读操作效率，Buffer Pool 被分为多个页面，这些页面可能包含多个行。为了缓存管理的效率，Buffer Pool 被实现为 Page 的链表。使用 LRU （Least Recently Used，最近最少使用）算法的变体将数据从缓存中老化淘汰出去。


知道如何利用 Buffer Pool，以将经常访问的数据保存在内存中是 MySQL 调整的重要方面。

# 缓冲池 LRU 算法

缓冲池 LRU 算法将缓冲池作为列表进行管理。当缓冲池空间不足，但有新页面需要添加到缓冲池时，将驱逐最近最少使用的页面，并将新页面添加到列表的 Midpoint（中点）。总列表分为两个子列表（Sublist）：

1. 最前面的是最近访问过的新页面（或者叫 young 页面，年轻页面）子列表

2. 末尾是最近访问的旧页面的子列表


<img src="https://img-blog.csdnimg.cn/20210201232619278.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM5MjkxOTE5,size_16,color_FFFFFF,t_70">


​
该算法将经常需要访问的页面保留在 New Sublist。Old Sublist 包含不常用的页面，这些页面是驱逐的候选对象。

通常情况，该算法遵循以下规则：

1.  的 Buffer Pool 用于 Old Sublist

2. 列表的 Midpoint 是 New Sublist 的 Tail 和 Old Sublist 的 Head 相交的边界。

3. 当 InnoDB 将页面读入缓冲池时，它首先将页面插入 Midpoint。通过用户触发的动作（比如 SQL 查询）或者预读操作可以对页面进行读取。

注意：官方这里说的读取（read）并不代表访问（access），请看下面。

4. 访问（access）Old Sublist 中的页面会让其变得 young，然后将其移至 New Sublist 的 Head。如果是因为用户触发的动作需要读取页面，则将立即进行第一次 access，使页面 young。如果是由于预读操作而读取了该页面，则第一次 access 不会立即访问，甚至在页面离开之前都不会发生！

注意：预读未必会让页面变 young。

5. 随着数据库的运行，通过将页面移动到列表的尾部，缓冲池中的页面将会“老化”。New 和 Old Sublist 都会随着其他页面的更新而老化。随着将页面插入 Midpoint，旧子列表中的页面也会老化。最终，未使用的页面到达子列表的尾部并被逐出。

注意：这里有个有趣的现象，页面是先插入到 Midpoint，而且这些页面插入之后属于 Old Sublist 的范围，所以他们很可能会马上“老化”，唯一让他们变得 young 的途径就是 access 一次。

默认情况下，查询（Query）读取的页面会立即移入新的子列表，这意味着它们在缓冲池停留的时间更长。

举个例子，mysqldump 操作或者 SELECT 不带 WHERE 子句的语句可能会将大量数据带入缓冲池，并驱逐出相当多的旧数据，即使加入 New Sublist Head 的新数据在 access 一次之后都不会再被使用了。同样，预读线程加载的页面，且仅仅访问过一次，那么也会移至 New Sublist 的开头。这些情况可能会将常用页面推到旧的子列表，然后被逐出。

所以，这似乎有个问题，有些 page 明明使用的频率不如其他页面，却可以插在其他页面前面，甚至逐出其他页面 ，官方给出了优化此行为的方法：让缓冲池的扫描具有“抵抗力”、配置 InnoDB 缓冲池预读（具体见官网）。

# 使用标准监视器监视 Buffer Pool


```bash
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992 # 为缓冲池分配的总内存（单位：字节）
Dictionary memory allocated 2774195 # 为 InnoDB 数据字典分配的总内存（单位：字节）
Buffer pool size   8192 # 分配给缓冲池的页数
Free buffers       957 # 缓冲池空闲列表的页数
Database pages     6896 # 缓冲池 LRU 列表的页数
Old database pages 2525 # 缓冲池 Old Sublist 的页数
Modified db pages  0 # 缓冲池中当前修改的页数
Pending reads      0 # 等待读入缓冲池的页数

# Pending writes LRU 来自于 LRU 列表底部待写的旧脏页数
# Pending writes flush list 检查点期间要刷新的缓冲池页面数
# Pending writes single page 缓冲池中暂挂的独立页面写入数
Pending writes: LRU 0, flush list 0, single page 0

# Pages made young 缓冲池 LRU 列表中变年轻的页面数（已移至 New Sublist 开头）
# Pages made not young 缓冲池 LRU 列表中非年轻的页数（在 Old Sublist 没有年轻的页面）
Pages made young 38061, not young 2052025
# youngs/s 每秒导致 LRU 列表中旧页面访问的平均次数
# non-youngs/s 
0.00 youngs/s, 0.00 non-youngs/s

# Pages read 从缓冲池读取的页面总数
# Pages created 从缓冲池中创建的页面总数
# Pages written 从缓冲池写入的页面总数
Pages read 416647, created 103098, written 239695
# reads/s 平均每秒缓冲池页面读取数
# creates/s 平均每秒创建缓冲池的数目
# writes/s 平均每秒缓冲池页面写入数
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 6896, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]

```
​