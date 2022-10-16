---
title: AbstractContext refresh 方法流程
date: 2022-07-22 08:51:23
categories:
- 框架
tags:
- Spring Framework
---
# AbstractContext refresh 方法流程
##  1. refresh 总体流程
IoC 容器刷新的方法是由 `AbstractApplicationContext.refresh` 实现，方法如下: 
```java
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 1. 为 refresh 准备上下文（与 finish 相对）
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        // 2. 通知子类，refresh 其内部的 beanFactory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        // 3. 准备 Bean Factory
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            // 4. 在 Context 的子类中可以后置处理 Bean Factory
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            // 5. 调用以 Bean 形式注册在 Context 中的 BeanFactoryPostProcessor
            // 官方这句注释有些不准确，因为其中会调用 AbstractContext 内部的几个 internal BeanFactoryPostProcessor
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            // 6. 注册 bean post processor
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            // 7. 为 context 初始化消息资源
            initMessageSource();

            // Initialize event multicaster for this context.
            // 8. 为 Context 初始化时间多播器
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            // 9. 在特定的 context 子类中初始化其他的特殊 bean
            onRefresh();

            // Check for listener beans and register them.
            // 10. 检查 Listener Bean 并注册
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            // 11. 实例化所有剩下来的单例
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            // 12. 最后一步：发布相关事件
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```
## 2. refresh 步骤
## 2.1. prepareRefresh
该方法执行的是 refresh 之前的预处理操作
```java
protected void prepareRefresh() {
    // 1. 设置值、状态
    this.startupDate = System.currentTimeMillis(); // 记录启动时间为当前时间
    this.closed.set(false); // 记录关闭状态为 false
    this.active.set(true); // 记录启动状态为 true

    if (logger.isInfoEnabled()) {
        logger.info("Refreshing " + this);
    }

    // 2. 初始化属性资源的空方法，留给子类覆盖
    // 在上下文环境中初始化一些占位符属性资源
    // 该方法 AbstractApplicationContext 是一个空实现，留给子类去自定义实现
    initPropertySources();

    // 3. 创建并获取 Environment 对象，验证需要的属性文件
    // 验证被标记为 required 的属性
    // see ConfigurablePropertyResolver#setRequiredProperties
    getEnvironment().validateRequiredProperties();

    // 4. 早期事件的集合
    // 一旦多转换器可用，就发布
    this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

如果是 Web 环境，会覆盖这种行为：

```java
protected void prepareRefresh() {
    // 清除缓存
    this.scanner.clearCache();
    super.prepareRefresh();
}
```

## 2.1. obtainFreshBeanFactory
通知子类，refresh 其内部的 beanFactory
```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 该方法是个抽象类，由子类实现
    // 1. 对于 GenericApplicationContext，仅仅是设置了一个序列化 ID
    refreshBeanFactory();
    // 该方法是个抽象类，有子类实现
    // 2. 对于 GenericApplicationContext，将内部的 DefaultListableBeanFactory 返回
    return getBeanFactory();
}
```

如果子类是 `GenericApplicationContext`，实现的代码清单如下：

```java
protected final void refreshBeanFactory() throws IllegalStateException {
    // 设置刷新标记
    if (!this.refreshed.compareAndSet(false, true)) {
        throw new IllegalStateException(
                "GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
    }
    // 设置一个序列 ID
    this.beanFactory.setSerializationId(getId());
}
```

- `AbstractApplicationContext`
	- `AbstractRefreshableApplicationContext`
		- `AbstractXmlApplicationContext`
			- `ClassPathXmlApplicationContext`
			- `FileSystemXmlApplicationContext`
	- `GenericApplicationContext`
		- `ServletWebServerApplicationContext`


## 2.3. prepareBeanFactory
源码注解上有一句话：Configure the factory's standard context characteristics, such as the context's ClassLoader and post-processors —— 配置工程的标准上下文特性，例如上下文的类加载器以及后置处理器（`BeanPostProcessor`）。包括：
- 设置 Bean `ClassLoader`
- 设置 `BeanExpressionResolver`
- 设置 `PropertyEditorRegistrar`
- 添加 `BeanPostProcessor`
- 设置忽略的依赖接口
- 注册可解析依赖
- 注册 Bean
- ...

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // Tell the internal bean factory to use the context's class loader etc.
    // 设置类加载器
    beanFactory.setBeanClassLoader(getClassLoader());
    // 表达式解析器
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // Configure the bean factory with context callbacks.
    // 添加部分的 bean 后置处理器
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));

    // 设置忽略的依赖接口，即这些接口的实现类不会通过接口类型自动注入
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    // 注册可以解析的依赖
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // Register early post-processor for detecting inner beans as ApplicationListeners.
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // Register default environment beans.
    // 注册 Environment
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    // 注册 SystemProperties，一个 Map<String, Object>
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    // 注册 SystemEnvironment，一个 Map<String, Object>
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
}
```

## 2.4. postProcessBeanFactory
`AbstractApplicationContext` 提供了一个空实现，留给子类具体实现

以下是  `ServletWebServerApplicationContext` 的实现：
```java
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 添加 BeanPostProcessor
    beanFactory.addBeanPostProcessor(new WebApplicationContextServletContextAwareProcessor(this));
    // 忽略依赖接口，不会自动绑定
    beanFactory.ignoreDependencyInterface(ServletContextAware.class);
    registerWebApplicationScopes();
}
```


## 2.5. invokeBeanFactoryPostProcessors
调用 `BeanFactoryPostProcessor`。总体来看调用逻辑分为两块：先处理 `BeanDefinitionRegistryPostProcessor`，然后处理 `BeanFactoryPostProcessor`。

> `BeanDefinitionRegistryPostProcessor` 也属于 `BeanFactoryPostProcessor`

例如，这里会调用 `ConfigurationClassPostProcessor` 处理 `@Configuration` 类。

> `ConfigurationClassPostProcessor` 属于 `BeanDefinitionRegistryPostProcessor`

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    // Bean Factory 后置处理器注册的委托组件
    // 委托其调用后置处理器
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

    // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
    // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
    if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }
}
```

以下是 `PostProcessorRegistrationDelegate` 调用逻辑:

参数 beanFactoryPostProcessors 一般是：

- `CachingMetadataReaderFactoryPostProcessor`
- `ConfigurationWarningsPostProcessor`
- `PropertySourceOrderingPostProcessor`

```java
public static void invokeBeanFactoryPostProcessors( ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

    // Invoke BeanDefinitionRegistryPostProcessors first, if any.
    // 首先调用 BeanDefinitionRegistryPostProcessor
    // 即 Bean Definition 注册表


    // 这个数据结构 processedBeans 用于存储已经处理过的 Bean，防止重复处理
    Set<String> processedBeans = new HashSet<>();

    if (beanFactory instanceof BeanDefinitionRegistry) {
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
        List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

        for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                BeanDefinitionRegistryPostProcessor registryProcessor =
                        (BeanDefinitionRegistryPostProcessor) postProcessor;
                // 进行后置处理
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                registryProcessors.add(registryProcessor);
            }
            else {
                regularPostProcessors.add(postProcessor);
            }
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let the bean factory post-processors apply to them!
        // Separate between BeanDefinitionRegistryPostProcessors that implement
        // PriorityOrdered, Ordered, and the rest.


        // 这个数据结构 currentRegistryProcessors 用于存储每一波的 Processor
        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

        
        // 获得所有的 BeanDefinitionRegistryPostProcessor
        String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);

        // 1. 首先，调用实现了 PriorityOrdered 的 BeanDefinitionRegistryPostProcessor
        for (String ppName : postProcessorNames) {
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                // 防止重复处理
                processedBeans.add(ppName);
            }
        }
        // 排序
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        // 执行
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();

        // 2. 然后，类似地，调用实现了 Ordered 接口的 BeanDefinitionRegistryPostProcessor
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            // 由于 PriorityOrdered 也是 Ordered，因此这里需要借助 processedBeans 结构防止重复处理
            if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();

        // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
        // 3. 最后调用所有其他 BeanDefinitionRegistryPostProcessor 直到没有一个
        // 因为有可能 BeanDefinitionRegistryPostProcessor 还会注册 BeanDefinitionRegistryPostProcessor
        boolean reiterate = true;
        while (reiterate) {
            reiterate = false;
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                if (!processedBeans.contains(ppName)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                    // 假设这里 BeanDefinitionRegistryPostProcessor 的会注册新的 BeanDefinitionRegistryPostProcessor
                    // 因此需要重新迭代
                    reiterate = true;
                }
            }

            // 最后一次迭代，currentRegistryProcessors 空的
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();
        }

        // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
        // 现在调用 postProcessBeanFactory（上面调用的都是 postProcessBeanDefinitionRegistry）
        invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
        invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
    }

    else {
        // Invoke factory processors registered with the context instance.
        invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let the bean factory post-processors apply to them!

    // 这里开始处理 BeanFactoryPostProcessor
    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

    // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
    // Ordered, and the rest.
    List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    List<String> orderedPostProcessorNames = new ArrayList<>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();
    for (String ppName : postProcessorNames) {
        if (processedBeans.contains(ppName)) {
            // skip - already processed in first phase above
            // 跳过，因为 BeanDefinitionRegistryPostProcessor 就是 BeanFactoryPostProcessor，已经在前面处理过了
        }
        else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        }
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }


    // 1. 首先，调用实现了 PriorityOrdered 的 BeanFactoryPostProcessors
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

    // 2. 接下来，调用实现了 Ordered 的 BeanFactoryPostProcessors
    List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
    for (String postProcessorName : orderedPostProcessorNames) {
        orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

    // 3. 最后，调用所有其他的 BeanFactoryPostProcessor
    List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
    for (String postProcessorName : nonOrderedPostProcessorNames) {
        nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

    // Clear cached merged bean definitions since the post-processors might have
    // modified the original metadata, e.g. replacing placeholders in values...
    beanFactory.clearMetadataCache();
}
```

## 2.6. registerBeanPostProcessors

注册 `BeanPostProcessor`。最终实际会注册到 `BeanFactory`（`AbstractBeanFactory`）的 `beanPostProcessors` 结构中，该数据类型为一个 `List<BeanPostProcessor>`：


> 不同类型的 `BeanPostProcessor` 在 `getBean` 的执行时机是不是一样的。

`MergedBeanDefinitionPostProcessor` 存放在 `internalPostProcessors`

```java
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    // 委托注册
    PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
```

```java
public static void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
    // 获得所有的 BeanPostProcessor
    // 默认的可能有：
    // 1. org.springframework.context.annotation.internalAutowiredAnnotationProcessor
    // 2. org.springframework.context.annotation.internalCommonAnnotationProcessor
    // 3. org.springframework.context.annotation.internalPersistenceAnnotationProcessor
    // 4. org.springframework.boot.context.properties.ConfigurationPropertiesBindingPostProcessor
    // 5. configurationPropertiesBeans
    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

    // Register BeanPostProcessorChecker that logs an info message when
    // a bean is created during BeanPostProcessor instantiation, i.e. when
    // a bean is not eligible for getting processed by all BeanPostProcessors.
    int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
    beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

    // Separate between BeanPostProcessors that implement PriorityOrdered,
    // Ordered, and the rest.
    List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
    List<String> orderedPostProcessorNames = new ArrayList<>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();

    // 遍历所有的 BeanPostProcessor
    for (String ppName : postProcessorNames) {
        if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            // 如果实现了 PriorityOrdered 接口, getBean 触发创建 bean
            BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
            priorityOrderedPostProcessors.add(pp);
            if (pp instanceof MergedBeanDefinitionPostProcessor) {
                internalPostProcessors.add(pp);
            }
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            // 如果实现了 Ordered 接口
            orderedPostProcessorNames.add(ppName);
        }
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }

    // 1. 首先，注册实现了 PriorityOrdered 接口的 BeanPostProcessor
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

    // 2. 接下来，注册实现了 Ordered 接口的 BeanPostProcessors
    List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
    for (String ppName : orderedPostProcessorNames) {
        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
        orderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
        }
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, orderedPostProcessors);

    // 3. 注册所有常规的 BeanPostProcessor，即没有实现任何优先级接口
    List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
    for (String ppName : nonOrderedPostProcessorNames) {
        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
        nonOrderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
        }
    }
    registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

    // Finally, re-register all internal BeanPostProcessors.
    // 4. 最终，重新注册所有内部的 BeanPostProcessor
    sortPostProcessors(internalPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, internalPostProcessors);

    // Re-register post-processor for detecting inner beans as ApplicationListeners,
    // moving it to the end of the processor chain (for picking up proxies etc).
    // 注册 ApplicationListenerDetector
    // 作用：
    // 在 Bean 创建完成后检查是否是 ApplicationListener
    // 如果是，则添加到容器的 applicationListeners 中
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```

## 2.7. initMessageSource
国际化功能；消息绑定；消息解析

```java
protected void initMessageSource() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	// 检查是否包含 "messageSource" 的 bean
	if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
		this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
		// Make MessageSource aware of parent MessageSource.
		if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
			HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
			if (hms.getParentMessageSource() == null) {
				// Only set parent context as parent MessageSource if no parent MessageSource
				// registered already.
				hms.setParentMessageSource(getInternalParentMessageSource());
			}
		}
		if (logger.isTraceEnabled()) {
			logger.trace("Using MessageSource [" + this.messageSource + "]");
		}
	}
	else {
		// Use empty MessageSource to be able to accept getMessage calls.
		DelegatingMessageSource dms = new DelegatingMessageSource();
		dms.setParentMessageSource(getInternalParentMessageSource());
		this.messageSource = dms;
		beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
		if (logger.isTraceEnabled()) {
			logger.trace("No '" + MESSAGE_SOURCE_BEAN_NAME + "' bean, using [" + this.messageSource + "]");
		}
	}
}
```

## 2.8. initApplicationEventMulticaster
初始化事件派发器。一般情况下都是容器给我们默认创建一个 `SimpleApplicationEventMulticaster`。

```java
protected void initApplicationEventMulticaster() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
		this.applicationEventMulticaster = beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
	}
	else {
		// 不存在则创建一个 SimpleApplicationEventMulticaster
		this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
		beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
	}
}
```

## 2.9. onRefresh
`AbstractApplicationContext` 的空实现，留给子类个性化实现


## 2.10. registerListeners

注册 `ApplicationListener`，向之前初始化的 `ApplicationEventMulticaster` 中注册大量的监听器。

```java
protected void registerListeners() {
    // Register statically specified listeners first.
    // 首先，静态地注册特定的 Listener
    for (ApplicationListener<?> listener : getApplicationListeners()) {
        getApplicationEventMulticaster().addApplicationListener(listener);
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let post-processors apply to them!
    String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
    for (String listenerBeanName : listenerBeanNames) {
        getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
    }

    // Publish early application events now that we finally have a multicaster...
    Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
    this.earlyApplicationEvents = null;
    if (!CollectionUtils.isEmpty(earlyEventsToProcess)) {
        for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
            getApplicationEventMulticaster().multicastEvent(earlyEvent);
        }
    }
}
```

## 2.11. finishBeanFactoryInitialization
初始化所有剩下的（非懒加载）单例 Bean

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // Initialize conversion service for this context.
    // 类型转换
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // Register a default embedded value resolver if no bean post-processor
    // (such as a PropertyPlaceholderConfigurer bean) registered any before:
    // at this point, primarily for resolution in annotation attribute values.
    // 值解析器
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    // Stop using the temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(null);

    // Allow for caching all bean definition metadata, not expecting further changes.
    beanFactory.freezeConfiguration();

    // Instantiate all remaining (non-lazy-init) singletons.
    // 实例化所有剩余的（非懒加载）单例
    beanFactory.preInstantiateSingletons();
}
```

下面的方法对剩余的 bean 进行了创建，其实 getBean 就是隐式包含一种对 bean 的创建。
```java
public void preInstantiateSingletons() throws BeansException {
    if (logger.isTraceEnabled()) {
        logger.trace("Pre-instantiating singletons in " + this);
    }

    // Iterate over a copy to allow for init methods which in turn register new bean definitions.
    // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
    // 所有的 Bean Definition 信息
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    // Trigger initialization of all non-lazy singleton beans...
    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // 必须是非抽象、单例、非懒加载
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            // 判断是否是 FactoryBean（是否实现了 FactoryBean）
            if (isFactoryBean(beanName)) {
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                if (bean instanceof FactoryBean) {
                    FactoryBean<?> factory = (FactoryBean<?>) bean;
                    boolean isEagerInit;
                    if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                        isEagerInit = AccessController.doPrivileged(
                                (PrivilegedAction<Boolean>) ((SmartFactoryBean<?>) factory)::isEagerInit,
                                getAccessControlContext());
                    }
                    else {
                        isEagerInit = (factory instanceof SmartFactoryBean &&
                                ((SmartFactoryBean<?>) factory).isEagerInit());
                    }
                    if (isEagerInit) {
                        getBean(beanName);
                    }
                }
            }
            else {
                getBean(beanName);
            }
        }
    }

    // Trigger post-initialization callback for all applicable beans...
    // 后置初始化回调所有的 Bean
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                    smartSingleton.afterSingletonsInstantiated();
                    return null;
                }, getAccessControlContext());
            }
            else {
                smartSingleton.afterSingletonsInstantiated();
            }
        }
    }
}
```


## 2.12. finishRefresh
完成 refresh 执行的操作
```java
protected void finishRefresh() {
    // Clear context-level resource caches (such as ASM metadata from scanning).
    clearResourceCaches();

    // Initialize lifecycle processor for this context.
    initLifecycleProcessor();

    // Propagate refresh to lifecycle processor first.
    getLifecycleProcessor().onRefresh();

    // 发布容器刷新完毕的事件
    publishEvent(new ContextRefreshedEvent(this));

    // Participate in LiveBeansView MBean, if active.
    LiveBeansView.registerApplicationContext(this);
}
```
