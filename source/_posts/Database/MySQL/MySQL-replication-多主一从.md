---
title: MySQL replication 多主一从
date: 2022-11-18 17:13:14
tags:
---

# 多主一从
数据汇集场景，确保场景内表结构一致

## master
**步骤一** 配置 my.cnf 必要选项，如果修改了需要重启 mysqld 使参数生效
```bash
[mysqld]
server-id = 1000
log_bin=mysql-bin
```

**步骤二** 配置 replication 用户
```sql
create user 'replication'@'%' identified with mysql_native_password by 'replication';
grant replication slave on *.* to 'replication'@'%';
flush privileges;
```

## slave
**步骤一** 配置 my.cnf 必要选项，如果修改了需要重启 mysqld 使参数生效
```bash
[mysqld]
server-id = 2000
```

**步骤二** 分别查看多个 master 的状态
```sql
show master status;
+------------------------------------+----------+--------------+------------------+-------------------+
| File                               | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------------------------+----------+--------------+------------------+-------------------+
| iZbp13qubkh4obhv5jxw91Z-bin.000001 |      155 | test         |                  |                   |
+------------------------------------+----------+--------------+------------------+-------------------+
```

**步骤三** 对 slave 进行配置
```sql
-- 查看是否有残余配置
show slave status\G;
-- 先停止再清除
stop slave;
reset slave all;
```
对不同的 master 建立通道，注意 for channel 应当不同
```sql
change master to
master_host = '112.124.1.100',
master_user = 'replication',
master_port = 3306,
master_password = 'replication',
master_log_file = 'mysql-bin.000006',
master_log_pos = 155,
master_connect_retry = 15,
master_retry_count = 0
for channel 'master1';
```
启动通道
```sql
start slave for channel 'master1';
start slave for channel 'master2';
```
查看状态
```sql
show slave status\G;
```
