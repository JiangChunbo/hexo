---
title: MyBatis Generator XmlFormatter
date: 2022-11-23 10:29:54
tags:
- MyBatis Generator
---

# 0. 参考引用

http://mybatis.org/generator/configreference/context.html

[CustomXmlFormatter.java](https://github.com/JiangChunbo/mybatis-generator-extension/blob/main/src/main/java/com/jiangchunbo/mybatis/generator/ext/CustomXmlFormatter.java)


# 说明

默认情况下，MyBatis Generator 的 XML 文件缩进 2 格。这是硬编码的，如果需要修改缩进到 4 格，需要改写默认的 `XmlFormatter`。官方的参考请查阅[这里](http://mybatis.org/generator/configreference/context.html)。


具体来说，需要自定义 XmlFormatter，然后配置 `<context>` 元素下面的 `<property>` 属性，例如:

```xml
<property name="xmlFormatter" value="com.jiangchunbo.mybatis.generator.ext.CustomXmlFormatter"/>
```
