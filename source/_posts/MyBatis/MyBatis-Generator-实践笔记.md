---
title: MyBatis Generator 实践笔记
date: 2022-10-11 15:18:11
tags:
---


# Maven 使用方式

## Maven 插件配置

```xml
<plugin>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-maven-plugin</artifactId>
    <version>1.4.1</version>
</plugin>
```

在连续构建环境中，你可能希望自动执行 MBG 作为 Maven 构建的一部分。这可以通过将 goal 配置为自动执行来实现。

```xml
<plugin>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-maven-plugin</artifactId>
    <version>1.4.1</version>
    <executions>
        <execution>
            <id>Generate MyBatis Artifacts</id>
            <goals>
            <goal>generate</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

如果你需要向插件的类路径添加一些东西（例如，JDBC 驱动程序），你可以通过向插件配置中添加依赖项来完成：

```xml
<plugin>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-maven-plugin</artifactId>
    <version>1.4.1</version>
    <executions>
        <execution>
            <id>Generate MyBatis Artifacts</id>
            <goals>
            <goal>generate</goal>
            </goals>
        </execution>
    </executions>
    <dependencies>
        <dependency>
            <groupId>org.hsqldb</groupId>
            <artifactId>hsqldb</artifactId>
            <version>2.3.4</version>
        </dependency>
    </dependencies>
</plugin>
```



## `<properties>`

使用 `<properties>` 可以将属性定义在外部文件.


```properties
target-project=src/main/java
driver-class=com.mysql.cj.jdbc.Driver
connection-url=jdbc:mysql://localhost:3306/self_building?characterEncoding=utf8&serverTimezone=UTC
user-id=root
password=
```

如果需要自定义生成文件名，则需要自定义插件:




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


## 参考配置

```properties
# java 根路径，可以是相对路径
java-path=D:/svn_repository/project/iorder-saas/trunk/application/dingzhi_jiashibo/src/main/java
# resource 路径，可以是相对路径
resource-path=D:/svn_repository/project/iorder-saas/trunk/application/dingzhi_jiashibo/src/main/resources
model-package=
mapper-package=
mapper-xml-package=
driver-class=org.postgresql.Driver
connection-url=jdbc:postgresql://127.0.0.1:5432/postgres
user-id=
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

## 参考配置

```properties
# java 根路径，可以是相对路径
java-path=D:/svn_repository/project/iorder-saas/trunk/application/dingzhi_jiashibo/src/main/java
# resource 路径，可以是相对路径
resource-path=D:/svn_repository/project/iorder-saas/trunk/application/dingzhi_jiashibo/src/main/resources
model-package=
mapper-package=
mapper-xml-package=
driver-class=org.postgresql.Driver
connection-url=jdbc:postgresql://127.0.0.1:5432/postgres
user-id=
password=
```

```xml
<generatorConfiguration>
    <properties resource="generatorConfig.properties"/>

    <context id="myContext" targetRuntime="MyBatis3">
        <property name="beginningDelimiter" value="`"/>
        <property name="endingDelimiter" value="`"/>

        <!--<plugin type="org.mybatis.generator.plugins.UnmergeableXmlMappersPlugin"/>-->
        <plugin type="com.jiangchunbo.RenameXmlMapperPlugin"/>
        <plugin type="com.jiangchunbo.LombokModelPlugin"/>

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

        <javaModelGenerator targetPackage="${model-package}" targetProject="${java-path}">
            <property name="enableSubPackages" value="true"/>
            <property name="trimStrings" value="true"/>
        </javaModelGenerator>

        <sqlMapGenerator targetPackage="${mapper-xml-package}" targetProject="${resource-path}">
            <property name="enableSubPackages" value="true"/>
        </sqlMapGenerator>

        <javaClientGenerator type="XMLMAPPER" targetPackage="${mapper-package}" targetProject="${java-path}">
            <property name="enableSubPackages" value="true"/>
        </javaClientGenerator>

        <table tableName="jsb_asset_install_assessment" domainObjectName="JsbAssetInstallAssessment"
               enableInsert="true"
               enableSelectByPrimaryKey="true"
               enableUpdateByPrimaryKey="true"
               enableDeleteByPrimaryKey="false"
               enableCountByExample="false"
               enableUpdateByExample="false"
               enableDeleteByExample="false"
               enableSelectByExample="false"
               selectByExampleQueryId="false"
        />
    </context>
</generatorConfiguration>
```


# 自定义插件

## XmlMapper

```java
public class RenameXmlMapperPlugin extends PluginAdapter {

    private final String searchString = "[A-Z][a-z]*";
    private Pattern pattern;

    @Override
    public boolean validate(List<String> warnings) {
        pattern = Pattern.compile(searchString);
        return true;
    }

    @Override
    public void initialized(IntrospectedTable introspectedTable) {
        String oldType = introspectedTable.getMyBatis3XmlMapperFileName();
        Matcher matcher = pattern.matcher(oldType);
        StringBuffer stringBuffer = new StringBuffer();
        while(matcher.find()) {
            if (matcher.start() == 0) {
                matcher.appendReplacement(stringBuffer, matcher.group(0).toLowerCase());
            } else {
                matcher.appendReplacement(stringBuffer, "_" + matcher.group(0).toLowerCase());
            }
        }
        stringBuffer.append("_mapping.xml");
        introspectedTable.setMyBatis3XmlMapperFileName(stringBuffer.toString());
    }
}
```


## 基于 Lombok 的 DO 对象

> 如果你的数据库字段名为单字母开头，如: `e_sales`，那么 MBG 生成的 Getter Setter 将会是 geteSales 和 seteSales，这比较奇怪，用 Lombok 替换可以替换为 getESales 和 setESales

```java
public class LombokModelPlugin extends PluginAdapter {

    @Override
    public boolean validate(List<String> warnings) {
        return true;
    }

    @Override
    public boolean modelBaseRecordClassGenerated(TopLevelClass topLevelClass, IntrospectedTable introspectedTable) {
        //该代码表示在生成class的时候，向topLevelClass添加一个@Setter和@Getter注解
        topLevelClass.addImportedType("lombok.Data");
        topLevelClass.addAnnotation("@Data");
        return super.modelBaseRecordClassGenerated(topLevelClass, introspectedTable);
    }

    @Override
    public boolean modelGetterMethodGenerated(Method method, TopLevelClass topLevelClass, IntrospectedColumn introspectedColumn, IntrospectedTable introspectedTable, ModelClassType modelClassType) {
        return false;
    }

    @Override
    public boolean modelSetterMethodGenerated(Method method, TopLevelClass topLevelClass, IntrospectedColumn introspectedColumn, IntrospectedTable introspectedTable, ModelClassType modelClassType) {
        return false;
    }
}
```

# 一些问题汇总

- 字段名和关键字冲突问题

配置界定符，MySQL 的界定符是 `` ` ``

```xml
<property name="beginningDelimiter" value="`"></property >
<property name="endingDelimiter" value="`"></property >
```

在 table 节点添加属性 `delimitAllColumns="true"`



- UnmergeableXmlMappersPlugin

如果配置了该插件，MBG 的合并将会失效。如果 overwrite 为 true，那么会覆盖源文件！如果为 false，将会生成新文件（不包含原来自定义的节点）。

> 根据一些特定的注解识别是否需要覆盖。

```xml
<plugin type="org.mybatis.generator.plugins.UnmergeableXmlMappersPlugin"/>
```


- overwrite 和 mergeable

是否合并 XML 文件，通过属性 `org.mybatis.generator.api.GeneratedXmlFile#isMergeable` 表示。是否合并也可以理解为是否使用同一个文件。

当配置了 `<overwrite>false</overwrite>`

- 如果 `mergeable=true` 合并，那么不会起作用
- 如果 `mergeable=false` 不合并，那么会生成新文件。之前的文件保留，但是后续需要自己做合并与删除。

当配置了 `<overwrite>true</overwrite>`

- 如果 `mergeable=true` 合并，那么会追加 XML 节点，保留原来的节点。这样会产生许多重复 id 的节点。
- 如果 `mergeable=false` 不合并，那么会覆盖已经存在的 XML 文件。导致原来的内容覆盖丢失。


如果配置了 `overwrite=true`，那么在不合并模式下，会覆盖 XML Mapper 文件。