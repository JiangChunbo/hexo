---
title: SSH 公私钥
date: 2022-11-03 09:58:44
tags:
---


```java
sudo vim /etc/ssh/sshd_config

# ----------------------------------------
# 是否允许 root 远程登录
PermitRootLogin yes

# 密码登录是否打开
PasswordAuthentication yes

# 开启公钥认证
RSAAuthentication yes # 这个参数可能没有 没关系
PubkeyAuthentication yes

# 存放登录用户公钥的文件位置
# 位置就是登录用户名的家目录下的 .ssh
# root 就是 /root/.ssh
# foo 就是 /home/foo/.ssh
AuthorizedKeysFile .ssh/authorized_keys
# ----------------------------------------
```


重启 

```bash
service sshd restart
```

- 客户端生成公私钥

ssh-keygen


- 客户端上传到服务器

```bash
ssh-copy-id -i user/.ssh/id_rsa.pub user@ip
# 或使用scp上传
```