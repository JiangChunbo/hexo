---
title: PostgreSQL 常用函数
date: 2022-11-07 11:47:17
tags:
---


## String Functions and Operators

### || 操作符

可以连接字符串；也可以连接非数组类型。


### concat

```sql
concat ( val1 "any" [, val2 "any" [, ...] ] )
```

连接所有参数的文本表示。NULL 参数被忽略。

与 || 的区别在于：

- concat 会忽略 null 值，将非 null 值连接
- || 如果有任意一个为 null， 整个表达式为 null


### to_char

格式化

日期/时间格式化的模板模式参考[官网](https://www.postgresql.org/docs/15/functions-formatting.html)


一些常用的格式化模式:

|Pattern|描述|
|-|-|
|YYYY|year (4 or more digits)|
|MM|month number (01–12)|
|DD|day of month (01–31)|
|HH24|hour of day (00–23)|
|MI|minute (00–59)|
|SS|second (00–59)|



### split_part

第 3 个参数是索引，从 1 开始。

```sql
select split_part('1,2,3', ',', 1);
```

> 字符串 `'1,2,3'` 可以替换为字段

```sql
 split_part
------------
 1
(1 row)
```

如果索引值不存在，也会返回

```sql
select split_part('1,2,3,', ',', 5);
```
```sql
 split_part
------------

(1 row)
```

取出最后一部分:

```sql
select split_part('/江苏/南京市/建邺区', '/', length(replace('/江苏/南京市/建邺区', '/', '//')) - length('/江苏/南京市/建邺区') + 1);
select split_part('江苏/南京市/建邺区', '/', length(replace('江苏/南京市/建邺区', '/', '//')) - length('江苏/南京市/建邺区') + 1);
```

```sql
 split_part
------------
 建邺区
(1 row)
```



### string_agg

聚合函数。必须配合 group by。

直接把一个表达式变成字符串
```sql
string_agg(expression, delimiter)
```

> 在某些场景下可能配合 JOIN 使用.