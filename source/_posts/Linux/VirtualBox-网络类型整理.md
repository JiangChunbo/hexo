---
title: VirtualBox 网络类型整理
date: 2022-10-04 19:01:24
tags:
---


# 网络地址转换NAT – Network Address Translation (NAT)

虚拟机没有自己独立的 IP

虚拟机不存在于真实的网络中、

虚拟机可以访问主机，但是主机无法访问虚拟机


VirtualBox 中 NAT 又分为: NAT 和 NAT Network

NAT 使用内置的网段, NAT Network 可以自定义网段

NAT 网关地址是 10.0.2.1, NAT Network 的网关地址是 x.x.x.1（自定义网段的第一个非零 IP）


> VMWare Station 会添加一个用于 NAT 的适配器，VirtualBox 并不添加 NAT 的适配器，因此 VMWare Station 可以自由设定 NAT 网络的网段，且不需要端口转发就能进行 SSH 连接。


# Host-Only 网络模式

虚拟机不能连接外网（因此不需要设置网关和 DNS）。宿主机和虚拟机们必须在同一网段才能互联。

网络模式为 Host-Only 时，可以在 File（管理） > Host Network Manager（主机网络管理器） 设置宿主机的 IP 等信息。（也可以直接在 Windows 网络连接的 VirtualBox Host-Only Network 适配器中修改。）

# 桥接网卡 – Brdged networking

虚拟机有独立的 IP，就像处于同一个局域网