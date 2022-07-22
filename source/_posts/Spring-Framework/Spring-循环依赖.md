---
title: Spring 循环依赖
date: 2022-07-10 10:19:35
categories:
- 框架
tags:
- Spring Framework
---


# Spring 循环依赖
## 三级缓存
singletonObjects
earlySingletonObjects
singletonFactories


对于 earlySingletonObjects 的使用场景存在于多循环依赖，比如 beanA 依赖于 beanB 和 beanC，beanB 和 beanC 分别依赖 beanA。在 beanB 进行属性注入 beanA 的时候，beanA 已经从 singletonFactories 构造出一个 earlySingletonObject 了，因此在 beanC 注入 beanA 的时候不必重复构造 beanA，只需从 earlySingletonObjects 中取得即可.

```java
A->B->A
A->C->A
```

## 场景

假设 beanA 依赖 beanB，beanB 依赖 beanA，以这种最朴素的场景为例

## 入口
假设程序以 beanA 开始解析
```java
AbstractApplicationContext.refresh()
-> AbstractApplicationContext.finishBeanFactoryInitialization(beanFactory)
-> DefaultLisableBeanFactory.preInstantiateSingletons()
-> AbstractBeanFactory.getBean(beanA)
-> AbstractBeanFactory.doGetBean(beanA)
```


## 流程分析

1. 检查 singletonObjects 是否存在 beanA。

```java
Object sharedInstance = getSingleton(beanName);
```
```java
public Object getSingleton(String beanName) {
    // 注意第二个参数为 true
    return getSingleton(beanName, true);
}
```
```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 此时 beanA 还没有开始创建，这里一定返回 null
    Object singletonObject = this.singletonObjects.get(beanName);
    // 由于 beanA 还没有开始创建，因此也不会存在于 singletonsCurrentlyInCreation
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```

2. 开始创建 beanA

```java
// 第二个参数是一个类似于 java.util.function.Supplier 的函数式接口
// 用于创建 beanA
sharedInstance = getSingleton(beanName, () -> {
    try {
        return createBean(beanName, mbd, args);
    }
    catch (BeansException ex) {
        destroySingleton(beanName);
        throw ex;
    }
});
```


```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "Bean name must not be null");
    synchronized (this.singletonObjects) {
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            if (this.singletonsCurrentlyInDestruction) {
                throw new BeanCreationNotAllowedException(beanName,
                        "Singleton bean creation not allowed while singletons of this factory are in destruction " +
                        "(Do not request a bean from a BeanFactory in a destroy method implementation!)");
            }
            if (logger.isDebugEnabled()) {
                logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
            }
            // 该方法会将 beanA 添加到 singletonsCurrentlyInCreation
            beforeSingletonCreation(beanName);
            boolean newSingleton = false;
            boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
            if (recordSuppressedExceptions) {
                this.suppressedExceptions = new LinkedHashSet<>();
            }
            try {
                // 此处调用函数式接口进行 beanA 的创建
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            }
            catch (IllegalStateException ex) {
                // Has the singleton object implicitly appeared in the meantime ->
                // if yes, proceed with it since the exception indicates that state.
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    throw ex;
                }
            }
            catch (BeanCreationException ex) {
                if (recordSuppressedExceptions) {
                    for (Exception suppressedException : this.suppressedExceptions) {
                        ex.addRelatedCause(suppressedException);
                    }
                }
                throw ex;
            }
            finally {
                if (recordSuppressedExceptions) {
                    this.suppressedExceptions = null;
                }
                afterSingletonCreation(beanName);
            }
            if (newSingleton) {
                addSingleton(beanName, singletonObject);
            }
        }
        return singletonObject;
    }
}
```


3. 创建 beanA

通过 getSingleton 传递的函数式接口调用链如下 

```java
AbstractAutowireCapableBeanFactory.createBean()
-> AbstractAutowireCapableBeanFactory.doCreateBean(beanName, mbdToUse, args)
```


4. 添加到 singletonFactories

`doCreateBean` 方法首先会进行 beanA 的实例化：
```java
instanceWrapper = createBeanInstance(beanName, mbd, args);
```

然后紧跟着将实例化的 beanA 以函数式接口 Supplier 的形式（实际上是 ObjectFactory）添加到 singletonFactories：

```java
// 这里的参数 bean 是刚刚实例化完毕的 beanA
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
```


5. 开始填充 beanA 的属性

```java
populateBean(beanName, mbd, instanceWrapper);
```


在填充过程中会调用一些 `InstantiationAwareBeanPostProcessor` 进行 `postProcessProperties` 操作，

如果你使用的是 `@Autowired` 进行属性绑定，那么 `AutowiredAnnotationBeanPostProcessor` 会处理关于 beanB 的属性绑定问题。


6. 解析依赖 beanB

`AutowiredAnnotationBeanPostProcessor` 的注入调用链如下，最终又会回到 beanFactory 的 getBean 方法：


```java
DefaultListableBeanFactory.doResolveDependency()
-> DependencyDescriptor.resolveCandidate("beanB", BeanB.class, beanFactory)
-> beanFactory.getBean("beanB")
```


7. 与步骤 1 相同

beanB 此时还没有创建，因此不会存在于 `singletonObjects` ，而且也不会存在于 `singletonsCurrentlyInCreation`


8. 将 beanB 添加到 `singletonsCurrentlyInCreation`
9. 实例化 beanB，将对象工程添加到 `singletonFactories`

10. 填充 beanB 属性，相关的 `InstantiationAwareBeanPostProcessor` 发挥作用。此时，发现 beanB 依赖 beanA，继续调用 beanFactory.getBean("beanA")

11. 与步骤 1 类似

由于 beanA 在创建前已经将自己放到 `singletonsCurrentlyInCreation` 中，而且将自己的对象工厂放到 `singletonFactories` 中了，因此会调用 `singletonFactories` 中的对象工厂方法获得一个 beanA，并且 beanA 的对象工厂会从 `singletonFactories` 移除，同时添加到 `earlySingletonObjects`

> 此时这个 beanA 属性还没有填充


这时候 getBean("beanA") 返回得到一个还未填充属性的 beanA


12. 回到 beanB 填充属性，将得到的 beanA 填充进自己的属性。接着，beanB 完成了自己的属性填充就可以将对象添加到 `singletonObjects` 中，并且移除 `singletonFactories` 和 `earlySingletonObjects` 相关的对象


13. 回到 beanA 填充属性，将得到的 beanB 填充进自己的属性。接着，beanA 完成了自己的属性填充就可以将对象添加到 `singletonObjects` 中，并且移除 `singletonFactories` 和 `earlySingletonObjects` 相关的对象