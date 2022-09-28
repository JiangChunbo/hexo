---
title: Kubernetes 笔记
date: 2022-09-26 11:06:16
tags:
---

POD 共享相同的 IP 地址和端口空间

同一 POD 中，容器运行的多个进程不能绑定到相同端口号，否则端口冲突。

一个 POD 中，所有容器也都具有相同的 loopback 网络接口，因此容器之间可以通过 localhost 与同一 POD 中的其他容器进行通信

