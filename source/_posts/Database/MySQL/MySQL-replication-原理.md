---
title: MySQL replication 原理
date: 2022-11-18 17:05:11
tags:
---


# MySQL Replication 实现原理
MySQL复制功能是通过 3 个线程实现的，1 个在复制源服务器上，另 2 个在副本服务器上。

这三个线程都是 start slave 之后，双方才出现的。

通过在 master 机器上查看线程
```bash
mysql> show processlist;
+-------+-----------------+-----------------+-------------+-------------+---------+---------------------------------------------------------------+------------------+
| Id    | User            | Host            | db          | Command     | Time    | State                                                         | Info             |
+-------+-----------------+-----------------+-------------+-------------+---------+---------------------------------------------------------------+------------------+
| 17091 | root            | localhost:60990 | NULL        | Binlog Dump |     408 | Master has sent all binlog to slave; waiting for more updates | NULL             |
+-------+-----------------+-----------------+-------------+-------------+---------+---------------------------------------------------------------+------------------+

```

通过在 slave 机器上查看线程
```bash
mysql> show processlist;
+----+-----------------+-----------------+-------------+---------+------+--------------------------------------------------------+------------------+
| Id | User            | Host            | db          | Command | Time | State                                                  | Info             |
+----+-----------------+-----------------+-------------+---------+------+--------------------------------------------------------+------------------+
|  9 | system user     | connecting host | NULL        | Connect |  138 | Waiting for master to send event                       | NULL             |
| 10 | system user     |                 | NULL        | Query   |   32 | Slave has read all relay log; waiting for more updates | NULL             |
+----+-----------------+-----------------+-------------+---------+------+--------------------------------------------------------+------------------+
```
## Binlog dump thread
binlog 转储线程。Master 创建一个线程，以便在 Slave 连接时将二进制日志内容发送到 Slave。

## Replication I/O thread
在 Slave 服务器上发出 start slave 语句时，Slave 将创建一个I / O线程，该线程连接到 Master 并要求它发送记录在其二进制日志中的更新。

Replication I / O线程读取源 Binlog Dump 线程发送的更新 （请参见上一项），并将它们复制到本地的中继日志文件中。

该线程的状态可以通过 `show slave status` 命令中的  Slave_IO_running 输出得知。 或者， `show status` 的输出Slave_running。


## Replication SQL thread
Slave 创建一个 SQL 线程以读取由 Replication I / O 线程写入的中继日志，并执行其中包含的事件。

在前面的描述中，每个 Replication 连接有三个线程。具有多个 Slave 的 Master 为每个当前连接的 Slave 创建一个二进制日志转储线程，当然，每个 Slave 都有自己的Replication I / O 和 SQL 线程。

