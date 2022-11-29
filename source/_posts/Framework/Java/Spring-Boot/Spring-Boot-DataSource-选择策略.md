---
title: Spring Boot DataSource 选择策略
date: 2022-11-29 11:21:35
tags:
- Spring Boot
---

# 0. 参考引用

https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-connect-to-production-database

# 选择算法
1. 我们更喜欢 HikariCP 的性能和并发。如果 HikariCP 可用，我们始终会选择它
2. 否则，如果 Tomcat 池化 `DataSource` 可用，我们将使用它
3. 如果 HikariCP 和 Tomcat 池化数据源都不可用，并且 Commons DBCP2 可用，我们会使用它

没有找到任何数据源，则抛出异常


请打开 `DataSourceAutoConfiguration` 类的定义，这是一个自动配置类，其中会尝试自动注入池化 `DataSource`:

```java
@Configuration(proxyBeanMethods = false)
@Conditional(PooledDataSourceCondition.class)
@ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
@Import({ DataSourceConfiguration.Hikari.class, DataSourceConfiguration.Tomcat.class,
        DataSourceConfiguration.Dbcp2.class, DataSourceConfiguration.Generic.class,
        DataSourceJmxConfiguration.class })
protected static class PooledDataSourceConfiguration {

}
```


Hikari

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(HikariDataSource.class)
@ConditionalOnMissingBean(DataSource.class)
@ConditionalOnProperty(
    name = "spring.datasource.type",
    havingValue = "com.zaxxer.hikari.HikariDataSource",
    matchIfMissing = true)
static class Hikari {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.hikari")
    HikariDataSource dataSource(DataSourceProperties properties) {
        HikariDataSource dataSource = createDataSource(properties, HikariDataSource.class);
        if (StringUtils.hasText(properties.getName())) {
            dataSource.setPoolName(properties.getName());
        }
        return dataSource;
    }

}
```

Tomcat JDBC Pool DataSource

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(org.apache.tomcat.jdbc.pool.DataSource.class)
@ConditionalOnMissingBean(DataSource.class)
@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "org.apache.tomcat.jdbc.pool.DataSource",
        matchIfMissing = true)
static class Tomcat {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.tomcat")
    org.apache.tomcat.jdbc.pool.DataSource dataSource(DataSourceProperties properties) {
        org.apache.tomcat.jdbc.pool.DataSource dataSource = createDataSource(properties,
                org.apache.tomcat.jdbc.pool.DataSource.class);
        DatabaseDriver databaseDriver = DatabaseDriver.fromJdbcUrl(properties.determineUrl());
        String validationQuery = databaseDriver.getValidationQuery();
        if (validationQuery != null) {
            dataSource.setTestOnBorrow(true);
            dataSource.setValidationQuery(validationQuery);
        }
        return dataSource;
    }

}
```


Dbcp2

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(org.apache.commons.dbcp2.BasicDataSource.class)
@ConditionalOnMissingBean(DataSource.class)
@ConditionalOnProperty(
    name = "spring.datasource.type",
    havingValue = "org.apache.commons.dbcp2.BasicDataSource",
    matchIfMissing = true)
static class Dbcp2 {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.dbcp2")
    org.apache.commons.dbcp2.BasicDataSource dataSource(DataSourceProperties properties) {
        return createDataSource(properties, org.apache.commons.dbcp2.BasicDataSource.class);
    }

}
```


```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingBean(DataSource.class)
@ConditionalOnProperty(name = "spring.datasource.type")
static class Generic {

    @Bean
    DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }

}
```