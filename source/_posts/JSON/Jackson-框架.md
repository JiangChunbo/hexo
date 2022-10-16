---
title: Jackson 框架
date: 2022-10-16 16:39:03
tags:
- JSON
---

# 参考引用



# Java Bean 字段名称转换



- 类注解

|Class|说明|Java属性名|JSON属性名|
|-|-|-|-|
|CamelCase|驼峰命名,首字母小写|personId|persionId|
|PascalCase|帕斯卡命名,首字母大写|personId|PersonId|
|SnakeCase|蛇形命名,大写转小写并以下划线连接|personId|person_id|
|KebabCase|短横线命名,大写转小写并以短横线连接|personId|person-id|

```java
@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)
```

- 字段注解

```java
@JSONField(name = "student_id")
```
