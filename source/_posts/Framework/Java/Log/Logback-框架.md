---
title: Logback 框架
date: 2022-10-27 09:07:56
tags:
---


### [Configuring loggers, or the `<logger>` element](https://logback.qos.ch/manual/configuration.html#loggerElement)


### [Configuring the root logger, or the `<root>` element](https://logback.qos.ch/manual/configuration.html#rootElement)


`<root>` 元素用于配置根 logger。它支持一个属性，即 *level* 属性。它不允许任何其他属性，因为可相加性标志不应用于根记录器。此外，由于根记录器已经被命名为“root”，它也不允许名称属性。level属性的值可以是不区分大小写的字符串TRACE、DEBUG、INFO、WARN、ERROR、ALL或OFF之一。注意，根记录器的级别不能设置为INHERITED或NULL。