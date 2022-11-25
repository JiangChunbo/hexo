---
title: MyBatis Spring MapperScannerConfigurer
date: 2022-11-25 09:31:53
tags:
- MyBatis Spring
---


`MapperScannerConfigurer` 是一个  `BeanDefinitionRegistryPostProcessor`。`BeanDefinitionRegistryPostProcessor` 的调用时机是 `refresh()` 方法中 `postProcessBeanFactory` 之后。其实现方法 `postProcessBeanDefinitionRegistry` 如下:

```java{.line-numbers}
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    if (this.processPropertyPlaceHolders) {
        processPropertyPlaceHolders();
    }

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
    scanner.registerFilters();
    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage,
            ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
}
```

可以看到，`MapperScannerConfigurer` 构造了一个 `org.mybatis.spring.mapper.ClassPathMapperScanner`，并调用了它的 `scan` 方法。


`ClassPathMapperScanner` 继承自 `org.springframework.context.annotation.ClassPathBeanDefinitionScanner`，因此具有一些扫描类路径资源的能力。



# 问题

**MyBatis-Spring 的扫描默认扫描哪些接口?**

先看 `MapperScannerConfigurer` 调用链：

`org.mybatis.spring.mapper.MapperScannerConfigurer`#`postProcessBeanDefinitionRegistry`

👉`org.springframework.context.annotation.ClassPathBeanDefinitionScanner`#`scan`

👉`org.mybatis.spring.mapper.ClassPathMapperScanner`#`doScan`


👉`org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider`#`findCandidateComponents`

👉`org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider`#`isCandidateComponent`



`isCandidateComponent` 是对父类 `org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider` 方法的重写: 

```java
protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
    return beanDefinition.getMetadata().isInterface() && beanDefinition.getMetadata().isIndependent();
}
```

只需要判断该类是不是接口，并且是"独立"的即可。


> <center><strong>什么是"独立"的类?</strong></center>
> 注释文档是这样描述: Determine whether the underlying class is independent, i.e. whether it is a top-level class or a nested class (static inner class) that can be constructed independently from an enclosing class.
> 
> 也就是顶级类或者嵌套类（静态内部类），相关信息参考{% post_link Language/Java/Java-类(接口)的种类 %}