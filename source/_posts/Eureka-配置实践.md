---
title: Eureka 配置实践
date: 2022-07-12 16:28:41
tags:
---


# Eureka Server
## 如何使用？
确保引入 maven 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```


## Config
Eureka 的 Server 配置没有特别好的文档，官网的指引也只是让你看源代码注释。

### Eureka Instance Config
- Instance Id
```java
String getInstanceId();
```

获得该实例注册到 eureka 的唯一 ID（在 appName 范围内）


- App Name
```java
String getAppname();
```

获得注册到 eureka 的应用名称


- lease renewal interval in seconds

```java
getLeaseRenewalIntervalInSeconds()
```

字面意思：租约续期间隔。

表示 eureka 客户端多久发送心跳给 eureka server，告诉它自己还活着。如果心跳在 `getLeaseExpirationDurationInSeconds()` 指定的期间还未收到，eureka server 就会从它的视图中移除该实例，从而禁用了该实例的流量。

> 该参数只是用于 Eureka Client 发送心跳间隔。需要确保与 Eureka Server 的 `getLeaseExpirationDurationInSeconds()` 参数值一致，否则 Eureka Server 无法正常工作。

- lease expiration duration in seconds

表示 Eureka Server 自收到某个实例最后一次心跳，在可以从它的视图中移除该实例从而禁用该实例流量之前，等待的时间，单位：秒。

设置该值太长可能意味着，即使该实例并不存活，也可以将流量路由到该实例。设置该值太小可能意味着，由于临时的网络故障，该实例可能会从流量中剔除。该值设置至少高于 `getLeaseRenewalIntervalInSeconds()` 指定的值。


> 如果该值比 `getLeaseRenewalIntervalInSeconds()` 小，那么实例将无法存活于注册表，即使注册成功，很快就被剔除。


### Eureka Server Config

- AWS Access Id

```java
String getAWSAccessId();
```
AWS 云 Access ID

- AWS Secret Key

```java
String getAWSSecretKey();
```
AWS 云 Secret Key


- EIPBindRebindRetries
```java
int getEIPBindRebindRetries();
```

- EIPBindingRetryIntervalMsWhenUnbound
```java
int getEIPBindingRetryIntervalMsWhenUnbound();
```

- EIPBindingRetryIntervalMs


- enable-self-preservation


```java
boolean shouldEnableSelfPreservation();
```

是否开启自我保护机制。

启用后，Eureka Server 将跟踪其应该从服务接收到的续约次数。任何时候，续约数量低于 `getRenewalPercentThreshold()` 定义的阈值百分比，Eureka Server 会关闭过期以避免危险。这有助于在 Eureka Client 与 Eureka Server 之间发生网络问题时维持注册表信息。

> 注意，自我保护是防止一些网络问题误杀。

- eviction-interval-timer-in-ms

清理无效节点的时间间隔。默认值 60000


> 可以将这个时间设置的短一些，进行快速下线。防止使用不可用的服务。

- expected-client-renewalI-interval-seconds

```java
int getExpectedClientRenewalIntervalSeconds();
```
期望客户端以这个间隔发送它们的心跳。

默认值：30

如果客户端以不同的频率发送心跳，例如每 15 秒发送一次，那么应该相应地调整此参数，否则，自我保护将无法按预期工作。

> 该参数用于计算内部的阈值。

- Response Cache Update Interval Ms

```java
long getResponseCacheUpdateIntervalMs();
```
获取更新 Eureka Client 负载缓存的时间间隔。

- Use Read Only Response Cache

```java
boolean shouldUseReadOnlyResponseCache();
```

字面意思：是否使用只读响应缓存

`com.netflix.eureka.registry.ResponseCache` 当前使用两级缓存策略来响应。带有过期策略的独写缓存，以及不会过期的只读缓存。

- Renewal Percent Threshold

在由 `getRenewalThresholdUpdateIntervalMs()` 指定的期间，期望从服务端续期次数的最小百分比。

如果续约降低到阈值以下，并且启用了 `shouldEnableSelfPreservation()` 则会禁用过期。

- getRenewalThresholdUpdateIntervalMs

由 `getRenewalPercentThreshold()` 指定的阈值，应该更新的间隔。阈值更新间隔


- shouldEnableSelfPreservation

检查 eureka server 是否启用了自我保护。

当启用，服务会跟踪它应该从微服务接受到的续约次数。在任何时候，续订次数的数量低于 `getRenewalPercentThreshold()` 定义的阈值百分比，服务器就会关闭过期以避免危险。这有助于在客户端和服务端之间产生网络问题时服务器维持注册信息。
