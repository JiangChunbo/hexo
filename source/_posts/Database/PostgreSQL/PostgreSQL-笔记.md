---
title: postgresql 笔记
date: 2022-09-28 11:29:46
tags:
---

# 参考引用

https://blog.csdn.net/lmmilove/article/details/122111192


## Linux 安装客户端

```bash
apt install postgresql-client
```


## psql

- 建立连接

```bash
psql --host=HOST --username=USERNAME --dbname=DBNAME
```


### psql 内置命令

|命令|含义|
|-|-|
|\c dbname|切换数据库|
|\l|查看所有数据库|
|\d|查看当前数据库中的表|
|\d table|查看表中的字段|
|\d+ table|查看表中的字段（详细）|
|\q|推出登录|


## pg_dump

- 导出表结构

```bash
pg_dump --host=HOST --username=USERNAME -s -t <TABLE> <DBNAME>
```

|选项|含义|
|-|-|
|-s|只导出表结构|
|-t|table, 跟上表名|



## SQL 

- 修改表字段

```sql
alter table table_name alter field_name type varchar(32) using field_name::varchar(32);
```

> 如果需要发生显式的类型转换，需要增加 USING 关键字。即使不需要发生显式的类型转换，也可以增加 USING 关键字（稳妥）。