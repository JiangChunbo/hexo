---
title: JSON-lib 框架
date: 2022-11-21 19:01:34
tags:
- JSON
---

# 参考引用

https://json-lib.sourceforge.net/


# maven 依赖

```xml
<dependency>
    <groupId>net.sf.json-lib</groupId>
    <artifactId>json-lib</artifactId>
    <version>2.2.3</version>
</dependency>
```


# 使用方式


## JSON >> Java Bean

```java
public class Bean {
    private Integer id;
    private String name;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

```java
String json = "{\"id\":1, \"name\": \"jcb\"}";
JSONObject jsonObject = JSONObject.fromObject(json);
Bean bean = (Bean) JSONObject.toBean(jsonObject, Bean.class);
System.out.println(bean);
```


使用时需要注意，该 Bean 不可以是：

- 不能是处于同一 java 文件下声明的类。因为必须具有 public 修饰符
- 不能是私有内部类。因为必须具有 public 修饰符