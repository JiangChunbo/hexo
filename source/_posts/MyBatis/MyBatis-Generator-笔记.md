---
title: MyBatis Generator 笔记
date: 2022-08-16 18:00:58
tags:
---

# MyBatis Generator 笔记


## XML 配置参考

### `<context>` 元素

#### 必需的属性

- id

该上下文的唯一标识符。该值将会在一些错误信息中使用到。

#### 可选的属性

- defaultModelType

如果 targer runtime 是 MyBatis3Simple, MyBatis3DynamicSql, 或者 MyBatis3Kotlin，该属性会被忽略。


- targetRuntime

该属性用于生成代码的运行时目标。

**MyBatisDynamicSql**


#### 支持的属性

- beginningDelimiter


- endingDelimiter


### `<javaClientGenerator>` 元素

`<javaClientGenerator>` 元素用于定义 Java 客户端生成器的属性。


#### 必需的属性


- type

该属性用来选择预定义的 Java 客户端生成器之一，或者指定一个用户提供的 Java 客户端生成器。


**XMLMAPPER**

生成的对象是 MyBatis 3.x mapper 基础设施的 Java 接口。接口将依赖于生成的 XML mapper 文件。


- targetPackage

这是生成的接口和实现类所在的包。


- targetProject

这用于为生成的接口和类指定目标项目。

### `<javaModelGenerator>` 元素

`<javaModelGenerator>` 元素用于定义 Java 模型生成器的属性。Java 模型生成器构建主键类，记录类，以及与自省表匹配的按照 Example 类查询。该元素是 `<context>` 元素必须的子元素。

#### 必需的属性

- targetPackage

这是生成的类将被放置的包。在默认的生成器中，属性 "enableSubPackages" 控制如何计算实际的包。如果为 true，则计算出的包是 targetPackage 加上表的 catalog 和 schema 的子包（如果存在）。

- targetProject

这用于为生成的对象指定目标项目。在 Eclipse 环境中运行时，这将会指定保存对象的项目和源文件夹。在其他环境中，该值应该是本地文件系统上的已经存在的目录。如果此目录不存在， MyBatis Generator 将不会创建此目录。


## 使用方式

创建配置文件，比如在 resources/mybatis-generator/ 下面创建 generatorConfig.xml 配置文件：


```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <properties resource="mybatis-generator/generator.properties"/>

    <context id="DB2Tables"  targetRuntime="MyBatis3Simple">

        <property name="beginningDelimiter" value="`"/>
        <property name="endingDelimiter" value="`"/>


        <!--注释-->
        <commentGenerator>
            <!--是否压制时间戳-->
            <property name="suppressDate" value="true"/>
            <!-- 是否去除自动生成的注释 true：是 ： false:否 -->
            <property name="suppressAllComments" value="true"/>
            <!-- 注解采用数据库的标注，suppressAllComments 必须设置为 false 才会生效 -->
            <property name="addRemarkComments" value="true"/>
        </commentGenerator>

        <!--数据库连接参数 -->
        <jdbcConnection
                driverClass="com.mysql.jdbc.Driver"
                connectionURL="jdbc:mysql://localhost:3306/xxx?characterEncoding=utf8&amp;serverTimezone=UTC"
                userId="xxx"
                password="xxx">
            <!-- mysql 获取数据库注解的方式，想要获取数据库注解必须添加 -->
            <property name="useInformationSchema" value="true"/>
        </jdbcConnection>

        <javaTypeResolver>
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>

        <!-- 实体类的包名和存放路径 -->
        <javaModelGenerator targetPackage="xxx" targetProject="src/main/java">
            <property name="enableSubPackages" value="true"/>
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>

        <!-- 生成映射文件*.xml的位置-->
        <sqlMapGenerator targetPackage="mapper" targetProject="src/main/resources">
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>

        <!-- 生成DAO的包名和位置 -->
        <javaClientGenerator type="XMLMAPPER" targetPackage="xxx" targetProject="src/main/java">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>

        <!-- tableName：数据库中的表名或视图名；domainObjectName：生成的实体类的类名-->
        <table tableName="book" domainObjectName="Book"
               enableCountByExample="false"
               enableUpdateByExample="false"
               enableDeleteByExample="false"
               enableSelectByExample="false"
               selectByExampleQueryId="false"/>
    </context>
</generatorConfiguration>
```


- 字段名和关键字冲突问题

配置界定符

```xml
<property name="beginningDelimiter" value="`"></property >
<property name="endingDelimiter" value="`"></property >
```

在 table 节点添加属性 `delimitAllColumns="true"`



- Mapper XML 文件覆盖

```xml
<plugin type="org.mybatis.generator.plugins.UnmergeableXmlMappersPlugin"/>
```