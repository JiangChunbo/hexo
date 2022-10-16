---
title: Java BigDecimal 方法记录
date: 2022-09-29 13:47:06
tags:
---




## 使用字符串构造一个 BigDecimal

```java
BigDecimal big = new BigDecimal("2.3513");
```


## 保留小数

一般比较建议使用枚举类型传参:

```java
// 5 进 1，如：2.65 得到 2.7
big.setScale(1, RoundingMode.HALF_UP);
// 5 不进位, 如: 2.65 得到 2.6
big.setScale(1, RoundingMode.HALF_DOWN);
// 截断多余小数位，如: 2.678 得到 2.6; -5.5 得到 -5
big.setScale(1, RoundingMode.DOWN);
```

下面这种方式，第二个参数为 int
```java
// 默认 java.math.BigDecimal#ROUND_UNNECESSARY
// 这种方法如果小数位大于 1，那么就会抛出异常
big.setScale(1);
// 5 进 1，如：2.65 得到 2.7
big.setScale(1, BigDecimal.ROUND_HALE_UP);
// 5 不进位, 如: 2.65 得到 2.6
big.setScale(1, BigDecimal.ROUND_HALF_DOWN);
// 截断多余小数位，如: 2.678 得到 2.6
big.setScale(1, BigDecimal.ROUND_DOWN);
```


## 判断符号


如果返回 -1，表示负数；
如果返回 0，表示 0；
如果返回 1，表示正数；

```java
big.signum();
```


## 相反数 `negate()`

```java
new BigDecimal(10).negate();
```

## 输出字符串

不使用科学计数法输出

```java
bigDecimal.toPlainString();
```