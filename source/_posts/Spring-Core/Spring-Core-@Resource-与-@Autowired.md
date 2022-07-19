---
title: Spring Bean @Autowired 与 JSR-250 @Resource 
date: 2022-07-19 09:11:46
tags:
- Spring Bean
---

# Spring Bean @Autowired 与 JSR-250 @Resource 

## 

## @Autowired

由 Spring 提供的注解，依赖注入的过程由 `AutowiredAnnotationBeanPostProcessor` 执行。

通常这一步骤发生在 `populateBean` 流程之中，使用特定的 `InstantiationAwareBeanPostProcessor` 进行属性注入



## @Resource

由 JSR-250 中提供的注解，依赖注入的过程由 `CommonAnnotationBeanPostProcessor` 执行。

通常这一步骤发生在 `populateBean` 流程之中，使用特定的 `InstantiationAwareBeanPostProcessor` 进行属性注入


1. 如果 `@Resource` 指定了 `name`，则按照 `name` 查找 Bean，找到则注入；找不到抛出异常
2. 如果 `@Resource` 没有指定 `name`，通过 Java 反射得到 `Field` 属性 `name`，找不到则按照类型匹配


> 由于显式指定了 `@Resource` 的 `name`，因此在找不到的情况下必须抛出异常，这可能是人为的疏漏；如果没有指定 `name`，那么容器会智能地按照属性名、类型地顺序依次寻找。


```java
protected Object autowireResource(BeanFactory factory, LookupElement element, @Nullable String requestingBeanName)
        throws NoSuchBeanDefinitionException {

    Object resource;
    Set<String> autowiredBeanNames;
    // 此处的 element.name 以及在构造器中赋值
    String name = element.name;

    if (factory instanceof AutowireCapableBeanFactory) {
        AutowireCapableBeanFactory beanFactory = (AutowireCapableBeanFactory) factory;
        DependencyDescriptor descriptor = element.getDependencyDescriptor();
        if (this.fallbackToDefaultTypeMatch && element.isDefaultName && !factory.containsBean(name)) {
            autowiredBeanNames = new LinkedHashSet<>();
            resource = beanFactory.resolveDependency(descriptor, requestingBeanName, autowiredBeanNames, null);
            if (resource == null) {
                throw new NoSuchBeanDefinitionException(element.getLookupType(), "No resolvable resource object");
            }
        }
        else {
            resource = beanFactory.resolveBeanByName(name, descriptor);
            autowiredBeanNames = Collections.singleton(name);
        }
    }
    else {
        resource = factory.getBean(name, element.lookupType);
        autowiredBeanNames = Collections.singleton(name);
    }

    if (factory instanceof ConfigurableBeanFactory) {
        ConfigurableBeanFactory beanFactory = (ConfigurableBeanFactory) factory;
        for (String autowiredBeanName : autowiredBeanNames) {
            if (requestingBeanName != null && beanFactory.containsBean(autowiredBeanName)) {
                beanFactory.registerDependentBean(autowiredBeanName, requestingBeanName);
            }
        }
    }

    return resource;
}
```