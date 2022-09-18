---
title: Spring Framework Data Access
date: 2022-08-26 11:50:01
tags:
---

# [Data Access](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#spring-data-tier)
---
## [1. Transaction Management](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#transaction)

### [1.1. Advantages of the Spring Framework’s Transaction Support Model](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#transaction-motivation)


### [1.1.1. Global Transactions](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#transaction-global)


## [1.2. Understanding the Spring Framework Transaction Abstraction](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#transaction-strategies)
Spring 事务抽象的关键在于事务策略概念。事务策略被定义在 `TransactionManager`，特别是，PlatformTransactionManager 用于命令式事务管理和 ReactiveTransactionManager 用于响应式事务管理。

PlatformTransactionManager 类似 Spring Framework IoC 容器的其他 bean。这种好处是，让 Spring Framework 事务称为一个抽象，即使你使用 JTA。你可以比直接使用 JTA 更轻松地测试事务代码。

同样，在 Spring 中，可以被任何 PlatformTransactionManager 接口额方法抛出的 TransactionException 是未检查的（也就是继承于 RuntimeException）。事务基础架构的错误总是致命的。在罕见的情况下，应用程序的代码可以从事务发生的故障中恢复，开发人员仍然可以选择 catch 和处理 TransactionException。


`TransactionDefinition` 接口指定如下内容：

- Propagation：通常，在一个事务范围中的所有代码运行在那个事务中。但是，如果当一个事务上下文已经存在时，运行了一个事务方法，则可以指定这种行为。例如，代码可以继续运行在现有事务中（通常的情况），或者暂停已存在的事务，创建一个新的事务。
- Isolation：隔离级别。该事务从其他事务的工作中隔离的程度。
- Timeout：超时。在超时之前该事务可以运行的时间，并且会自动由事务底层机制回滚。
- Read-only status：你可以在代码读取但不修改数据时使用只读状态。在某些情况下，只读事务时有用的优化，如 Hibernate。

这些设置反映了标准的事务概念。

### [1.4. Declarative Transaction Management](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#transaction-declarative)
> 大多数 Spring 框架用户选择声明式事务管理。此选项对应用程序代码影响最小，是最符合非侵入式轻量容器的理想方式。


基于 Spring 面向切面编程（AOP），Spring 框架的声明式事务管理是可能的。但是，由于事务切面代码随着 Spring Framework 发布而来，并且可以在样板中使用，AOP 的概念没有必要理解为充分利用这段代码。

Spring Framework 的声明式事务管理类似于 EJB CMT，因此你可以指定事务行为到单个方法级别。如有必要，你可以在事务上下文中调用 setRollBackOnly()。


#### [1.4.1. Understanding the Spring Framework’s Declarative Transaction Implementation](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#tx-decl-explained)
仅仅使用 `@Transactional` 注解是没有用的，还需要添加 `@EnableTransactionManagement` 到配置类中。

掌握 Spring Framework 声明式事务支持最重要的概念是通过 AOP 代理开启的支持和由元数据（XML 或者 注解）驱动的 transactional advice。AOP 和事务元数据的组合产生了 AOP 代理，其使用 TransactionInterceptor 与适当的 TransactionManager 实现来驱动围绕方法调用事务。

Spring 框架的 `TransactionInterceptor` 为命令式和响应式编程模型提供事务管理。拦截器通过检查方法返回类型来检测所需的事务管理。


#### [1.4.6. Using @Transactional](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#transaction-declarative-annotations)
除了基于注解的声明式方式进行事务配置之外，你还可以使用基于注解的方式。直接在 Java 代码中声明事务语义上更贴近受影响的代码。

使用在类级别上，表示注解默认作用域声明类的所有方法（包括子类）。或者，每个方法可以被单独注解。注意，类级别的注解不会使用祖先类到类的层次结构；在这样的场景中，需要在本地重新声明，一边参与子类级注解。

在 xml 配种，标签 `<tx:annotation-driven/>`提供相似的简便方式：
```xml
<tx:annotation-driven transaction-manager="txManager"/>
```
> - 如果你想绑定的 `TransactionManager` 名称就是 `transactionManager`，你可以省略配置 `<tx:annotation-driven/>` 标签中的 `transaction-manager` 属性；否则你不得不指定 `transaction-manager` 属性。

##### @Transactional Settings
`@Transactional` 注解是指定接口，类或者方法必须具有事务语义。默认的 `@Transactional` 设置如下：

- 传播类型 `PROPAGATION_REQUIRED`
- 隔离级别 `ISOLATION_DEFAULT`
- 事务是读写型
- 事务超时默认为底层事务系统的默认超时时间，如果不支持超时，则没有
- 任何 `RuntimeException` 触发回滚，任何检查型 `Exception` 并不会。

&nbsp;
##### [Custom Composed Annotations](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#tx-custom-attributes)

#### [1.4.7. Transaction Propagation](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#tx-propagation)

本节描述 Spring 中事务传播的一些语义。请注意，本节不是对事务传播本身的介绍。相反，它详细描述了 Spring 中关于事务传播的一些语义。

在 Spring 管理的事务中，要注意物理事务和逻辑事务之间的区别，以及传播设置如何应用的区别。

##### [Understanding `PROPAGATION_REQUIRED`](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#tx-propagation-required)

<img src="https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/images/tx_prop_required.png">

`PROPAGATION_REQUIRED` 强制执行一个物理事务，如果不存在事务，则本地为当前范围开启事务，或者参与定义为更大范围的已经存在的外部事务。在同一个线程内的公用调用栈安排中，这是一个很好的默认值（例如，一个委托给几个仓库方法的服务门面，其中所有的底层资源都必须参与服务级别事务）

当传播设置为 `PROPAGATION_REQUIRED` 时，会为每个设置应用的方法创建一个逻辑事务范围。每个此类逻辑事务范围可以单独决定仅回滚状态，外部事务范围逻辑上独立于内部事务范围。在标准的 `PROPAGATION_REQUIRED` 行为下，所有这些范围都映射相同的物理事务。内部事务范围的仅回滚标记确实会影响到外部事务实际提交的机会。



##### [Understanding `PROPAGATION_REQUIRES_NEW`](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#tx-propagation-requires_new)
相较于 `PROPAGATION_REQUIRED`，`PROPAGATION_REQUIRES_NEW` 总是为每个受影响的事务范围使用独立的物理事务，从不参与到外部范围的现有事务中去。在这种安排下，底层资源事务是不同的，因此，可以独立提交、回滚，外部事务不受内部事务回滚状态影响，并且内部事务的锁在完成后立即释放。这种独立的内部事务也可以声明自己的隔离级别，超时，以及是否只读，而不继承外部事物的特征。


##### [Understanding `PROPAGATION_NESTED`](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#tx-propagation-nested)
`PROPAGATION_NESTED` 使用一个具有多个可以回滚保存点的物理事务。这种部分回滚让内部事务范围触发其范围内的回滚，并且即使某些操作已经回滚，外部事务仍能够继续物理事务。这个设置通常映射到 JDBC 保存点，因此它只适用于 JDBC 资源事务。参见 Spring 的 [`DataSourceTransactionManager`](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/javadoc-api/org/springframework/jdbc/datasource/DataSourceTransactionManager.html)


#### [1.5.3. Using the TransactionManager](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#transaction-programmatic-tm)

##### [Using the PlatformTransactionManager](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#transaction-programmatic-ptm)
对于强制性的事务，你可以直接使用 `org.springframework.transaction.PlatformTransactionManager` 来管理事务。为此，将你要使用的 `PlatformTransactionManager` 的实现类传递给你的 bean。然后，通过使用 `TransactionDefinition` 和 `TransactionStatus` 对象，你可以启动，回滚以及提交事务。

##### [Using the ReactiveTransactionManager](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#transaction-programmatic-rtm)
当使用响应式事务时，你可以直接使用 `org.springframework.transaction.ReactiveTransactionManager` 管理你的事务。


### [1.6. Choosing Between Programmatic and Declarative Transaction Management](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#tx-decl-vs-prog)
当你具有少量的事务操作时，编程式事务管理通常是个不错的主意。



&nbsp;
## [2. DAO Support](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#dao)

### [2.1. Consistent Exception Hierarchy](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#dao-exceptions)

Spring 提供了特定异常到自己异常类层次结构的转换。该类层次结构将 `DataAccessException` 作为根异常。

## [3. Data Access with JDBC](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#jdbc)

### [3.1. Choosing an Approach for JDBC Database Access](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#jdbc-choose-style)
你可以选择及中方来构建 JDBC 数据库访问基础。除了三种 JdbcTemplate 之外，一个新的 SimpleJdbcInsert 和 SimpleJdbcCall 方法优化了数据库元数据，并且 RDBMS 对象风格采取了一种更面向对象的方式，类似于 JDO 查询设计。

- `JdbcTemplate` 是经典且最流行的 Spring JDBC 方式。这种 "最低级别" 的方法和所有其他封装的方法使用 `JdbcTemplate`
- `NamedParameterJdbcTemplate` 包装了一个 `JdbcTemplate` 以提供命名参数而不是传统的 JDBC ? 占位符。当你的 SQL 语句有多个参数时，这种方法提供了更好的文档性且易于使用。
- `SimpleJdbcInsert` 和 `SimpleJdbcCall` 优化了数据库元数据来限制必要配置的数量。这种方法简化了编码，你只需要提供表名称，或者存储过程名称，并且提供一个参数与列名匹配的映射。


### [3.3. Using the JDBC Core Classes to Control Basic JDBC Processing and Error Handling](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#jdbc-core)

#### [3.3.1. Using JdbcTemplate](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#jdbc-JdbcTemplate)
`JdbcTemplate` 是 JDBC `core` 包下的核心类。它处理资源的创建与释放，这可以帮助你避免一些常见的错误，比如忘记关闭连接。它执行核心 JDBC 工作流的基本任务（如语句的创建和执行），让应用程序代码来提供 SQL 和获取结果集。

`JdbcTemplate` 可以做如下的事情：

- 运行 SQL 查询
- 更新语句以及存储过程调用
- 执行 `ResultSet` 的迭代，以及提取返回参数值的结果
- 捕获 JDBC 异常，并将其转换为定义在 `org.springframework.dao` 包下的更通用，更具信息意义的异常层次

当你使用 `JdbcTemplate` 时，你只需要实现一个回调接口，给出明确定义的约定。给定一个 `JdbcTemplate` 类提供的 `Connection`，`PreparedStatementCreator` 回调接口会创建一个 预编译的语句，提供了 SQL 以及任何必要的参数。`CallableStatementCreator` 接口也是如此，它创建可调用的语句。`RowCallbackHandler` 接口从 `ResultSet` 的每一行中提取值。


你可以在你的 DAO 实现中通过给予一个 `DataSource` 的引用直接实例化 `JdbcTempalte` 并使用；或者，你可以配置它到 Spring IoC 容器里，并将其作为一个 bean 引用给 DAO。

> `DataSource` 应当总是配置为一个 Spring Ioc 容器中的 bean。

此类发出的所有 SQL 都以 debug 级别的日志记录下来，


##### [Querying (SELECT)](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#jdbc-JdbcTemplate-examples-query)


##### [Updating (INSERT, UPDATE, and DELETE) with JdbcTemplate](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#jdbc-JdbcTemplate-examples-update)



#### [3.3.3. Using SQLExceptionTranslator](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/data-access.html#jdbc-SQLExceptionTranslator)
SQLExceptionTranslator 是一个接口，用于 SQLExceptions 和 Spring 的 DataAccessException 之间进行转换。

SQLErrorCodeSQLExceptionTranslator 是 SQLExceptionTranslator 默认的实现。此实现使用特定的供应商代码，比 SQLState 更精确。错误代码转换基于名为 SQLErrorCodes 的 JavaBean 类中保存的代码。此类由 SQLErrorCodesFactory 创建和填充，是基于 sql-error-codes.xml 的配置文件的内容创建的工厂。
