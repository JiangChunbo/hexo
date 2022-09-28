---
title: Docker 笔记
date: 2022-09-24 22:21:01
tags:
---


# 安装
[官方安装参考](https://docs.docker.com/engine/install/centos/)

**步骤一** 卸载旧版本
```bash
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

**步骤二** 配置 yum 并安装 docker
```bash
sudo yum install -y yum-utils
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce docker-ce-cli containerd.io
```

**步骤三** 启动
```bash
sudo systemctl start docker
```

## 阿里云镜像加速
https://cr.console.aliyun.com/cn-hangzhou/instances/repositories


# 命令
## `docker images`
如果只需要输出特定镜像信息，则使用 `docker images <REPOSITORY>` 即可
```bash
$ docker images
REPOSITORY    TAG       IMAGE ID       CREATED         SIZE
mysql         5.6       eb4e842271a4   4 days ago      303MB
```

|字段|含义|
|:---|:---|
|REPOSITORY|镜像仓库源|
|TAG|镜像的标签||
|IMAGE ID|镜像 ID|
|CREATED|创建时间|
|SIZE|镜像大小|

1. 同一个仓库源可以有多个 TAG，代表仓库源的不同版本

2. 如果不指定一个镜像的版本 TAG，默认使用 latest 镜像


## `docker search`
从 docker hub 上搜索

列出收藏数不小于 10 的镜像数
```bash
docker search -f stars=10 java
# 旧版 docker 命令
docker search -s 10 java
```

## `docker pull`
拉取镜像

`docker pull <repository>:<tag>`

如果不指定一个镜像的版本 TAG，默认使用 latest 镜像


## `docker rmi`
删除镜像，支持多个参数
`docker rmi <repository>:<tag> [<repository>:<tag> ...]`

如果不指定一个镜像的版本 TAG，默认使用 latest 镜像。所以，想删除 mysql:5.6，使用 `docker rmi mysql:5.6`。

如果有容器正在运行，且需要删除，则使用 `-f` 参数


## `docker run`
创建容器并启动运行
`docker run [OPTIONS] IMAGE [COMMAND] [ARG...]`

|选项|含义|
|:---|:---|
|--name="容器名"|为容器指定一个名称，未指定会随机生成|
|-d|后台运行|
|-i|交互模式，通常和 -t 一起使用|
|-t|为容器分配一个伪输入终端|
|-P|随机端口映射|
|-p|指定端口映射|


## `docker ps`
|选项|含义|
|:---|:---|
|-a|列出所有的的容器，包括正在运行的和已经退出的|
|-l|列出最近创建的容器|
|-n|显示最近创建的 n 个容器|
|-q|静默模式，只显示容器编号|
|--no-trunc|不截断输出|

## `docker start`
docker start <CONTAINER ID | NAMES>

## `退出`
exit 关闭离开

CTRL + P + Q 保持运行离开

## `docker restart`

docker restart <CONTAINER ID | NAMES>


## `docker stop`
docker stop <CONTAINER ID | NAMES>

## `docker kill`
docker kill <CONTAINER ID | NAMES>

## `docker rm`
docker rm <CONTAINER ID | NAMES>

一次性删除多个容器
```bash
docker rm -f $(docker ps -a -q)
docker ps -a -q | xargs docker rm
```

## `docker logs`

显示日志

docker logs -f -t --tail <CONTAINER ID>

|选项|含义|
|---|---|
|-t|显示时间|
|-f|追加|
|-tail n|显示倒数 n 行|

## `docker top`
查看容器进程

docker top <CONTAINER ID | NAMES>

## `docker inspect`
审查容器

## `docker exec`
在容器中打开新的终端，并且可以启动新的进程

语法格式 docker exec -t <CONTAINER ID | NAMES>

## `docker cp`
docker cp <CONTAINER ID | NAMES>:<容器内路径> <目的主机路径>

## `docker commit`

提交容器副本使之成为一个新的镜像

`docker commit -m="提交的描述信息" -a="作者名" <CONTAINER ID | NAMES> <NEW NAMES>:[标签名]`



## `docker exec`

进入容器

docker exec -it b7a9f5eb6b85 sh