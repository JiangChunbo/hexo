---
title: List Array String 转换操作注意
date: 2022-10-26 13:52:02
tags:
---

String: 通常使用 String 类型表示多元素，一般可能考虑界定符分隔元素。如: "1,2,3"，其中, "," 表示界定符。


- 界定符字符串 > Array


假设有一个字符串标识一组 ID，名称为 `ids`，

```java
String[] idArray = ids.split(",");
```

> 注意，`ids` 如果为空字符串，会得到一个包含空字符的数组。因此可能需要判断是否为空。

```java
String[] idArray = ids != null && ids.trim().length() > 0 ? ids.split(",") : new String[0];
```

如果还希望转换为 `Integer[]` 或者 `Long[]` 等:
l
```java
Arrays.stream(idArray).map(Integer::parseInt).toArray(Integer[]::new)
```