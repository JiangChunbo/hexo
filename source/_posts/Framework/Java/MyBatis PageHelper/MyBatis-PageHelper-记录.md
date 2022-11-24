---
title: MyBatis PageHelper 记录
date: 2022-11-23 08:45:41
tags:
---

# 0. 参考引用

https://github.com/pagehelper/Mybatis-PageHelper
https://github.com/pagehelper/Mybatis-PageHelper/blob/master/wikis/zh/HowToUse.md


# 如何使用?


## 配置插件


```xml
<plugins>
    <plugin interceptor="com.github.pagehelper.PageInterceptor">
    </plugin>
</plugins>
```

## 代码使用

官方推荐的方式:

```java
PageHelper.startPage(1, 10);
List<User> list = userMapper.selectIf(1);

PageHelper.offsetPage(1, 10);
List<User> list = userMapper.selectIf(1);
```
