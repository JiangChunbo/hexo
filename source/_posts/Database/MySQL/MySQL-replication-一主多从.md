---
title: MySQL replication 一主多从
date: 2022-11-18 17:09:57
tags:
---

# 一主多从
## master
**步骤一** 检查配置 my.cnf 必要选项，如果修改了需要重启 mysqld 使参数生效
```bash
[mysqld]
log_bin=mysql-bin
server-id=1000
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
server-id=2000
```

## 配置
**步骤一** 登录 master，查看 master 状态
```bash
show master status;
+------------------------------------+----------+--------------+------------------+-------------------+
| File                               | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------------------------+----------+--------------+------------------+-------------------+
| iZbp13qubkh4obhv5jxw91Z-bin.000001 |      155 | test         |                  |                   |
+------------------------------------+----------+--------------+------------------+-------------------+
```

**步骤二** 登录 slave
```sql
-- 查看是否有残余配置
show slave status\G;
-- 先停止再清除
stop slave;
reset slave all;
-- 根据 master 的状态进行配置
change master to
master_host = '<IP>',
master_port = 13306,
master_user = 'replication',
master_password = 'root',
master_log_file = 'iZbp13qubkh4obhv5jxw91Z-bin.000001',
master_log_pos = 155;
-- 启动 slave
start slave;
show slave status\G;
```
注意 Slave_IO_Running 和 Slave_Slave_Running 需要均显示为 Yes，才表示成功，否则留意错误提示。
