---
title: Spring Framework Integration
date: 2022-07-27 16:11:28
tags:
---

# [Integration](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#spring-integration)
---
## [Message Conversion](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#rest-message-conversion)

WebMvcConfigurationSupport 内含默认的 Converter

BufferedImageHttpMessageConverter 返回图片
```java
@RequestMapping(path = "/image/{id}", produces = MediaType.IMAGE_JPEG_VALUE)
public BufferedImage getImage(@PathVariable Integer id) throws Exception {
    Weather weather = weatherMapper.getById(id);
    return ImageIO.read(weather.getPicture());
}
```


## [6. Mail](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#mail)
Spring Boot 的使用方式：

(1) 配置并将 `JavaMailSender` 注入到 IOC 容器，类似一个 Mail 工厂配置，可以 getSession，也可以 createMimeMessage
```java
@Configuration
public class JavaMailConfig {
    @Bean(name = "javaMailSender")
    public JavaMailSenderImpl javaMailSender() {
        JavaMailSenderImpl javaMailSender = new JavaMailSenderImpl(); // 唯一的实现
        javaMailSender.setHost("smtp.qq.com"); // 设置 SMTP 主机
        javaMailSender.setUsername("945086245@qq.com");
        javaMailSender.setPassword("password");
        javaMailSender.setDefaultEncoding("UTF-8");
        return javaMailSender;
    }
}
```

(2) 使用 javaMailSender 发送。MimeMessageHelper 类似 Builder 设计，但是没有做链式调用。

```java
MimeMessage mimeMessage = javaMailSender.createMimeMessage();
MimeMessageHelper mimeMessageHelper = new MimeMessageHelper(mimeMessage, true);
mimeMessageHelper.setFrom("945086245@qq.com"); // 发送人
mimeMessageHelper.setTo(email); // 接收人
mimeMessageHelper.setSubject("验证码");
mimeMessageHelper.setText("您的验证码是: 9836, 如非本人操作请忽视。");
javaMailSender.send(mimeMessage);
```

## [7. Task Execution and Scheduling](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#scheduling)

Spring Framework 分别使用 `TaskExecutor` 和 `TaskScheduler` 接口进行

&nbsp;
## [8. Cache Abstraction](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#cache)
### [8.1. Understanding the Cache Abstraction](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#cache-strategies)
**Buffer 和 Cache**

一般地，buffer 可以翻译为缓冲区，用于高速与低速的实体之间的中间存储，缓冲区至少对知晓它的一方是可见的。

cache 可以翻译为缓存，根据定义是隐藏的，任何一方都不知晓。

**Spring 提供了一些缓存抽象的实现**：

- SimpleCacheConfiguration
- EhCacheCacheConfiguration
- GenericCacheConfiguration
- RedisCacheConfiguration
- ...


SpringBoot `CacheAutoConfiguration` 中使用 `@Import` 导入了 `CacheConfigurationImportSelector.class`，其中导入了枚举类 `CacheType` 中所有的缓存类型。


**SimpleCacheConfiguration**

底层使用 `concurrentMap` 实现，见 `SimpleCacheConfiguration`  注入 `ConcurrentMapCacheManager`。`ConcurrentMapCacheManager ` 属性 `dynamic` 可以配置 `cacheName` 是否可以动态生成，默认为 true。




### [8.2. Declarative Annotation-based Caching](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#cache-annotations)
Spring 缓存抽象提供了一组 Java 注解：

- `@Cacheable`：触发缓存填充
- `@CacheEvict`：触发缓存驱逐
- `@CachePut`：在不干扰方法执行的情况下更新缓存
- `@Caching`：重新组合要应用于方法的多个缓存操作
- `@CacheConfig`：在类级别共享一些常见的缓存相关设置


#### [8.2.1. The @Cacheable Annotation](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#cache-annotations-cacheable)

|注解属性|说明|
|:---|:---|
|value / cacheNames|缓存的名字|
|key|缓存数据使用的 key，默认使用方法参数|
|keyGenerator|自定义 key 生成器，实现 KeyGenerator|
|cacheManager|指定缓存管理器|
|condition|指定符合条件的情况下才缓存|
|unless|缓存的否定条件|
|sync|是否使用异步模式|

##### [Default Key Generation](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#cache-annotations-cacheable-default-key)
key 默认采用 `SimpleKeyGenerator` 生成。以方法参数为标识。


##### [Synchronized Caching](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#cache-annotations-cacheable-synchronized)
Spring Core 框架的 `CacheManager` 实现都支持 sync。其他缓存库未必。


##### [Available Caching SpEL Evaluation Context](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#cache-spel-context)
Cache SpEL 可用元数据，具体见官方表。



#### [8.2.2. The @CachePut Annotation](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#cache-annotations-put)
每次都调用方法，缓存结果。


#### [8.2.3. The @CacheEvict annotation](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#cache-annotations-evict)
清空缓存

#### [8.2.6. Enabling Caching Annotations](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#cache-annotation-enable)
```java
@Configuration
@EnableCaching
public class AppConfig {
}
```

### [8.3. JCache (JSR-107) Annotations](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#cache-jsr-107)
从 Spring 4.1 开始，Spring 缓存抽象完全支持 JCache 注解

#### [8.3.1. Feature Summary](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#cache-jsr-107-summary)
|Spring|JSR-107|备注|
|:---|:---|:---|
|@Cacheable|@CacheResult|
|@CachePut|@CachePut|
|@CacheEvict|@CacheRemove|
|@CacheEvict(allEntries=true)|@CacheRemoveAll|
|@CacheConfig|@CacheDefaults|


JCache 的 CacheResolver 概念上与 Spring CacheResolver 接口相同，只是 JCache 仅支持单个 Cache。默认，Simple 实现会根据注解上的名称检索要使用的 Cache，如果注解上没有指定名称，则会自动生成一个默认值。

#### [8.3.2. Enabling JSR-107 Support](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#enabling-jsr-107-support)
如果类路径同时存在 JSR-107 API 和 spring-context-support，`@EnableCaching` 和 `cache:annotation-driven` 都会自启用 JCache。

