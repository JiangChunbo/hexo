---
title: Spring Cloud Netflix
date: 2022-07-12 16:26:07
categories:
- 框架
tags:
- Spring Cloud
- Netflix
---

# [Spring Cloud Netflix](https://docs.spring.io/spring-cloud-netflix/docs/2.2.10.RELEASE/reference/html/)

## [1. Service Discovery: Eureka Clients](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#service-discovery-eureka-clients)
服务发现是微服务架构的关键原则之一。手工配置每个客户端或基于某种约定是非常脆弱的。Eureka 是 Netflix 服务发现服务端和客户端。服务可以被配置和部署，变得高可用，每个服务都会复制其他已经注册服务的状态。

### [1.1. How to Include Eureka Client](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#netflix-eureka-client-starter)
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

### [1.2. Registering with Eureka](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#registering-with-eureka)
当 Eureka Client 注册 Eureka Server 时，它会提供自己的元数据，比如：主机，端口，健康指示器 URL，主页等详细信息。Eureka Server 接收属于某个服务的每个实例的心跳消息。如果心跳在可配置的时间表上失败，则通常从注册表中删除实例。

以下实例展示了最小的 Eureka Client 应用：

```java
@SpringBootApplication
@RestController
public class Application {

    @RequestMapping("/")
    public String home() {
        return "Hello world";
    }

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}
```


请注意，前面的示例展示了一个普通的 Spring Boot 应用。通过包含 `spring-cloud-starter-netflix-eureka-client` 于类路径，你的应用会自动注册到 Eureka Server。需要配置的就是定位 Eureka Server，如下示例所示：

```yml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

在前面的示例中，`defaultZone` 是一个魔术字符串备用值，它为任何没有表示首选项的客户端提供服务 URL（换句话说，这是一个有用的默认值）。

> `defaultZone` 属性是大小写敏感，并且需要驼峰格式，因为 `serviceUrl` 属性是一个 `Map<String, String>`。因此，`defaultZone` 属性不遵循通常的 Spring Boot 蛇形约定 `default-zone`。


默认的应用名（即，服务 ID），虚拟机主机名，以及非安全端口（从 `Environment` 中获取）分别是 `${spring.application.name}`，`${spring.application.name}`，以及 `${server.port}`。


### [1.3. Authenticating with the Eureka Server](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#authenticating-with-the-eureka-server)
如果 eureka.client.serviceUrl.defaultZone 之一有凭证嵌入其中，则会自动把 HTTP basic 认证添加到 eureka 客户端。

**注意** 这里并不是给 URL 添加凭证，只是激活 HTTP basic 用于请求的认证。


### [1.4. Status Page and Health Indicator](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#status-page-and-health-indicator)

### [1.5. Registering a Secure Application](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#registering-a-secure-application)

如果你的应用希望通过 HTTPS 连接，你可以在 `EurekaInstanceConfig` 设置两个标记：

- `eureka.instance.[nonSecurePortEnabled]=[false]`
- `eureka.instance.[securePortEnabled]=[true]`

### [1.6. Eureka’s Health Checks](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#eurekas-health-checks)

### [1.7. Eureka Metadata for Instances and Clients](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#eureka-metadata-for-instances-and-clients)

#### [1.7.1. Using Eureka on Cloud Foundry](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#using-eureka-on-cloud-foundry)
#### [1.7.2. Using Eureka on AWS](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#using-eureka-on-aws)
如果你计划将应用程序部署到 AWS 云上，则必须将 Eureka 实例配置为 AWS-aware。你可以通过自定义 `EurekaInstanceConfigBean` 来做到这一点，如下所示：
```java
@Bean
@Profile("!default")
public EurekaInstanceConfigBean eurekaInstanceConfig(InetUtils inetUtils) {
  EurekaInstanceConfigBean b = new EurekaInstanceConfigBean(inetUtils);
  AmazonInfo info = AmazonInfo.Builder.newBuilder().autoBuild("eureka");
  b.setDataCenterInfo(info);
  return b;
}
```

### [1.7.3. Changing the Eureka Instance ID](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#changing-the-eureka-instance-id)
一般地，NetFlix Eureka 实例以 host name 注册为其主机名。Spring Cloud Eureka 提供了一个明确的默认值：
```php
${spring.cloud.client.hostname}:${spring.application.name}:${spring.application.instance_id:${server.port}}
```
例如：`myhost:myappname:8080`

在 Spring Cloud 中，你可以通过提供唯一标识符 `eureka.instance.instanceId` 来覆盖该值：
```yml
eureka:
  instance:
    instanceId: ${spring.application.name}:${vcap.application.instance_id:${spring.application.instance_id:${random.value}}}
```

### [1.8. Using the EurekaClient](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#using-the-eurekaclient)

一旦你拥有一个发现客户端的应用程序，你可以使用它从 Eureka Server 发现服务实例。这样做的一种方法是使用本地 `com.netflix.discovery.EurekaClient`

## [1.9. Alternatives to the Native Netflix EurekaClient](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#alternatives-to-the-native-netflix-eurekaclient)

你无需使用原生的 NetFlix 的 `EurekaClient`。而且，在某种形式的包装之后通常会更容易使用它。通过逻辑 eureka 服务标识符（VIPs）而不是物理 URL，Spring Cloud 支持 Feign（一个 REST 客户端构建器）以及 Spring RestTemplate。


## [1.10. Why Is It so Slow to Register a Service?](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#why-is-it-so-slow-to-register-a-service)

作为一个实例，还涉及到向注册中心（通过 client 的 serviceUrl）的定期心跳，默认时间为 30 秒。一个服务，直到该实例，注册中心，客户端缓存中都有相同的元数据（因此需要 3 个心跳），客户端才能发现其不可用。你可以通过设置 `eureka.instance.leaseRenewalIntervalInSeconds` 来更改周期。将其设置为小于 30 的值，可以加快客户端连接到其他服务的进程。在生产中，由于注册中心的内部计算对租赁续期做了假设，因此最好坚持使用默认值。

# [2. Service Discovery: Eureka Server](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#spring-cloud-eureka-server)
[参考配置](https://gitee.com/jiang_chun_bo/cannedbread-parent/tree/master/cannedbread-eureka-server)
## [2.1. How to Include Eureka Server](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#netflix-eureka-server-starter)
要在你的项目中包含 Eureka Server，请使用 group ID 为 `org.springframework.cloud` 以及 artifact ID 为 `spring-cloud-starter-netflix-eureka-server` 的 starter。参见 [Spring Cloud Project page](https://projects.spring.io/spring-cloud/)


## [2.2. How to Run a Eureka Server](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#spring-cloud-running-eureka-server)
服务端有一个主页和 HTTP API （在 /eureka/* 下）。

&nbsp;
## [2.3. High Availability, Zones and Regions](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#spring-cloud-eureka-server-zones-and-regions)
Eureka 服务端并没有后端存储，但是注册表中的服务实例都必须发送心跳，来保持它们的注册最新（这可以在内存中完成）。客户端还有一个内存中的 Eureka 注册缓存，因此对于每个服务的请求，不必跳转到注册表。

默认情况下，每个 Eureka 服务端也是 Eureka 客户端，至少需要一个服务 URL 来定位对等体。如果没有提供服务 URL，服务跑起来并开始工作，就会填充大量的噪音到日志中（无法与对等体注册的异常）。

&nbsp;
## [2.4. Standalone Mode](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#spring-cloud-eureka-server-standalone-mode)
双缓存和心跳机制，两者组合可以让独立 Eureka Server 对于失败具有相当的弹性，只要有某种监视器或者弹性的运行时间保持存活。在独立模式下，你可能更希望关闭客户端行为，避免不断尝试并无法达到对等体的错误。

```yml
server:
  port: 8761
eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

**注意** serviceUrl 需要指向本地实例相同的主机。


## [2.5. Peer Awareness](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#spring-cloud-eureka-server-peer-awareness)
通过运行多个实例，并让他们互相注册，eureka 可以变得更具有弹性和可用性。实际上，这是默认的行为，所以你只需要向对等体添加有效的 serviceUrl 即可。

为了在单个主机上体验对等感知，可以操作 `/etc/hosts` 文件来解析主机名。实际上，如果运行在一台已知主机名的机器上时，没有必要配置 `eureka.instance.hostname`。默认地，会使用 `java.net.InetAddress` 寻找。

你可以向一个系统添加多个对等体，只要它们通过至少边缘彼此连接，它们之间就会同步注册信息。如果对等体物理分离，那么系统原则上存在脑裂的问题。


## [2.6. When to Prefer IP Address](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#spring-cloud-eureka-server-prefer-ip-address)
在某些情况下，eureka 更偏向于建议使用服务的 IP 而不是主机名。设置 `eureka.instance.preferIpAddress` 为 true，那么当应用注册到 eureka 时，将会使用 IP 地址而不是主机名。

> 如果 Java 无法确定 hostname，那么就会发送 IP 地址给 eureka。


## [2.8. Disabling Ribbon with Eureka Server and Client starters](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#disabling-ribbon-with-eureka-server-and-client-starters)

`spring-cloud-starter-netflix-eureka-server` 和 `spring-cloud-starter-netflix-eureka-client` y与 `spring-cloud-starter-netflix-ribbon` 一起出现。由于 Ribbon 负载均衡处于维护模式，因此我们建议切换使用 Spring Cloud LoadBalancer，也包含在 Eureka starter 中。

为此，你可以将 `spring.cloud.loadbalancer.ribbon.enabled` 属性设置为 `false`。

然后，你还可以从构建文件中地 Eureka starter 中排除 ribbon 相关的依赖，像这样：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-ribbon</artifactId>
        </exclusion>
        <exclusion>
            <groupId>com.netflix.ribbon</groupId>
            <artifactId>ribbon-eureka</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

## [3. Circuit Breaker: Spring Cloud Circuit Breaker With Hystrix](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#circuit-breaker-spring-cloud-circuit-breaker-with-hystrix)

### [3.1. Disabling Spring Cloud Circuit Breaker Hystrix](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#disabling-spring-cloud-circuit-breaker-hystrix)
你可以通过设置 `spring.cloud.circuitbreaker.hystrix.enabled` 为 `false` 来禁用自动配置。
```java
spring.cloud.circuitbreaker.hystrix.enabled=false
```

### [3.2. Configuring Hystrix Circuit Breakers](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#configuring-hystrix-circuit-breakers)
#### [3.2.1. Default Configuration](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#default-configuration)

为了给您所有的断路器提供默认配置，请创建传递一个 `HystrixCircuitBreakerFactory` 或者 `ReactiveHystrixCircuitBreakerFactory` 的 `Customize` Bean。`configureDefault` 方法可用于提供默认配置。

```java
@Bean
public Customizer<HystrixCircuitBreakerFactory> defaultConfig() {
    return factory -> factory.configureDefault(id -> HystrixCommand.Setter
            .withGroupKey(HystrixCommandGroupKey.Factory.asKey(id))
            .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
            .withExecutionTimeoutInMilliseconds(4000)));
}
```

##### [Reactive Example](https://docs.spring.io/spring-cloud-netflix/docs/2.2.10.RELEASE/reference/html/#reactive-example)

```java
@Bean
public Customizer<ReactiveHystrixCircuitBreakerFactory> defaultConfig() {
    return factory -> factory.configureDefault(id -> HystrixObservableCommand.Setter
            .withGroupKey(HystrixCommandGroupKey.Factory.asKey(id))
            .andCommandPropertiesDefaults(HystrixCommandProperties.Setter()
                    .withExecutionTimeoutInMilliseconds(4000)));
}
```

#### [3.2.2. Specific Circuit Breaker Configuration](https://docs.spring.io/spring-cloud-netflix/docs/2.2.10.RELEASE/reference/html/#specific-circuit-breaker-configuration)

类似于提供默认配置，您可以传递一个 `HystrixCircuitBreakerFactory` 创建一个 `Customize` Bean。

```java
@Bean
public Customizer<HystrixCircuitBreakerFactory> customizer() {
    return factory -> factory.configure(builder -> builder.commandProperties(
                    HystrixCommandProperties.Setter().withExecutionTimeoutInMilliseconds(2000)), "foo", "bar");
}
```

##### [Reactive Example](https://docs.spring.io/spring-cloud-netflix/docs/2.2.10.RELEASE/reference/html/#reactive-example-2)

```java
@Bean
public Customizer<ReactiveHystrixCircuitBreakerFactory> customizer() {
    return factory -> factory.configure(builder -> builder.commandProperties(
                    HystrixCommandProperties.Setter().withExecutionTimeoutInMilliseconds(2000)), "foo", "bar");
}
```

## [4. Circuit Breaker: Hystrix Clients](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#circuit-breaker-hystrix-clients)
Netflix 已经创建了一个名为 Hystrix 的库，该库实现了 [circuit breaker pattern](https://martinfowler.com/bliki/CircuitBreaker.html)（熔断器模式）。在微服务体系结构中，通常具有多层服务调用，如下示例所示：

<img src="https://raw.githubusercontent.com/spring-cloud/spring-cloud-netflix/main/docs/src/main/asciidoc/images/Hystrix.png">

**Figure 1. Microservice Graph**

较低级别的服务失败可能导致级联失败，直至用户。当对特定服务调用超过 `circuitBreaker.requestVolumeThreshold`（默认是 20 次请求），并且，在一个由 `metrics.rollingStats.timeInMilliseconds` 定义滑动窗口（默认为 10 秒）内，失败百分比大于 `circuitBreaker.errorThresholdPercentage`（默认 >50%）， 熔断器会打开，并且不进行调用。在出错并且熔断器打开的情况下，开发者可以提供一个 fallback。

<img src="https://raw.githubusercontent.com/spring-cloud/spring-cloud-netflix/main/docs/src/main/asciidoc/images/HystrixFallback.png">

**Figure 2. Hystrix fallback prevents cascading failures**

## [4.1. How to Include Hystrix](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#how-to-include-hystrix)

若要在你的项目中包含 Hystrix，请使用 group ID 为 `org.springframework.cloud`，artifact ID 为 `spring-cloud-starter-netflix-hystrix` 的 starter。

下面示例展示了一个极简的带有 Hystrix 熔断器的 Eureka 服务：
```java
@SpringBootApplication
@EnableCircuitBreaker
public class Application {

    public static void main(String[] args) {
        new SpringApplicationBuilder(Application.class).web(true).run(args);
    }

}

@Component
public class StoreIntegration {

    @HystrixCommand(fallbackMethod = "defaultStores")
    public Object getStores(Map<String, Object> parameters) {
        //do stuff that might fail
    }

    public Object defaultStores(Map<String, Object> parameters) {
        return /* something useful */;
    }
}
```


`@HystrixCommand` 由名为 ["javanica"](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-javanica) 的 Netflix contrib 库提供。Spring Cloud 以代理的方式包装具有该注解的 Spring Bean，用于连接到 Hystrix 熔断器。熔断器计算何时打开和关闭回路，以及在失败的情况下做什么。

要配置 `@HystrixCommand`，你可以以 `@HystrixProperty` 注解列表的形式使用 `commandProperties` 属性。有关更多详情，请参见[此处](https://github.com/Netflix/Hystrix/tree/master/hystrix-contrib/hystrix-javanica#configuration)。有关可用属性的详细信息，参见 [Hystrix wiki](https://github.com/Netflix/Hystrix/wiki/Configuration)


## [4.2. Propagating the Security Context or Using Spring Scopes](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#netflix-hystrix-starter)

如果你希望一些线程的本地上下文传播到 `@HystrixCommand`，默认声明并不起作用，因为它在线程池中执行命令（在超时情况下）。你可以通过配置或者直接在注解上，请求 Hystrix 使用不同的 "Isolation Strategy"，切换 Hystrix 使用相同的线程作为调用者。以下示例演示了在注解中设置线程：

```java
@HystrixCommand(fallbackMethod = "stubMyService",
    commandProperties = {
      @HystrixProperty(name="execution.isolation.strategy", value="SEMAPHORE")
    }
)
...
```

### [4.3. Health Indicator](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#health-indicator)

连接熔断器的状态也可以暴露在调用应用的 `/health` 端点上，如下示例所示：

```json
{
    "hystrix": {
        "openCircuitBreakers": [
            "StoreIntegration::getStoresByLocationLink"
        ],
        "status": "CIRCUIT_OPEN"
    },
    "status": "UP"
}
```

### [4.4. Hystrix Metrics Stream](https://docs.spring.io/spring-cloud-netflix/docs/2.2.10.RELEASE/reference/html/#hystrix-metrics-stream)

若要启动 Hystrix metrics stream，请包含关于 `spring-boot-starter-actuator` 的依赖，并设置 `management.endpoints.web.exposure.include=hystrix.stream`。这样做会暴露 `/actuator/hystrix.stream` 作为管理端点，如下示例所示：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

## [5. Circuit Breaker: Hystrix Dashboard](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#circuit-breaker-hystrix-dashboard)
只需要引入以下依赖即可：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
```

配置类添加以下注解：
```java
@EnableHystrixDashboard
```


## [6. Hystrix Timeouts And Ribbon Clients](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#hystrix-timeouts-and-ribbon-clients)
当使用 Hystrix command 包装你希望的 Ribbon 客户端时，确保你的 Hystrix 配置的 timeout 比 Ribbon 配置的 timeout 更长，包括任何可能的重试。例如，如果你的 Ribbon 连接 timeout 是 1 秒，并且 Ribbon 客户端可能重试请求 3 次，那么你的 Hystrix 超时时间应该略高于 3 秒。

> **作者的话** 如果上述示例配置低于 3 秒，那么 Ribbon 可能还未完成 3 次重试就已经结束，并不满足配置预期。


### [6.1. How to Include the Hystrix Dashboard](https://docs.spring.io/spring-cloud-netflix/docs/2.2.10.RELEASE/reference/html/#netflix-hystrix-dashboard-starter)

若要在你的项目包含 Hystrix Dashboard，请使用 group ID 为 `org.springframework.cloud`，artifact ID 为 `spring-cloud-starter-netflix-hystrix-dashboard` 的 starter。

若要运行 Hystrix Dashboard，请使用 `@EnableHystrixDashboard` 注解你的 Spring Boot 主类。然后访问 `/hystrix`，并将 dashboard 指向一个实例在 Hystrix 客户端应用中的 `/hystrix.stream` 端点。

## [7. Client Side Load Balancer: Ribbon](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#spring-cloud-ribbon)

Ribbon 是一个客户端的负载均衡，可以对 HTTP 和 TCP 客户端行为进行大量控制。Feign 已经使用了 Ribbon，因此，如果你使用 `@FeignClient`，这部分也会使用。

> Ribbon 是一个进程内 LB

Ribbon 的核心概念是有名客户端。每个负载均衡器都是组件集合的一部分，组件们在一起工作，按需要连接到远程服务器，并且整体有一个名字，这是你作为开发者赋予的名字。根据需要，Spring Cloud 通过使用 `RibbonClientConfiguration` 以 `ApplicationContext` 为每个有名客户端创建一个新的整体。这包含一个 `ILoadBalancer`，一个 `RestClient`，以及一个 `ServerListFiter`。


### [7.1. How to Include Ribbon](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#netflix-ribbon-starter)

要在你的项目使用 ribbon，请使用 group ID 为 `org.springframework.cloud`，artifact ID 为 `spring-cloud-starter-netflix-ribbon` 的 starter。有关使用当前 Spring Cloud Release Train 设置你的构建系统详情见 [Spring Cloud Project page](https://projects.spring.io/spring-cloud/)


> 如果你依赖了 eureka，则 ribbon 也会同时引入，无需显式引入


### [7.2. Customizing the Ribbon Client](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#customizing-the-ribbon-client)

你可以通过使用 `<client>.ribbon.*` 之中的外部属性配置 Ribbon 客户端的一些部分，这类似于使用原生的 Netflix API，但你可以使用 Spring Boot 配置文件。原生选项可以在 [`CommonClientConfigKey`](https://github.com/Netflix/ribbon/blob/master/ribbon-core/src/main/java/com/netflix/client/config/CommonClientConfigKey.java)  （ribbon-core 的一部分）以静态字段查看。

Spring Cloud 还让你通过使用 `@RibbonClient` 声明额外配置（除了 `RibbonClientConfiguration`）来完全控制客户端，如下所示：

```java
@Configuration
@RibbonClient(name = "custom", configuration = CustomConfiguration.class)
public class TestConfiguration {
}
```

在这个案例中，客户端由已经存在于 `RibbonClientConfiguration` 的组件，以及在 `CustomConfiguration` （后者通常覆盖前者）中的一些组成。

> `CustomConfiguration` 类必须是 `@Configuration` 类，但是注意，不要放在主应用上下文的 `@ComponentScan` 中。否则，它将被所有的 `@RibbonClients` 共享。如果你使用 `@ComponentScan`（或者 `@SpringBootApplication`），你需要采取一些方法以避免将其包括在内（例如，你可以将其放入分开的，不重叠的包中，或者在 `@ComponentScan` 中指定要明确扫描的包）。


### [7.3. Customizing the Default for All Ribbon Clients](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#customizing-the-default-for-all-ribbon-clients)

可以通过使用 `@RibbonClients` 注解，注册一个默认的配置，为所有的 Ribbon 客户端提供默认配置，如下示例所示：

```java
@RibbonClients(defaultConfiguration = DefaultRibbonConfig.class)
public class RibbonClientDefaultConfigurationTestsConfig {

    public static class BazServiceList extends ConfigurationBasedServerList {

        public BazServiceList(IClientConfig config) {
            super.initWithNiwsConfig(config);
        }

    }

}

@Configuration(proxyBeanMethods = false)
class DefaultRibbonConfig {

    @Bean
    public IRule ribbonRule() {
        return new BestAvailableRule();
    }

    @Bean
    public IPing ribbonPing() {
        return new PingUrl();
    }

    @Bean
    public ServerList<Server> ribbonServerList(IClientConfig config) {
        return new RibbonClientDefaultConfigurationTestsConfig.BazServiceList(config);
    }

    @Bean
    public ServerListSubsetFilter serverListFilter() {
        ServerListSubsetFilter filter = new ServerListSubsetFilter();
        return filter;
    }

}
```

### [7.4. Customizing the Ribbon Client by Setting Properties](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#customizing-the-ribbon-client-by-setting-properties)

从 1.2.0 开始，Spring Cloud Netflix 现在支持通过设置兼容 [Ribbon documentation](https://github.com/Netflix/ribbon/wiki/Working-with-load-balancers#components-of-load-balancer) 属性来定制化 Ribbon 客户端。

这使你可以在不同环境中更改启动时行为。

下面的列表展示了支持的属性：

- `<clientName>.ribbon.NFLoadBalancerClassName`：应该实现了 `ILoadBalancer`
- `<clientName>.ribbon.NFLoadBalancerRuleClassName`：应该实现了 `IRule`
- `<clientName>.ribbon.NFLoadBalancerPingClassName`：应该实现了 `IPing`
- `<clientName>.ribbon.NIWSServerListClassName`：应该实现了 `ServerList`
- `<clientName>.ribbon.NIWSServerListFilterClassName`：应该实现了 `ServerListFilter`

> 这些属性中定义的类，相较于使用 `@RibbonClient(configuration=MyRibbonConfig.class)` 定义的 Bean，以及由 Spring Cloud Netflix 提供的默认值，具有更高优先级

要设置名为 `users` 的服务名的 `IRule`


### [7.6. Example: How to Use Ribbon Without Eureka](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#spring-cloud-ribbon-without-eureka)

Eureka 是一种便捷的方式，它抽象化远程服务的发现，因此你不必在客户端中进行硬编码。但是，如果你不想使用 Eureka，Ribbon 和 Feign 也可以工作。假设你已经声明了 "stores" 的 `@RibbonClient`，并且 Eureka 没有使用（甚至不在类路径）。Ribbon 客户端默认到配置好的服务列表。你可以按如下方式提供配置：

**application.yml**

```yml
stores:
  ribbon:
    listOfServers: example.com,google.com
```

### [7.7. Example: Disable Eureka Use in Ribbon](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#example-disable-eureka-use-in-ribbon)

将 `ribbon.eureka.enabled` 属性设置为 `false`，显式地禁用 Ribbon 中 Eureka 的使用，如下示例所示：

**application.yml**
```yml
ribbon:
  eureka:
   enabled: false
```


### [7.8. Using the Ribbon API Directly](https://docs.spring.io/spring-cloud-netflix/docs/2.2.9.RELEASE/reference/html/#using-the-ribbon-api-directly)

你也可以直接使用 `LoadBalanceClient`，如下示例所示：

```java
public class MyClass {
    @Autowired
    private LoadBalancerClient loadBalancer;

    public void doStuff() {
        ServiceInstance instance = loadBalancer.choose("stores");
        URI storesUri = URI.create(String.format("https://%s:%s", instance.getHost(), instance.getPort()));
        // ... do something with the URI
    }
}
```