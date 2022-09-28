---
title: Linux 磁盘笔记
date: 2022-09-22 22:02:38
tags:
---

```bash
df -h
[root@linux30 ~]# df -h
文件系统                 容量  已用  可用 已用% 挂载点
devtmpfs                 1.9G     0  1.9G    0% /dev
tmpfs                    1.9G     0  1.9G    0% /dev/shm
tmpfs                    1.9G   12M  1.9G    1% /run
tmpfs                    1.9G     0  1.9G    0% /sys/fs/cgroup
dev/mapper/centos-root   17G   17G  246M   99% /
/dev/sda1               1014M  195M  820M   20% /boot
tmpfs                    378M     0  378M    0% /run/user/0
```