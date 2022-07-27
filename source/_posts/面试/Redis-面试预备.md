---
title: Redis 面试预备
date: 2022-07-24 18:38:41
tags:
- Redis
---

# Redis 面试预备

## 为什么用 Redis

- 高并发
- 高可用


## 缓存

缓存穿透: 缓存中查不到，数据库中也差不多

解决方案: 1. 对参数进行合法性校验 2. 将数据库中没有查到的也写入到缓存

为了防止缓存垃圾，这一类的缓存可以设置短一些

> BloomFilter ，MySQL 的 id 引入该过滤器。




## 使用场景

1. 缓存


2. 记录用户在线人数


使用 zSet 结构


用户每次调用接口，在鉴权的时候刷新一次该用户 user_id 对应的 score


定时脚本，删除过期的数据:

```php
$redis->zRemRangeByScore(REDIS_ONLINE_USER, 0, time() - 5200);
```