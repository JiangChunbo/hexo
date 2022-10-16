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
    // 主要资源类，Set 去重
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 推断 Web 环境，底层通过 classpath 是否包含特定类检测，NONE, SERVLET, REACTIVE
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 设置应用上下文初始化器 （从 META-INF/spring.factories）
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 设置监听器 （从 META-INF/spring.factories）
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 推断主应用类，通过 stackTrace
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

> 上面之所以要加载 `ApplicationListener` 保存到 `SpringApplication`，是因为下面要传递给 `ApplicationEventMulticaster`。`SpringApplication` 作为 `EventPublishingRunListener` 参数传入。


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

    // 立即发布 ApplicationStartingEvent 应用启动事件
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


该方法返回一个 `SpringApplicationRunListeners` 实例，注意末尾有一个 "s"。

```java
// SpringApplication
private SpringApplicationRunListeners getRunListeners(String[] args) {
    Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
    // 传入的 this 参数就是 SpringApplication
    return new SpringApplicationRunListeners(logger,
            getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```

> 照应前面所说，之所以将 `SpringApplication` 传入，作为后续的构造器参数，是因为后面需要往 `ApplicationEventMulticaster` 加入 `ApplicationListener`，而 `ApplicationListener` 已经在 `SpringApplication` 的构造器加载完毕

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
    // 获得所有发现的 ApplicationListener
    for (ApplicationListener<?> listener : application.getListeners()) {
        // 添加到事件多播器
        this.initialMulticaster.addApplicationListener(listener);
    }
}
```


## listeners.starting()

在获得了 `SpringApplicationRunListeners` 之后立即发布一个事件，告诉所有 `ApplicationListener` 容器正在启动。

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
    // 构造一个事件广播
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

获得关心该事件的 `ApplicationListener`

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


## prepareEnvironment

准备环境。也会发布事件

```java
private ConfigurableEnvironment prepareEnvironment(
        SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments) {
    // Create and configure the environment
    // 根据 webApplicationType 实例化 Environment，注意其间不断调用父类默认构造器
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    // 配置环境，设置 ConversionService；添加命令行
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    ConfigurationPropertySources.attach(environment);
    // 发布事件
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


补充一下 `StandardServletEnvironment` 构造器做了什么:
```java
// AbstractEnvironment
public AbstractEnvironment() {
    // customizePropertySources 是一个空实现
    // 这里传入了 this 属性，但可以不传?
    customizePropertySources(this.propertySources);
}
```

以下是 `StandardServletEnvironment` 对于 `customizePropertySources` 的覆盖（定制化）：

```java
// StandardServletEnvironment
protected void customizePropertySources(MutablePropertySources propertySources) {
    // 添加 servletConfigInitParams
    propertySources.addLast(new StubPropertySource(SERVLET_CONFIG_PROPERTY_SOURCE_NAME));
    // 添加 servletContextInitParams
    propertySources.addLast(new StubPropertySource(SERVLET_CONTEXT_PROPERTY_SOURCE_NAME));
    if (JndiLocatorDelegate.isDefaultJndiEnvironmentAvailable()) {
        propertySources.addLast(new JndiPropertySource(JNDI_PROPERTY_SOURCE_NAME));
    }
    // 注意，这里也调用了父类 StandardEnvironement 的方法
    super.customizePropertySources(propertySources);
}
```

以下是 `StandardEnvironement` 关于 `customizePropertySources` 方法的覆盖（定制化）: 

```java
// StandardEnvironement
protected void customizePropertySources(MutablePropertySources propertySources) {
    // 添加系统属性
    propertySources.addLast(
            new PropertiesPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));
    // 添加环境变量
    propertySources.addLast(
            new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
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