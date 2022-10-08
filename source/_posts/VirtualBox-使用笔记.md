---
title: VirtualBox 使用笔记
date: 2022-09-22 21:43:02
tags:
- VirtualBox
---

## 安装增强的问题

- 安装提示 modprobe vboxguest failed


参考 https://www.cnblogs.com/mychangee/p/12087954.html

```bash
yum install -y kernel-devel gcc //安装kernel-devel和gcc编译工具链
yum -y upgrade kernel kernel-devel //更新kernel和kernel-devel到最新版本
reboot //重启，重启时，选择最新版本的内核启动
```


- 共享粘贴板

Settings > General > Advanced > 共享粘贴板



- 增强功能

鼠标自由切换；
共享粘贴板



## 一些视图显示的问题

一些操作系统，CentOS 本身支持分辨率的调整

View > Adjust Window Size 自适应分辨率





## 参考

https://blog.csdn.net/yuexiaxiaoxi27172319/article/details/121079338

- 使用 VBoxManage 修改

```java
VBoxManage.exe modifyhd "D:\ubuntu18.vdi" --resize 60000
```


- 使用 gparted 调整分区

http://gparted.sourceforge.net/download.php