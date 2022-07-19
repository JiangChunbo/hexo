---
title: Spring Cloud Gateway
date: 2022-07-15 16:28:39
categories:
- 框架
tags:
- Spring Cloud
---

# [Spring Cloud Gateway](https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/)

## [1. How to Include Spring Cloud Gateway](https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/#gateway-starter)


```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

如果你包含了 starter，但是你不想启用网关，则可以设置 `spring.cloud.gateway.enabled=false`


## [2. Glossary](https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/#glossary)

- **Route**: 网关的基本构建。它由 ID，目标 URI，谓词集合，以及过滤器集合定义。如果聚合谓词为 true，则匹配路由。
- **Predicate**: 这是 Java 8 Function Predicate。输入类型是 `Spring Framework ServerWebExchange`。这使你可以匹配来自 HTTP 请求中的任何东西，例如 Header 或者参数。
- **Filter**: 这些是一些 `GatewayFilter` 实例，它们由特定的工厂构建出来。在这里，你可以在发送下游请求之前或之后修改请求和响应。


## [4. Configuring Route Predicate Factories and Gateway Filter Factories](https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/#configuring-route-predicate-factories-and-gateway-filter-factories)

配置谓词和过滤器有两种方式：快捷方式和完全扩展参数。下面大多数示例都是用快捷方式。

### [4.1. Shortcut Configuration](https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/#shortcut-configuration)

快捷方式配置由过滤器名称识别，跟着一个等号（`=`），后面跟着由逗号（`,`）分割的参数值。

**application.yml**

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: https://example.org
        predicates:
        - Cookie=mycookie,mycookievalue
```
上面的样例定义了具有两个参数的 `Cookie` 路由谓词工厂，参数分别是，cookie name `mycookie`，以及要匹配 `mycookievalue` 的值。


# [5. Route Predicate Factories](https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/#gateway-request-predicates-factories)

Spring Cloud Gateway 包含许多内置的路由谓词工厂，所有这些谓词匹配不同的 HTTP 请求属性

## [5.1. The After Route Predicate Factory](https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/#the-after-route-predicate-factory)
`After` 路由谓词工厂携带一个参数，即一个 `datetime`（其实是 Java 的 `ZonedDateTime`）。该谓词匹配发生在指定时间之后的请求。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: after_route
        uri: http://www.baidu.com
        predicates:
          - After=2021-06-03T10:00:00.000+08:00
```


## [5.2. The Before Route Predicate Factory](https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/#the-before-route-predicate-factory)
`Before` 路由谓词工厂携带一个参数，即一个 `datetime`（其实是 Java 的 `ZonedDateTime`）。该谓词匹配发生在指定时间之前的请求。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: before_route
        uri: http://www.baidu.com
        predicates:
          - Before=2021-06-03T10:00:00.000+08:00
```

## [5.3. The Between Route Predicate Factory](https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/#the-between-route-predicate-factory)

`Between` 路由谓词工厂携带两个参数，`datetime1` 和 `datetime2`，他们都是 Java 的 `ZonedDateTime` 对象。该谓词匹配发生在 `datetime1` 之后，以及 `datetime2` 之前的请求。`datetime2` 参数必须在 `datetime1` 之后。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: method_route
        uri: http://www.baidu.com
        predicates:
          - Between=2017-01-20T17:42:47.789-07:00[America/Denver], 2017-01-21T17:42:47.789-07:00[America/Denver]
```

## [5.4. The Cookie Route Predicate Factory](https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/#the-cookie-route-predicate-factory)
`Cookie` 路由谓词工厂携带两个参数，`name` 和 `regexp`（本质是 Java 的正则表达式）。该谓词匹配具有给定名称，并且值匹配正则表达式的 cookie。

```yml
spring:
  cloud:
    gateway:
      routes:
        - id: method_route
          uri: https://www.baidu.com
          predicates:
            - Cookie=x-token, \w+
```

## [5.5. The Header Route Predicate Factory](https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/#the-header-route-predicate-factory)
`Header` 路由谓词工厂携带两个参数，`name` 和 `regexp`（本质是 Java 的正则表达式）。该谓词匹配具有给定名称，并且值匹配正则表达式的 header。

```yml
spring:
  cloud:
    gateway:
      routes:
        - id: method_route
          uri: http://www.baidu.com
          predicates:
            - Header=X-Request-Id, \d+
```


## [5.6. The Host Route Predicate Factory](https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/#the-host-route-predicate-factory)
`Header` 路由谓词工厂携带一个参数：主机名 `patterns` 列表。模式是一种 Ant 风格的带有 `.` 作为分隔符的模式。该谓词匹配模式中的的 `Host` 头部。

```yml
spring:
  cloud:
    gateway:
      routes:
        - id: method_route
          uri: http://www.baidu.com
          predicates:
            - Header=X-Request-Id, \d+
```

> 消息头部的字段名是大小写不敏感的，也就是头部 X-Request-Id，也可以传递 x-request-id
> RFC 2616 https://www.rfc-editor.org/rfc/inline-errata/rfc2616.html，查询 (":") 关键字附近文字可以找到：Field names are case-insensitive.


## [5.7. The Method Route Predicate Factory](https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/#the-method-route-predicate-factory)
`Method` 路由谓词工厂携带一个参数 `methods`，该参数是一个或者多个参数：待匹配的 HTTP 方法。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: method_route
        uri: http://www.baidu.com
        predicates:
          - Method=GET,POST
```

## [5.8. The Path Route Predicate Factory](https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/#the-path-route-predicate-factory)

```yml
spring:
  cloud:
    gateway:
      routes:
        - id: method_route
          uri: http://localhost:8080
          predicates:
            - Path=/echo/{segment}
```


## [5.9. The Query Route Predicate Factory](https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/#the-query-route-predicate-factory)
`Query` 路由谓词工厂携带两个参数，一个必需的 `param` 和一个可选的 `regexp`（本质是 Java 的正则表达式）。

```yml
spring:
  cloud:
    gateway:
      routes:
        - id: method_route
          uri: https://www.baidu.com
          predicates:
            - Query=id
```

## [5.10. The RemoteAddr Route Predicate Factory](https://docs.spring.io/spring-cloud-gateway/docs/2.2.9.RELEASE/reference/html/#the-remoteaddr-route-predicate-factory)
`RemoteAddr` 路由谓词工厂携带一个 `sources` 列表，这是 CIDR 符号字符串，例如 `192.168.0.1/16`，此处 `192.168.0.1` 是 IP 地址，16 是子网掩码。

```yml
spring:
  cloud:
    gateway:
      routes:
      - id: remoteaddr_route
        uri: http://www.baidu.com
        predicates:
          - RemoteAddr=192.168.0.187/24
```