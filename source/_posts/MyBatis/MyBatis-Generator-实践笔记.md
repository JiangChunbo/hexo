---
title: MyBatis Generator 实践笔记
date: 2022-10-11 15:18:11
tags:
---


# 使用方式




```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-maven-plugin</artifactId>
            <version>1.3.7</version>
            <configuration>
                <overwrite>true</overwrite>
                <configurationFile>src/main/resources/generatorConfig.xml</configurationFile>
            </configuration>
            <dependencies>
                <dependency>
                    <groupId>mysql</groupId>
                    <artifactId>mysql-connector-java</artifactId>
                    <version>8.0.20</version>
                </dependency>
            </dependencies>
        </plugin>
    </plugins>
</build>
```

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-maven-plugin</artifactId>
            <version>1.3.7</version>
            <configuration>
                <overwrite>true</overwrite>
                <configurationFile>src/main/resources/generatorConfig.xml</configurationFile>
            </configuration>
            <dependencies>
                <dependency>
                    <groupId>mysql</groupId>
                    <artifactId>mysql-connector-java</artifactId>
                    <version>8.0.20</version>
                </dependency>
            </dependencies>
        </plugin>
    </plugins>
</build>
```



# MySQL

```properties
target-project=src/main/java
driver-class=com.mysql.cj.jdbc.Driver
connection-url=jdbc:mysql://localhost:3306/self_building?characterEncoding=utf8&serverTimezone=UTC
user-id=root
password=
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <properties resource="generatorConfig.properties"/>

    <context id="myContext" targetRuntime="MyBatis3Simple">
        <property name="beginningDelimiter" value="`"/>
        <property name="endingDelimiter" value="`"/>

        <plugin type="org.mybatis.generator.plugins.UnmergeableXmlMappersPlugin"/>

        <commentGenerator>
            <property name="suppressDate" value="true"/>
            <property name="suppressAllComments" value="true"/>
            <property name="addRemarkComments" value="true"/>
        </commentGenerator>

        <jdbcConnection
                driverClass="${driver-class}"
                connectionURL="${connection-url}"
                userId="${user-id}"
                password="${password}">
            <property name="useInformationSchema" value="true"/>
        </jdbcConnection>

        <javaTypeResolver>
            <property name="forceBigDecimals" value="false"/>
        </javaTypeResolver>

        <javaModelGenerator targetPackage="model" targetProject="${target-project}">
            <property name="enableSubPackages" value="true"/>
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>

        <sqlMapGenerator targetPackage="mapper" targetProject="${target-project}">
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>

        <javaClientGenerator type="XMLMAPPER" targetPackage="mapper" targetProject="${target-project}">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>

        <!-- tableName：数据库中的表名或视图名；domainObjectName：生成的实体类的类名-->
        <table tableName="app" domainObjectName="App"
               enableCountByExample="false"
               enableUpdateByExample="false"
               enableDeleteByExample="false"
               enableSelectByExample="false"
               selectByExampleQueryId="false"/>
    </context>
</generatorConfiguration>
```

# PostGreSQL

# 一些问题汇总

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