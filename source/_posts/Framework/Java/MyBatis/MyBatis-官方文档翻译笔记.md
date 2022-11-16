---
title: MyBatis 官方文档翻译笔记
date: 2022-07-15 18:08:57
categories:
- 框架
tags:
- MyBatis
---

# [MyBatis](https://mybatis.org/mybatis-3/)
---
## [1. Getting started](https://mybatis.org/mybatis-3/getting-started.html#Getting_started)
### [1.1. Installation](https://mybatis.org/mybatis-3/getting-started.html#Installation)
进行最基本的 MyBatis 应用开发需要导入的依赖，最新版本可以参考[github](https://github.com/mybatis/mybatis-3/releases)
```xml
<dependency>
  <groupId>org.mybatis</groupId>
  <artifactId>mybatis</artifactId>
  <version>3.5.8</version>
</dependency>
```


### [1.2. Building SqlSessionFactory from XML](https://mybatis.org/mybatis-3/getting-started.html#Building_SqlSessionFactory_from_XML)

从 XML 构建 SqlSessionFactory。所有 mybatis 应用程序围绕 `SqlSessionFactory` 实例。可以通过 `SqlSessionFactoryBuilder` 获取到 `SqlSessionFactory` 实例。`SqlSessionFactoryBuilder` 可以从以下几个来源进行构建：

- Mybatis configuration xml
- Configuration object

从 XML 文件构建 `SqlSessionFactory` 实例，建议使用**类路径资源**，但实际上也可以使用 InputStream。

**mybatis-config.xml** 可以参考[官网](https://mybatis.org/mybatis-3/getting-started.html)


### [1.3. Building SqlSessionFactory without XML](https://mybatis.org/mybatis-3/getting-started.html#Building_SqlSessionFactory_without_XML)
可以参考官网的案例。但是，并不支持这种做法，因为高级映射仍然需要 XML 的支持。


### [1.4. Acquiring a SqlSession from SqlSessionFactory](https://mybatis.org/mybatis-3/getting-started.html#Acquiring_a_SqlSession_from_SqlSessionFactory)
通过以下代码可以创建一个 Session（注意关闭）：
```java
SqlSession session = sqlSessionFactory.openSession();
```


### [1.5. Exploring Mapped SQL Statements](https://mybatis.org/mybatis-3/getting-started.html#Exploring_Mapped_SQL_Statements)
语句可以由 XML 或者 Annotation 定义。

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
  PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="org.mybatis.example.BlogMapper">
  <select id="selectBlog" resultType="Blog">
    select * from Blog where id = #{id}
  </select>
</mapper>
```
上述例子，在命名空间 `org.mybatis.example.BlogMapper` 中，定义了名为 `selectBlog` 的映射语句，允许你通过完全限定名称 `org.mybatis.example.BlogMapper.selectBlog` 来调用：

```java
Blog blog = session.selectOne("org.mybatis.example.BlogMapper.selectBlog", 101);
```

这与在完全限定的 Java 类上调用一个方法是非常类似的，这也是如此设计的原因。使用与映射的 select 语句在名称，参数，返回类型相匹配的方法，可以直接映射到与命名空间名称相同的 Mapper 类。这允许你简单地将方法调用在映射器接口上：
```java
BlogMapper mapper = session.getMapper(BlogMapper.class);
Blog blog = mapper.selectBlog(101);
```
第二种方法有很多优势。首先，它不依赖字符串，更安全。其次，如果你的 IDE 由代码补全，可以利用它跳转到你映射的 SQL 语句。

**Namespaces** 
以前版本的 MyBatis 命名空间是可选的，现在强制需要命名空间，具有隔离语句的目的。

命名空间让接口进行绑定，并且，即使你现在并不认为你会使用，你也应该遵循这种实践，防止你改变想法。一旦使用命名空间，把它放到正确的 Java 包命名空间下能清理你的代码，并且长期内提高 MyBatis 的可用性


对于简单的语句，可以使用注解方式定义：
```java
public interface BlogMapper {
  @Select("SELECT * FROM blog WHERE id = #{id}")
  Blog selectBlog(int id);
}
```
>不过，对于复杂的语句，建议使用 XML 方式定义，否则显得混乱。


## [2. Configuration XML](https://mybatis.org/mybatis-3/configuration.html)

### [2.1. properties](https://mybatis.org/mybatis-3/configuration.html#properties)
`<properties>` 除了支持嵌入内置属性，还支持引入外部 Java 属性文件，只需通过 resource 或者 url 属性设置即可。resource 是以类路径为基础路径的相对路径，url 为标准的路径位置，如 file:///。
```xml
<properties resource="org/mybatis/example/config.properties">
  <property name="username" value="dev_user"/>
  <property name="password" value="F2Fa3!33TYyg"/>
</properties>
```
**注意点** 
- resource 和 url 不可以同时设置，否则抛出异常。
- 如果配置了相同的属性，则后面的会覆盖前面的；如果是分散在属性文件和子元素属性中，那么由于属性文件后加载，因此属性文件会覆盖子元素属性。
- `<properties>` 标签必需位于 `configuration` 子元素第一个



属性配置完，可以在配置文件中使用属性，以替代需要动态配置的值。例如：
```xml
<dataSource type="POOLED">
  <property name="driver" value="${driver}"/>
  <property name="url" value="${url}"/>
  <property name="username" value="${username}"/>
  <property name="password" value="${password}"/>
</dataSource>
```

属性的来源也可以是由程序传递给 SqlSessionFactoryBuilder.build() 方法：

```java
SqlSessionFactory factory = sqlSessionFactoryBuilder.build(reader, props);
// ... or ...
SqlSessionFactory factory = new SqlSessionFactoryBuilder.build(reader, environment, props);
```



### [2.2. settings](https://mybatis.org/mybatis-3/configuration.html#settings)

|参数|描述|有效值|默认值|
|:---|:---|:---|:---|
|cacheEnabled|全局启用或禁用此配置下任何映射器中的任何缓存|
|lazyLoadingEnabled|全局懒加载。该值可以被 `fetchType` 取代。|
|aggressiveLazyLoading|启用后，任何方法调用都会加载对象的延迟属性。否则，每个属性按需加载。具体见 `lazyLoadTriggerMethods`||false (true in ≤3.4.1)|
|useColumnLabel|使用列标签而不是列名。不同的驱动表现不同。|
|mapUnderscoreToCamelCase|下划线到驼峰转换|
|autoMappingBehavior|是否自动映射，Spring MyBatis 默认为 PARTIAL，注意。|
|defaultExecutorType|配置默认的执行器。SIMPLE REUSE BATCH|
|localCacheScope|本地缓存的作用域|SESSION \| STATEMENT|SESSION |
|logImpl|指定 MyBatis 应该使用的日志实现。如果未设置，则自动发现。|SLF4J \| LOG4J \| LOG4J2 \| JDK_LOGGING \| COMMONS_LOGGING \| STDOUT_LOGGING \| NO_LOGGING|Not set|


### [2.3. typeAliases](https://mybatis.org/mybatis-3/configuration.html#typeAliases)
别名是 Java 类型的短名称，可以简单地减少完全限定类名的冗余输入。支持 `<typeAlias>` 和 `<package>` 配置

```xml
<typeAliases>
  <typeAlias alias="Author" type="domain.blog.Author"/>
</typeAliases>
```
指定 MyBatis 搜索 bean 的包，例如：
```xml
<typeAliases>
  <package name="domain.blog"/>
</typeAliases>
```
在包中的**所有 bean**，都会使用 bean 的小写非限定类名作为别名。其中，`domain.blog.Author` 将被注册为 `author`。特殊地，如果发现 @Alias 注解，则其值将用作别名：
```java
@Alias("author")
public class Author {
    ...
}
```

有许多内置的 Java 类型的别名。它们都是大小写不敏感的。具体参见类 `TypeAliasRegistry`



### [2.4. typeHandlers](https://mybatis.org/mybatis-3/configuration.html#typeHandlers)
typeHandler 用于两个地方：

- 当 MyBatis 在 PreparedStatement 设置参数时。参见 `DefaultParameterHandler`
- 从 ResultSet 检索值。参见 `DefaultResultSetHandler`

> 自从  3.4.5, MyBatis 默认支持 JSR-310 (Date and Time API)

在 MyBatis 中，存在一个 `TypeHandlerRegistry` 组件，其维护了一个 Map 结构：
```java
private final Map<Type, Map<JdbcType, TypeHandler<?>>> typeHandlerMap = new ConcurrentHashMap<>();
```
该 Map 的 key 是 `java.lang.reflect.Type` 类型对象，value 又是一个 Map（称为 sub map），通常用于传入一个 Class 对象，得到 Jdbc 类型支持的类型处理器，sub map 通常还会存储一个 key 为 null 的键值对，用于匹配默认的处理器。


> typeHandlers 几乎很少自己定义扩展，但少部分情况还是比较有效的，例如：不涉及查询的一对多关系、setString 的乱码问题。


### [2.5. Handling Enums](https://mybatis.org/mybatis-3/configuration.html#Handling_Enums)


### [2.6. objectFactory](https://mybatis.org/mybatis-3/configuration.html#objectFactory)
使用 ObjectFactory 进行创建结果对象的新实例。不仅支持无参构造器的创建，也支持有参构造器创建。



### [2.7. plugins](https://mybatis.org/mybatis-3/configuration.html#plugins)


### [2.8. environments](https://mybatis.org/mybatis-3/configuration.html#environments)
MyBatis 可以配置多个环境。这有助于你以任何原因将 SQL 映射到不同的数据库。例如，你可能对于开发，测试和生产环境由不同的配置。或者，你可能有多个表结构相同的生产数据库，并且你希望为两者使用相同的 SQL 映射。

**注意** 尽管你可以配置多个环境，但是你只可以为每个 SqlSessionFactory 选择一个。因此，如果要连接到多个数据库，则需要为每个数据库创建一个 SqlSessionFactory。

- **每个数据库一个 SqlSessionFactory 实例**

传递给 build() 方法参数 environment 可以指定环境，否则使用 default 环境。

```java
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(reader, environment);
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(reader, environment, properties);
```


**transactionManager**

MyBatis 包含了两种事务管理器，JDBC | MANAGED


**dataSource**

dataSource 元素使用标准 JDBC DataSource 接口配置 JDBC Connection 对象。

大多数 MyBatis 应用程序将根据示例中配置 dataSource。但是，它不是必需的。但是，要促进懒加载，数据源是需要的。

有三种内置的数据源类型，UNPOOLED | POOLED | JNDI

**UNPOOLED** 每次请求时，数据源都是简单地打开和关闭一个链接。虽然它有点慢，但是对于不需要立即可用连接地简单应用来说，是个不错地选择。不同的数据库也不同，因此对于一些池化不太重要的数据库，该配置会是比较理想的。UNPOOLED 数据源具有以下属性需要配置：

- `driver` 
- `url`
- `username`
- `password`
- `defaultTransactionIsolationLevel` 连接的默认事务隔离级别
- `defaultNetworkTimeout` 默认网络超时值，以等待数据库操作完成，单位毫秒


**POOLED** DataSource 池化 JDBC Connection 对象，以避免创建新连接实例所需要的初始化和认证时间。这是一种当前 web 应用中流行的方式，可以获得最快的响应。

除了上面的（UNPOOLED）属性之外，还有许多属性可用于配置 POOLED 数据源：

- `poolMaximumActiveConnections`
- `poolMaximumIdleConnections`
- `poolMaximumCheckoutTime`
- `poolTimeToWait`
- `poolMaximumLocalBadConnectionTolerance`
- `poolPingQuery` ping 查询被发送到数据库以验证连接处于良好的工作状态，并且已准备好接受请求。默认值："NO PING QUERY SET"，这回引起大多数数据库驱动以错误信息产生失败。
- `poolPingEnabled` 开启或禁用 ping 查询。如果开启，你必须用一个有效的 SQL 语句（最好很快）设置 `poolPingQuery` 属性。默认：false
- `poolPingConnectionsNotUsedFor`


**JNDI** 此 DataSource 的实现旨在与容器（如 EJB 或者应用服务器）一起使用，该数据源可以配置内部或外部数据源，并且在 JNDI 上下文中对其进行引用。该数据源配置只需要两个属性：

- initial_context
- data_source


### [2.9. databaseIdProvider](https://mybatis.org/mybatis-3/configuration.html#databaseIdProvider)


### [2.10. mappers](https://mybatis.org/mybatis-3/configuration.html#mappers)
定义映射的 SQL 语句，首先，我们需要告诉 MyBatis 在哪里找到他们，你可以使用：

- 类路径相对资源引用
- 完全限定 url 引用，包括 `file:///`
- 类名
- 包名


#### 方式1 类路径相对资源引用
```xml
<mapper resource="org/mybatis/builder/AuthorMapper.xml"/>
```

#### 方式2 完全限定 url 引用
```xml
<mapper url="file:///var/mappers/AuthorMapper.xml"/>
```

#### 方式3 类名
注册的配置结构如下：
```xml
<mapper class="org.mybatis.builder.AuthorMapper"/>
```

**注册源码分析**

依靠组件 `MapperRegistry` 进行注册。不需要尝试配置两个一样的类，否则你会得到异常。

依靠 `MapperAnnotationBuilder` 解析注解。注解解析部分，通过反射 Class.getMethod() 获取到接口所有的方法。

MapperAnnotationBuilder 会委派内部的 MapperBuilderAssistant 进行语句添加，还会缓存到 MapperBuilderAssistant  中


#### 方式4 包名
注册该包下所有接口为 mapper
```xml
<package name="org.mybatis.builder"/>
```
你在指定非 package 的配置时，只能配置 url，name，resource 之一的属性，否则你会得到异常。




## [3. Mapper XML Files](https://mybatis.org/mybatis-3/sqlmap-xml.html#)
### [3.1. select](https://mybatis.org/mybatis-3/sqlmap-xml.html#select)
对于形如 `#{id}` 这样的标记，这告诉 MyBatis 需要创建一个 PreparedStatement 的参数。使用 JDBC，这样的参数会在 SQL 中识别为 "?"，传递给 PreparedStatement。

|属性|描述|
|:---|:---|
|id|在该 namesapce 下的唯一标识符，可以用于引用该语句，即 Statement ID。|
|parameterType|将传递到此语句的参数的完全限定类名或别名。可选，因为 MyBatis 可以通过传递给语句的实际参数得到 `TypeHandler`|
|resultType|从此语句返回的预期类型的完全限定类名或别名。在集合的情况下，这应该是集合包含的类型，而不是集合类型。使用 `resultType` 或者 `resultMap`，不可以都用。|
|resultMap|外部 `resultMap` 的引用。使用 `resultMap` 或者 `resultType`，不可以都用。|
|flushCache|设置为 true 将导致当调用此语句时会刷新 Local 和 2 级缓存。对于 select 默认是 false
|useCache|设置为 true 将导致该语句的结果缓存在 2 级缓存。对于 select 语句默认为: `true`|
|timeout|驱动等待数据库返回数据，在抛出异常之前的超时时间，默认 `unset`（驱动决定）|
|fetchSize||
|statementType|可选值：STATEMENT, PREPARED, CALLABLE。默认值：PREPARED|
|resultSetType||
|databaseId||
|resultOrdered||
|resultSets||

### [3.2. insert, update and delete](https://mybatis.org/mybatis-3/sqlmap-xml.html#insert.2C_update_and_delete)

MyBatis 加载 Mapper 的时候会将读取到的语句信息都存储到 Configuration 的属性 `mappedStatements` 中：
```java
public Map<String, MappedStatement> mappedStatements;
```

其中，`MappedStatement` 又能够获取 `BoundSql`



### 一对多关联
- collection

支持分步查询和关联查询

```xml
<resultMap type="com.cannedbread.dictionary.pojo.domain.DictionaryTypeDO" id="doctionaryType">
	<id column="t_id" property="id" javaType="java.lang.String"/>
	<result column="t_type" property="type"/>
	<result column="t_description" property="description"/>
	<collection property="dictionaryTypeList" ofType="com.cannedbread.dictionary.pojo.domain.DictionaryValueDO">
		<id column="v_id" property="id" javaType="java.lang.String"/>
		<result column="v_label" property="label"/>
		<result column="v_value" property="value"/>
	</collection>
</resultMap>
```


### [3.3. Parameters](https://mybatis.org/mybatis-3/sqlmap-xml.html#Parameters)
参数，指传递给语句的参数。参数可以认为有 2 种：

- 简易（原始）数据类型。 Integer、String 等没有属性的值。

- 复合数据类型。可以理解为是由简易数据类型组合而成的类。


通常，我们使用 `#{id}` 这种方式引用参数。但是，参数可以更具体地引用：
```java
#{property,javaType=int,jdbcType:NUMBERIC}
```
`javaType` 基本可以总是从参数对象中确定，除非该对象是 `HashMap`，那么需要显式指定 `javaType` 确保使用正确的 TypeHandler。

- 传入 List

引用名称必须为 `list`，获取长度：`list.size`，获取第 n 个元素：list[n]

```xml
<if test="list != null and list.size != 0"></if>
<foreach collection="list" open="(" close=")" separator="," item="item">#{item}</foreach>
```

- 传入多个值

多个参数会封装成 Map，key 为 param1, param2, ...

引用方式：`#{param1}`、`#{param2.password}`

- 传入 Map


对 XML 中映射的语句进行解析之后，能够得到一个 `List<ParameterMapping>`，其中收集所有 `#{}` 占位符。

这些结构之后会传递给 `StatementHandler`，通常是 `PreparedStatementHandler`，之后通过 handler 进行参数设置。


### [3.4. Result Maps](https://mybatis.org/mybatis-3/sqlmap-xml.html#Result_Maps)
对于如下映射语句例子：
```xml
<select id="selectUsers" resultType="map">
  select id, username, hashedPassword
  from some_table
  where id = #{id}
</select>
```

根据 `resultType` 属性所指定的，这样的语句会简单地将所有地列自动映射到 `HashMap` 的 key 上。但是，在大多数场景下，`HashMap`并不是一个好的领域模型。应用程序更有可能使用 Java Bean 或者 POJO。考虑如下的 Java Bean：
```java
public class User {
  private int id;
  private String username;
  private String hashedPassword;
  // Getter Setter
}
```
基于 Java Bean 规范，上面的类有 3 个属性：id, username, hashedPassword。这些与 SELECT 语句中的列名**完全匹配**。这样，Java Bean 可以像 HashMap 一样，简单地映射到 ResultSet。

在这些情况下，MyBatis 自动在幕后创建 ResultMap 以将列名基于名称地自动映射到 Java Bean 的属性。


**列名与属性名不匹配的情况如何处理 ?**

- 标准 SQL 语法，SELECT 语句别名
- 使用自定义 `<Result Map>`


#### [3.4.1. Advanced Result Maps](https://mybatis.org/mybatis-3/sqlmap-xml.html#Advanced_Result_Maps)

Result Map 是 MyBatis 强大的工具，可以自由映射结果。


#### [3.4.2. resultMap](https://mybatis.org/mybatis-3/sqlmap-xml.html#resultMap)
`<resultMap>` 元素的概览：

- `<constructor>` 用于在实例化时将结果注入类的构造函数
	- `idArg` ID 参数，标识 ID 将有助于提升整体性能
	- `arg` 普通结果参数注入
- `<id>` 一个 ID 结果，标识 ID 将会提高整体性能
- `<result>` 普通结果参数
- `<association>` 复合类型关联，许多结果会汇总到该类型
	- 嵌套结果映射 - association 本身就是 `resultMap`，或者可以引用其他的 `resultMap`
- `<collection>` 复合类型的集合
	- 嵌套结果映射 - collection 本身就是 `resultMap`，或者可以引用其他的 `resultMap`
- `<discriminator>` 使用结果的值来确定要使用的 `resultMap`


|属性|描述|
|:---|:---|
|id|当前命名空间下引用此 result map 的唯一标识符|
|type|Java 的完全限定名，或者别名|
|autoMapping|如果存在该属性，MyBatis 会为该 result map 启用或禁用自动映射|


#### [3.4.3. id & result](https://mybatis.org/mybatis-3/sqlmap-xml.html#id_.26_result)
id 和 result 都可以映射一个列的值到一个简单数据类型的字段（String,int,double,Date,...）



#### [association](https://mybatis.org/mybatis-3/sqlmap-xml.html#association)
association 元素处理 "has-one" 类型关系。例如，一个博客有一个作者。`<association>` 映射像其他 `<result>` 一样工作。你可以指定 `property`, `javaType`, `jdbcType`, `typeHandler`。

association 不同之处在于，你需要告诉 MyBatis 如何加载关联数据，支持 2 种方式：

- Nested Select：嵌套查询，或者嵌套子查询。通过执行另一个映射 SQL 语句返回所需的**复合类型**。
- Nested Results：嵌套结果。通过使用嵌套的结果映射来处理 join 结果的重复子集。


|属性|描述|
|:---|:---|
|property|字段或属性。如果给定名称存在 Java Bean 匹配的属性，则会使用。否则，MyBatis 会寻找给定名称的字段。|
|javaType|完全限定的 Java 类名（或者 alias）。如果你映射到 Java Bean，MyBatis 会弄清楚类型。如果你映射到 HashMap，你需要指定 javaType 以确保所需的行为。|
|jdbcType||
|typeHandler||



#### Nested Select for Association
|属性|描述|
|:---|:----|
|column|将要传递给嵌套语句的列名或者别名。**可以确定唯一一条记录。**<br>注意：为了处理复合键，可以使用语法 `column="prop1=col1,prop2=col2"` 指定多个列名传递给嵌套的 select 语句|
|select|映射语句的 ID，将会加载需要的复杂类型|
|fetchType|可选。有效值是 lazy 和 eager。如果存在，它会覆盖全局配置参数 `lazyLoadingEnabled`|


> 嵌套 SELECT 必须确保结果集的数量小于等于 1 个，否则你会得到异常：
> org.apache.ibatis.executor.ExecutorException: Statement returned more than one row, where no more than one was expected.

**官方提示** 虽然这种方式比较简单，但是，对于大量数据集或者列表性能并不好。这种问题也被称之为 "N + 1 Select 问题"。简而言之，N + 1 Select 问题是由类似如下引起的：

- 你执行单个 SQL 检索出一个记录列表（+1）
- 对于每个返回的记录，你执行 select 语句来加载每个的细节（N）

该问题会导致大量 SQL 执行，并不是可取的。MyBatis 可以拦截在这些查询语句，因此你可能会忽略这些语句的开销。但是，如果你加载这样的列表，然后立即迭代它访问嵌套数据，则会调用所有延迟加载，因此性能可能非常糟糕。

> 这也是在开发过程中比较忌讳的，在循环体中发起 SQL 查询


#### [Nested Results for Association](https://mybatis.org/mybatis-3/sqlmap-xml.html#Nested_Results_for_Association)
|属性|描述|
|:---|:----|
|resultMap|ID|
|columnPrefix|当 join 多张表时，你往往会使用别名来避免结果重复列名。指定 `columnPrefix` 允许你在映射时添加统一的前缀|
|notNullColumn||
|autoMapping|如果该属性存在，MyBatis 在映射该属性时，将会启用或者禁用自动映射|



#### Multiple ResultSets for Association

#### collection
- 嵌套 select
- 使用 join + 嵌套结果

#### Nested Select for Collection
`<collection>` 元素将会使用新的属性 `ofType`，该属性适用于区分 Java Bean 属性类型和 collection 包含的类型。


#### Nested Results for Collection
需要注意 collection 子元素 id 的重要性。


#### Multiple ResultSets for Collection

#### discriminator

### Auto-mapping
MyBatis 可以自动映射结果，你也可以自己构建 result map，你甚至可以结合两种。

通常，数据库列使用大写字母且单词之间用下划线，而 Java 属性通常遵循驼峰命名约定。可以设置 `mapUnderscoreToCamelCase ` 为 true 完成自动映射。

即使存在 `resultMap`，自动映射也会发生。对于每个结果映射，在结果集中但没有进行手工映射的列，将会被自动映射，之后进行手工映射。

有 3 个映射级别：

- `NONE` 禁用自动映射。即，只会赋值手动映射属性。
- `PARTIAL` 除了其中定义的嵌套结果映射，其他都会自动映射
- `FULL` 自动映射所有



## [3.6. cache](https://mybatis.org/mybatis-3/sqlmap-xml.html#cache) 
默认地，只有 Local Cache 是启用的，用于缓存会话的数据。要启用全局二级缓存，你只需添加一行文字到你的 SQL Mapper 文件：
```xml
<cache/>
```

> 二级缓存需要配置 `cacheEnabled` 为 `true` 才会开启，一般默认是开启的，但建议显式设置。另外，使用二级缓存，还需要实体类实现 `Serializable` 接口，否则将会抛出异常。

这个简单的语句效果如下：

- 缓存所有映射语句文件中 SELECT 语句的结果
- 所有 insert, update, delete 语句将刷新缓存
- 缓存将使用最近最少使用（LRU）算法进行驱逐
- 缓存不会以基于计划的时间顺序刷新
- 缓存将存储 1024 个列表或对象的引用（无论方法返回什么）
- 缓存会被视为读/写缓存，这意味，被检索的对象不会被共享，可以被调用者安全的修改，不会干扰其他线程可能的修改

### Using a Custom Cache
实现 Cache 接口，并在 Mapper 文件中指定 type：
```xml
<cache type="com.domain.something.MyCustomCache"/>
```

### cache-ref
在不同 namespace 之间共享相同的缓存配置和实例：
```xml
<cache-ref namespace="com.someone.application.data.SomeMapper"/>
```


## [5. Java API](https://mybatis.org/mybatis-3/java-api.html)

### [5.2. SqlSessions](https://mybatis.org/mybatis-3/java-api.html#sqlSessions)
通过 `SqlSession`，你可以执行命令，获得 Mapper，管理事务。`SqlSession` 由 `SqlSessionFactory` 创建。


#### 5.2.1. SqlSessionFactoryBuilder
`SqlSessionFactoryBuilder` 有 5 个 build() 方法，每个方法允许你从不同来源构建 `SqlSessionFactory`。

```java
SqlSessionFactory build(InputStream inputStream)
SqlSessionFactory build(InputStream inputStream, String environment)
SqlSessionFactory build(InputStream inputStream, Properties properties)
SqlSessionFactory build(InputStream inputStream, String env, Properties props)
SqlSessionFactory build(Configuration config)
```

`environment` MyBatis 将会使用的环境，如果没有调用包含 environment 参数的方法，则使用默认环境。

`properties` MyBatis 将会加载的属性，可使用 `${propName}` 进行配置



**关于 properties 加载的顺序** 

- 首先，读取 properties 元素主体中指定的属性
- 其次，加载于 properties 元素的 resource 类路径资源或者 url 属性指定的资源将会被读取
- 最后，作为方法参数传递的 properties 最后读取

后加载的属性会覆盖前者


#### 5.2.2. SqlSessionFactory
SqlSessionFactory 有 6 种方法用于创建 SqlSession 实例。一般地，当你选择用哪个方法时，需要考虑以下事情：

- **Transaction**：你是否希望为 session 使用事务范围，或者使用 auto-commit（这通常意味着数据库或 JDBC 没有事务）？
- **Connection**：从 MyBatis 配置好的数据源获取还是自己提供
- **Execution**：重用 PreparedStatement 还是批处理

```java
SqlSession openSession()
SqlSession openSession(boolean autoCommit)
SqlSession openSession(Connection connection)
SqlSession openSession(TransactionIsolationLevel level)
SqlSession openSession(ExecutorType execType, TransactionIsolationLevel level)
SqlSession openSession(ExecutorType execType)
SqlSession openSession(ExecutorType execType, boolean autoCommit)
SqlSession openSession(ExecutorType execType, Connection connection)
Configuration getConfiguration();
```
默认地，无参的 openSession() 将会创建具有如下特征的 SqlSession：

- 启动事务，即不会 auto commit
- Connection 对象从 DataSource 中获取
- 事务隔离级别是驱动或者数据源使用的默认值
- 不会重用 PreparedStatements，不会批量更新


ExecutorType 参数定义了 3 个可选值：

- ExecutorType.SIMPLE：朴素的执行器。为每个执行语句创建一个 PreparedStatement。
- ExecutorType.REUSE：这种类型会重用 PreparedStatement。
- ExecutorType.BATCH：批量更新，同时，如果执行语句之间有 SELECT 语句，也会在必要的地方划清。



#### [5.2.3. SqlSession](https://mybatis.org/mybatis-3/java-api.html#SqlSession)

##### [5.2.3.1. Statement Execution Methods](https://mybatis.org/mybatis-3/java-api.html#Statement_Execution_Methods)
SqlSession 具有许多方法，用于执行 SQL 映射文件中定义的 SELECT, INSERT, UPDATE, DELETE 语句。每个方法都有一个 `statement` 参数（Statement ID），并且可以有 parameter 参数，参数可以是原始类型（自动装箱），或者 Java Bean，POJO，Map。
```java
<T> T selectOne(String statement);
<T> T selectOne(String statement, Object parameter);

<E> List<E> selectList(String statement);
<E> List<E> selectList(String statement, Object parameter);
<E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds);

<T> Cursor<T> selectCursor(String statement);
<T> Cursor<T> selectCursor(String statement, Object parameter);
<T> Cursor<T> selectCursor(String statement, Object parameter, RowBounds rowBounds);

<K, V> Map<K, V> selectMap(String statement, String mapKey);
<K,V> Map<K,V> selectMap(String statement, Object parameter, String mapKey);
<K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds);

void select(String statement, ResultHandler handler);
void select(String statement, Object parameter, ResultHandler handler);
void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler);

int insert(String statement);
int insert(String statement, Object parameter);

int update(String statement);
int update(String statement, Object parameter);

int delete(String statement);
int delete(String statement, Object parameter)
```

> - `selectOne` 和 `selectList` 不同点在于， `selectOne` 必须返回一个对象或者 null（没有对象）。如果超过一个对象，则抛出异常。

如果你不知道有多少对象，使用 `selectList` 总是安全的。

如果你想检查对象是否存在，你最好返回一个计数值（0 或 1）。

`selectMap` 是一个特殊的情况，它会基于结果中的某个属性，将结果列表转换为一个 Map。

因为不是所有方法都需要参数，因此这些方法都有一个无参数的重载方法。

`insert`, `update`, `delete` 的返回值表示的是影响行数。



有几种高级的 `select` 方法，它们允许你限制要返回的行范围，或者提供自定义的结果处理逻辑（ResultHandler），通常用于非常大的数据集（MySQL 几乎不使用）。
```java
<E> List<E> selectList (String statement, Object parameter, RowBounds rowBounds)
<T> Cursor<T> selectCursor(String statement, Object parameter, RowBounds rowBounds)
<K,V> Map<K,V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowbounds)
void select (String statement, Object parameter, ResultHandler<T> handler)
void select (String statement, Object parameter, RowBounds rowBounds, ResultHandler<T> handler)
```
`RowBounds` 参数会让 MyBatis 跳过指定的记录数，同时限制要返回的记录数。

> - 在这里，不同的驱动能够获得不同的效率。为了最佳性能，需要使用结果集类型为 `SCROLL_SENSITIVE ` 或者 `SCROLL_INSENSITIVE `

`ResultHandler` 参数允许你按照你的方式处理每一行。你可以把它添加到 List，创建一个 Map，Set，或者抛出买个结果，而只保留计算的计算的总数。你可以使用 ResultHandler 做任何事情，这就是 MyBatis 内部使用来构造结果集列表的。


ResultHandler 接口定义：

```java
public interface ResultHandler<T> {
  void handleResult(ResultContext<? extends T> context);
}
```
`ResultContext` 参数用于访问结果对象本身，这是被创建的结果对象的数量计数，以及你可以使用 `stop()` 方法停止 MyBatis 加载更多结果。

使用 `ResultHandler` 有两点你应该了解：

- 使用 `ResultHandler` 调用的方法种的数据不会被缓存
- 当使用高级 `resultMap` 时，MyBatis 有可能需要多行来构建 1 个对象。如果使用 `ResultHandler`，可能会给你一个关联或者集合未被填充的对象。




##### [Local Cache](https://mybatis.org/mybatis-3/java-api.html#Local_Cache)
MyBatis 使用了 2 个缓存，Local Cache 和 二级缓存。

每次 MyBatis 创建新的会话时，也会创建一个 Local Cache 附加到会话中。在**本次会话**中执行的任何查询都将存储在本地缓存中，因此具有相同输入参数的相同查询，以后都不会查询数据库。当 update, commit, rollback 和 close 时，Local Cache 会清除。


###### 1. Local Cache 原理 👇

一般地，执行查询方法的时候都是调用 BaseExecutor 的 query() 方法，其方法内部会先考虑是否从 Local Cache 中获取数据，如果不从缓存中获取，则调用内部的 doQuery() 方法


###### Local Cache 的 Key 🐱

org.apache.ibatis.cache.CacheKey 是 Local Cache 的 key


###### 何时可能会失效 Local Cache（失效）？ 🐌
- 不同的会话

不同的 Session 本身不共享 Local Cache。我并不认为这是失效的原因。这反而是一种使用错误的方式。

- 执行 update 操作

如果执行了更新操作会清空 Local Cache
```java
public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    clearLocalCache();
    return doUpdate(ms, parameter);
}
```

- 手动调用 DefaultSqlSession.clearCache()

底层会调用 executor 的 clearLocalCache() 方法

- 设置了语句属性 `flushCache` 为 `true`，缓存全清

- 设置 flushCacheRequired
- 配置 localCacheScope 为 STATEMENT
- commit 和 rollback

##### Ensuring that SqlSession is Closed
你必须确保关闭所有你打开的会话。推荐方式是使用 try 包裹资源。

##### Using Mappers

##### Mapper Annotations

|注解|目标|XML 等价物|描述|
|:---|:---|:---|:---|
|@CacheNamespace|Class|`<cache>`|为给定的命名空间配置缓存|
|@CacheNamespaceRef|Class|`<cacheRef>`|引用其他命名空间的缓存。注意：xml mapper 文件声明的缓存是隔离的|
|@MapKey|Method||用于返回类型是 Map 的方法。基于对象的属性将结果 List 转换为 Map。注解的属性 `value` 作为 Map 的 key|





# [MyBatisPlus](https://mp.baomidou.com/)
---
## Spring
```xml
<dependency>
	<groupId>com.baomidou</groupId>
	<artifactId>mybatis-plus-boot-starter</artifactId>
</dependency>
```

```java
@Configuration
@MapperScan(basePackages = {"com.cannedbread.dictionary.mapper"})
public class MybatisPlusConfiguration {
    @Bean
    public PaginationInterceptor paginationInterceptor() {
        return new PaginationInterceptor();
    }
}
```



# 分页插件
---
- PageHelper

```xml
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
    <version>1.2.12</version>
</dependency>
```

使用方法：
(1) PageHelper.start()，传入 pageNum 当前页码，pageSize 每页显示数目。
(2) 调用 Mapper 方法查询
(3) 使用 pageInfo 构造器传入查询结果集

```java
public PageInfo<ProjectArea> listPageByExample(int pageNum, int pageSize, ProjectArea example) {
	PageHelper.startPage(pageNum, pageSize);
	List<ProjectArea> result = projectAreaMapper.selectList(example);
	PageInfo<ProjectArea> pageInfo = new PageInfo<>(result);
	return pageInfo;
}
```


# 源码分析
## 各种 Logger
- PreparedStatementLogger
- ConnectionLogger




## StatementHandler
### PreparedStatementHandler
执行 SQL 的时候会将 statement 强制转换为 PreparedStatement 类型，并进行执行。
```java
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
	PreparedStatement ps = (PreparedStatement) statement;
	ps.execute();
	return resultSetHandler.handleResultSets(ps);
}
```



## SqlSource

- RawSqlSource
- DynamicSqlSource
- StaticSqlSource，生成 BoundSQL

### RawSqlSource
RawSqlSource 拥有一个 SqlSource 的属性，它是 StaticSqlSource：
```java
public class RawSqlSource implements SqlSource {
	private final SqlSource sqlSource;
}
```



**如何判断一个 SqlNode 是否是动态的 ?**

- 如果是纯文本节点，且存在 `${}`，认为是动态
- 如果是元素节点，则一定是动态

具体代码见 XMLScriptBuilder.parseDynamicTags > TextSqlNode.isDynamic



**为什么 RawSqlSource、DynamicSqlSource 不直接生成 BoundSQL，反而通过   StaticSqlSource 生成?**

设计的问题


DynamicSqlSource 解析是对 SqlNode 的解析，其中会存储根节点 rootSqlNode，节点可以认为是类似 DOM 的节点。

`rootSqlNode.apply(context);` 即遍历 SqlNode 树进行解析，最终生成 context。


## SqlNode
- MixedSqlNode

SQL 语法树从结构来看是树形，但是 MyBatis 会将同一层的所有节点用 MixedSqlNode 包装，因此，从节点的角度看，是一个链表。

```java
public boolean apply(DynamicContext context) {
    contents.forEach(node -> node.apply(context));
    return true;
}
```

- StaticTextSqlNode 静态文本
- TextSqlNode 包含了表达式的文本，如：select * from ${table}

- IfSqlNode

计算表达式的值，如果为 true，则继续解析，一般 if 表达式下面就是静态文本了，所以大部分情况则是直接追加 SQL
```java
public boolean apply(DynamicContext context) {
    if (evaluator.evaluateBoolean(test, context.getBindings())) {
        contents.apply(context);
        return true;
    }
    return false;
}
```

- ForEachNode

解析完毕之后会生成特定的序列，其中每个每个

该解析一定会添加 open 与 close，所以如果在拼接 in 语句的时候没有元素，可能导致 SQL 错误


- TrimSqlNode

从进入该节点开始，将会使用 `FilteredDynamicContext` 包装原来的 context，之后的结果会缓存在 `FilteredDynamicContext` 的 `sqlBuffer` 中，直至该节点解析完毕，最后使用 filteredDynamicContext.applyAll() 真正进行应用。
```java
public boolean apply(DynamicContext context) {
    TrimSqlNode.FilteredDynamicContext filteredDynamicContext = new TrimSqlNode.FilteredDynamicContext(context);
    boolean result = contents.apply(filteredDynamicContext);
    filteredDynamicContext.applyAll();
    return result;
}
```

**为什么加这一个缓存 ?**

如果直接追加 SQL 到最终的结果上，最后还需要做一步 trim，此时已经不方便 trim 了。


- WhereSqlNode

将会删除特定的前缀，并追加前缀 WHERE

```java
private static List<String> prefixList = Arrays.asList("AND ","OR ","AND\n", "OR\n", "AND\r", "OR\r", "AND\t", "OR\t");
```

```java
public class WhereSqlNode extends TrimSqlNode {
	public WhereSqlNode(Configuration configuration, SqlNode contents) {
		super(configuration, contents, "WHERE", prefixList, null, null);
	}
}
```


## rootSqlNode.apply(context)


## MapperProxy
MapperProxy 是 MyBatis 中 InvacationHandler 的实现类。其中，包含一个 SqlSessionTemplate，SqlSessionTemplate 内部又包含一个 DefaultSqlSession 

**为什么 SqlSessionTemplate 又使用代理对象 sqlSessionProxy 去执行方法？**

为了实现拦截操作：
```java
this.sqlSessionProxy = (SqlSession) newProxyInstance(SqlSessionFactory.class.getClassLoader(),
    new Class[] { SqlSession.class }, new SqlSessionInterceptor());
```




## CachingExecutor 的执行
### query
(1) 获得 BoundSql

BoundSql 

(2) 创建缓存 key

创建是由 BaseExecutor 完成的，其中影响因素有：

- MappedStatement.id，即方法全限定名
- rowBounds.offset
- rowBounds.limit
- boundSql.sql
- parameterMappings 的参数值
- parameterMappings.environment.id

(3) flushCacheIfRequired
(4) 使用代理 Executor 执行查询


## SimpleExecutor
### queryFromDatabase
(1) 获取 Configuration
(2) 创建 StatementHandler
(3) prepareStatement
(4) StatementHandler.query
