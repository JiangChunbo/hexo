---
title: MySQL InnoDB 锁的产生时机
date: 2022-11-30 14:17:42
tags:
---


# 0. 参考引用

https://cloud.tencent.com/developer/article/1844928

现有一个 `class` 表，具有字段 `id`, `no`, `title`。`id` 为主键，`no` 列为唯一索引，`title` 为普通索引，数据如下:

```bash
+----+----+-------+
| id | no | title |
+----+----+-------+
|  1 |  1 | 01班  |
|  6 |  6 | 06班  |
|  7 |  7 | 07班  |
|  8 |  8 | 08班  |
|  9 |  9 | 09班  |
| 10 | 10 | 10班  |
| 14 | 14 | 14班  |
+----+----+-------+
```


# 相同检索条件下，不同索引的表现

## 等于(=)存在的值

当你在事务中根据索引**等值匹配**检索 1 条**存在的**记录, 意图获得 X 锁时:

### PRIMARY

```sql
begin;
update class set title = title where id = 1;
```

这时候需要请求 2 个锁:
- 表级 IX 锁
- PRIMARY KEY = 1 的行锁



### UNIQUE

```sql
update class set title = title where `no` = 1;
```

相较于使用 PRIMARY KEY，多 1 个锁，这时候需要请求 3 个锁:
- 表级 IX 锁
- UNIQUE INDEX = 1 的行锁
- PRIMARY KEY = 1 的行锁

> 匹配上的 UNIQUE INDEX 还需要锁定对应的 PRIMARY KEY


### NORMAL

```sql
update class set title = title where `title` = '10班';
```
这时候需要请求 4 个锁:
- 表级 IX 锁
- NORMAL INDEX = '10班' 的临键锁，区间范围是 `('9班', '10班']`
- PRIMARY KEY = 10 的行锁
- NORMAL INDEX = '14班' 的间隙锁，区间范围是 `('10班', '14班')`

> 这似乎意味着，命中的普通索引会锁住他所在的相邻两个区间，左右不包含。

## 等于(=)不存在的值（介于两索引之间）


当你在事务中根据索引检索 1 条**不存在的**且**介于两个存在索引之间**记录, 意图获得 X 锁时:

### PRIMARY

```sql
begin;
update class set title = title where id = 3;
```

此时需要请求 2 个锁: 

- 表级 IX 锁
- PRIMARY KEY = 6 的间隙锁，区间是 (1, 6)

> 由于并不需要锁住 6，因此不需要使用临键锁

### UNIQUE

```sql
update class set title = title where `no` = 3;
```

此时需要请求 2 个锁: 
- 表级 IX 锁
- UNIQUE INDEX = 6 的间隙锁，区间是 $(1, 6)$ 



### NORMAL

```sql
update class set title = title where `title` = '3班';
```

此时需要请求 2 个锁: 
- 表级 IX 锁
- NORMAL INDEX = '6班' 的间隙锁，区间是 ('1班', '6班')


## 等于(=)多个存在的值

当你在事务中根据索引**等值匹配**检索 n (n > 1) 条**存在的**记录, 意图获得 X 锁时:

### PRIMARY

```sql
update class set title = title where id in (1, 2);
```

这与等值匹配 1 个值的时候没什么区别，这时候需要请求 3 个锁:

- 表级 IX 锁
- PRIMARY KEY = 1 的行锁
- PRIMARY KEY = 2 的行锁

### UNIQUE

```sql
update class set title = title where `no` in (7,8);
```

此时需要请求 5 个锁:
- 表级 IX 锁
- 锁定 UNIQUE KEY = 7 的行级 X 锁
- 锁定 PRIMARY KEY = 7 的行级 X 锁
- 锁定 UNIQUE KEY = 8 的行级 X 锁
- 锁定 PRIMARY KEY = 8 的行级 X 锁


### NORMAL

```sql
update class set title = title where `no` in ('1班', '10班');
```

此时需要请求 7 个锁:

- 表级 IX 锁

3 个关于 `'1班'`的锁:

- NORMAL INDEX = '1班' 的临键锁，锁定范围 $(negative infinity, '1班']$
- PRIMARY KEY = 1 的行锁
- NORMAL INDEX = '6班' 的间隙锁，锁定范围 $('1班', '6班')$

3 个关于 `'10班'`的锁:

- NORMAL INDEX = '10班' 的临键锁，锁定范围 $('9班', '10班']$
- PRIMARY KEY = 1 的行锁
- NORMAL INDEX = '14班' 的间隙锁，锁定范围 $('10班', '14班']$





## 等于不存在的值（小于最小索引）

当你在事务中根据索引检索 1 条**不存在的**且**小于最小索引**的记录, 意图获得 X 锁时:

### PRIMARY KEY

```sql
update class set title = '一班' where id = 0;
```

类似上面，此时也需要请求 2 个锁，只是第二个锁左区间是负无穷：
- 表级 IX 锁
- PRIMARY KEY = 1 的间隙锁，区间是 $($*negative infinity*$, 1)$

### UNIQUE KEY

```sql
update class set title = '一班' where `no` = -1;
```

此时需要请求 2 个锁：
- 表级 IX 锁
- 定的区间是 $(-∞, 1)$ 的间隙锁。

## 等于不存在的值（大于最大索引）


当你在事务中根据 Primary Key 检索 1 条**不存在的**且**大于最大索引**的记录, 意图获得 X 锁时:

### PRIMARY

```sql
update class set title = title where id = 99;
```

注意，这有点特别，但依然需要请求 2 个锁:
- 表级 IX 锁
- PRIMARY KEY = supremum pseudo-record 的临键锁，区间是 $(14, $ *positive infinity*$)$

> 临键锁是记录锁和间隙锁的组合，一般可以理解为是一个左开右闭的区间，但是这里右区间的值是一个上确界伪记录，称之为 supremum pseudo-record，官方似乎将这种锁划定到临键锁。
> 关于更多 supremum record 参见官网 [supremum record 术语表](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_supremum_record) 以及 [Next-Key Locks](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html#innodb-next-key-locks)

### UNIQUE


```sql
update class set title = title where `no` = 99;
```

需要请求 2 个锁:
- 表级 IX 锁
- UNIQUE KEY = supremum pseudo-record 的临键锁，区间是 $(14, $ *positive infinity*$)$

### NORMAL

```sql
update class set title = title where `title` = '99班';
```
需要请求 2 个锁:
- 表级 IX 锁
- NORMAL INDEX = supremum pseudo-record 的临键锁，区间是 ('14班', *positive infinity*)

## 大于（等于）存在的值

当你在事务中根据索引检索一个**大于（等于）**某个**存在的值**范围的记录，意图获得 X 锁时:
### PRIMARY

```sql
update class set title = '一班' where id >= 10;
```
此时至少请求 3 个锁:
- 表级 IX 锁
- PRIMARY KEY = 14 的临键锁，区间是 $(10, 14]$
- PRIMARY KEY = supremum pseudo-record 的临键锁，区间是 $(14,$ *positive infinity*$)$

如果是大于等于，需要将 10 单独锁住:

- PRIMARY KEY = 10 的记录锁



### UNIQUE
```sql
update class set title = '一班' where `no` >= 10;
```

- 表级 IX 锁
- PRIMARY KEY = 10 的行锁，**如果带等于**
- UNIQUE INDEX = 10 的临键锁，区间是 (9, 10]
- UNIQUE INDEX = 14 的临键锁，区间是 (10, 14]
- PRIMARY KEY = 14 的行锁，

## 大于（等于）不存在的值

当你在事务中根据 PRIMARY KEY 检索一个**大于（等于）**某个**不存在的值**范围的记录，意图获得 X 锁时:

```sql
update class set title = '一班' where id >= 12;
```

此时会请求 3 个锁:

- 表级 IX 锁
- PRIMARY KEY = 14 的临键锁，区间是 $(10, 14]$
- PRIMARY KEY = supremum pseudo-record 的临键锁，区间是 $(14,$ *positive infinity*$)$

> 因为不存在 12 的索引，锁定最接近的下一个索引值 14 所在的区间 $(10, 14]$。同时，由于 14 是从小到大最后一个索引值，获得一个上确界伪记录 X 锁，防止其他事务 INSERT。
> 值得注意的是，由于锁定的区间覆盖了值 11，因此甚至连 11 都无法 INSERT，尽管 11 并不在 id > 12 的条件中。


## 小于（等于）存在的值

当你在事务中根据 PRIMARY KEY 检索 1 个**小于（等于）**某个**存在的值**范围的记录，意图获得 X 锁时:

```sql
update class set title = '一班' where id <= 6;
```


此时会请求 3 个锁:

- 表级 IX 锁
- PRIMARY KEY = 6 的临键锁，区间 $(1, 6]$；如果没有等于匹配，那么退化为间隙锁，区间是 $(1, 6)$
- PRIMARY KEY = 1 的临键锁，区间 $($*negative infinity*$, 1]$


## 小于（等于）不存在的值

当你在事务中根据 PRIMARY KEY 检索 1 个**小于（等于）**某个**不存在的值**范围的记录，意图获得 X 锁时:

```sql
update class set title = '一班' where id <= 3;
```

此时会请求 3 个锁:

- 表级 IX 锁
- PRIMARY KEY = 6 的间隙锁，区间 $(1, 6)$
- PRIMARY KEY = 1 的临键锁，区间 $($*negative infinity*$, 1]$