---
title: MyBatis Mapper 方法参数处理流程
date: 2022-11-17 10:20:00
tags:
- MyBatis
---

# 参考引用

org.apache.ibatis.reflection.ParamNameResolver#wrapToMapIfCollection

# 结论

- 如果传入的是单个参数
    - 参数类型是 `Collection`，可以使用 `collection` 参数名
    - 参数类型是 `List`，可以使用 `list` 参数名
    - 参数类型是数组，可以使用 `array` 参数名

> 为了较好的语义性，在单个参数是集合或者数组时，可以使用 `@Param` 进行命名。

# 分析


应当知道，当程序调用 Mapper 接口的方法时，将会调用 `MapperProxy` 的 invoke 方法，其中会对方法进行解析并缓存，缓存的数据结构是一个 Map，位于 `MapperProxy` 。

```java
private final Map<Method, MapperMethodInvoker> methodCache;
// 之前版本
private final Map<Method, MapperMethod> methodCache;
```

`methodCache` 的 key 是 Java 反射的 `Method`，value 是 `MapperMethodInvoker`。

> - 在之前的版本中，value 是 `MapperMethod`，其实新版是为了兼容 default 方法做出的扩展。只需要记住 `MapperMethodInvoker` 是用于执行方法的。

`MapperMethodInvoker` 有 2 个实现类，`PlainMethodInvoker` 和 `DefaultMethodInvoker`。这里我们只需要了解 `PlainMethodInvoker`，因为它是 SQL Statement 的执行入口。`PlainMethodInvoker` 将 `MapperMethod` 包装起来，在构造 `PlainMethodInvoker` 时需要传入一个 `MapperMethod` 对象。

`MapperMethod` 的构造器如下：
```java
public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
	this.command = new SqlCommand(config, mapperInterface, method);
	this.method = new MethodSignature(config, mapperInterface, method);
}
```

其中的 `MethodSignature` 跟参数处理密切相关，观察 `MethodSignature` 构造方法：
```java
public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
	Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
	if (resolvedReturnType instanceof Class<?>) {
		this.returnType = (Class<?>) resolvedReturnType;
	} else if (resolvedReturnType instanceof ParameterizedType) {
		this.returnType = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
	} else {
		this.returnType = method.getReturnType();
	}
	this.returnsVoid = void.class.equals(this.returnType);
	this.returnsMany = configuration.getObjectFactory().isCollection(this.returnType) || this.returnType.isArray();
	this.returnsCursor = Cursor.class.equals(this.returnType);
	this.returnsOptional = Optional.class.equals(this.returnType);
	this.mapKey = getMapKey(method);
	this.returnsMap = this.mapKey != null;
	this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
	this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
	// 创建参数名解析器
	this.paramNameResolver = new ParamNameResolver(configuration, method);
}
```

主要关注其中一个组件 `ParamNameResolver`，`ParamNameResolver` 构造方法如下，它接收一个 `Configuration`，以及一个反射的 `Method`。

```java
public ParamNameResolver(Configuration config, Method method) {
	// configuration 就是为了获得一些全局的配置
	this.useActualParamName = config.isUseActualParamName();
	final Class<?>[] paramTypes = method.getParameterTypes();
	final Annotation[][] paramAnnotations = method.getParameterAnnotations();
	final SortedMap<Integer, String> map = new TreeMap<>();
	int paramCount = paramAnnotations.length;
	// get names from @Param annotations
	for (int paramIndex = 0; paramIndex < paramCount; paramIndex++) {
		if (isSpecialParameter(paramTypes[paramIndex])) {
			// skip special parameters
			continue;
		}

		// 寻找该参数是否有 @Param 注解
		String name = null;
		for (Annotation annotation : paramAnnotations[paramIndex]) {
			if (annotation instanceof Param) {
				// 该属性记录整个方法有没有任何一个参数被 @Param 注解，便于后面针对 1 个参数时的处理
				this.hasParamAnnotation = true;
				name = ((Param) annotation).value();
				break;
			}
		}

		if (name == null) {
			// 未找到
			if (useActualParamName) {
				// 使用实际的参数名
				name = getActualParamName(method, paramIndex);
			}
			if (name == null) {
				// use the parameter index as the name ("0", "1", ...)
				// gcode issue #71
				name = String.valueOf(map.size());
			}
		}
		map.put(paramIndex, name);
	}
	this.names = Collections.unmodifiableSortedMap(map);
}
```

经过初步的参数准备之后，会得到一个名称为 `names` 的 Map 结构，其 key 为 `paramIndex`，value 为 参数名。

> - 我比较疑惑的一点是 names 既然索引从 0 开始，为什么不设计成一个 List。


这些工作准备好之后，当调用 MapperMethod 的 execute 方法时，将会派上用场。因为我们需要传递一个 MyBatis 可以识别的参数数据结构来给 SqlSession，便于它将 `#{}` 替换为准确的参数类型。所以，在传递给 SqlSession 参数的时候，会对参数再次进行处理。

```java
// MethodSignature.java
public Object convertArgsToSqlCommandParam(Object[] args) {
	return paramNameResolver.getNamedParams(args);
}
```
这里，我们再次使用 `ParamNameResolver`，调用其 `getNamedParams` 方法获得参数：
```java
// ParamNameResolver.javca
public Object getNamedParams(Object[] args) {
	// 获得之前准备的 names
	final int paramCount = this.names.size();
	if (args == null || paramCount == 0) {
		return null;
	} else if (!hasParamAnnotation && paramCount == 1) {
		// 如果只有一个参数，而且没有被 @Param 注解
		Object value = args[names.firstKey()];
		// 包装
		return wrapToMapIfCollection(value, useActualParamName ? names.get(0) : null);
	} else {
		final Map<String, Object> param = new ParamMap<>();
		int i = 0;
		for (Map.Entry<Integer, String> entry : names.entrySet()) {
			param.put(entry.getValue(), args[entry.getKey()]);
			// add generic param names (param1, param2, ...)
			// 添加通用的名称
			final String genericParamName = GENERIC_NAME_PREFIX + (i + 1);
			// ensure not to overwrite parameter named with @Param
			// 保证 @Param 的高优先级，不能被覆盖了
			if (!names.containsValue(genericParamName)) {
				param.put(genericParamName, args[entry.getKey()]);
			}
			i++;
		}
		return param;
	}
}

public static Object wrapToMapIfCollection(Object object, String actualParamName) {
	if (object instanceof Collection) {
		ParamMap<Object> map = new ParamMap<>();
		map.put("collection", object);
		if (object instanceof List) {
			map.put("list", object);
		}
		Optional.ofNullable(actualParamName).ifPresent(name -> map.put(name, object));
		return map;
	} else if (object != null && object.getClass().isArray()) {
		ParamMap<Object> map = new ParamMap<>();
		map.put("array", object);
		Optional.ofNullable(actualParamName).ifPresent(name -> map.put(name, object));
		return map;
	}
	return object;
}
```

在经过 `getNamedParams` 处理过后，将会得到传递给 sqlSession 的 parameter 参数，这个参数有以下取值：
- 可以是 null
- 可以是一个 Java Bean
- 可以是一个 ParamMap


`parameter` 传递给 SqlSession 之后，不管是 query，还是 update，最终都会将参数传递给 MappedStatement 的 getBoundSql 方法获取 BoundSql。

```java
// MappedStatement.java
public BoundSql getBoundSql(Object parameterObject) {
	BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
	List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
	if (parameterMappings == null || parameterMappings.isEmpty()) {
		boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(),
				parameterObject);
	}
	// check for nested result maps in parameter mappings (issue #30)
	for (ParameterMapping pm : boundSql.getParameterMappings()) {
		String rmId = pm.getResultMapId();
		if (rmId != null) {
			ResultMap rm = configuration.getResultMap(rmId);
			if (rm != null) {
				hasNestedResultMaps |= rm.hasNestedResultMaps();
			}
		}
	}
	return boundSql;
}
```

由 `MappedStatement.getBoundSql` 方法可知，它又将 `parameter` 传递给 `SqlSource.getBoundSql`，SqlSource 一般分为 `RawSqlSource` 和 `DynamicSqlSource`。`RawSqlSource` 表示那些没有动态标签（如：`<if/>`）的 SQL，`DynamicSqlSource` 则表示动态 SQL。


RawSqlSource 的 getBoundSql 非常简单，将实现细节委托给了其内部的 sqlSource。
```java
// RawSqlSource.java
public BoundSql getBoundSql(Object parameterObject) {
	return sqlSource.getBoundSql(parameterObject);
}
```

`RawSqlSource` 内部的 `SqlSource` 是在其实例化的时候构建的，通过构造一个 `SqlSourceBuilder`，调用其 `parse` 方法返回一个 sqlSource。
```java
public RawSqlSource(Configuration configuration, String sql, Class<?> parameterType) {
	SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
	Class<?> clazz = parameterType == null ? Object.class : parameterType;
	this.sqlSource = sqlSourceParser.parse(sql, clazz, new HashMap<>());
}
```

`SqlSourceBuilder.parse` 方法返回的其实是一个 `StaticSqlSource`
```java
public SqlSource parse(String originalSql, Class<?> parameterType, Map<String, Object> additionalParameters) {
	// 创建一个参数映射token的处理器，传入下面的 GenericTokenParser, 可以将 #{} 以及中间的部分转换为 ?
	ParameterMappingTokenHandler handler = new ParameterMappingTokenHandler(configuration, parameterType,
			additionalParameters);
	GenericTokenParser parser = new GenericTokenParser("#{", "}", handler);
	String sql;
	if (configuration.isShrinkWhitespacesInSql()) {
		sql = parser.parse(removeExtraWhitespaces(originalSql));
	} else {
		sql = parser.parse(originalSql);
	}
	return new StaticSqlSource(configuration, sql, handler.getParameterMappings());
}
```

经过 `GenericTokenParser` 和 `ParameterMappingTokenHandler` 配合解析，原始 Sql 中的 `#{}` 将全部替换为 `?`，并且在解析过程 中，`ParameterMappingTokenHandler` 会得到一份 `List<ParameterMapping>` 参数映射列表，保存着每个占位符所需要的参数详细信息。


得到 `StaticSqlSource` 之后，便可以由它创建一个 `BoundSql`。
```java
public BoundSql getBoundSql(Object parameterObject) {
	return new BoundSql(configuration, sql, parameterMappings, parameterObject);
}
```


`BoudSql` 的在对 Statement 进行预处理的时候发挥作用。
```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
	Statement stmt;
	Connection connection = getConnection(statementLog);
	stmt = handler.prepare(connection, transaction.getTimeout());
	handler.parameterize(stmt);
	return stmt;
}
```

主要观察 StatementHandler 的参数化方法 parameterize。
```java
// PreparedStatementHandler.java
public void parameterize(Statement statement) throws SQLException {
	parameterHandler.setParameters((PreparedStatement) statement);
}
```

```java
public void setParameters(PreparedStatement ps) {
	ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
	// 获得 BoundSql 中存储的参数映射列表
	List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
	if (parameterMappings != null) {
		for (int i = 0; i < parameterMappings.size(); i++) {
			ParameterMapping parameterMapping = parameterMappings.get(i);
			if (parameterMapping.getMode() != ParameterMode.OUT) {
				Object value;
				String propertyName = parameterMapping.getProperty();
				if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
					value = boundSql.getAdditionalParameter(propertyName);
				} else if (parameterObject == null) {
					value = null;
				} else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
					value = parameterObject;
				} else {
					MetaObject metaObject = configuration.newMetaObject(parameterObject);
					value = metaObject.getValue(propertyName);
				}
				TypeHandler typeHandler = parameterMapping.getTypeHandler();
				JdbcType jdbcType = parameterMapping.getJdbcType();
				if (value == null && jdbcType == null) {
					jdbcType = configuration.getJdbcTypeForNull();
				}
				try {
					typeHandler.setParameter(ps, i + 1, value, jdbcType);
				} catch (TypeException | SQLException e) {
					throw new TypeException(
							"Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
				}
			}
		}
	}
}
```

