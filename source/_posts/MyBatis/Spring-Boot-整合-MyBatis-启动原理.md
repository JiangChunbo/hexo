---
title: Spring Boot 整合 MyBatis 启动原理
date: 2022-07-15 16:54:14
categories:
- 框架
tags:
- Spring Boot
- MyBatis
---


# Spring Boot 整合 MyBatis 启动原理

需要知道，MyBatis 是通过 JDK 动态代理技术创建 Mapper 接口的代理类的。

MyBatis 整合 Spring Boot 需要解决的是如何将自己创建的代理对象（`java.lang.reflect.Proxy`）交给 Spring 容器管理，这与将一个类（Class）交给 Spring 管理有所不同。

> 将对象交给 Spring 容器管理，我们可以选择注入 Bean Definition，然后让 Spring 完成对象构造、配置、初始化等操作，然后放到 `singletonObjects` 单例池中，也可以选择直接放到单例池中，也就是不构造 Bean Definition，也不会存在于 `beanDefinitionMap`，这就是下面提及的 `SingletonBeanRegistry.registerSingleton` 方式

将自定义的对象交给 Spring 容器管理，一般考虑的方式是：

- @Bean
- factory method
- SingletonBeanRegistry.registerSingleton
- FactoryBean



对于 `@Bean` 方式，我们可以使用这种方式：

```java
@Bean
public UserMapper videoMapper(SqlSession sqlSession) {
    // 添加到 MapperRegistry.knownMappers，否则 SqlSession.getMapper 会抛出异常
    sqlSession.getConfiguration().addMapper(UserMapper.class);
    return sqlSession.getMapper(UserMapper.class);
}
```
上述方式的确可行，但是我们每定义一个 Mapper 就需要写一个 `@Bean` 方法，还是比较麻烦的。


factory method 注入的方式与 `@Bean` 大同小异。

MyBatis 整合进 Spring 选择的是 `FactoryBean` 方式，通过向容器注入 `FactoryBean` 这种特殊的 Bean Definitnion，进而注入 `Mapper`

比较朴素的想法是，对于每个 `Mapper` 我们都为它定义一个 `FactoryBean`，但是这样工作量太大，MyBatis 通过动态 Class 实现了只需要一个 `MapperFactoryBean` 就可以构造出不同的 `Mapper` 实例

> 底层对应 `MapperFactoryBean` 的类属性 `mapperInterface`，表示不同的 `Mapper` 接口的 `Class` 对象

在使用 Spring 整合 MyBatis 的时候，通常会使用如下配置，这也是官网 [Getting Started](http://mybatis.org/spring/getting-started.html) 提供的案例：

```xml
<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
  <property name="mapperInterface" value="org.mybatis.spring.sample.mapper.UserMapper" />
  <property name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
```

MyBatis Spring Boot Starter 为了解决上述这种按需配置 `MapperFactoryBean` 的繁琐步骤，引入了 Spring 类路径扫描机制。

## Spring Boot 加载 Mapper

1. Spring Boot 启动过程中，会调用 `AbstractApplicationContext` 的 `refresh` 方法，其中一个过程是 `invokeBeanFactoryPostProcessors`，它会调用 `ConfigurationClassPostProcessor` 后置处理器加载具有 `@Configuration` 注解的 Bean。
2. `ConfigurationClassPostProcessor` 关注的是 `@Configuration` 类型的 Bean，它会判断该 Bean 是否包含 `@Import` 注解。假设我们将 `@MapperScan` 添加在主启动类上（一般是具有 `@SpringBootApplication` 注解），那么将会读取主启动类注解，发现具有 `@MapperScan`，而 `@MapperScan` 嵌套了 `@Import` 注解，value 为 `MapperScannerRegistrar.class`，因此，将会构造`MapperScannerRegistrar` Bean Definition 进入 Spring 容器



当 `@Configuration` 类解析完毕加载到容器后，就会执行 `load` 方法，用于加载这些 `@Configuration` 类相关的 Bean、Resource 等（如 `@Import`, `@ImportResource`）：

```java
this.reader.loadBeanDefinitions(configClasses)
```

当 `load` 完毕，此时 `MapperScannerRegistrar` 已经加载到 Spring 容器，而且 `load` 会主动触发 `MapperScannerRegistrar` 的回调方法 `registerBeanDefinitions`。


`MapperScannerRegistrar` 是 `ImportBeanDefinitionRegistrars` 的子类，其关键方法 `registerBeanDefinitions` 如下：

```java
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    // 获得 @MapperScan 的全部属性
    AnnotationAttributes mapperScanAttrs = AnnotationAttributes
        .fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
    if (mapperScanAttrs != null) {
        registerBeanDefinitions(importingClassMetadata, mapperScanAttrs, registry,
            generateBaseBeanName(importingClassMetadata, 0));
    }
}
```

在底层构造了一个 `MapperScannerConfigurer` Bean Definitnion，并注册到容器。

```java
void registerBeanDefinitions(AnnotationMetadata annoMeta, AnnotationAttributes annoAttrs, BeanDefinitionRegistry registry, String beanName) {
    // 构造 MapperScannerConfigurer Bean Definition
    BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);
    builder.addPropertyValue("processPropertyPlaceHolders", true);

    Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
    if (!Annotation.class.equals(annotationClass)) {
        builder.addPropertyValue("annotationClass", annotationClass);
    }

    Class<?> markerInterface = annoAttrs.getClass("markerInterface");
    if (!Class.class.equals(markerInterface)) {
        builder.addPropertyValue("markerInterface", markerInterface);
    }

    Class<? extends BeanNameGenerator> generatorClass = annoAttrs.getClass("nameGenerator");
    if (!BeanNameGenerator.class.equals(generatorClass)) {
        builder.addPropertyValue("nameGenerator", BeanUtils.instantiateClass(generatorClass));
    }

    Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs.getClass("factoryBean");
    if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
        builder.addPropertyValue("mapperFactoryBeanClass", mapperFactoryBeanClass);
    }

    String sqlSessionTemplateRef = annoAttrs.getString("sqlSessionTemplateRef");
    if (StringUtils.hasText(sqlSessionTemplateRef)) {
        builder.addPropertyValue("sqlSessionTemplateBeanName", annoAttrs.getString("sqlSessionTemplateRef"));
    }

    String sqlSessionFactoryRef = annoAttrs.getString("sqlSessionFactoryRef");
    if (StringUtils.hasText(sqlSessionFactoryRef)) {
        builder.addPropertyValue("sqlSessionFactoryBeanName", annoAttrs.getString("sqlSessionFactoryRef"));
    }

    // 收集 basePackage
    List<String> basePackages = new ArrayList<>();
    basePackages.addAll(
        Arrays.stream(annoAttrs.getStringArray("value")).filter(StringUtils::hasText).collect(Collectors.toList()));

    basePackages.addAll(Arrays.stream(annoAttrs.getStringArray("basePackages")).filter(StringUtils::hasText)
        .collect(Collectors.toList()));

    basePackages.addAll(Arrays.stream(annoAttrs.getClassArray("basePackageClasses")).map(ClassUtils::getPackageName)
        .collect(Collectors.toList()));

    if (basePackages.isEmpty()) {
        basePackages.add(getDefaultBasePackage(annoMeta));
    }

    String lazyInitialization = annoAttrs.getString("lazyInitialization");
    if (StringUtils.hasText(lazyInitialization)) {
        builder.addPropertyValue("lazyInitialization", lazyInitialization);
    }

    String defaultScope = annoAttrs.getString("defaultScope");
    if (!AbstractBeanDefinition.SCOPE_DEFAULT.equals(defaultScope)) {
        builder.addPropertyValue("defaultScope", defaultScope);
    }

    // 设置 basePackage 属性，用逗号(,)分隔
    builder.addPropertyValue("basePackage", StringUtils.collectionToCommaDelimitedString(basePackages));

    registry.registerBeanDefinition(beanName, builder.getBeanDefinition());
}
```

`MapperScannerConfigurer` 是 `BeanDefinitionRegistryPostProcessor` 的实现类，也就说其调用时机在 refresh 方法的 `invokeBeanFactoryPostProcessors` 中，它执行了类路径扫描，并注册了相关的 Mapper，关键方法如下：

```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    if (this.processPropertyPlaceHolders) {
        processPropertyPlaceHolders();
    }
    // 创建扫描器
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    scanner.setMapperFactoryBeanClass(this.mapperFactoryBeanClass);
    if (StringUtils.hasText(lazyInitialization)) {
        scanner.setLazyInitialization(Boolean.valueOf(lazyInitialization));
    }
    if (StringUtils.hasText(defaultScope)) {
        scanner.setDefaultScope(defaultScope);
    }
    scanner.registerFilters();

    // 这里将之前用逗号(,)分隔的 basePackage 字符串分解成 String[]
    scanner.scan(
        StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
}
```

`ClassPathBeanDefinitionScanner` 的 scan 方法如下：

```java
public int scan(String... basePackages) {
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

    doScan(basePackages);

    // Register annotation config processors, if necessary.
    if (this.includeAnnotationConfig) {
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    }

    return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
}
```

`ClassPathBeanDefinitionScanner` 的 doScan 方法如下，它返回扫描到的所有 `BeanDefinitionHolder` 的集合：

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Assert.notEmpty(basePackages, "At least one base package must be specified");
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
    for (String basePackage : basePackages) {
        Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
        for (BeanDefinition candidate : candidates) {
            ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
            candidate.setScope(scopeMetadata.getScopeName());
            String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
            if (candidate instanceof AbstractBeanDefinition) {
                postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
            }
            if (candidate instanceof AnnotatedBeanDefinition) {
                AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
            }
            if (checkCandidate(beanName, candidate)) {
                BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                definitionHolder =
                        AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                beanDefinitions.add(definitionHolder);
                registerBeanDefinition(definitionHolder, this.registry);
            }
        }
    }
    return beanDefinitions;
}
```

MyBatis 覆盖了原来的 `doScan` 方法，因为我们不能把扫描到的 `Mapper` 接口交给 Spring 容器，否则后续的实例化将无法进行。因此，MyBatis 在调用 `super.doScan()` 方法得到扫描到的 `Set<BeanDefinitionHolder>` 之后又进行后置处理：

```java
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    // 调用 Spring 内置提供的 doScan 方法
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
        LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages)
            + "' package. Please check your configuration.");
    } else {
        // 后置处理
        processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
}
```

MyBatis 的 `ClassPathMapperScanner` 后置处理方法如下：

```java
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) { AbstractBeanDefinition definition;
    BeanDefinitionRegistry registry = getRegistry();
    for (BeanDefinitionHolder holder : beanDefinitions) {
        definition = (AbstractBeanDefinition) holder.getBeanDefinition();
        boolean scopedProxy = false;
        if (ScopedProxyFactoryBean.class.getName().equals(definition.getBeanClassName())) {
            definition = (AbstractBeanDefinition) Optional
                .ofNullable(((RootBeanDefinition) definition).getDecoratedDefinition())
                .map(BeanDefinitionHolder::getBeanDefinition).orElseThrow(() -> new IllegalStateException(
                    "The target bean definition of scoped proxy bean not found. Root bean definition[" + holder + "]"));
            scopedProxy = true;
        }
        // 扫描到的 Mapper 接口，因此这里获取到的应该是形如 UserMapper 之类的
        String beanClassName = definition.getBeanClassName();
        LOGGER.debug(() -> "Creating MapperFactoryBean with name '" + holder.getBeanName() + "' and '" + beanClassName
            + "' mapperInterface");

        // Mapper 接口是 Bean 原始的类型，但是实际类型是 MapperFactoryBean
        // 将原始类型（Mapper）作为构造器参数传入
        definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); // issue #59
        // 将 BeanClass 强制修改为 MapperFactoryBean 类型
        definition.setBeanClass(this.mapperFactoryBeanClass);

        definition.getPropertyValues().add("addToConfig", this.addToConfig);

        // Attribute for MockitoPostProcessor
        // https://github.com/mybatis/spring-boot-starter/issues/475
        definition.setAttribute(FACTORY_BEAN_OBJECT_TYPE, beanClassName);

        boolean explicitFactoryUsed = false;
        if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
        definition.getPropertyValues().add("sqlSessionFactory",
            new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
        explicitFactoryUsed = true;
        } else if (this.sqlSessionFactory != null) {
        definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
        explicitFactoryUsed = true;
        }

        if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
        if (explicitFactoryUsed) {
            LOGGER.warn(
                () -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate",
            new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
        explicitFactoryUsed = true;
        } else if (this.sqlSessionTemplate != null) {
        if (explicitFactoryUsed) {
            LOGGER.warn(
                () -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
        explicitFactoryUsed = true;
        }

        if (!explicitFactoryUsed) {
        LOGGER.debug(() -> "Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
        }

        definition.setLazyInit(lazyInitialization);

        if (scopedProxy) {
        continue;
        }

        if (ConfigurableBeanFactory.SCOPE_SINGLETON.equals(definition.getScope()) && defaultScope != null) {
        definition.setScope(defaultScope);
        }

        if (!definition.isSingleton()) {
        BeanDefinitionHolder proxyHolder = ScopedProxyUtils.createScopedProxy(holder, registry, true);
        if (registry.containsBeanDefinition(proxyHolder.getBeanName())) {
            registry.removeBeanDefinition(proxyHolder.getBeanName());
        }
        registry.registerBeanDefinition(proxyHolder.getBeanName(), proxyHolder.getBeanDefinition());
        }

    }
}
```


### AutoConfiguredMapperScannerRegistrar


如果没有发现 `MapperScannerConfigurer` 就会导入 `AutoConfiguredMapperScannerRegistrar`：

```java
@org.springframework.context.annotation.Configuration
@Import(AutoConfiguredMapperScannerRegistrar.class)
@ConditionalOnMissingBean({ MapperFactoryBean.class, MapperScannerConfigurer.class })
public static class MapperScannerRegistrarNotFoundConfiguration implements InitializingBean {

    @Override
    public void afterPropertiesSet() {
        logger.debug(
            "Not found configuration for registering mapper bean using @MapperScan, MapperFactoryBean and MapperScannerConfigurer.");
    }

}
```

`AutoConfiguredMapperScannerRegistrar` 的定义如下：

```java
public static class AutoConfiguredMapperScannerRegistrar implements BeanFactoryAware, ImportBeanDefinitionRegistrar {

    private BeanFactory beanFactory;

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        if (!AutoConfigurationPackages.has(this.beanFactory)) {
        logger.debug("Could not determine auto-configuration package, automatic mapper scanning disabled.");
        return;
        }

        logger.debug("Searching for mappers annotated with @Mapper");

        List<String> packages = AutoConfigurationPackages.get(this.beanFactory);
        if (logger.isDebugEnabled()) {
        packages.forEach(pkg -> logger.debug("Using auto-configuration base package '{}'", pkg));
        }

        BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);
        builder.addPropertyValue("processPropertyPlaceHolders", true);
        builder.addPropertyValue("annotationClass", Mapper.class);
        builder.addPropertyValue("basePackage", StringUtils.collectionToCommaDelimitedString(packages));
        BeanWrapper beanWrapper = new BeanWrapperImpl(MapperScannerConfigurer.class);
        Set<String> propertyNames = Stream.of(beanWrapper.getPropertyDescriptors()).map(PropertyDescriptor::getName)
            .collect(Collectors.toSet());
        if (propertyNames.contains("lazyInitialization")) {
        // Need to mybatis-spring 2.0.2+
        builder.addPropertyValue("lazyInitialization", "${mybatis.lazy-initialization:false}");
        }
        if (propertyNames.contains("defaultScope")) {
        // Need to mybatis-spring 2.0.6+
        builder.addPropertyValue("defaultScope", "${mybatis.mapper-default-scope:}");
        }
        registry.registerBeanDefinition(MapperScannerConfigurer.class.getName(), builder.getBeanDefinition());
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }`

}
```

分析：

实现了 `BeanFactoryAware` 接口，因此该 Bean 在构造的过程中会自动调用 `setBeanFactory` 方法，使得该 Bean 获得 beanFactory 的引用

实现了 `ImportBeanDefinitionRegistrar` 接口，