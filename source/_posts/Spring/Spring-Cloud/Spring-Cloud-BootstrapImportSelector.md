---
title: Spring Cloud BootstrapImportSelector
date: 2022-07-22 15:17:55
categories:
- 框架
tags:
- Spring Cloud
---

# Spring Cloud BootstrapImportSelector




在 `BootstrapApplicationListener` 监听器中，会将 `BootstrapImportSelectorConfiguration` 配置类注入到 IoC 容器，该配置类有一个注解 `@Import`，将 `BootstrapImportSelector` 类注入到容器：

```java
@Configuration(proxyBeanMethods = false)
@Import(BootstrapImportSelector.class)
public class BootstrapImportSelectorConfiguration {

}
```


`BootstrapImportSelector` 获得 spring.factories 文件中 key 为 `org.springframework.cloud.bootstrap.BootstrapConfiguration` 的配置组件。


## 构建 bootstrapServiceContext 流程

在 `SpringApplication` 运行 run 方法的过程中会调用 `prepareEnvironment` 进行环境准备，其中会调用所有的监听器 `SpringApplicationRunListener` 进行环境准备，调用链如下：

```java
SpringApplication.run()
-> SpringApplication.prepareEnvironment()
-> listeners.environmentPrepared()
```

一般来说，默认只有一个监听器 `EventPublishingRunListener`，以下是 `SpringApplicationRunListeners` 的调用入口：

```java
void environmentPrepared(ConfigurableEnvironment environment) {
    for (SpringApplicationRunListener listener : this.listeners) {
        listener.environmentPrepared(environment);
    }
}
```

`EventPublishingRunListener` 会调用内部的 `initialMulticaster` 进行事件广播，代码清单如下：

```java
public void environmentPrepared(ConfigurableEnvironment environment) {
    this.initialMulticaster
            .multicastEvent(new ApplicationEnvironmentPreparedEvent(this.application, this.args, environment));
}
```


接下来会进入 `SimpleApplicationEventMulticaster` 的广播事件逻辑，将会获取所有的 `ApplicationListener`（这也包括 `BootstrapApplicationListener`）：

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

调用事件逻辑链如下：

```java
SimpleApplicationEventMulticaster.invokeListener()
-> SimpleApplicationEventMulticaster.doInvokeListener()
-> ApplicationListener.onApplicationEvent(event)
```


下面 `BootstrapApplicationListener` 开始处理事件:

```java
public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
    ConfigurableEnvironment environment = event.getEnvironment();
    if (!environment.getProperty("spring.cloud.bootstrap.enabled", Boolean.class,
            true)) {
        return;
    }
    // don't listen to events in a bootstrap context
    if (environment.getPropertySources().contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
        return;
    }
    ConfigurableApplicationContext context = null;
    String configName = environment
            .resolvePlaceholders("${spring.cloud.bootstrap.name:bootstrap}");
    for (ApplicationContextInitializer<?> initializer : event.getSpringApplication()
            .getInitializers()) {
        if (initializer instanceof ParentContextApplicationContextInitializer) {
            context = findBootstrapContext(
                    (ParentContextApplicationContextInitializer) initializer,
                    configName);
        }
    }
    if (context == null) {
        // 这里调用 bootstrapServiceContext 创建了一个上下文
        context = bootstrapServiceContext(environment, event.getSpringApplication(), configName);
        event.getSpringApplication().addListeners(new CloseContextOnFailureApplicationListener(context));
    }

    apply(context, event.getSpringApplication(), environment);
}
```

下面是 BoostrapApplicationListener 创建上下文的方法：

```java
private ConfigurableApplicationContext bootstrapServiceContext(
        ConfigurableEnvironment environment, final SpringApplication application,
        String configName) {
    StandardEnvironment bootstrapEnvironment = new StandardEnvironment();
    MutablePropertySources bootstrapProperties = bootstrapEnvironment
            .getPropertySources();
    for (PropertySource<?> source : bootstrapProperties) {
        bootstrapProperties.remove(source.getName());
    }
    String configLocation = environment
            .resolvePlaceholders("${spring.cloud.bootstrap.location:}");
    String configAdditionalLocation = environment
            .resolvePlaceholders("${spring.cloud.bootstrap.additional-location:}");
    Map<String, Object> bootstrapMap = new HashMap<>();
    bootstrapMap.put("spring.config.name", configName);
    // if an app (or test) uses spring.main.web-application-type=reactive, bootstrap
    // will fail
    // force the environment to use none, because if though it is set below in the
    // builder
    // the environment overrides it

    // 如果一个应用（或者 test）使用 spring.main.web-application-type=reactive，bootstrap 就会失败
    // 强制环境使用 none，因为如果在构建器中设置为 below，环境就会覆盖???
    bootstrapMap.put("spring.main.web-application-type", "none");
    if (StringUtils.hasText(configLocation)) {
        bootstrapMap.put("spring.config.location", configLocation);
    }
    if (StringUtils.hasText(configAdditionalLocation)) {
        bootstrapMap.put("spring.config.additional-location",
                configAdditionalLocation);
    }
    bootstrapProperties.addFirst(
            new MapPropertySource(BOOTSTRAP_PROPERTY_SOURCE_NAME, bootstrapMap));
    for (PropertySource<?> source : environment.getPropertySources()) {
        if (source instanceof StubPropertySource) {
            continue;
        }
        bootstrapProperties.addLast(source);
    }
    // TODO: is it possible or sensible to share a ResourceLoader?
    SpringApplicationBuilder builder = new SpringApplicationBuilder()
            .profiles(environment.getActiveProfiles()).bannerMode(Mode.OFF)
            .environment(bootstrapEnvironment)
            // Don't use the default properties in this builder
            .registerShutdownHook(false).logStartupInfo(false)
            .web(WebApplicationType.NONE);
    final SpringApplication builderApplication = builder.application();
    if (builderApplication.getMainApplicationClass() == null) {
        // gh_425:
        // SpringApplication cannot deduce the MainApplicationClass here
        // if it is booted from SpringBootServletInitializer due to the
        // absense of the "main" method in stackTraces.
        // But luckily this method's second parameter "application" here
        // carries the real MainApplicationClass which has been explicitly
        // set by SpringBootServletInitializer itself already.
        builder.main(application.getMainApplicationClass());
    }
    if (environment.getPropertySources().contains("refreshArgs")) {
        // If we are doing a context refresh, really we only want to refresh the
        // Environment, and there are some toxic listeners (like the
        // LoggingApplicationListener) that affect global static state, so we need a
        // way to switch those off.
        builderApplication
                .setListeners(filterListeners(builderApplication.getListeners()));
    }
    // 这里配了一个 BootstrapImportSelectorConfiguration
    builder.sources(BootstrapImportSelectorConfiguration.class);
    final ConfigurableApplicationContext context = builder.run();
    // gh-214 using spring.application.name=bootstrap to set the context id via
    // `ContextIdApplicationContextInitializer` prevents apps from getting the actual
    // spring.application.name
    // during the bootstrap phase.
    context.setId("bootstrap");
    // Make the bootstrap context a parent of the app context
    addAncestorInitializer(application, context);
    // It only has properties in it now that we don't want in the parent so remove
    // it (and it will be added back later)
    bootstrapProperties.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
    mergeDefaultProperties(environment.getPropertySources(), bootstrapProperties);
    return context;
}
```


## `BootstrapImportSelector` 如何进行 `selectImports` 操作


`bootstrapServiceContext` 与一般的 `ApplicationContext` 类似，也需要进行容器 `refresh` 操作。


其中会调用许多 `BeanFactoryPostProcessor` 进行处理，包括 `ConfigurationClassPostProcessor`，Bean Definition `bootstrapImportSelectorConfiguration` 就是其处理对象，也可以说是候选人 `candidates` 之一。

候选人经过 `ConfigurationClassParser` 解析，解析之后会得到许多 `ConfigurationClass`。解析的过程主要分两个步骤：解析、延迟处理：

> 对于那些实现了 `DeferredImportSelector` 接口的 `ImportSelector`，会在延迟处理中处理

```java
// ConfigurationClassParser
public void parse(Set<BeanDefinitionHolder> configCandidates) {
    for (BeanDefinitionHolder holder : configCandidates) {
        BeanDefinition bd = holder.getBeanDefinition();
        // 以下的 parse 方法也会特殊过滤 DeferredImportSelector
        try {
            if (bd instanceof AnnotatedBeanDefinition) {
                parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
            }
            else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
            }
            else {
                parse(bd.getBeanClassName(), holder.getBeanName());
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
        }
    }
    // 延迟处理 ImportSelector
    this.deferredImportSelectorHandler.process();
}
```


`parse` 方法执行结束，接着使用 `ConfigurationClassBeanDefinitionReader` 加载所有的 `ConfigurationClass`。一般这里会解析到如下：

- org.springframework.cloud.bootstrap.BootstrapImportSelectorConfiguration
- org/springframework/cloud/bootstrap/config/PropertySourceBootstrapConfiguration.class
- org/springframework/cloud/bootstrap/encrypt/EncryptionBootstrapConfiguration.class
- org/springframework/cloud/autoconfigure/ConfigurationPropertiesRebinderAutoConfiguration.class
- org/springframework/boot/autoconfigure/context/PropertyPlaceholderAutoConfiguration.class
- org/springframework/cloud/util/random/CachedRandomPropertySourceAutoConfiguration.class


针对这些 `ConfigurationClass`，以此进行 Bean Definition 的加载：

```java
// ConfigurationClassBeanDefinitionReader.java
private void loadBeanDefinitionsForConfigurationClass(
        ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {
    if (trackedConditionEvaluator.shouldSkip(configClass)) {
        String beanName = configClass.getBeanName();
        if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
            this.registry.removeBeanDefinition(beanName);
        }
        this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
        return;
    }

    if (configClass.isImported()) {
        registerBeanDefinitionForImportedConfigurationClass(configClass);
    }
    for (BeanMethod beanMethod : configClass.getBeanMethods()) {
        loadBeanDefinitionsForBeanMethod(beanMethod);
    }

    // 用于处理导入资源
    loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
    // 用于处理 
    loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```