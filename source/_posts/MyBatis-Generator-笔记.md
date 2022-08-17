---
title: MyBatis Generator 笔记
date: 2022-08-16 18:00:58
tags:
---


- 字段名和关键字冲突问题

配置界定符

```xml
<property name="beginningDelimiter" value="`"></property >
<property name="endingDelimiter" value="`"></property >
```

在 table 节点添加属性 `delimitAllColumns="true"`