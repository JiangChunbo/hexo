---
title: Linux 设置静态 IP
date: 2022-09-26 22:05:30
tags:
- Linux
---


1. 编辑网卡配置

vi /etc/sysconfig/network-scripts/ifcfg-ens33


2. 编辑配置

```bash
ONBOOT=yes
BOOTPROTO=static
IPADDR=192.168.59.59
NETMASK=255.255.255.0
GATEWAY=192.168.59.2
DNS1=8.8.8.8
```


3. 重启网络 

```bash
systemctl restart network
```