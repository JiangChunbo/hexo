---
title: Spring Cloud Config 配置实践
date: 2022-12-01 18:40:26
tags:
---

# Git 

## Config Server 配置

准备一个 Git 仓库远端地址:

比如: https://gitee.com/jiang_chun_bo/spring-cloud-config.git


Maven 依赖:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

**application.properties**

```properties
server.port=8888
spring.cloud.config.server.git.uri=https://github.com/JiangChunbo/canned-bread.git
spring.cloud.config.server.git.default-label=master
spring.cloud.config.server.git.search-paths=cloud-config-server/src/main/resources/
spring.cloud.config.server.git.username=945086245@qq.com
spring.cloud.config.server.git.password='{cipher}3290c2fd481619115fcda1d1c9843c6c033320e8e9b2e3150ab90e421a014743df8fb0336b9c6579a2d7233e5bb930d3'
```

启动类:

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigEurekaApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigEurekaApplication.class, args);
    }
}
```


## Client 配置

Maven 依赖:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```


**bootstrap.properties**
```properties

```