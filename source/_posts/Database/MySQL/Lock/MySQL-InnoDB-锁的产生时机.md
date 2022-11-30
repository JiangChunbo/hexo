---
title: MySQL InnoDB 锁的产生时机
date: 2022-11-30 14:17:42
tags:
---

现有一个 `class` 表，具有字段 `id`, `no`, `title`。`id` 为主键，`no` 列为唯一索引，数据如下:

```bash
+----+----+-------+
| id | no | title |
+----+----+-------+
|  1 |  1 | 1班   |
|  6 |  6 | 1班   |
|  7 |  7 | 2班   |
|  8 |  8 | 3班   |
|  9 |  9 | 1班   |
| 10 | 10 | 2班   |
| 14 | 14 | 3班   |
+----+----+-------+
```

# Primary Key


当你在事务中根据 Primary Key 检索 1 条**存在的**记录, 意图获得 X 锁时:

```sql
update class set title = '一班' where id = 1;
```

这时候需要请求 2 个锁:
- 表级 IX 锁
- 锁定 Primary Key = 1 的行锁


当你在事务中根据 Primary Key 检索 n (n > 1) 条**存在的**记录, 意图获得 X 锁时:

```sql
update class set title = '一班' where id in (1, 2);
```

类似地，由于 in 语句存在 2 个值，这时候需要请求 3 个锁:

- 表级 IX 锁
- PRIMARY KEY = 1 的行锁
- PRIMARY KEY = 2 的行锁



当你在事务中根据 Primary Key 检索 1 条**不存在的**且**介于两个存在索引之间**记录, 意图获得 X 锁时:

```sql
update class set title = '一班' where id = 3;
```

此时需要请求 2 个锁: 

- 表级 IX 锁
- 区间是 $(1, 6)$ 的间隙锁


当你在事务中根据 Primary Key 检索 1 条**不存在的**且**小于最小索引**的记录, 意图获得 X 锁时:

```sql
update class set title = '一班' where id = 0;
```

类似上面，此时也需要请求 2 个锁，只是第二个锁左区间是负无穷：
- 表级 IX 锁
- 区间是 $(-∞, 1)$ 的间隙锁

当你在事务中根据 Primary Key 检索 1 条**不存在的**且**大于最大索引**的记录, 意图获得 X 锁时:


```sql
update class set title = '一班' where id = 99;
```

注意，这有点特别，但依然需要请求 2 个锁:
- 表级 IX 锁
- 区间是 $(14, $ *positive infinity* $)$ 的临键锁，此时右区间是一个 supremum pseudo-record

> 临键锁是记录锁和间隙锁的组合，一般可以理解为是一个左开右闭的区间，但是这里右区间的值是一个上确界伪记录。
> 关于更多 supremum record 参见官网 [supremum record 术语表](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_supremum_record) 以及 [Next-Key Locks](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html#innodb-next-key-locks)


当你在事务中根据 PRIMARY KEY 检索一个**大于**某值范围的记录，意图获得 X 锁时:

```sql
update class set title = '一班' where id > 12;
```

此时会请求 3 个锁:

- 表级 IX 锁
- PRIMARY KEY = 14 的行级锁
- 上确界伪记录 X 锁 supremum pseudo-record


当你在事务中根据 PRIMARY KEY 检索 1 个**小于**某值范围的记录，意图获得 X 锁时:

```sql
update class set title = '一班' where id < 3;
```

此时会请求 3 个锁:

- 表级 IX 锁
- PRIMARY KEY = 1 的行级锁
- 锁定 $(1, 6)$ 区间的间隙锁

> 如果进行具有**小于**语义的检索（如: `<`,`<=`,`between`），InnoDB 会找到这个

# UNIQUE Index



这与 Primary Key 的比较类似。



当你在事务中根据 Unique Key 检索一条**存在的**记录, 意图获得 X 锁时:

```sql
update class set title = '一班' where `no` = 9;
```

这时候需要请求 3 个锁:
- 表级 IX 锁
- 锁定 Unique Key = 9 的行级 X 锁
- 锁定 Primary Key = 11 的行级 X 锁





当你在事务中根据 Unique Key 检索多条**存在的**记录, 意图获得 X 锁时:

```sql
update class set title = '一班' where `no` in (7,8,9);
```

此时需要请求 7 个锁:
- 表级 IX 锁
- 锁定 UNIQUE KEY = 7 的行级 X 锁
- 锁定 PRIMARY KEY = 7 的行级 X 锁
- 锁定 UNIQUE KEY = 8 的行级 X 锁
- 锁定 PRIMARY KEY = 8 的行级 X 锁
- 锁定 UNIQUE KEY = 9 的行级 X 锁
- 锁定 PRIMARY KEY = 9 的行级 X 锁


当你在事务中根据 UNIQUE KEY 检索一条**不存在的**且**介于两个存在索引之间**记录, 意图获得 X 锁时:

```sql
update class set title = '一班' where `no` = 3;
```

此时需要请求 2 个锁: 
- 表级 IX 锁
- 锁定 PRIMARY KEY 区间在 $(1, 6)$ 的间隙锁

> **注意** 如果记录不存在，将会锁定 PRIMARY KEY


当你在事务中根据 UNIQUE KEY 检索一条**不存在的**且**小于最小索引**的记录, 意图获得 X 锁时:

```sql
update class set title = '一班' where `no` = -1;
```

此时需要请求 2 个锁：
- 表级 IX 锁
- 定的区间是 $(-∞, 1)$ 的间隙锁。

当你在事务中根据 UNIQUE KEY 检索一条**不存在的**且**大于最大索引**的记录, 意图获得 X 锁时:


```sql
update class set title = '一班' where id = 99;
```

此时请求 2 个锁:
- 表级 IX 锁；
- 1 个伪记录锁 supremum pseudo-record。