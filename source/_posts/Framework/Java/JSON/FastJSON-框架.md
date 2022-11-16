---
title: FastJSON 框架
date: 2022-10-16 16:33:40
tags:
- JSON
---


# 参考引用

https://github.com/alibaba/fastjson


# Java Bean 字段名称转换

- 类注解

|enum 类型|说明|Java属性名|JSON属性名|
|-|-|-|-|
|CamelCase|驼峰命名,首字母小写|UserName|userName|
|PascalCase|帕斯卡命名,首字母大写|userName|UserName|
|SnakeCase|蛇形命名,大写转小写并以下划线连接|userName|user_name|
|KebabCase|短横线命名,大写转小写并以短横线连接|userName|user-name|

```java
@JSONType(naming = PropertyNamingStrategy.SnakeCase)
```

- 字段注解

```java
@JSONField(name = "student_id")
```

