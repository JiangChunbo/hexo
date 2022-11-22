---
title: MyBatis BoundSql 记录
date: 2022-11-22 11:50:42
tags:
- MyBatis
---

# 0. 参考引用


# 由谁创建

BoundSql 是由 SqlSource 创建。更准确地说，是由 StaticSqlSource

```java
public interface SqlSource {
  BoundSql getBoundSql(Object parameterObject);
}
```



# 杂项问题

## 关于 additionalParameters

`additionalParameters` 是 BoundSql 中的一个属性，类型是 HashMap。

其中存储的属性可能有:

- `_parameter`
- `_databaseId`
- 一些 `__frch_` 前缀的 index 或者 item


```java
public BoundSql getBoundSql(Object parameterObject) {
    DynamicContext context = new DynamicContext(configuration, parameterObject);
    rootSqlNode.apply(context);
    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
    Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
    context.getBindings().forEach(boundSql::setAdditionalParameter);
    return boundSql;
}
```

以下是各个版本不同的判断存在的方法: 

```java
// 3.3.0
public boolean hasAdditionalParameter(String name) {
	return this.metaParameters.hasGetter(name);
}
```

```java
// 3.4.1
public boolean hasAdditionalParameter(String name) {
    PropertyTokenizer prop = new PropertyTokenizer(name);
    String indexedName = prop.getIndexedName();
    return additionalParameters.containsKey(indexedName);
}
```


```java
// 3.4.2 ~ 3.5.6
public boolean hasAdditionalParameter(String name) {
    String paramName = (new PropertyTokenizer(name)).getName();
        return this.additionalParameters.containsKey(paramName);
}
```


```java
public Object getAdditionalParameter(String name) {
    return metaParameters.getValue(name);
}
```