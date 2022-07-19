---
title: Spring Cloud OpenFeign
date: 2022-07-13 00:35:38
categories:
- 框架
tags:
- Spring Cloud
- OpenFeign
---

# Spring Cloud OpenFeign

## [1. Declarative REST Client: Feign](https://docs.spring.io/spring-cloud-openfeign/docs/2.2.10.RELEASE/reference/html/#spring-cloud-feign)

Feign 是一个声明式 Web 服务客户端。它使得编写 Web 服务客户端更容易。使用 Feign 创建一个接口并注解它。它具有可插入的注解支持，包括 Feign 注解和 JAX-RS 注解。Feign 还支持可插拔的编码器和解码器。Spring Cloud 增加了对 Spring MVC 注解的支持，以及支持使用 Spring Web 中默认项相同的 `HttpMessageConverters`。Spring Cloud 集成了 Ribbon 以及 Eureka，Spring Cloud CiruitBreaker，以及 Spring Cloud LoadBalancer，以便在使用 Feign 时提供负载均衡的 HTTP 客户端。


### [1.1. How to Include Feign](https://docs.spring.io/spring-cloud-openfeign/docs/2.2.10.RELEASE/reference/html/#netflix-feign-starter)

要在项目中包含 Feign，请使用 group 为 `org.springframework.cloud`，artifact id 为 `spring-cloud-starter-openfeign` 的 starter。有关使用当前 Spring Cloud Release Train 设置你的构建系统的详情，参见 [Spring Cloud Project page](https://projects.spring.io/spring-cloud/)

示例 Spring Boot 应用：

```java
@SpringBootApplication
@EnableFeignClients
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}
```

**StoreClient.java**

```java
@FeignClient("stores")
public interface StoreClient {
    @RequestMapping(method = RequestMethod.GET, value = "/stores")
    List<Store> getStores();

    @RequestMapping(method = RequestMethod.GET, value = "/stores")
    Page<Store> getStores(Pageable pageable);

    @RequestMapping(method = RequestMethod.POST, value = "/stores/{storeId}", consumes = "application/json")
    Store update(@PathVariable("storeId") Long storeId, Store store);
}
```

在 `@FeignClient` 注解中，字符串 value（即上面的 "stores"）是一个任意的 client name，用于创建一个 Ribbon 负载均衡或者 Spring Cloud LoadBalancer。你还可以使用 `url` 属性（绝对值或者一个主机名）指定一个 URL。应用上下文中 Bean 的名称是接口的完全限定名。为了指定你自己的别名，你可以使用 `@FeignClient` 注解的 `qualifiers` 值。

上面的负载均衡客户端将会希望发现 "stores" 服务的物理地址。如果你的应用是 Eureka 客户端，则它将在 Eureka 服务注册表中解析该服务。如果你不想使用 Eureka，则可以使用 `SimpleDiscoveryClient` 在外部配置中简单地配置服务器列表。


### [1.2. Overriding Feign Defaults](https://docs.spring.io/spring-cloud-openfeign/docs/2.2.10.RELEASE/reference/html/#spring-cloud-feign-overriding-defaults)

Spring Cloud 的 Feign 支持中的一个核心概念就是有名客户端。每个 Feign 客户端都是一群组件的部分，它们共同协作按需与远程服务进行连接，并且这个整体有一个你作为开发者使用 `@FeignClient` 注解赋予的名字。Spring Cloud 使用 `FeignClientsConfiguration` 为每个有名客户端按需地，以 `ApplicationContext` 的形式创建一个全新的整体。这包含一个 `feign.Decoder`，一个 `feign.Encoder`，以及一个 `feign.Contract`。可以使用 `@FeignClient` 注解的 `contextId` 属性覆盖该整体的名称。

Spring Cloud 使你可以使用 `@FeignClient` 声明额外的配置（覆盖 `FeignClientsConfiguration`）完全控制 feign 客户端。

示例：

```java
@FeignClient(name = "stores", configuration = FooConfiguration.class)
public interface StoreClient {
    //..
}
```

在这种情况下，客户端是由已经存在于 `FeignClientsConfiguration` 中的组件，以及 `FooConfiguration` （后者覆盖前者）中的种种组成。

> `FooConfiguration` 不需要 `@Configuration` 注解。但是，如果这样做，请将其从任何 `@ComponentScan` 中排除，否则将包含此配置，因为它将将成为 `feign.Decoder`，`feign.Encoder`，`feign.Contract` 等默认源。


`name` 和 `url` 属性也支持占位符。


#### [1.2.1. SpringEncoder configuration](https://docs.spring.io/spring-cloud-openfeign/docs/2.2.10.RELEASE/reference/html/#springencoder-configuration)

在我们提供的 `SpringEncoder` 中，我们为二进制内容类型设置了 `null` 字符集，并未所有其他的设置了 `UTF-8`。

你可以通过设置 `feign.encoder.charset-from-content-type` 的值为 `true`来修改此行为，以从 `Content-Type` 头部 charset 获得字符集。

### [1.3. Timeout Handling](https://docs.spring.io/spring-cloud-openfeign/docs/2.2.10.RELEASE/reference/html/#timeout-handling)

我们可以在默认和有名客户端上配置超时。OpenFeign 可与两个超时参数一起使用：

- `connectTime`
- `readTimeout`


### [1.5. Feign Hystrix Support](https://docs.spring.io/spring-cloud-openfeign/docs/2.2.10.RELEASE/reference/html/#spring-cloud-feign-hystrix)

如果 Hystrix 在类路径，并且 `feign.hystrix.enabled=true`，Feign 将用熔断器包装所有方法。还提供返回一个 `com.netflix.hystrix.HystrixCommand`。这使你可以使用响应式模式。


### [1.6. Feign Hystrix Fallbacks](https://docs.spring.io/spring-cloud-openfeign/docs/2.2.10.RELEASE/reference/html/#spring-cloud-feign-hystrix-fallback)


### [1.10. Feign Inheritance Support](https://docs.spring.io/spring-cloud-openfeign/docs/2.2.10.RELEASE/reference/html/#spring-cloud-feign-inheritance)

Feign 通过单继承接口支持样板 API。这允许你将通用操作分组为方便的基础接口。

**UserService.java**

```java
public interface UserService {

    @RequestMapping(method = RequestMethod.GET, value ="/users/{id}")
    User getUser(@PathVariable("id") long id);
}
```

**UserResource.java**

```java
@RestController
public class UserResource implements UserService {

}
```

**UserClient.java**

```java
package project.user;

@FeignClient("users")
public interface UserClient extends UserService {

}
```

### [1.3. Timeout Handling](https://docs.spring.io/spring-cloud-openfeign/docs/2.2.10.RELEASE/reference/html/#timeout-handling)

我们可以在默认和有名客户端上配置 timeout 属性。OpenFeign 用这两个 timeout 参数进行工作：

- `connectTimeout` 防止由于较长的服务端处理时间导致的阻塞。
- `readTimeout` 在连接建立的时候使用，并且当返回响应花费太久的时候会触发


> 如果服务没有运行或者不可用，数据包可能以连接被拒绝结束。通信以错误信息或者后背方式结束。如果 `connectTimeout` 设置非常低，这就有可能在 `connectTimeout` 之前就结束通信。执行查找以及接收这样的数据包的时间可能会产生较大一部分延迟。可以基于涉及到 DNS 查找的远程主机修改该值。

当启用 Hystrix 后，超时配置默认是 1000 毫秒。因此，它可能发生在我们前面配置的客户端超时之前。增加该超时，防止发生这种情况。

```yml
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 5000
  hystrix:
    enabled: true

hystrix:
  command:
    default:
      execution:
        timeout:
          enabled: true
        isolation:
          thread:
            timeoutInMilliseconds: 60000
```

> 当启用 Hystrix 的 timeout 且它的 timeout 设置得比 feign client 更长时，`HystrixTimeoutException` 会包装一个 feign 异常。否则，唯一的区别就是异常的原因。`HystrixTimeoutException` 的目的是包装先发生的任何运行时异常，并抛出自身实例。


### [1.5. Feign Hystrix Support](https://docs.spring.io/spring-cloud-openfeign/docs/2.2.10.RELEASE/reference/html/#spring-cloud-feign-hystrix)

如果 Hystrix 在类路径中，且 `feign.hystrix.enabled=true`，Feign 将使用熔断器包装所有的方法。还可以返回一个 `HystrixCommand`。


### [1.12. Feign logging](https://docs.spring.io/spring-cloud-openfeign/docs/2.2.10.RELEASE/reference/html/#feign-logging)

为每个 Feign 客户端创建了一个 logger。默认情况下，logger 的名字是用于创建 Feign 客户端的接口的完全限定类名。Feign 日志仅仅响应 `DEBUG` 级别。


**application.yml**
```properties
logging.level.project.user.UserClient: DEBUG
```

你可以为每个客户端配置的 `Logger.Level` 对象，告诉 Feign 多少要记录。选项如下：

- `NONE`，不记录日志（DEFAULT）
- `BASIC`，仅仅记录请求方法，URL，响应状态码，以及执行时间
- `HEADERS`，记录基本信息，以及请求头和响应头
- `FULL`，记录来自请求或响应的头部


### [1.13. Feign @QueryMap support](https://docs.spring.io/spring-cloud-openfeign/docs/2.2.10.RELEASE/reference/html/#feign-querymap-support)

