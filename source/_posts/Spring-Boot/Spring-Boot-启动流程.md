---
title: Spring Boot 启动流程
date: 2022-07-24 13:03:48
tags:
---

# Spring Boot 启动流程


## Spring Boot 入口

一般会使用静态方法，也可以自己 new 一个 `SpringApplication`，或者使用 Buidler 定制化。

```java
@SpringBootApplication
public class StartupApplication {

    public static void main(String[] args) {
        SpringApplication.run(StartupApplication.class, args);
    }
}
```


SpringApplication 构造器:

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    // 资源加载器，默认 null
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    // 主要加载资源类，Set 去重
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 推断 Web 环境，底层通过 classpath 检测，NONE, SERVLET, REACTIVE
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 设置应用上下文初始化器 （从 META-INF/spring.factories）
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 设置监听器 （从 META-INF/spring.factories）
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 推断主应用类，通过 stackTrace
    this.mainApplicationClass = deduceMainApplicationClass();
}
```


`SpringApplication` 运行方法：

```java
public ConfigurableApplicationContext run(String... args) {
    // 1. 创建一个秒表，并启动
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    // 设置系统属性 java.awt.headless
    configureHeadlessProperty();
    // 创建所有 SpringApplicationRunListener（EventPublishingRunListener）
    // 将所有 SpringApplicationRunListener 封装到 SpringApplicationRunListeners
    // 底层会读取 spring.factories 的 org.springframework.boot.SpringApplicationRunListener
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 发布 ApplicationStartingEvent 应用启动事件
    listeners.starting();
    try {
        // 初始化默认应用参数类
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 根据 SpringApplicationRunListener 和应用参数准备 spring Environment
        // 发布 ApplicationEnvironmentPreparedEvent 事件
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        // 将要忽略的 bean 的参数打开
        configureIgnoreBeanInfo(environment);
        // 创建 Banner 打印类
        Banner printedBanner = printBanner(environment);
        // 创建应用上下文（借助构造器推断的 webApplicationType 选择类）
        context = createApplicationContext();
        // 准备上下文
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        // 刷新
        refreshContext(context);
        // 后置处理
        afterRefresh(context, applicationArguments);
        // 停止秒表
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        // 发布应用上下文启动完毕事件 ApplicationStartedEvent
        listeners.started(context);
        // 执行所有的 Runner
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        // 发布 ApplicationReadyEvent 事件
        listeners.running(context);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```

## StopWatch

计时器/秒表

```java
StopWatch stopWatch = new StopWatch();
stopWatch.start();
```


## configureHeadlessProperty

设置属性 `java.awt.headless`，如果不存在默认值设置为 true

```java
private void configureHeadlessProperty() {
    System.setProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS,
            System.getProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS, Boolean.toString(this.headless)));
}
```

## getRunListeners

> 这里是 `SpringApplicationRunListener`，注意与 `ApplicationListener` 区别


```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
    Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
    return new SpringApplicationRunListeners(logger,
            getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```
一般情况下，只有 spring-boot 下面的 spring.factories 配置了一个 `EventPublishingRunListener`，其构造器如下：

```java
public EventPublishingRunListener(SpringApplication application, String[] args) {
    this.application = application;
    this.args = args;
    // 事件多播器
    this.initialMulticaster = new SimpleApplicationEventMulticaster();
    // application.getListeners() 获得 ApplicationListener
    for (ApplicationListener<?> listener : application.getListeners()) {
        // 添加到事件多播器
        this.initialMulticaster.addApplicationListener(listener);
    }
}
```


## listeners.starting()

通常这里只有 1 个 listeners，即 `EventPublishingRunListener`。

```java
void starting() {
    for (SpringApplicationRunListener listener : this.listeners) {
        listener.starting();
    }
}
```

`EventPublishingRunListener` 底层调用内部的 `Multicaster` 进行广播。

```java
public void starting() {
    this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
}
```

```java
public void multicastEvent(ApplicationEvent event) {
    // 调用 resolveDefaultEventType，解析得到 ResolvableType
    multicastEvent(event, resolveDefaultEventType(event));
}
```

```java
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    Executor executor = getTaskExecutor();
    for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        if (executor != null) {
            executor.execute(() -> invokeListener(listener, event));
        }
        else {
            invokeListener(listener, event);
        }
    }
}
```

```java
protected Collection<ApplicationListener<?>> getApplicationListeners(ApplicationEvent event, ResolvableType eventType) {
    // 获得 event 中的 source，一般为 SpringApplication
    Object source = event.getSource();
    Class<?> sourceType = (source != null ? source.getClass() : null);
    // 获得 key
    ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);

    // Potential new retriever to populate
    // retriever 检索器
    CachedListenerRetriever newRetriever = null;

    // Quick check for existing entry on ConcurrentHashMap
    CachedListenerRetriever existingRetriever = this.retrieverCache.get(cacheKey);
    if (existingRetriever == null) {
        // Caching a new ListenerRetriever if possible
        if (this.beanClassLoader == null ||
                (ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
                        (sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
            newRetriever = new CachedListenerRetriever();
            // 用新的覆盖
            existingRetriever = this.retrieverCache.putIfAbsent(cacheKey, newRetriever);
            if (existingRetriever != null) {
                newRetriever = null;  // no need to populate it in retrieveApplicationListeners
            }
        }
    }

    if (existingRetriever != null) {
        Collection<ApplicationListener<?>> result = existingRetriever.getApplicationListeners();
        if (result != null) {
            return result;
        }
        // If result is null, the existing retriever is not fully populated yet by another thread.
        // Proceed like caching wasn't possible for this current local attempt.
    }

    return retrieveApplicationListeners(eventType, sourceType, newRetriever);
}
```

默认情况下，这里会根据 Starting 事件匹配到 4 个：

- LoggingApplicationListener
- BackgroundPreinitializer
- DelegatingApplicationListener
- LiquibaseServiceLocatorApplicationListener


## java

```java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments) {
    // Create and configure the environment
    // 根据 webApplicationType 创建
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    // 配置环境，设置 ConversionService、
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    ConfigurationPropertySources.attach(environment);
    listeners.environmentPrepared(environment);
    bindToSpringApplication(environment);
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
                deduceEnvironmentClass());
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```


## prepareContext

```java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
        SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
    context.setEnvironment(environment);
    postProcessApplicationContext(context);
    applyInitializers(context);
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    // Add boot specific singleton beans
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    if (beanFactory instanceof DefaultListableBeanFactory) {
        ((DefaultListableBeanFactory) beanFactory)
                .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    if (this.lazyInitialization) {
        context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
    }
    // Load the sources
    // 加载资源，包括启动类
    Set<Object> sources = getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    load(context, sources.toArray(new Object[0]));
    listeners.contextLoaded(context);
}
```