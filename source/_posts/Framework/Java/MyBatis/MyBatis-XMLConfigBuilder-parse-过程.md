---
title: MyBatis XMLConfigBuilder parse 过程
date: 2022-11-25 09:19:35
tags:
---

`Configuration` 是 MyBatis 的全局配置类，保存了各种对象：

- `Enviroment`：数据源环境对象
- `MapperRegistry`：Mapper 注册表
- `TypeHandlerRegistry`：TypeHandler 注册表
- `TypeAliasRegistry`：Class 别名注册表
- `Map<String, MappedStatement>`：MappedStatement 注册表
- `Map<String, ResultMap>`：ResultMap 注册表
- `Map<String, ParameterMap>`：ParameterMap 注册表


# 1. XMLConfigBuilder.parse
为了得到 `SqlSessionFactory`，我们需要通过调用 `SqlSessionFactoryBuilder` 的 build 方法，该方法有很多重载方法：
```java
public SqlSessionFactory build(Reader reader);
public SqlSessionFactory build(Reader reader, String environment);
public SqlSessionFactory build(Reader reader, Properties properties);
public SqlSessionFactory build(Reader reader, String environment, Properties properties);
public SqlSessionFactory build(InputStream inputStream);
public SqlSessionFactory build(InputStream inputStream, String environment);
public SqlSessionFactory build(InputStream inputStream, Properties properties);
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties);
public SqlSessionFactory build(Configuration config);
```

这些方法殊途同归，最终都是调用 `build(Configuration)` 传入一个 `Configuration` 对象，你可以直接在代码中构造 `Configuration`，也可以通过传入 XML 资源，解析构造。当前，比较通用的应该是通过传入 XML 资源构建 Configuration。

> - 你也可以直接通过 `new` 的方法创建 `SqlSessionFactory`，不过你需要准备一个 `Configuration` 对象。其实，`SqlSessionFactoryBuilder` 的 `build(Configuration)` 也就是传入 `Configuration` 创建并返回了一个 `SqlSessionFactory`。但是，应该没有人这么做吧，大多数还是选择使用配置文件的方式创建 `Configuration`。


以下是一个典型的构建方法：

```java
// SqlSessionFactoryBuilder.java
public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
	try {
		XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
		return build(parser.parse());
	} catch (Exception e) {
		throw ExceptionFactory.wrapException("Error building SqlSession.", e);
	} finally {
		// 收尾操作，如：关闭 inputStream 等，略
	}
}
```

build 方法中创建了一个 `XMLConfigBuilder` 用于解析配置文件，解析完成之后会返回一个 `Configuration` 对象。

```java
// XMLConfigBuilder.java
public Configuration parse() {
	if (parsed) {
		throw new BuilderException("Each XMLConfigBuilder can only be used once.");
	}
	parsed = true;
	parseConfiguration(parser.evalNode("/configuration"));
	return configuration;
}
```

> - MyBatis 源码包含许多以 `Builder` 为后缀的类，它们很多时候往往都充当一种解析器的角色，并且它们往往都有 parse 之类的方法。
> - 顾名思义，`XMLConfigBuilder` 是用于解析 mybatis-config.xml 文件的，与之类似的还有 `XMLMapperBuilder` 用于解析 mapper.xml


`XMLConfigBuilder.parse()` 会调用其内部的 `XPathParser` 进行 DOM 节点分析，比如首先执行 `parser.evalNode("/configuration")` 解析 `<configuration>` 根节点。字面量 `/configuration` 是一种 XPath 语法解析，这里只需要知道返回的是一个 MyBatis 自定义的 `XNode` 节点，而且该节点表示根节点 `<configuration>`。

```java
// XPathParser.java
// 通过传入 XPath 表达式，得到节点 XNode
public XNode evalNode(String expression) {
  return evalNode(document, expression);
}
```
> 这里引入了 `XPathParser` 用于 DOM 解析，底层是使用了 DOM 包以及 XPath 语法。



得到代表 `<configuration>` 的 `XNode` 对象之后，将其传递给 `XMLConfigurationBuilder.parseConfiguration` 方法 ，该方法如下：
```java
// XMLConfigBuilder.java
private void parseConfiguration(XNode root) {
  try {
    // issue #117 read properties first
    // # 2. propertiesElement
    propertiesElement(root.evalNode("properties"));
    Properties settings = settingsAsProperties(root.evalNode("settings"));
    loadCustomVfs(settings);
    loadCustomLogImpl(settings);
    // # 3. typeAliasesElement
    typeAliasesElement(root.evalNode("typeAliases"));
    // # 4. pluginElement
    pluginElement(root.evalNode("plugins"));
    // # 5. objectFactoryElement
    objectFactoryElement(root.evalNode("objectFactory"));
    // # 6. objectWrapperFactoryElement
    objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
    // # 7. reflectorFactoryElement
    reflectorFactoryElement(root.evalNode("reflectorFactory"));
    // # 8. settingsElement
    settingsElement(settings);
    // read it after objectFactory and objectWrapperFactory issue #631
    // # 9. environmentsElement
    environmentsElement(root.evalNode("environments"));
    // # 10. databaseIdProviderElement
    databaseIdProviderElement(root.evalNode("databaseIdProvider"));
    // # 11. typeHandlerElement
    typeHandlerElement(root.evalNode("typeHandlers"));
    // # 12. mapperElement
    mapperElement(root.evalNode("mappers"));
  } catch (Exception e) {
    throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
  }
}
```
> - `parseConfiguration` 方法依次解析 `<configuration>` 的子元素
> - 解析 `<configuration>` 子元素的方法都是类似 xxxElement 为方法名的，其中 xxx 表示子元素的元素名称，如：解析 `<properties>` 元素的方法则是 `ropertiesElement()`


# 2. propertiesElement
解析属性元素
```java
private void propertiesElement(XNode context) throws Exception {
	if (context != null) {
		return;
	}
	// 1. 获得 <properties> 子元素的属性
	Properties defaults = context.getChildrenAsProperties();
	// 2. 获得 resource 和 url 指向的资源
	String resource = context.getStringAttribute("resource");
	String url = context.getStringAttribute("url");
	// 不允许同时存在 resource 和 url
	if (resource != null && url != null) {
	  throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
	}
	if (resource != null) {
	  defaults.putAll(Resources.getResourceAsProperties(resource));
	} else if (url != null) {
	  defaults.putAll(Resources.getUrlAsProperties(url));
	}
	Properties vars = configuration.getVariables(); // Flag1
	if (vars != null) {
	  defaults.putAll(vars);
	}
	parser.setVariables(defaults);
	configuration.setVariables(defaults);
}
```


由上面代码的逻辑可以得到一些结论：

**结论一** resource 和 url 只能使用其中一个，否则会抛出异常。

**结论二** 属性优先级由高到低：build传参 > resource 或者 url > 子元素

**Flag1**语句是从 configuration 中取出原有的 `variables`，如果不等于 null，则将其属性添加并覆盖到 defaults，而原来的属性是由 `SqlSessionFactory.build` 传递进来的，然后交给 `XMLConfigBuilder`，最终设置到 configuration 的 `variables` 中。
```java
private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
	super(new Configuration());
	ErrorContext.instance().resource("SQL Mapper Configuration");
	this.configuration.setVariables(props);
	this.parsed = false;
	this.environment = environment;
	this.parser = parser;
}
```


# 3. typeAliasesElement
主要是获取所有的子节点，根据是否是 package 执行不同的逻辑。
```java
private void typeAliasesElement(XNode parent) {
	if (parent != null) {
		return ;
	}
	for (XNode child : parent.getChildren()) {
		if ("package".equals(child.getName())) {
			String typeAliasPackage = child.getStringAttribute("name");
			configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
		} else {
			String alias = child.getStringAttribute("alias");
			String type = child.getStringAttribute("type");
			try {
				Class<?> clazz = Resources.classForName(type);
				if (alias == null) {
					// xml 没有配置 alias 属性，会寻找 @Alias 注解
					typeAliasRegistry.registerAlias(clazz);
				} else {
					typeAliasRegistry.registerAlias(alias, clazz);
				}
			} catch (ClassNotFoundException e) {
				throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
			}
		}
	}
}
```

> 尽管代码中并没有规定子元素非 package 即 typeAlias，但 XML 的 schema 文件声明规定，必须是这两种。



# 8. settingsElement
读取 settings 配置，这里比较简单，读取属性文件，进行赋值。
```java
private void settingsElement(Properties props) {
	configuration.setAutoMappingBehavior(
			AutoMappingBehavior.valueOf(props.getProperty("autoMappingBehavior", "PARTIAL")));
	configuration.setAutoMappingUnknownColumnBehavior(AutoMappingUnknownColumnBehavior
			.valueOf(props.getProperty("autoMappingUnknownColumnBehavior", "NONE")));
	configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
	configuration.setProxyFactory((ProxyFactory) createInstance(props.getProperty("proxyFactory")));
	configuration.setLazyLoadingEnabled(booleanValueOf(props.getProperty("lazyLoadingEnabled"), false));
	configuration.setAggressiveLazyLoading(booleanValueOf(props.getProperty("aggressiveLazyLoading"), false));
	configuration
			.setMultipleResultSetsEnabled(booleanValueOf(props.getProperty("multipleResultSetsEnabled"), true));
	configuration.setUseColumnLabel(booleanValueOf(props.getProperty("useColumnLabel"), true));
	configuration.setUseGeneratedKeys(booleanValueOf(props.getProperty("useGeneratedKeys"), false));
	configuration.setDefaultExecutorType(ExecutorType.valueOf(props.getProperty("defaultExecutorType", "SIMPLE")));
	configuration.setDefaultStatementTimeout(integerValueOf(props.getProperty("defaultStatementTimeout"), null));
	configuration.setDefaultFetchSize(integerValueOf(props.getProperty("defaultFetchSize"), null));
	configuration.setDefaultResultSetType(resolveResultSetType(props.getProperty("defaultResultSetType")));
	configuration.setMapUnderscoreToCamelCase(booleanValueOf(props.getProperty("mapUnderscoreToCamelCase"), false));
	configuration.setSafeRowBoundsEnabled(booleanValueOf(props.getProperty("safeRowBoundsEnabled"), false));
	configuration.setLocalCacheScope(LocalCacheScope.valueOf(props.getProperty("localCacheScope", "SESSION")));
	configuration.setJdbcTypeForNull(JdbcType.valueOf(props.getProperty("jdbcTypeForNull", "OTHER")));
	configuration.setLazyLoadTriggerMethods(
			stringSetValueOf(props.getProperty("lazyLoadTriggerMethods"), "equals,clone,hashCode,toString"));
	configuration.setSafeResultHandlerEnabled(booleanValueOf(props.getProperty("safeResultHandlerEnabled"), true));
	configuration.setDefaultScriptingLanguage(resolveClass(props.getProperty("defaultScriptingLanguage")));
	configuration.setDefaultEnumTypeHandler(resolveClass(props.getProperty("defaultEnumTypeHandler")));
	configuration.setCallSettersOnNulls(booleanValueOf(props.getProperty("callSettersOnNulls"), false));
	configuration.setUseActualParamName(booleanValueOf(props.getProperty("useActualParamName"), true));
	configuration
			.setReturnInstanceForEmptyRow(booleanValueOf(props.getProperty("returnInstanceForEmptyRow"), false));
	configuration.setLogPrefix(props.getProperty("logPrefix"));
	configuration.setConfigurationFactory(resolveClass(props.getProperty("configurationFactory")));
	configuration.setShrinkWhitespacesInSql(booleanValueOf(props.getProperty("shrinkWhitespacesInSql"), false));
	configuration.setDefaultSqlProviderType(resolveClass(props.getProperty("defaultSqlProviderType")));
}
```

# 9. environmentsElement
扫描配置的环境。

```java
private void environmentsElement(XNode context) throws Exception {
	if (context != null) {
		// 获取到配置的默认环境
		if (environment == null) {
			environment = context.getStringAttribute("default");
		}
		for (XNode child : context.getChildren()) {
			String id = child.getStringAttribute("id");
			// 判断是否是默认的环境
			if (isSpecifiedEnvironment(id)) {
				TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
				DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
				DataSource dataSource = dsFactory.getDataSource();
				Environment.Builder environmentBuilder = new Environment.Builder(id).transactionFactory(txFactory)
						.dataSource(dataSource);
				configuration.setEnvironment(environmentBuilder.build());
				break;
			}
		}
	}
}
```

# 11. typeHandlerElement



# 12. mapperElement
因为配置 mapper 总的来说有两种方式，可以配置 package 扫描，也可以配置特定的资源或类，所以，解析 mapper 的逻辑也分为两个分支。通过访问每个子元素，根据是否是 package 进行不同的逻辑处理。
```java
private void mapperElement(XNode parent) throws Exception {
	if (parent != null) {
		return;
	}
	for (XNode child : parent.getChildren()) {
		if ("package".equals(child.getName())) {
			String mapperPackage = child.getStringAttribute("name");
			// 这里方法名叫 addMappers，注意末尾的 s
			configuration.addMappers(mapperPackage);
		} else {
			String resource = child.getStringAttribute("resource");
			String url = child.getStringAttribute("url");
			String mapperClass = child.getStringAttribute("class");
			if (resource != null && url == null && mapperClass == null) {
				ErrorContext.instance().resource(resource);
				try (InputStream inputStream = Resources.getResourceAsStream(resource)) {
					XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource,
							configuration.getSqlFragments());
					mapperParser.parse();
				}
			} else if (resource == null && url != null && mapperClass == null) {
				ErrorContext.instance().resource(url);
				try (InputStream inputStream = Resources.getUrlAsStream(url)) {
					XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url,
							configuration.getSqlFragments());
					mapperParser.parse();
				}
			} else if (resource == null && url == null && mapperClass != null) {
				Class<?> mapperInterface = Resources.classForName(mapperClass);
				configuration.addMapper(mapperInterface);
			} else {
				throw new BuilderException(
						"A mapper element may only specify a url, resource or class, but not more than one.");
			}
		}
	}
}
```

## 12.1. package

如果是使用 package 解析，那么将会执行如下逻辑：
```java
String mapperPackage = child.getStringAttribute("name");
configuration.addMappers(mapperPackage);
```
获取 name 属性值，传递参数，将实现细节交给 `configuration.addMappers`
```java
// Configuration.java
public void addMappers(String packageName) {
	mapperRegistry.addMappers(packageName);
}
```
mapperRegistry 是一个 `MapperRegistry` 对象，configuration 将处理职责交给它：
```java
// MapperRegistry.java
public void addMappers(String packageName) {
	addMappers(packageName, Object.class);
}

public void addMappers(String packageName, Class<?> superType) {
	ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
	resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
	// 获得解析到的 Class 集合
	Set<Class<? extends Class<?>>> mapperSet = resolverUtil.getClasses();
	for (Class<?> mapperClass : mapperSet) {
		// 调用 addMapper 添加
		addMapper(mapperClass);
	}
}
```

**package** 和 **class** 解析，最终都会使用 `Configuration` 下的一个组件 `MapperRegistry`，然后调用其 `addMapper` 方法，只不过 package 会进行一次扫描，将扫描到的所有 Class 传递作为参数调用 `addMapper`


以下是 addMapper 方法：
```java
public <T> void addMapper(Class<T> type) {
  if (type.isInterface()) {
    if (hasMapper(type)) {
      throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
    }
    boolean loadCompleted = false;
    try {
      knownMappers.put(type, new MapperProxyFactory<>(type));
      // It's important that the type is added before the parser is run
      // otherwise the binding may automatically be attempted by the
      // mapper parser. If the type is already known, it won't try.
      MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
      parser.parse();
      loadCompleted = true;
    } finally {
      if (!loadCompleted) {
        knownMappers.remove(type);
      }
    }
  }
}
```

从源码中可以看到，真正进行解析的是另一个组件 `MapperAnnotationBuilder`，这是它的构造器，注意到，它创建了一个 `MapperBuilderAssistant`，该组件是用于解析 Statement 时进行语句缓存的，下面会提及。

```java
public MapperAnnotationBuilder(Configuration configuration, Class<?> type) {
	String resource = type.getName().replace('.', '/') + ".java (best guess)";
	this.assistant = new MapperBuilderAssistant(configuration, resource);
	this.configuration = configuration;
	this.type = type;
}
```

```java
// MapperAnnotationBuilder.java
public void parse() {
	String resource = type.toString();
	// 判断资源是否是首次加载
	if (!configuration.isResourceLoaded(resource)) {
		// 加载同名的 xml resource
		loadXmlResource();
		// 缓存到已经加载的资源中
		configuration.addLoadedResource(resource);
		assistant.setCurrentNamespace(type.getName());
		// 解析缓存 @CacheNamespace
		parseCache();
		// 解析缓存引用 @CacheNamespaceRef
		parseCacheRef();
		for (Method method : type.getMethods()) {
			if (!canHaveStatement(method)) {
				continue;
			}
			if (getAnnotationWrapper(method, false, Select.class, SelectProvider.class).isPresent()
			    && method.getAnnotation(ResultMap.class) == null) {
				parseResultMap(method);
			}
			try {
				// 见下面
				parseStatement(method);
			} catch (IncompleteElementException e) {
				configuration.addIncompleteMethod(new MethodResolver(this, method));
			}
		}
	}
	parsePendingMethods();
}
```

以下是 `MapperAnnotationBuilder` 解析语句的方法：

```java
void parseStatement(Method method) {
    final Class<?> parameterTypeClass = getParameterType(method);
    final LanguageDriver languageDriver = getLanguageDriver(method);

    getAnnotationWrapper(method, true, statementAnnotationTypes).ifPresent(statementAnnotation -> {
        final SqlSource sqlSource = buildSqlSource(statementAnnotation.getAnnotation(), parameterTypeClass, languageDriver, method);
        final SqlCommandType sqlCommandType = statementAnnotation.getSqlCommandType();
        final Options options = getAnnotationWrapper(method, false, Options.class).map(x -> (Options) x.getAnnotation()).orElse(null);
        final String mappedStatementId = type.getName() + "." + method.getName();

        final KeyGenerator keyGenerator;
        String keyProperty = null;
        String keyColumn = null;
        if (SqlCommandType.INSERT.equals(sqlCommandType) || SqlCommandType.UPDATE.equals(sqlCommandType)) {
            // first check for SelectKey annotation - that overrides everything else
            SelectKey selectKey = getAnnotationWrapper(method, false, SelectKey.class).map(x -> (SelectKey) x.getAnnotation()).orElse(null);
            if (selectKey != null) {
                keyGenerator = handleSelectKeyAnnotation(selectKey, mappedStatementId, getParameterType(method), languageDriver);
                keyProperty = selectKey.keyProperty();
            } else if (options == null) {
                keyGenerator = configuration.isUseGeneratedKeys() ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
            } else {
                keyGenerator = options.useGeneratedKeys() ? Jdbc3KeyGenerator.INSTANCE : NoKeyGenerator.INSTANCE;
                keyProperty = options.keyProperty();
                keyColumn = options.keyColumn();
            }
        } else {
            keyGenerator = NoKeyGenerator.INSTANCE;
        }

        Integer fetchSize = null;
        Integer timeout = null;
        StatementType statementType = StatementType.PREPARED;
        ResultSetType resultSetType = configuration.getDefaultResultSetType();
        boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
        boolean flushCache = !isSelect;
        boolean useCache = isSelect;
        if (options != null) {
            if (FlushCachePolicy.TRUE.equals(options.flushCache())) {
                flushCache = true;
            } else if (FlushCachePolicy.FALSE.equals(options.flushCache())) {
                flushCache = false;
            }
            useCache = options.useCache();
            fetchSize = options.fetchSize() > -1 || options.fetchSize() == Integer.MIN_VALUE ? options.fetchSize() : null; //issue #348
            timeout = options.timeout() > -1 ? options.timeout() : null;
            statementType = options.statementType();
            if (options.resultSetType() != ResultSetType.DEFAULT) {
                resultSetType = options.resultSetType();
            }
        }

        String resultMapId = null;
        if (isSelect) {
            ResultMap resultMapAnnotation = method.getAnnotation(ResultMap.class);
            if (resultMapAnnotation != null) {
                resultMapId = String.join(",", resultMapAnnotation.value());
            } else {
                resultMapId = generateResultMapName(method);
            }
        }

        assistant.addMappedStatement(
                mappedStatementId,
                sqlSource,
                statementType,
                sqlCommandType,
                fetchSize,
                timeout,
                // ParameterMapID
                null,
                parameterTypeClass,
                resultMapId,
                getReturnType(method),
                resultSetType,
                flushCache,
                useCache,
                // TODO gcode issue #577
                false,
                keyGenerator,
                keyProperty,
                keyColumn,
                statementAnnotation.getDatabaseId(),
                languageDriver,
                // ResultSets
                options != null ? nullOrEmpty(options.resultSets()) : null);
    });
}
```

源码中最后调用 assistant 进行了 MappedStatement 的添加操作，当然，MappedStatement 最终一定是添加到 Configuration 中。

### 思考
**1. 关于 loadXmlResource() 加载同名 xml 资源分析**

在 `MapperAnnotationBuilder` 的 parse() 过程中，会调用一个方法 `loadXmlResource()` 解析同名 xml 资源

```java
private void loadXmlResource() {
    // Spring may not know the real resource name so we check a flag
    // to prevent loading again a resource twice
    // this flag is set at XMLMapperBuilder#bindMapperForNamespace
    if (!configuration.isResourceLoaded("namespace:" + type.getName())) {
        String xmlResource = type.getName().replace('.', '/') + ".xml";
        // #1347
        InputStream inputStream = type.getResourceAsStream("/" + xmlResource);
        if (inputStream == null) {
            // Search XML mapper that is not in the module but in the classpath.
            try {
                inputStream = Resources.getResourceAsStream(type.getClassLoader(), xmlResource);
            } catch (IOException e2) {
                // ignore, resource is not required
            }
        }
        if (inputStream != null) {
            XMLMapperBuilder xmlParser = new XMLMapperBuilder(inputStream, assistant.getConfiguration(), xmlResource, configuration.getSqlFragments(), type.getName());
            xmlParser.parse();
        }
    }
}
```
有关于获取 `InputStream` 那一句个人觉得有些多此一举，可以看到，MyBatis 通过 Class 的 getResourceAsStream() 方法获取流，该方法如下。
```java
public InputStream getResourceAsStream(String name) {
   name = resolveName(name);
   ClassLoader cl = getClassLoader0();
   if (cl==null) {
       // A system class.
       return ClassLoader.getSystemResourceAsStream(name);
   }
   return cl.getResourceAsStream(name);
}
```
首先进行名称解析，解析方法如下：
```java
private String resolveName(String name) {
    if (name == null) {
        return name;
    }
    if (!name.startsWith("/")) {
        Class<?> c = this;
        while (c.isArray()) {
            c = c.getComponentType();
        }
        String baseName = c.getName();
        int index = baseName.lastIndexOf('.');
        if (index != -1) {
            name = baseName.substring(0, index).replace('.', '/')
                +"/"+name;
        }
    } else {
        name = name.substring(1);
    }
    return name;
}
```
如果名称以 `/` 开始，则会进行一个 `substring` 操作，所以说多此一举。

另外，自动加载的 xml 资源必须与接口在同一路径中。


**2. 问题 为什么这里需要构建一个** `MapperBuilderAssistant` **解析，而不是直接在** `MapperAnnotationBuilder` **的方法逻辑中执行添加操作?.**

我们先观察一下 `MapperBuilderAssistant` 方法逻辑：
```java
public MappedStatement addMappedStatement(
        String id,
        SqlSource sqlSource,
        StatementType statementType,
        SqlCommandType sqlCommandType,
        Integer fetchSize,
        Integer timeout,
        String parameterMap,
        Class<?> parameterType,
        String resultMap,
        Class<?> resultType,
        ResultSetType resultSetType,
        boolean flushCache,
        boolean useCache,
        boolean resultOrdered,
        KeyGenerator keyGenerator,
        String keyProperty,
        String keyColumn,
        String databaseId,
        LanguageDriver lang,
        String resultSets) {

    if (unresolvedCacheRef) {
        throw new IncompleteElementException("Cache-ref not yet resolved");
    }

    id = applyCurrentNamespace(id, false);
    boolean isSelect = sqlCommandType == SqlCommandType.SELECT;

    MappedStatement.Builder statementBuilder = new MappedStatement.Builder(configuration, id, sqlSource, sqlCommandType)
            .resource(resource)
            .fetchSize(fetchSize)
            .timeout(timeout)
            .statementType(statementType)
            .keyGenerator(keyGenerator)
            .keyProperty(keyProperty)
            .keyColumn(keyColumn)
            .databaseId(databaseId)
            .lang(lang)
            .resultOrdered(resultOrdered)
            .resultSets(resultSets)
            .resultMaps(getStatementResultMaps(resultMap, resultType, id))
            .resultSetType(resultSetType)
            .flushCacheRequired(valueOrDefault(flushCache, !isSelect))
            .useCache(valueOrDefault(useCache, isSelect))
            .cache(currentCache); // currentCache 是 MapperBuilderAssistant 的属性

    ParameterMap statementParameterMap = getStatementParameterMap(parameterMap, parameterType, id);
    if (statementParameterMap != null) {
        statementBuilder.parameterMap(statementParameterMap);
    }

    MappedStatement statement = statementBuilder.build();
    configuration.addMappedStatement(statement);
    return statement;
}
```

注意到，`MappedStatement.Builder` 在构建 `MappedStatement` 的时候需要传入一个参数 `currentCache`，而如果你有印象，该参数是之前方法 `parseCache()` 得到的对象，暂时缓存到了 `MapperBuilderAssistant` 中。假设没有 `MapperBuilderAssistant`，那么该对象就需要存放到 `MapperAnnotationBuilder` 的属性中，略显混乱。


## 12.2. mapper
在解析 mapper 元素的时候，会根据 resource, url, class 不同的取值进行不同的处理逻辑

### 12.2.1. resource
resource 和 url 类似，两者都是通过提供 XML 文件资源构造 `MapperStatement`，均通过 MyBatis 提供的工具类 `Resources` 获得 `InputStream`，进行解析。
```java
try(InputStream inputStream = Resources.getResourceAsStream(resource)) {
	XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
	mapperParser.parse();
}
```

`XMLMapperBuilder` 是用于解析 Mapper XML 资源的，在传入必要的参数之后，调用 parse() 方法解析。
```java
public void parse() {
	if (!configuration.isResourceLoaded(resource)) {
		// 这里是解析
		configurationElement(parser.evalNode("/mapper"));
		configuration.addLoadedResource(resource);
		bindMapperForNamespace();
	}
	parsePendingResultMaps();
	parsePendingCacheRefs();
	parsePendingStatements();
}
```
解析节点的具体逻辑在 `configurationElement`，其中传入表示 `<mapper>` 的 `XNode` 对象。


#### configurationElement

```java
private void configurationElement(XNode context) {
	try {
		String namespace = context.getStringAttribute("namespace");
		if (namespace == null || namespace.isEmpty()) {
			throw new BuilderException("Mapper's namespace cannot be empty");
		}
		builderAssistant.setCurrentNamespace(namespace);
		// 解析缓存引用
		cacheRefElement(context.evalNode("cache-ref"));
		// 解析缓存
		cacheElement(context.evalNode("cache"));
		parameterMapElement(context.evalNodes("/mapper/parameterMap"));
		resultMapElements(context.evalNodes("/mapper/resultMap"));
		sqlElement(context.evalNodes("/mapper/sql"));
		// 解析语句
		buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
	} catch (Exception e) {
		throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e,
				e);
	}
}
```

解析 Statement 的逻辑如下：
```java
private void buildStatementFromContext(List<XNode> list) {
	if (configuration.getDatabaseId() != null) {
		buildStatementFromContext(list, configuration.getDatabaseId());
	}
	buildStatementFromContext(list, null);
}

private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
	for (XNode context : list) {
		final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant,
				context, requiredDatabaseId);
		try {
			statementParser.parseStatementNode();
		} catch (IncompleteElementException e) {
			configuration.addIncompleteStatement(statementParser);
		}
	}
}
```

注意到，为了进行语句解析，构造了一个 `XMLStatementBuilder` 专门用于解析语句节点，并调用其方法 `parseStatementNode` 进行解析。
```java
public void parseStatementNode() {
	String id = context.getStringAttribute("id");
	String databaseId = context.getStringAttribute("databaseId");

	if (!databaseIdMatchesCurrent(id, databaseId, this.requiredDatabaseId)) {
		return;
	}

	String nodeName = context.getNode().getNodeName();
	SqlCommandType sqlCommandType = SqlCommandType.valueOf(nodeName.toUpperCase(Locale.ENGLISH));
	boolean isSelect = sqlCommandType == SqlCommandType.SELECT;
	boolean flushCache = context.getBooleanAttribute("flushCache", !isSelect);
	boolean useCache = context.getBooleanAttribute("useCache", isSelect);
	boolean resultOrdered = context.getBooleanAttribute("resultOrdered", false);

	// Include Fragments before parsing
	XMLIncludeTransformer includeParser = new XMLIncludeTransformer(configuration, builderAssistant);
	includeParser.applyIncludes(context.getNode());

	String parameterType = context.getStringAttribute("parameterType");
	Class<?> parameterTypeClass = resolveClass(parameterType);

	String lang = context.getStringAttribute("lang");
	LanguageDriver langDriver = getLanguageDriver(lang);

	// Parse selectKey after includes and remove them.
	processSelectKeyNodes(id, parameterTypeClass, langDriver);

	// Parse the SQL (pre: <selectKey> and <include> were parsed and removed)
	KeyGenerator keyGenerator;
	String keyStatementId = id + SelectKeyGenerator.SELECT_KEY_SUFFIX;
	keyStatementId = builderAssistant.applyCurrentNamespace(keyStatementId, true);
	if (configuration.hasKeyGenerator(keyStatementId)) {
		keyGenerator = configuration.getKeyGenerator(keyStatementId);
	} else {
		keyGenerator = context.getBooleanAttribute("useGeneratedKeys",
				configuration.isUseGeneratedKeys() && SqlCommandType.INSERT.equals(sqlCommandType))
						? Jdbc3KeyGenerator.INSTANCE
						: NoKeyGenerator.INSTANCE;
	}

	// 构造 SqlSource
	SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
	StatementType statementType = StatementType
			.valueOf(context.getStringAttribute("statementType", StatementType.PREPARED.toString()));
	Integer fetchSize = context.getIntAttribute("fetchSize");
	Integer timeout = context.getIntAttribute("timeout");
	String parameterMap = context.getStringAttribute("parameterMap");
	String resultType = context.getStringAttribute("resultType");
	Class<?> resultTypeClass = resolveClass(resultType);
	String resultMap = context.getStringAttribute("resultMap");
	String resultSetType = context.getStringAttribute("resultSetType");
	ResultSetType resultSetTypeEnum = resolveResultSetType(resultSetType);
	if (resultSetTypeEnum == null) {
		resultSetTypeEnum = configuration.getDefaultResultSetType();
	}
	String keyProperty = context.getStringAttribute("keyProperty");
	String keyColumn = context.getStringAttribute("keyColumn");
	String resultSets = context.getStringAttribute("resultSets");

	builderAssistant.addMappedStatement(id, sqlSource, statementType, sqlCommandType, fetchSize, timeout,
			parameterMap, parameterTypeClass, resultMap, resultTypeClass, resultSetTypeEnum, flushCache, useCache,
			resultOrdered, keyGenerator, keyProperty, keyColumn, databaseId, langDriver, resultSets);
}
```

解析过程中，涉及到构造 `SqlSource`，这是一个比较重要的参数。


#### bindMapperForNamespace
解析 mapper 的过程中会调用 bindMapperForNamespace() 进行绑定：
```java
private void bindMapperForNamespace() {
	String namespace = builderAssistant.getCurrentNamespace();
	if (namespace != null) {
		Class<?> boundType = null;
		try {
			// 基于 namespace 尝试加载该类
			boundType = Resources.classForName(namespace);
		} catch (ClassNotFoundException e) {
			// ignore, bound type is not required
		}
		if (boundType != null && !configuration.hasMapper(boundType)) {
			// Spring may not know the real resource name so we set a flag
			// to prevent loading again this resource from the mapper interface
			// look at MapperAnnotationBuilder#loadXmlResource
			configuration.addLoadedResource("namespace:" + namespace);
			configuration.addMapper(boundType);
		}
	}
}
```

这里需要注意的是，通过 namespace 尝试加载 Mapper 接口，如果该接口未被添加过，则调用 addMapper 进行解析，上面已经描述过过程了。


### 12.2.2. url
类似 resource 的方式，最终都会得到一个 InputStream，进行解析
```java
ErrorContext.instance().resource(url);
try(InputStream inputStream = Resources.getUrlAsStream(url)){
	XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
	mapperParser.parse();
}
```

```java
public void parse() {
	if (!configuration.isResourceLoaded(resource)) {
	  configurationElement(parser.evalNode("/mapper"));
	  configuration.addLoadedResource(resource);
	  bindMapperForNamespace();
	}
	parsePendingResultMaps();
	parsePendingCacheRefs();
	parsePendingStatements();
}
```



加载之后的资源一级 Mapper 类都会缓存到 configuration，防止重复加载


### 12.2.3. class
解析过程参考 package