---
title: MyBatis Spring 覆盖自定义 Environment
date: 2022-11-24 14:08:24
tags:
---

MyBatis 的官方文档提及到: 

> 如果你打算将 MyBatis 与 Spring 一起使用，则不需要配置任何 `TransactionManager`，因为 Spring 模块将设置自己的 `TransactionManager`，覆盖之前设置的任何配置。

其实，如果你使用 MyBatis-Spring，你无需配置 `<environment>` 节点，配置了也无效。因为后续会使用 Spring 的 `SpringManagedTransactionFactory` 以及 IoC 容器内的 `DataSource` 覆盖配置项。具体见 `SqlSessionFactoryBean.buildSqlSessionFactory` 的执行逻辑:



```java{.line-numbers}
protected SqlSessionFactory buildSqlSessionFactory() throws Exception {

    final Configuration targetConfiguration;

    XMLConfigBuilder xmlConfigBuilder = null;
    if (this.configuration != null) {
        targetConfiguration = this.configuration;
        if (targetConfiguration.getVariables() == null) {
            targetConfiguration.setVariables(this.configurationProperties);
        } else if (this.configurationProperties != null) {
            targetConfiguration.getVariables().putAll(this.configurationProperties);
        }
    } else if (this.configLocation != null) {
        xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null,
                this.configurationProperties);
        targetConfiguration = xmlConfigBuilder.getConfiguration();
    } else {
        LOGGER.debug(
                () -> "Property 'configuration' or 'configLocation' not specified, using default MyBatis Configuration");
        targetConfiguration = new Configuration();
        Optional.ofNullable(this.configurationProperties).ifPresent(targetConfiguration::setVariables);
    }

    Optional.ofNullable(this.objectFactory).ifPresent(targetConfiguration::setObjectFactory);
    Optional.ofNullable(this.objectWrapperFactory).ifPresent(targetConfiguration::setObjectWrapperFactory);
    Optional.ofNullable(this.vfs).ifPresent(targetConfiguration::setVfsImpl);

    if (hasLength(this.typeAliasesPackage)) {
        scanClasses(this.typeAliasesPackage, this.typeAliasesSuperType).stream()
                .filter(clazz -> !clazz.isAnonymousClass()).filter(clazz -> !clazz.isInterface())
                .filter(clazz -> !clazz.isMemberClass())
                .forEach(targetConfiguration.getTypeAliasRegistry()::registerAlias);
    }

    if (!isEmpty(this.typeAliases)) {
        Stream.of(this.typeAliases).forEach(typeAlias -> {
            targetConfiguration.getTypeAliasRegistry().registerAlias(typeAlias);
            LOGGER.debug(() -> "Registered type alias: '" + typeAlias + "'");
        });
    }

    if (!isEmpty(this.plugins)) {
        Stream.of(this.plugins).forEach(plugin -> {
            targetConfiguration.addInterceptor(plugin);
            LOGGER.debug(() -> "Registered plugin: '" + plugin + "'");
        });
    }

    if (hasLength(this.typeHandlersPackage)) {
        scanClasses(this.typeHandlersPackage, TypeHandler.class).stream().filter(clazz -> !clazz.isAnonymousClass())
                .filter(clazz -> !clazz.isInterface()).filter(clazz -> !Modifier.isAbstract(clazz.getModifiers()))
                .forEach(targetConfiguration.getTypeHandlerRegistry()::register);
    }

    if (!isEmpty(this.typeHandlers)) {
        Stream.of(this.typeHandlers).forEach(typeHandler -> {
            targetConfiguration.getTypeHandlerRegistry().register(typeHandler);
            LOGGER.debug(() -> "Registered type handler: '" + typeHandler + "'");
        });
    }

    targetConfiguration.setDefaultEnumTypeHandler(defaultEnumTypeHandler);

    if (!isEmpty(this.scriptingLanguageDrivers)) {
        Stream.of(this.scriptingLanguageDrivers).forEach(languageDriver -> {
            targetConfiguration.getLanguageRegistry().register(languageDriver);
            LOGGER.debug(() -> "Registered scripting language driver: '" + languageDriver + "'");
        });
    }
    Optional.ofNullable(this.defaultScriptingLanguageDriver)
            .ifPresent(targetConfiguration::setDefaultScriptingLanguage);

    if (this.databaseIdProvider != null) {// fix #64 set databaseId before parse mapper xmls
        try {
            targetConfiguration.setDatabaseId(this.databaseIdProvider.getDatabaseId(this.dataSource));
        } catch (SQLException e) {
            throw new NestedIOException("Failed getting a databaseId", e);
        }
    }

    Optional.ofNullable(this.cache).ifPresent(targetConfiguration::addCache);

    if (xmlConfigBuilder != null) {
        try {
            xmlConfigBuilder.parse();
            LOGGER.debug(() -> "Parsed configuration file: '" + this.configLocation + "'");
        } catch (Exception ex) {
            throw new NestedIOException("Failed to parse config resource: " + this.configLocation, ex);
        } finally {
            ErrorContext.instance().reset();
        }
    }

    targetConfiguration.setEnvironment(new Environment(this.environment,
            this.transactionFactory == null ? new SpringManagedTransactionFactory() : this.transactionFactory,
            this.dataSource));

    if (this.mapperLocations != null) {
        if (this.mapperLocations.length == 0) {
            LOGGER.warn(() -> "Property 'mapperLocations' was specified but matching resources are not found.");
        } else {
            for (Resource mapperLocation : this.mapperLocations) {
                if (mapperLocation == null) {
                    continue;
                }
                try {
                    XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(),
                            targetConfiguration, mapperLocation.toString(), targetConfiguration.getSqlFragments());
                    xmlMapperBuilder.parse();
                } catch (Exception e) {
                    throw new NestedIOException("Failed to parse mapping resource: '" + mapperLocation + "'", e);
                } finally {
                    ErrorContext.instance().reset();
                }
                LOGGER.debug(() -> "Parsed mapper file: '" + mapperLocation + "'");
            }
        }
    } else {
        LOGGER.debug(() -> "Property 'mapperLocations' was not specified.");
    }

    return this.sqlSessionFactoryBuilder.build(targetConfiguration);
}
```


第 `85` 行显示 `xmlConfigBuilder` 将会从配置文件中读取配置，但随即 `94` 行就对 `Environment` 进行了重新赋值。因此，从 `<environment>` 节点读取的信息会被覆盖。