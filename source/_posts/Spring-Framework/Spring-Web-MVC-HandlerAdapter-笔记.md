---
title: Spring Web MVC HandlerAdapter 笔记
date: 2022-07-23 08:15:49
tags:
- Spring Framework
---

# Spring Boot
Spring Boot 默认的 HandlerAdapter 有：

- `RequestMappingHandlerAdapter`，注解 `@RequestMapping`
- `HandlerFunctionAdapter`，端点
- `HttpRequestHandlerAdapter`，用于一些静态资源
- `SimpleControllerHandlerAdapter`，用于 `Controller` 接口

> `AnnotationMethodHandlerAdapter` 是之前使用的一个，现在已经废弃

这些实现类需要关注的就是它们的方法：
```java
boolean supports(Object handler);
```


## RequestMappingHandlerAdapter

`RequestMappingHandlerAdapter` 是 `AbstractHandlerMethodAdapter` 的子类，父类也已经实现了 `supports()` 方法：

```java
public final boolean supports(Object handler) {
	return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
}
```

该方法的返回值由两个条件决定：

- 第一个是 handler 是否是 `HandlerMethod` 的实例。如果之前的 HandlerMapping 是 `RequestMappingHandlerMapping`，那么它返回的一定是 `HandlerMethod`。
- 第二个是由子类实现的 `supportsInternal()` 方法，不过 `RequestMappingHandlerAdapter` 始终返回 `true`


## HttpRequestHandlerAdapter

一般用于适配静态资源请求