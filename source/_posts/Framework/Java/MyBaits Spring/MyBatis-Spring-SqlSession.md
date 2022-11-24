---
title: MyBatis Spring SqlSession
date: 2022-11-24 19:51:00
tags:
- MyBatis Spring
---


# Using an SqlSession

在 MyBatis 中，你使用 `SqlSessionFactory` 创建 `SqlSession`。一旦你有了会话，就可以使用它来执行映射语句，提交连接，或者回滚连接，最后，当不再需要它时，就关闭会话。

使用 MyBatis-Spring，你不必直接使用 `SqlSessionFactory`，因为你的 Bean 可以被注入一个线程安全的 `SqlSession`，该 `SqlSession` 可以根据 Spring 的事务配置自动提交、回滚以及关闭会话。

## SqlSessionTemplate

`SqlSessionTemplate` 是 MyBatis-Spring 的核心。它实现了 `SqlSession`，这意味着，可以替代代码中任何 `SqlSession` 的现有使用。`SqlSessionTemplate` 是线程安全的，可以由多个 DAO 或者映射器共享。

> 就是说，只需要一个 `SqlSessionTemplate` 就可以服务多个 `SqlSession`。

当调用 SQL 方法时，包括从 mapper 通过 `getMapper()` 返回的任何方法，`SqlSessionTemplate` 将确保使用的 `SqlSession` 是与当前 Spring 事务相关联的 `SqlSession`。此外，它还管理会话的生命周期，包括必要时关闭，提交，或者回滚会话。它还会将 MyBatis 的异常转换为 Spring 的 `DataAccessException`。

应该总是使用 `SqlSessionTemplate` 而不是默认的 MyBatis 实现 `DefaultSqlSession`，因为该 template 可以参与 Spring 事务，并且对于通过注入的多个 mapper 类是线程安全的。在同一个应用程序中的两个类之间切换可能会导致数据完整性问题。

`SqlSessionTemplate` 可以使用 `SqlSessionFactory` 作为构造函数参数来构造: 

```xml
<bean id="sqlSession" class="org.mybatis.spring.SqlSessionTemplate">
  <constructor-arg index="0" ref="sqlSessionFactory" />
</bean>
```

```java
@Configuration
public class MyBatisConfig {
  @Bean
  public SqlSessionTemplate sqlSession() throws Exception {
    return new SqlSessionTemplate(sqlSessionFactory());
  }
}
```

该 Bean 现在可以直接注入到你的 DAO Bean 中。你在 Bean 中需要一个 `SqlSession` 属性，只要像下面这样:

```java
public class UserDaoImpl implements UserDao {

  private SqlSession sqlSession;

  public void setSqlSession(SqlSession sqlSession) {
    this.sqlSession = sqlSession;
  }

  public User getUser(String userId) {
    return sqlSession.selectOne("org.mybatis.spring.sample.mapper.UserMapper.getUser", userId);
  }
}
```

然后，按如下方法注入 `SqlSessionTemplate`:

```xml
<bean id="userDao" class="org.mybatis.spring.sample.dao.UserDaoImpl">
  <property name="sqlSession" ref="sqlSession" />
</bean>
```

`SqlSessionTemplate` 还有一个构造函数，它接受 `ExecutorType` 作为参数。例如，这允许你在 Spring 的配置文件中使用以下内容来构造一个批处理 `SqlSession`: 

```xml
<bean id="sqlSession" class="org.mybatis.spring.SqlSessionTemplate">
  <constructor-arg index="0" ref="sqlSessionFactory" />
  <constructor-arg index="1" value="BATCH" />
</bean>
```

```java
@Configuration
public class MyBatisConfig {
  @Bean
  public SqlSessionTemplate sqlSession() throws Exception {
    return new SqlSessionTemplate(sqlSessionFactory(), ExecutorType.BATCH);
  }
}
```

## SqlSessionDaoSupport

`SqlSessionDaoSupport` 是一个抽象支持类，它为你提供一个 `SqlSession`。调用 `getSqlSession()` 将得到一个 `SqlSessionTemplate`，然后可以使用它来执行 SQL 方法，如下所示:

```java
public class UserDaoImpl extends SqlSessionDaoSupport implements UserDao {
  public User getUser(String userId) {
    return getSqlSession().selectOne("org.mybatis.spring.sample.mapper.UserMapper.getUser", userId);
  }
}
```

通常 `MapperFactoryBean` 比这个类更受欢迎，因为它不需要额外的代码。但是，如果你需要在你的 DAO 中执行其他非 Mybatis 的工作，并且需要具体的类，那么这个类是有用的。


`SqlSessionDaoSupport` 要求设置一个 `sqlSessionFactory` 或者一个 `sqlSessionTemplate` 属性。如果两个属性都设置了，那么会忽略 `sqlSessionFactory`。

假设有一个 `UserDaoImpl` 类，它是 `SqlSessionDaoSupport` 的子类，它可以像下面这样在 Spring 中配置:

```xml
<bean id="userDao" class="org.mybatis.spring.sample.dao.UserDaoImpl">
  <property name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
```