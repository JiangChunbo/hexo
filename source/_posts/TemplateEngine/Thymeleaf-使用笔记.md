---
title: Thymeleaf 使用笔记
date: 2022-09-20 20:01:57
tags:
---


## Spring Boot

如果使用 Spring Boot，默认模板引擎的路径是 `classpath:/template`，但是无法直接访问，需要通过控制器 > 解析模板才可以访问。

> 如果没有进行控制器跳转，那么 Spring Boot 将会使用 `ResourceHttpRequestHandler` 来处理你的请求；如果静态路径又找不到对应的资源，则返回 404.
