---
title: MyBatis Generator 官方翻译笔记
date: 2022-08-16 18:00:58
tags:
---

# 参考引用

http://mybatis.org/generator/


# 依赖

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-maven-plugin</artifactId>
            <version>1.3.7</version>
        </plugin>
    <plugins>    
</build>
```

# XML 配置参考

## `<context>`

`<context>` 元素用于指定生成一组对象的环境。可以在 `<generatorConfiguration>` 元素中列出多个 `<context>` 元素，以允许在同一次运行 MyBatis Generator（MBG）中从不同的数据库或使用不同的生成参数生成对象。

### Required Attributes

- id

该上下文的唯一标识符。该值将会在一些错误信息中使用到。

### Optional Attributes

- defaultModelType

如果 targer runtime 是 MyBatis3Simple, MyBatis3DynamicSql, 或者 MyBatis3Kotlin，该属性会被忽略。


- targetRuntime

该属性用于生成代码的运行时目标。

**MyBatisDynamicSql**

**MyBatis3** 使用该值，MBG 将生成与 MyBatis 3.0+，JSE 5.0+ 兼容的对象。这些生成的对象中的 "by example" 方法实际上支持无限制的动态 where 字句。此外，这些生成器生成的 Java 对象支持许多 JSE 5.0 特性，包括参数化类型和注解。


**MyBatis3Simple** 使用该值，MBG 将生成与 MyBatis 3.0+，JSE 5.0+ 兼容的对象。这个目标运行时生成的映射程序时非常基本的 CRUD 操作，只不过没有 "by example" 以及没有动态 SQL。这些生成器生成的 Java 对象支持许多 JSE 5.0 特性，包括参数化类型和注解。


### Supported Properties

- beginningDelimiter


- endingDelimiter


## `<commentGenerator>`

- suppressAllComments
此属性用于指定 MBG 是否在生成的代码中包含任何注释。

- suppressDate
此属性用于指定 MBG 是否将生成的时间戳在生成的注释中包括。

- addRemarkComments
此属性用于指定 MBG 是否将在生成的注释中包含来自 db 表的表备注和列备注。

> 警告: 如果 `suppressAllComments` 选项为 true，那么该选项会被忽略。即，该选项优先级比较低。

## `<javaClientGenerator>`

`<javaClientGenerator>` 元素用于定义 Java 客户端生成器的属性。


### Required Attributes


- type

该属性用来选择预定义的 Java 客户端生成器之一，或者指定一个用户提供的 Java 客户端生成器。


**XMLMAPPER**

生成的对象是 MyBatis 3.x mapper 基础设施的 Java 接口。接口将依赖于生成的 XML mapper 文件。


- targetPackage

这是生成的接口和实现类所在的包。


- targetProject

这用于为生成的接口和类指定目标项目。


### Supported Properties

- enableSubPackages
该属性用于选择 MyBatis Generator 是否将基于内置表的 catelog 和 schema 为对象生成不同的 Java 包。




## `<javaModelGenerator>`

`<javaModelGenerator>` 元素用于定义 Java 模型生成器的属性。Java 模型生成器构建主键类，记录类，以及与自省表匹配的按照 Example 类查询。该元素是 `<context>` 元素必须的子元素。

### 必需的属性

- targetPackage

这是生成的类将被放置的包。在默认的生成器中，属性 "enableSubPackages" 控制如何计算实际的包。如果为 true，则计算出的包是 targetPackage 加上表的 catalog 和 schema 的子包（如果存在）。

- targetProject

这用于为生成的对象指定目标项目。在 Eclipse 环境中运行时，这将会指定保存对象的项目和源文件夹。在其他环境中，该值应该是本地文件系统上的已经存在的目录。如果此目录不存在， MyBatis Generator 将不会创建此目录。


### 可选

- enableSubPackages

该属性用于选择 MyBatis Generator 是否将基于内置表的 catelog 和 schema 为对象生成不同的 Java 包。

> 建议: false


- trimStrings
该属性用于选择 MyBatis Generator 是否添加代码以修剪从数据库返回的字符字段的空白。

> 建议: true。尽管说数据库设计出现空白字符应该是一种设计失误。



## `<jdbcConnection>`

`<jdbcConnection>` 元素用于指定

## `<properties>`

`<properties>` 元素用于指定用于解析配置的外部属性文件。配置中的任何属性都将接受以 `${property}` 格式的属性。


`<properties>` 元素是 `<generatorConfiguration>` 元素的子元素。

### 必须的属性

- resource
属性文件的完全限定名称

- url
用于属性文件的 URL 值

