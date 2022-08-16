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

Spring Framework 分别使用 `TaskExecutor` 和 `TaskScheduler` 接口为异步执行和任务调度提供了抽象。Spring 还具有那些支持线程池的接口实现或在应用程序服务器环境中委托 CommonJ。最终，通用接口之间的这些实现的使用抽象出了 Java SE5，Java SE 6 以及 Java EE 环境的差异。

Spring 还具有支持使用 `Timer` 调度的集成类，以及 Quartz Scheduler。你可以通过使用 `FactoryBean`，可选的分别对 `Timer` 或者 `Trigger` 实例的引用设置这两个调度器。此外，用于 Quartz 调度器和 `Timer` 的便利类是可用的，它让你调用现存的目标对象的方法（类似于普通的 `MethodInvokingFactoryBean` 操作）。

### [7.1. The Spring TaskExecutor Abstraction](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#scheduling-task-executor)

Executor 是 JDK 用于线程池概念的名称。之所以叫 "executor" 是因为实际上不能保证底层的实现就是池。一个 executor 可以是单线程，甚至是同步的。Spring 的抽象隐藏了 Java SE 和 Java EE 环境之间的实现细节。

Spring 的 `TaskExecutor` 接口与 `java.util.concurrent.Executor` 接口相同。实际上，最初其主要存在的原因就是在使用线程池的时候抽象出对 Java 5 的需求。该接口只有一个方法（`execute(Runnable task)`），该方法接受给予线程池的语义和配置的任务。


#### [7.1.1. TaskExecutor Types](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#scheduling-task-executor-types)

Spring 包含了许多预置的 `TaskExecutor` 实现。很有可能，你永远不需要实现自己的类。Spring 提供的变体如下：

- `SyncTaskExecutor`: 此实现不会异步运行调用。相反，每个调用都发生在调用线程中。它主要用于不需要多线程的情况，例如在简单的测试用例中。


- `ThreadPoolTaskExecutor`: 此实现最常用。它暴露了用于配置 `java.util.concurrent.ThreadPoolExecutor` 的 Bean 属性，并将其包装在 `TaskExecutor` 中。如果你需要适应其他类型的 `java.util.concurrent.Executor`，我们建议你改用 `ConcurrentTaskExecutor`。


#### [7.1.2. Using a TaskExecutor](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#scheduling-task-executor-usage)

Spring 的 `TaskExecutor` 实现被用作简单的 Java Bean。在下面的示例中，我们定义了一个使用 `ThreadPoolTaskExecutor` 的 Bean，异步地打印一组消息：

```java
import org.springframework.core.task.TaskExecutor;

public class TaskExecutorExample {

    private class MessagePrinterTask implements Runnable {

        private String message;

        public MessagePrinterTask(String message) {
            this.message = message;
        }

        public void run() {
            System.out.println(message);
        }
    }

    private TaskExecutor taskExecutor;

    public TaskExecutorExample(TaskExecutor taskExecutor) {
        this.taskExecutor = taskExecutor;
    }

    public void printMessages() {
        for(int i = 0; i < 25; i++) {
            taskExecutor.execute(new MessagePrinterTask("Message" + i));
        }
    }
}
```

如你所见，你没有从池中检索线程并自己执行，而是将你的 `Runnable` 添加到队列中。然后 `TaskExecutor` 使用其内部规则来决定任务何时运行。

为了配置 `TaskExecutor` 使用的规则，我们将公开简单的 Bean 属性：

```xml
<bean id="taskExecutor" class="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor">
    <property name="corePoolSize" value="5"/>
    <property name="maxPoolSize" value="10"/>
    <property name="queueCapacity" value="25"/>
</bean>

<bean id="taskExecutorExample" class="TaskExecutorExample">
    <constructor-arg ref="taskExecutor"/>
</bean>
```

### [7.2. The Spring TaskScheduler Abstraction](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/integration.html#scheduling-task-scheduler)

除了 `TaskExecutor` 抽象外，



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

