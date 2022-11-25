---
title: MyBatis Spring Mapper
date: 2022-11-25 08:36:23
tags:
---

# Injecting Mappers

无需使用 `SqlSessionDaoSupport` 或者 `SqlSessionTemplate` 手动编写数据访问对象（DAO），MyBatis-Spring 可以创建一个线程安全的 mapper，你可以直接将它注入到其他 Bean 中:

```xml
<bean id="fooService" class="org.mybatis.spring.sample.service.FooServiceImpl">
  <constructor-arg ref="userMapper" />
</bean>
```

注入之后，mapper 就可以在应用程序逻辑中使用了:

```java
public class FooServiceImpl implements FooService {

  private final UserMapper userMapper;

  public FooServiceImpl(UserMapper userMapper) {
    this.userMapper = userMapper;
  }

  public User doSomeBusinessStuff(String userId) {
    return this.userMapper.getUser(userId);
  }
}
```

> **注意** 这段代码中没有 `SqlSession` 或者 MyBatis 引用。也不需要创建，打开，关闭会话，MyBatis-Spring 会处理这些。


## Registering a mapper

注册 mapper 的方式取决于你使用的是经典的 XML 配置，还是新的 3.0+ Java Config（又名 `@Configuration`）。

### With XML Config

通过在你的 XML 配置文件中包含 `MapperFactoryBean` 注册一个 mapper，像下面这样:

```xml
<bean id="userMapper" class="org.mybatis.spring.mapper.MapperFactoryBean">
  <property name="mapperInterface" value="org.mybatis.spring.sample.mapper.UserMapper" />
  <property name="sqlSessionFactory" ref="sqlSessionFactory" />
</bean>
```

如果 `UserMapper` 在与 mapper 接口相同的类路径位置有一个相应的 MyBatis XML mapper 文件，那么它将由 `MapperFactoryBean` 自动解析。无需在 MyBatis 配置文件中指定 mapper，除非映射器 XML 文件位于不同的类路径位置。有关更多信息，请参见 `SqlSessionFactoryBean` 的 `configLocation` 属性。

注意，`MapperFactoryBean` 需要一个 `SqlSessionFactory` 或者一个 `SqlSessionTemplate`。这些可以通过各自的 `sqlSessionFactory` 以及 `sqlSessionTemplate` 属性进行设置。如果设置了这两个属性，`SqlSessionFactory` 将被忽略。由于 `SqlSessionTemplate` 需要设置 session factory，所以 `MapperFactoryBean` 将使用这个工厂。

### With Java Config

```java
@Configuration
public class MyBatisConfig {
  @Bean
  public MapperFactoryBean<UserMapper> userMapper() throws Exception {
    MapperFactoryBean<UserMapper> factoryBean = new MapperFactoryBean<>(UserMapper.class);
    factoryBean.setSqlSessionFactory(sqlSessionFactory());
    return factoryBean;
  }
}
```


## Scanning for mappers

没有必要一个一个地注册所有的 mapper。相反，你可以让 MyBatis-Spring 为它们扫描你的类路径。

有三种不同的方式:

- 使用 `<mybatis:scan>` 元素
- 使用注解 `@MapperScan`
- 使用一个经典的 Spring xml 文件，并注册 `MapperScannerConfigurer`

`<mybatis:scan/>` 和 `@MapperScan` 都是 MyBatis-Spring 1.2.0 引入的特性。`@MapperScan` 需要 Spring 3.1+。

自 2.0.2 开始，mapper 扫描特性支持一个选项（`lazy-initialization`），其控制 mapper Bean 的延迟初始化启用/禁用。添加这个选项的因素是由于 Spring Boot 2.2 支持的延迟初始化控制特性。该选项的默认值是 false（= 不使用延迟初始化）。如果开发人员想对 mapper Bean 使用延迟初始化，它应该被显式地设置为 `true`。


### `@MapperScan`

如果你正在使用 Spring Java 配置（也就是 `@Configuration`），你会更愿意使用 `@MapperScan` 而不是 `<mybatis:scan/>`。

`@MapperScan` 注解使用如下:

```java
@Configuration
@MapperScan("org.mybatis.spring.sample.mapper")
public class AppConfig {
  // ...
}
```

这个注解相较于上一章我们看到的 `<mybatis:scan/>`，以相同、精确的方式工作。它还允许你通过其属性 `markerInterface` 和 `annotaionClass` 指定标记接口。你还可以通过其属性 `SqlSessionFactory` 和 `SqlSessionTemplate` 来提供特定的 `SqlSessionFactory` 或 `SqlSessionTemplate`。

> **注意** 如果 `basePackageClasses` 或者 `basePackages` 没有定义，则将从声明此注解的类的包中进行扫描。


### MapperScannerConfigurer

`MapperScannerConfigurer` 是一个 `BeanDefinitionRegistryPostProcessor`，它可以作为一个普通 Bean 包含在一个经典的 XML 应用程序上下文中。要设置 `MapperScannerConfigurer`，请将以下内容添加到 Spring 配置中:

```xml
<bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
  <property name="basePackage" value="org.mybatis.spring.sample.mapper" />
</bean>
```

如果你需要指定一个特定的 `sqlSessionFactory` 或 `sqlSessionTemplate`，请注意，需要的是 **bean 名称**，而不是 bean 引用，因此使用 value 属性而不是通常的 `ref`:

```xml
<property name="sqlSessionFactoryBeanName" value="sqlSessionFactory" />
```

> **注意** `sqlSessionFactoryBean` 和 `sqlSessionTemplateBean` 属性是 MyBatis-Spring 1.0.2 之前唯一可用的选项，但鉴于 `MapperScannerConfigurer` 在启动过程中较早运行，`PropertyPlaceholderConfigurer` 经常出现错误。为此，已弃用该属性，建议使用新属性 `sqlSessionFactoryBeanName` 和 `sqlSessionTemplateBeanName`。