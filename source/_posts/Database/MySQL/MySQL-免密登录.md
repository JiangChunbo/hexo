---
title: MySQL 免密登录
date: 2022-11-28 19:20:04
tags:
---


编辑 my.cnf 配置文件

在 `[mysqld]` 下面添加以下选项:

```ini
skip-grant-tables
```

重新启动 mysql

```bash
systemctl restart mysql
```

直接免密登录

```bash
mysql
```

需要先刷一下权限，否则执行 `ALTER` 你会得到如下错误:

```bash
ERROR 1290 (HY000): The MySQL server is running with the --skip-grant-tables option so it cannot execute this statement
```

修改新密码

```sql
alter user root@localhost identified by 'new_password';
```

删除选项 `skip-grant-tables`，重启服务即可。