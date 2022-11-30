---
title: MySQL 锁的观测方式
date: 2022-11-30 13:27:53
tags:
- MySQL
---

# 0. 参考引用

[The data_locks Table](https://dev.mysql.com/doc/refman/8.0/en/performance-schema-data-locks-table.html)


# 观察锁



MySQL 8.0 主要通过表 `performance_schema` 下的 `data_locks` 和 `data_lock_waits` 观察锁。


## `data_locks`

`data_locks` 显示的是持有和请求的数据所。


```sql
mysql> show create table performance_schema.data_locks\G
*************************** 1. row ***************************
       Table: data_locks
Create Table: CREATE TABLE `data_locks` (
  `ENGINE` varchar(32) NOT NULL,
  `ENGINE_LOCK_ID` varchar(128) NOT NULL,
  `ENGINE_TRANSACTION_ID` bigint unsigned DEFAULT NULL,
  `THREAD_ID` bigint unsigned DEFAULT NULL,
  `EVENT_ID` bigint unsigned DEFAULT NULL,
  `OBJECT_SCHEMA` varchar(64) DEFAULT NULL,
  `OBJECT_NAME` varchar(64) DEFAULT NULL,
  `PARTITION_NAME` varchar(64) DEFAULT NULL,
  `SUBPARTITION_NAME` varchar(64) DEFAULT NULL,
  `INDEX_NAME` varchar(64) DEFAULT NULL,
  `OBJECT_INSTANCE_BEGIN` bigint unsigned NOT NULL,
  `LOCK_TYPE` varchar(32) NOT NULL,
  `LOCK_MODE` varchar(32) NOT NULL,
  `LOCK_STATUS` varchar(32) NOT NULL,
  `LOCK_DATA` varchar(8192) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci DEFAULT NULL,
  PRIMARY KEY (`ENGINE_LOCK_ID`,`ENGINE`),
  KEY `ENGINE_TRANSACTION_ID` (`ENGINE_TRANSACTION_ID`,`ENGINE`),
  KEY `THREAD_ID` (`THREAD_ID`,`EVENT_ID`),
  KEY `OBJECT_SCHEMA` (`OBJECT_SCHEMA`,`OBJECT_NAME`,`PARTITION_NAME`,`SUBPARTITION_NAME`)
) ENGINE=PERFORMANCE_SCHEMA DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

字段说明：

|字段名|说明|
|:---|:---|
|ENGINE|持有或者请求锁的存储引擎|
|ENGINE_LOCK_ID|存储引擎持有或请求的锁 ID。元组 (ENGINE_LOCK_ID, ENGINE) 是唯一的。|
|ENGINE_TRANSACTION_ID|存储引擎请求锁的事务 ID（锁的持有者）|
|THREAD_ID|线程 ID|
|EVENT_ID|造成锁的 EVENT_ID|
|OBJECT_SCHEMA|对应锁表的 schema 名称（数据库名）|
|OBJECT_NAME|锁定表的名称。|
|PARTITION_NAME|锁定分区的名称（如果有），否则 NULL。|
|SUBPARTITION_NAME|对应锁的子分区名|
|INDEX_NAME|锁定索引的名称（如果有），否则 NULL。<br><br>实际上，InnoDB 始终创建索引（GEN_CLUST_INDEX），因此对于 InnoDB 表，INDEX_NAME 非 NULL|
|OBJECT_INSTANCE_BEGIN|锁在内存中的地址。|
|LOCK_TYPE|锁的类型。<br><br>取值与存储引擎相关。对 InnoDB 而言，允许的值是用于行级的锁 RECORD，以及用于表级的锁 TABLE。|
|LOCK_MODE|锁是如何被请求的。<br><br>取值与存储引擎相关。对于 InnoDB，允许的值是 S[,GAP], X[,GAP], IS[,GAP], IX[,GAP], AUTO_INC, UNKNOWN。除 AUTO_INC 和 UNKNOWN 以外的锁模式表示间隙锁，如果存在。|
|LOCK_STATUS|请求锁的状态。<br><br>取值与存储引擎相关。对于 InnoDB，允许的值是 GRANTED（锁被持有）以及 WAITING（锁被等待）。|
|LOCK_DATA|与锁相关的数据。|





## LOCK_MODE 和 LOCK_DATA

以下表格说明了不同 LOCK_MODE 下 LOCK_DATA 表示的含义:

|LOCK_MODE|LOCK_DATA|锁住的数据范围|
|:-|:-|:-|
|X,REC_NOT_GAP|15|索引值为 15 的行锁|
|X,GAP|15|索引值为 15 之前的那个间隙（不包括 15）|
|X|15|索引值为 15 之前的那个间隙（包括 15）|


&nbsp;
# 实践
以 directory 表作为测试对象，字段 `id` 是主键
```sql
mysql> show create table directory\G
*************************** 1. row ***************************
       Table: directory
Create Table: CREATE TABLE `directory` (
  `id` int NOT NULL AUTO_INCREMENT COMMENT '主键',
  `book_id` int NOT NULL COMMENT 'book ID',
  `level` tinyint NOT NULL COMMENT '层级',
  `parent_id` int NOT NULL COMMENT '上级',
  `title` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '章节名称',
  `weight` int NOT NULL COMMENT '权重',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `is_del` tinyint NOT NULL DEFAULT '0' COMMENT '是否删除',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新名称',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=16 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci
```

&nbsp;
**session1**

开启一个事务，执行 UPDATE 操作，查询 data_locks 表，存储引擎获取了 question_bank.directory 的 IX 锁，之后再获取了 X 记录锁。

> 必须先获得意向锁，才能获得记录锁（行锁）。

```sql
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update directory set weight=10 where id = 3;
Query OK, 0 rows affected (0.00 sec)
Rows matched: 1  Changed: 0  Warnings: 0

mysql> select * from performance_schema.data_locks\G
*************************** 1. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140490196079640:1278:140490125172848
ENGINE_TRANSACTION_ID: 215652
            THREAD_ID: 148803
             EVENT_ID: 20
        OBJECT_SCHEMA: question_bank  --> 数据库名
          OBJECT_NAME: directory  ---> 表名
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: NULL
OBJECT_INSTANCE_BEGIN: 140490125172848
            LOCK_TYPE: TABLE ---> 表锁
            LOCK_MODE: IX  ---> 意向排他锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: NULL  ---> 因为是表锁，故没有指定锁数据
*************************** 2. row ***************************
               ENGINE: INNODB
       ENGINE_LOCK_ID: 140490196079640:221:4:4:140490125169856
ENGINE_TRANSACTION_ID: 215652
            THREAD_ID: 148803
             EVENT_ID: 20
        OBJECT_SCHEMA: question_bank
          OBJECT_NAME: directory
       PARTITION_NAME: NULL
    SUBPARTITION_NAME: NULL
           INDEX_NAME: PRIMARY
OBJECT_INSTANCE_BEGIN: 140490125169856
            LOCK_TYPE: RECORD
            LOCK_MODE: X,REC_NOT_GAP  ---> 记录锁
          LOCK_STATUS: GRANTED
            LOCK_DATA: 3  ---> 索引值 3
2 rows in set (0.00 sec)
```
