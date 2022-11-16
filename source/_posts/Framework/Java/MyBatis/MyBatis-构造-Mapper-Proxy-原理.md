---
title: MyBatis 构造 Mapper Proxy 原理
date: 2022-07-16 08:10:44
categories:
- 框架
tags:
- MyBatis
---


在 MyBatis 应用中，我们定义的 `Mapper` 接口，最终都会转换为 JDK 动态代理对象 `Proxy`。

假设有 UserMapper.xml：

```java
SqlSession sqlSession = sqlSessionFactory.openSession();
UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
```
在 MyBatis 的框架中，提供了 `SqlSession` 的默认实现类 `DefaultSqlSession`，关注其实现方法 `getMapper()`：

```java
// DefaultSqlSession.java
public <T> T getMapper(Class<T> type) {
    return configuration.getMapper(type, this);
}
```

方法 getMapper 内部调用了属性 `Configuration` 的 getMapper 方法，将代码细节委托给 `Configuration` 实现。
```java
// Configuration.java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
}
```

`Configuration` 内部的实现是将 getMapper 的细节委托给 `MapperRegistry` 实现，顾名思义，Mapper 注册表。
```java
// MapperRegistry.java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    // 确保该类型被识别
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
        throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
        return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
        throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
}
```
由 `MapperRegistry` 的实现可以知道，其内部维护了一个 `knownMappers` 结构，用于进行 `Mapper` 接口到 `MapperProxyFactory` 的映射：
```java
private final Map<Class<?>, MapperProxyFactory<?>> knownMappers = new HashMap<>();
```

`MapperProxyFactory`，由类名可以知道这是一个批量生产 MapperProxy 的工厂类，关注其如何生产 MapperProxy

```java
// MapperProxyFactory.java
public T newInstance(SqlSession sqlSession) {
    // 创建一个 InvocationHandler
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
}
protected T newInstance(MapperProxy<T> mapperProxy) {
    // 调用 JDK 动态代理的方法创建代理类
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}
```
首先，创建了一个 `MapperProxy`，这是一个由 MyBatis 提供的 `InvocationHandler` 的实现类，然后将其传递给 `Proxy.newProxyInstance` 方法创建 Proxy 并返回。


> 从名字上看，MapperProxyFactory 似乎是生产 MapperProxy 的工厂，但在过程中，MapperProxy 只是扮演了代理类的 Handler 角色，MapperProxyFactory 真正生产的应该是代理类。

