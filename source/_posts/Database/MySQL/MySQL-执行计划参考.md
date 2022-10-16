---
title: MySQL 执行计划参考
date: 2022-10-02 07:29:46
tags:
- MySQL
---

# 参考文章
[官方文档](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)

EXPLAIN 语句提供有关 MySQL 如何执行语句的信息。EXPLAIN 工作于 `SELECT`, `DELETE`, `INSERT`, `REPLACE`, `UPDATE` 语句。
EXPLAIN 为每一个使用了 SELECT 语句的表返回一行信息。在输出信息中，它按照 MySQL 在执行语句的时候读取的表的顺序进行输出。这意味着，MySQL 从第 1 张表读取一行，那么它从第 2 张表中寻找匹配的行，然后从第三张表，等等。


## id
SELECT 标识符。这是本次查询中 SELECT 的序列号。如果行是其他行的 UNION 结果，该值可以为 NULL。在这种情况下，table 列显示诸如 *<union M, N>*，标识该行是 id 值为 M 和 N 行的 UNION。

## select_type
SELECT 的类型，可以是下表中的任意一种。

|select_type 值|意义|
|:---|:---|
|SIMPLE|简单 SELECT（没有使用 UNION 或者子查询）|
|PRIMARY|最外层查询|
|UNION|UNION 查询中第二或后面的 SELECT 语句|
|DEPENDENT UNION|UNION 查询中第二或后面的 SELECT 语句，依赖于外部查询|
|UNION RESULT|UNION 的结果|
|SUBQUERY|子查询中第一个 SELECT|
|DEPENDENT SUBQUERY|子查询第一个 SELECT，依赖于外部查询|
|DERIVED|派生表|
|DEPENDENT DERIVED|依赖于其他表的派生表|
|MATERIALIZED||
|UNCACHEABLE SUBQUERY||
|UNCACHEABLE UNION||


## table
该行输出所引用的表名称。可以是下面的值中的一个：

- <union ***M,N***>: 该行引用 id 值为 ***M*** 和 ***N*** 的行 UNION。

- <derived ***N***>: 引用 id 值为 ***N*** 的行的派生表结果。例如，一个派生表可能源自于 FROM 子句的子查询。

- <subquery ***N***>: 引用 id 值为 ***N*** 的行的 materialized 子查询结果。


## type

EXPLAIN 的列 type 输出描述了表如何连接。下列描述了连接类型，顺序由好到差：

- system
表只有一行（= 系统表）。这是 const 连接的特例。

- const
当比较 PRIMARY KEY 或者 UNIQUE 索引**所有部分**与常量值的时候，const 被使用。这里之所以用“所有部分”，是因为 PRIMARY KEY 和 UNIQUE 都可以基于多字段建立。

- eq_ref
从 B 表中读取仅匹配的一行，与 A 表的每一行进行组合。除了 system 和 const 类型，这可能是最好的连接类型。当索引的**所有部分**被用于连接，而且索引是 PRIMARY KEY 或者 UNIQUE NOT NULL 索引，eq_ref 被使用。

eq_ref 用于使用等于运算符的索引列。比较值可以是常量或者表达式，该表达式使用前表读取的列。比如下面例子：
```sql
SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column=other_table.column;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column_part1=other_table.column
  AND ref_table.key_column_part2=1;
```
> 对于第 2 个 SQL，必须充分使用索引的“所有部分”，可以是列，可以是常量；而且，至少有一个列，否则全是常量就变成 const 连接类型了

- ref
从 B 表中读取匹配的所有行，与 A 表的每一行进行组合。如果连接仅仅使用最左前缀，或者 key 不是 PRIMARY KEY 或者 UNIQUE 索引，ref 将被使用。总之，如果连接无法选择单列，而是多列，就有可能是 ref。

ref 可以用于 = 或者 <=> 操作符进行比较的索引列。

-fultext
使用全文索引执行连接

- ref or null
省略

- range
使用索引来选择行，只有那些在给定范围内的行数据会被取出。输出行中的 key 标识哪个索引被使用到。key_len 包含使用的最长 key 的长度。对于此类型，ref 列为 null。

range 用于 key 列与常量进行比较，涉及的运算符有 =, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, LIKE, IN()

- index
除了索引树被扫描以外，index 连接类型和 ALL 相同。发生在两种情况下：
	- 如果索引对于查询来说是覆盖索引，并且可用于满足表所需的所有数据，则仅仅扫描索引树。在这种情况下，Extra 列显示 Using index。仅扫描索引树通常比 ALL 更快，因为索引的大小通常小于表数据。
	- 以索引的顺序，使用来自索引的读取执行全表扫描来检索数据。Using index 不会出现在 Extra 列。

当查询仅使用单个索引的一部分时，MySQL 使用 index 类型。

- ALL
从 B 表进行全表扫描，与 A 表每一行进行组合。

## key
key 列表示 MySQL 实际决定使用的 key（索引）。如果 MySQL 决定使用 possible keys 索引之一来检索行，那么，那个索引将被列为 key 值。

对于 InnoDB 来说，辅助索引可能会覆盖被选择的列，即使查询也取出主键，因为 InnoDB 将主键和每个辅助索引存储在一起。如果 key 是 NULL，说明 MySQL 找不到更有效地执行查询地索引。

## key_len
key_len 标识 MySQL 决定使用的 key 的长度。key_len 可以让你确定联合索引实际使用的部分。如果 key 列是 NULL，那么 key_len 也是 NULL。

由于 key 的存储不同，对于可以为 NULL 的列会比 NOT NULL 列的 key 长度大 1 位。