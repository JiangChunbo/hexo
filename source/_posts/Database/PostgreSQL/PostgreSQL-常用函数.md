---
title: PostgreSQL 常用函数
date: 2022-11-07 11:47:17
tags:
---





- to_char

格式化

日期/时间格式化的模板模式参考[官网](https://www.postgresql.org/docs/15/functions-formatting.html)



- split_part

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



- string_agg

聚合函数。必须配合 group by。

直接把一个表达式变成字符串
```sql
string_agg(expression, delimiter)
```

> 在某些场景下可能配合 JOIN 使用.