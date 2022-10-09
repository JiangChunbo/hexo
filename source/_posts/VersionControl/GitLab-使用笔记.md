---
title: GitLab 使用笔记
date: 2022-09-22 21:27:54
tags:
---


# 参考文档

https://docs.gitlab.com/runner/register/

https://docs.gitlab.com/ee/ci/yaml/gitlab_ci_yaml.html

# Docker 安装

异步启动

```bash
sudo docker run \
    --detach \
    --hostname gitlab.mczaiyun.top \
    --publish 443:443 \
    --publish 80:80 \
    --publish 222:22 \
    --name gitlab \
    --restart always \
    --volume /srv/gitlab/config:/etc/gitlab \
    --volume /srv/gitlab/logs:/var/log/gitlab \
    --volume /srv/gitlab/data:/var/opt/gitlab \
    gitlab/gitlab-ce:latest

``` 

--detach   后台运行
--hostname 指定服务域名
--publish 映射端口
--name gitlab
--restart 自动重启，docker 启动的时候，容器也会跟着启动
--volume 目录映射



默认账号 root，默认密码存放在 `/etc/gitlab/initial_root_password`，该文件有时效性，需要及时修改



```bash
sudo docker run \
    -d \
    --name gitlab-runner \
    --restart always \
    -v /srv/gitlab-runner/config:/etc/gitlab-runner \
    -v /var/run/docker.sock:/var/run/docker.sock \
    gitlab/gitlab-runner:latest
```




## Gitlab 修改 root 初始密码且忘记密码

```bash
# gitlab-rails console -e production
 
 
irb(main):003:0> User.all
 
=> #<ActiveRecord::Relation [#<User id:1 @root>]>
 
irb(main):004:0> user=User.where(id:1).first
 
=> #<User id:1 @root>
 
irb(main):008:0> user.password='12345678'
 
=> "12345678"
 
irb(main):009:0> user.password_confirmation='12345678'
 
=> "12345678"
 
irb(main):010:0> user.save!
 
=> true
```

## Docker 容器中修改 Gitlab 的 hostname

1. 关闭 docker


2. 进入容器的配置文件夹，修改 config.v2.json、hostname、hosts 文件
