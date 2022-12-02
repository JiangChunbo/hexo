---
title: SSH 命令
date: 2022-12-02 10:07:38
tags:
---


|Option|含义|
|:---|:---|
|-C|Compression，表示进行数据压缩|
|-f|Fork，命令执行之后进入后台。一般与 -N 一起使用。|
|-N|不执行远端命令。适用于仅仅做端口转发。|
|-g|允许远程主机链接本地转发端口|
|-i|Identity_file，指定私钥文件。适用于公钥认证。|
|-L|Local，将本地端口转发到远程主机端口|
|-R|Remote，将远程主机端口转发到本地端口|


以用户名 user，登录远程主机，指定端口：
```bash
ssh [-p port] user@host
```
将本地主机的给定端口转发到远程主机的给定主机和端口：
```bash
ssh –C –f –N –g –i <identity_file>  –L [bind_address:]port:host:hostport [user@]hostname
```
将远程主机的给定端口转发到本地主机的给定主机和端口：
```bash
ssh –C –f –N –g –i <identity_file> –R [bind_address:]port:host:hostport [user@]hostname
```


远程执行命令:

```bash
ssh root@112.124.1.100 "source /etc/profile; pid=$(ps -ef | grep cloud-config-server-0.0.1-SNAPSHOT.jar | egrep -v grep | awk '{print $2}');kill -9 $pid;java -jar /opt/cannedbread/cloud-config-server-0.0.1-SNAPSHOT.jar > /opt/cannedbread/cloud-config-server-0.0.1-SNAPSHOT.jar.log 2>&1 &"
```