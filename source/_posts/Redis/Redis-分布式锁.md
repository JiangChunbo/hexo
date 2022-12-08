---
title: Redis 分布式锁
date: 2022-11-10 23:25:07
tags:
---



## 分布式锁的实现

尝试获得锁使用 `SETNX` 命令，key 可以认为是锁的名字，value 是每个操作者持有的唯一值。

> 锁的名字应该是一个约定好的共识。

```bash
> SETNX lockKey uniqueValue
(integer) 1
> SETNX lockKey uniqueValue
(integer) 0
```

第一个执行 `SETNX` 的请求可以拿到锁。后面的请求都因为 key 已经存在而无法获得锁。

光是 `SETNX` 是不够的，还需要给锁设置一个过期时间，防止某些原因请求线程无法释放锁，因此，通常使用以下方法:
```bash
> SET lockKey uniqueValue EX 3 NX
OK
```


释放锁就是 `DEL` 对应的 key。

```bash
> DEL lockKey
(integer) 1
```

通常使用 Lua 脚本保证释放锁的原子性。因为 Redis 在执行 Lua 脚本时，是原子性的。

```lua
// 释放锁时，先比较锁对应的 value 值是否相等，避免锁的误释放（防止释放了别人的锁）
// 避免误释放的前提是 value 唯一性
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```



设置过期时间又会引入一个问题，如果业务时间大于锁过期时间，产生锁提前过期（释放）问题，此时其他请求可以获得锁。

解决方法: 锁续约。Redission