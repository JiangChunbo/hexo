---
title: MyBatis Generator 实践笔记
date: 2022-10-11 15:18:11
tags:
---


# 参考引用

https://developer.aliyun.com/article/828451


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

```xml
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
```

```xml
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
```



## `<properties>`

使用 `<properties>` 可以将属性定义在外部文件.


```properties
# 实体类的配置
java-model-generator.target-package=
java-model-generator.target-project=
# XML Mapper 配置
sql-map-generator.target-package=
sql-map-generator.target-project=
# DAO 配置
java-client-generator.target-package=
java-client-generator.target-project=
# 数据库相关配置
driver-class=
connection-url=
user-id=
password=
```


## `<context>`

```xml
<context id="myContext" targetRuntime="MyBatis3">
    <!-- PostgreSQL 界定符，不需要双引号就注释掉 -->
    <property name="beginningDelimiter" value="&quot;" />
    <property name="endingDelimiter" value="&quot;" />


    <!-- 关于注解的时间格式化格式，给 SimpleDateFormat 用 -->
    <property name="dateFormat" value="yyyy-MM-dd HH:mm:ss" />

    <!-- 指定编码。不指定可能会使用默认的 GBK -->
    <property name="javaFileEncoding" value="UTF-8" />

    <!-- <plugin type="org.mybatis.generator.plugins.UnmergeableXmlMappersPlugin"/> -->
    <!-- <plugin type="com.jiangchunbo.mybatis.generator.plugins.BasePlugin"/> -->
    <plugin
        type="com.jiangchunbo.mybatis.generator.plugins.RenameXmlMapperPlugin" />
    <!-- <plugin
        type="com.jiangchunbo.mybatis.generator.plugins.LombokModelPlugin" /> -->

    <commentGenerator type="com.jiangchunbo.mybatis.generator.plugins.CustomCommentGenerator">
        <!-- 压制日期。如果为 true，则注释部分不显示日期 -->
        <property name="suppressDate" value="false" />
        <!-- 压制所有注释。如果为 true，则注释不显示，将会导致合并代码失效 -->
        <property name="suppressAllComments" value="false" />
        <property name="addRemarkComments" value="true" />
        <property name="dateFormat"
            value="yyyy-MM-dd HH:mm:ss" />
    </commentGenerator>

    <!-- 数据库配置 -->
    <jdbcConnection driverClass="${driver-class}"
        connectionURL="${connection-url}" userId="${user-id}"
        password="${password}">
        <property name="useInformationSchema" value="true" />
    </jdbcConnection>

    <!-- 
        Java 类型处理器
        处理 DB 类型到 Java 类型，默认使用 JavaTypeResolverDefaultImpl
        注意一点，默认会先尝试使用Integer，Long，Short等来对应DECIMAL和 NUMERIC数据类型
     -->
    <javaTypeResolver>
        <!-- 
            true：使用BigDecimal对应DECIMAL和 NUMERIC数据类型
            false：默认,
                scale>0;length>18：使用BigDecimal;
                scale=0;length[10,18]：使用Long；
                scale=0;length[5,9]：使用Integer；
                scale=0;length<5：使用Short；
         -->
        <property name="forceBigDecimals" value="false" />
    </javaTypeResolver>

    <!--
         java模型创建器，是必须要的元素
        负责：1，key类（见context的defaultModelType）；2，java类；3，查询类
        targetPackage：生成的类要放的包，真实的包受enableSubPackages属性控制；
        targetProject：目标项目，指定一个存在的目录下，生成的内容会放到指定目录中，如果目录不存在，MBG不会自动建目录
     -->
    <javaModelGenerator
        targetPackage="${java-model-generator.target-package}"
        targetProject="${java-model-generator.target-project}">
        <property name="enableSubPackages" value="true" />
        <property name="trimStrings" value="true" />
    </javaModelGenerator>

    <sqlMapGenerator
        targetPackage="${sql-map-generator.target-package}"
        targetProject="${sql-map-generator.target-project}">
        <property name="enableSubPackages" value="true" />
    </sqlMapGenerator>

    <javaClientGenerator type="XMLMAPPER"
        targetPackage="${java-client-generator.target-package}"
        targetProject="${java-client-generator.target-project}">
        <property name="enableSubPackages" value="true" />
    </javaClientGenerator>

    <table tableName="abnormalstock_subtask_duration"
        domainObjectName="JnjSubtaskDuration" enableInsert="true"
        enableSelectByPrimaryKey="true"
        enableUpdateByPrimaryKey="true"
        enableDeleteByPrimaryKey="false"
        enableCountByExample="false" enableUpdateByExample="false"
        enableDeleteByExample="false" enableSelectByExample="false"
        selectByExampleQueryId="false" />
</context>
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