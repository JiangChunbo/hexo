---
title: VirtualBox CentOS 设置静态 IP
date: 2022-09-26 22:05:30
tags:
- Virtualization
---


# 参考指引

https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/7/html-single/networking_guide/index#sec-Configuring_IP_Networking_with_ifcg_Files

关于 PREFIX 和 NETMASK https://forums.centos.org/viewtopic.php?t=45322


# 步骤


## 可选步骤

如果你使用 Host-Only 网卡模式，打开主机网络管理器，配置适配器:

<img src="https://img-blog.csdnimg.cn/5eb37e372aee468cb094b89df937ffa4.png">

点击创建新的适配器:

<img src="https://img-blog.csdnimg.cn/4ab177c13a7b49a9919bf2104f9a45fc.png">


创建完毕之后，宿主机的适配器也会产生新的项目:

<img src="https://img-blog.csdnimg.cn/5c6d03f3b8cd45e382bf6b7b825e2612.png">


配置相关的属性，比如手动配置适配器的 IP 地址，子网掩码，DHCP 服务器等: 

<img src="https://img-blog.csdnimg.cn/acda068b13f14def828f111a8fdebe25.png">

> 此处可以不启用 DHCP 服务，因为虚拟机选择的是静态 IP


## 关键步骤

1. 编辑网卡配置文件

```bash
vim /etc/sysconfig/network-scripts/ifcfg-eth1
```

> 需要确认自己 Host-Only 的网卡文件是哪一个，然后编辑它


2. 编辑配置

```properties
DEVICE=eth1
BOOTPROTO=static
ONBOOT=yes
PREFIX=24
IPADDR=192.168.101.101
```
> - 比如上图的适配器是 192.168.101.1，为了让虚拟机和宿主机在同一个网段，我们也需要配置相同网段的 `IPADDR`
> - 仅需要上面这几个配置就足够了


3. 重启 network 服务

```bash
systemctl restart network
```