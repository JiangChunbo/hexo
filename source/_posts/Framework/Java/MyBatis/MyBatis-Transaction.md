---
title: MyBatis Transaction
date: 2022-11-24 08:51:36
tags:
- MyBatis
---

# Transaction

## 何时构建 Transaction

答案是，在 `openSession` 的时候构建了 `Transaction` 对象。
```java{.line-numbers}
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level,
        boolean autoCommit) {
    Transaction tx = null;
    try {
        final Environment environment = configuration.getEnvironment();
        final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
        tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
        final Executor executor = configuration.newExecutor(tx, execType);
        return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
        closeTransaction(tx); // may have fetched a connection so lets call close()
        throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

第 `6` 行显示，从 `TransactionFactory` 构造了一个 `Transaction` 对象。

在 openConnection 的时候并没有获取连接，更没有开启事务，只是根据配置文件构建了事务对象，更关键的点是执行器的构造方法，构建出来的事务对象通过构造方法被注入到了执行器当中，这一点对后续mybatis的事务体系相当重要。

MyBatis 默认提供两种事务:`JdbcTransaction` 和 `ManagedTransaction`；如果将 MyBatis 和 Spring 一起使用，则提供了一个 `SpringManagedTransaction`


# mybatis-spring

> **注意** 如果你打算将 MyBatis 与 Spring 一起使用，则不需要配置任何 `TransactionManager`，因为 Spring 模块将设置自己的 `TransactionManager`，覆盖之前设置的任何配置。原因参见[MyBatis Spring 覆盖自定义 Environment](../MyBatis-Spring-覆盖自定义-Environment)

## Transactions

使用 MyBatis-Spring 的一个主要原因是它允许 MyBatis 参与 Spring 事务。MyBatis-Spring 没有创建一个特定于 MyBatis 的新事务管理器，而是利用了 Spring 现有的 `DataSourceTransactionManager`。

一旦配置了 Spring 事务管理器，你就可以像往常一样在 Spring 中配置事务。`@Transactional` 注解和 AOP 样式的配置都支持。将创建一个 `SqlSession` 对象，并在事务持续期间使用它。当事务完成时，将根据情况提交或回滚此会话。

一旦设置了事务，MyBatis-Spring 将透明地管理它们。在 DAO 类中不需要额外的代码。


## Standard Configuration

要启用 Spring 事务处理，只需在 Spring 配置文件中创建一个 `DataSourceTransactionManager`:

```xml
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
  <constructor-arg ref="dataSource" />
</bean>
```

```xml
@Configuration
public class DataSourceConfig {
    @Bean
    public DataSourceTransactionManager transactionManager() {
      return new DataSourceTransactionManager(dataSource());
    }
}
```

指定的 `DataSource` 可以是 Spring 通常使用的任何 JDBC 数据源。折包括连接池以及通过 JNDI 查找获得的数据源 `DataSource`。

注意，为事务管理器指定的数据源必须与用于创建 `SqlSessionFactoryBean` 的数据源相同，否则事务管理将无法工作。


## Container Managed Transactions
如果你正在使用 JEE 容器，并且希望 Spring 参与容器管理事务（CMT），那么应该用 `JtaTransactionManager` 或折它的一个特定于容器的子类来配置 Spring。最简单的方法是使用 Spring 事务名称空间或 `JtaTransactionManagerFactoryBean`:

```xml
<tx:jta-transaction-manager />
```

```java
@Configuration
public class DataSourceConfig {
  @Bean
  public JtaTransactionManager transactionManager() {
    return new JtaTransactionManagerFactoryBean().getObject();
  }
}
```

## Programmatic Transaction Management

MyBatis `SqlSession` 提供特定的方法以编程方式处理事务。但是，当你使用 MyBatis-Spring 时，你的 Bean 将会以 Spring 管理的 `SqlSession` 或者是一个 Spring 管理的 mapper 注入。这意味着 Spring 将始终处理你的事务。

不能在 Spring 管理的 SqlSession 上调用 `SqlSession.commit()`，`SqlSession.rollback()`或者是 `SqlSession.close()`。如果尝试这样做，则会抛出 `UnsupportedOperationException` 异常。注意，这些方法不会在注入的 mapper 类中公开。

无论你的 JDBC 连接的 autocommit 设置无论，`SqlSession` 数据方法的任何执行或者对 Spring 事务之外的 mapper 方法的任何调用都是自动提交。


这段代码展示了如何使用 `PlatformTransactionManager` 手动处理事务。

```java
public class UserService {
  private final PlatformTransactionManager transactionManager;
  public UserService(PlatformTransactionManager transactionManager) {
    this.transactionManager = transactionManager;
  }
  public void createUser() {
    TransactionStatus txStatus =
        transactionManager.getTransaction(new DefaultTransactionDefinition());
    try {
      userMapper.insertUser(user);
    } catch (Exception e) {
      transactionManager.rollback(txStatus);
      throw e;
    }
    transactionManager.commit(txStatus);
  }
}
```

你可以使用 `TransactionTemplate` 忽略调用提交和回滚方法。


```java
public class UserService {
  private final PlatformTransactionManager transactionManager;
  public UserService(PlatformTransactionManager transactionManager) {
    this.transactionManager = transactionManager;
  }
  public void createUser() {
    TransactionTemplate transactionTemplate = new TransactionTemplate(transactionManager);
    transactionTemplate.execute(txStatus -> {
      userMapper.insertUser(user);
      return null;
    });
  }
}
```

注意，这段代码使用了一个 mapper，但是它还是会以一个 `SqlSession` 来工作。


Spring 将会以 `SqlSessionTemplate` 取代原来的 `DefaultSqlSession`。

`SqlSessionTemplate` 是一种代理模式的设计思想，其所有的方法全部由内部属性 `sqlSessionProxy` 执行。

`sqlSessionProxy` 本质上也是一个 JDK 动态代理对象。其真正的执行者是 `SqlSessionInterceptor`。

