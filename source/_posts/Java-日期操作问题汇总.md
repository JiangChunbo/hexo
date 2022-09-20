---
title: Java 日期操作问题汇总
date: 2022-09-20 11:24:42
tags:
---


## Date


获得本月第一天

```java
Calendar calendar = Calendar.getInstance();
calendar.set(Calendar.DAY_OF_MONTH, 1);
Date date = calendar.getTime();
```

获得本月最后一天

```java
Calendar calendar = Calendar.getInstance();
calendar.add(Calendar.MONTH, 1);
calendar.set(Calendar.DAY_OF_MONTH, 1);
calendar.add(Calendar.DAY_OF_MONTH, -1);
Date date = calendar.getTime();
```

