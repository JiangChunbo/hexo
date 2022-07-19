---
title: Hystrix 配置实践
date: 2022-07-15 11:50:30
tags:
---



涉及到断路器的参数（HystrixCommandProperties）：

|参数|描述|默认值|
|:---|:---|:---|
|circuitBreaker.enabled|是否开启断路器|true|
|circuitBreaker.requestVolumeThreshold|请求总数阈值。这意味着，如果 hystrix 命令在休眠窗口期间调用次数不足 20 次，即使请求都失败，断路器也不会打开|20|
|circuitBreaker.sleepWindowInMilliseconds|休眠窗口|5000|
|circuitBreaker.errorThresholdPercentage|错误阈值。当请求总数在休眠窗口内超过了阈值，断路器就会打开。|50|