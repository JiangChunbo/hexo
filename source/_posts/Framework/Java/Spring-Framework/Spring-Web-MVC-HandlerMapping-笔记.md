---
title: Spring Web MVC HandlerMapping 笔记
date: 2022-07-22 23:28:16
tags:
---

# Spring Web MVC HandlerMapping 笔记


# HandlerMapping
## 1. 介绍
`HandlerMapping` 用于将请求映射到 handler，以及前置处理拦截器和后置处理拦截器。该映射基于某些规则，其中的细节随着 `HandlerMapping` 的实现有所不同。

两个主要的 `HandlerMapping` 的实现类：

(1) `RequestMappingHandlerMapping `，它支持 `@RequestMapping` 注解的方法

(2) `SimpleUrlHandlerMapping`，它维护 URI 路径模式到 handler 的指定注册

> 参考官方文档 [1.1.2. Special Bean Types](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-servlet-special-bean-types)



## 2. Spring Boot
默认情况下，Spring Boot 提供 5 个默认的 `HandlerMapping`，他们分别是：

- `RequestMappingHandlerMapping`
- `BeanNameUrlHandlerMapping`
- `RouterFunctionMapping`
- `SimpleUrlHandlerMapping`
- `WelcomePageHandlerMapping`

上述这几个 `HandlerMapping` 都是在 `WebMvcAutoConfiguration` 内部类 `EnableWebMvcConfiguration` 中配置的。

为了能够通过类名立即能知道该类实现了 `HandlerMapping` 接口，这些实现类都以 `HandlerMapping` 为结尾，以前缀标识其作用。


> 通常 `SimpleUrlHandlerMapping` 会返回 `ResourceHttpRequestHandler`；`RequestMappingHandlerMapping` 返回 `HandlerMethod`



**如何找到当前请求使用哪个 `HandlerMapping` 处理？**

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
	if (this.handlerMappings != null) {
		for (HandlerMapping mapping : this.handlerMappings) {
			HandlerExecutionChain handler = mapping.getHandler(request);
            // 通过返回值是否为 null 确定
			if (handler != null) {
				return handler;
			}
		}
	}
	return null;
}
```

### AbstractHandlerMapping
`AbstractHandlerMapping` 是所有 `HandlerMapping` 的父类，以下是所有请求获取 handler 的统一入口。

```java
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
	// getHandlerInternal 是抽象方法，留给子类实现
	
	Object handler = getHandlerInternal(request);
	// 如果子类无法找到 handler 则使用默认 handler
	if (handler == null) {
		handler = getDefaultHandler();
	}
	// 如果没有默认 handler，则返回 null
	if (handler == null) {
		return null;
	}
	// Bean name or resolved handler?
	if (handler instanceof String) {
		String handlerName = (String) handler;
		handler = obtainApplicationContext().getBean(handlerName);
	}

	HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
	if (CorsUtils.isCorsRequest(request)) {
		CorsConfiguration globalConfig = this.globalCorsConfigSource.getCorsConfiguration(request);
		CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
		CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
		executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
	}
	return executionChain;
}
```

`getHandlerInternal` 是抽象方法，由子类实现，因此官方才会有这样一句话：

> The mapping is based on some criteria, the details of which vary by `HandlerMapping` implementation.

### RequestMappingHandlerMapping
`RequestMappingHandlerMapping` 是 `RequestMappingInfoHandlerMapping` 的子类。`RequestMappingInfoHandlerMapping` 实现了 `getHandlerInternal`：

```java
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
	request.removeAttribute(PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
	try {
		return super.getHandlerInternal(request);
	}
	finally {
		ProducesRequestCondition.clearMediaTypesAttribute(request);
	}
}
```
`RequestMappingInfoHandlerMapping` 并没有做什么具体的操作，他只是调用父类 `AbstractHandlerMethodMapping` 的 `getHandlerInternal` 方法：

```java
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
	String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
	request.setAttribute(LOOKUP_PATH, lookupPath);
	this.mappingRegistry.acquireReadLock();
	try {
		HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
		return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
	}
	finally {
		this.mappingRegistry.releaseReadLock();
	}
}
```

上述代码比较重要的是 `lookupHandlerMethod` 方法调用，它负责寻找 handler 方法：

```java
protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
	List<Match> matches = new ArrayList<>();
	// 先通过 url 找到匹配项
	List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
	// 接着添加匹配的映射（根据具体条件，如 HTTP Method 等）
	if (directPathMatches != null) {
		addMatchingMappings(directPathMatches, matches, request);
	}
	if (matches.isEmpty()) {
		// No choice but to go through all mappings...
		addMatchingMappings(this.mappingRegistry.getMappings().keySet(), matches, request);
	}

	if (!matches.isEmpty()) {
		Match bestMatch = matches.get(0);

		// 如果匹配项不止 1 个
		if (matches.size() > 1) {	
			Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
			// 排序，取出第 0 个
			matches.sort(comparator);
			bestMatch = matches.get(0);
			if (logger.isTraceEnabled()) {
				logger.trace(matches.size() + " matching mappings: " + matches);
			}
			if (CorsUtils.isPreFlightRequest(request)) {
				return PREFLIGHT_AMBIGUOUS_MATCH;
			}
			Match secondBestMatch = matches.get(1);
			if (comparator.compare(bestMatch, secondBestMatch) == 0) {
				Method m1 = bestMatch.handlerMethod.getMethod();
				Method m2 = secondBestMatch.handlerMethod.getMethod();
				String uri = request.getRequestURI();
				// 无法决定最佳匹配，抛异常！！！
				throw new IllegalStateException(
						"Ambiguous handler methods mapped for '" + uri + "': {" + m1 + ", " + m2 + "}");
			}
		}
		// 设置一些属性等
		request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.handlerMethod);
		handleMatch(bestMatch.mapping, lookupPath, request);
		return bestMatch.handlerMethod;
	}
	else {
		return handleNoMatch(this.mappingRegistry.getMappings().keySet(), lookupPath, request);
	}
}
```

上述代码具体做的就是找到一个最佳匹配 handlerMethod 并返回，如果有两个并列的最佳匹配 handlerMethod，则抛出异常，Spring 无法决定选择哪个，因此，建议不要定义具有歧义的 handler。



### RouterFunctionMapping
用于功能性端点的映射，可以阅读[官网](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web-reactive.html#webflux-fn)

### SimpleUrlHandlerMapping

尽管类名称叫 `SimpleUrlHandlerMapping`，但是其 beanName 为 `resourceHandlerMapping`


`SimpleUrlHandlerMapping` 是 `AbstractUrlHandlerMapping` 的子类，所有请求直接经过其父类 `getHandlerInternal` 方法：

```java
protected Object getHandlerInternal(HttpServletRequest request) throws Exception {
	// 获得需要寻找的路径
	String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
	request.setAttribute(LOOKUP_PATH, lookupPath);
	// 寻找 handler
	Object handler = lookupHandler(lookupPath, request);
	if (handler == null) {
		// We need to care for the default handler directly, since we need to
		// expose the PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE for it as well.
		Object rawHandler = null;
		if ("/".equals(lookupPath)) {
			rawHandler = getRootHandler();
		}
		if (rawHandler == null) {
			rawHandler = getDefaultHandler();
		}
		if (rawHandler != null) {
			// Bean name or resolved handler?
			if (rawHandler instanceof String) {
				String handlerName = (String) rawHandler;
				rawHandler = obtainApplicationContext().getBean(handlerName);
			}
			validateHandler(rawHandler, request);
			handler = buildPathExposingHandler(rawHandler, lookupPath, lookupPath, null);
		}
	}
	return handler;
}
```

```java
protected Object lookupHandler(String urlPath, HttpServletRequest request) throws Exception {
	// 通过 map 直接匹配，可以自行通过 WebMvcConfigurer 注册更多项
	// 一般匹配不到，因为我们几乎都是通过 ** 等通配符配置
	Object handler = this.handlerMap.get(urlPath);  // 1
	
	if (handler != null) {
		// Bean name or resolved handler?
		// 如果匹配到...
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = obtainApplicationContext().getBean(handlerName);
		}
		validateHandler(handler, request);
		return buildPathExposingHandler(handler, urlPath, urlPath, null);
	}

	// Pattern match?
	// 没有直接匹配到，进行模式匹配
	List<String> matchingPatterns = new ArrayList<>();
	for (String registeredPattern : this.handlerMap.keySet()) {
		if (getPathMatcher().match(registeredPattern, urlPath)) {
			matchingPatterns.add(registeredPattern);
		}
		else if (useTrailingSlashMatch()) {
			if (!registeredPattern.endsWith("/") && getPathMatcher().match(registeredPattern + "/", urlPath)) {
				matchingPatterns.add(registeredPattern + "/");
			}
		}
	}

	String bestMatch = null;
	Comparator<String> patternComparator = getPathMatcher().getPatternComparator(urlPath);
	if (!matchingPatterns.isEmpty()) {
		// 排序
		matchingPatterns.sort(patternComparator);
		if (logger.isTraceEnabled() && matchingPatterns.size() > 1) {
			logger.trace("Matching patterns " + matchingPatterns);
		}
		// 认为第一个是最好的匹配
		bestMatch = matchingPatterns.get(0);
	}
	if (bestMatch != null) {
		handler = this.handlerMap.get(bestMatch);
		if (handler == null) {
			// 匹配不到。这里是为了处理上面 useTrailingSlashMatch 的问题
			// 去除末尾斜杠
			if (bestMatch.endsWith("/")) {
				handler = this.handlerMap.get(bestMatch.substring(0, bestMatch.length() - 1));
			}
			if (handler == null) {
				throw new IllegalStateException("Could not find handler for best pattern match [" + bestMatch + "]");
			}
		}
		// Bean name or resolved handler?
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = obtainApplicationContext().getBean(handlerName);
		}
		validateHandler(handler, request);
		String pathWithinMapping = getPathMatcher().extractPathWithinPattern(bestMatch, urlPath);

		// There might be multiple 'best patterns', let's make sure we have the correct URI template variables
		// for all of them
		// 可能有多个最佳匹配，我们需要确定对于每个都有正确的 uri 模板变量
		Map<String, String> uriTemplateVariables = new LinkedHashMap<>();
		for (String matchingPattern : matchingPatterns) {
			// 这里可能出现并列最佳匹配
			if (patternComparator.compare(bestMatch, matchingPattern) == 0) {
				Map<String, String> vars = getPathMatcher().extractUriTemplateVariables(matchingPattern, urlPath);
				Map<String, String> decodedVars = getUrlPathHelper().decodePathVariables(request, vars);
				uriTemplateVariables.putAll(decodedVars);
			}
		}
		if (logger.isTraceEnabled() && uriTemplateVariables.size() > 0) {
			logger.trace("URI variables " + uriTemplateVariables);
		}
		return buildPathExposingHandler(handler, bestMatch, pathWithinMapping, uriTemplateVariables);
	}

	// No handler found...
	return null;
}
```

**useTrailingSlashMatch**

在以上代码的模式匹配部分，使用了 `useTrailingSlashMatch()` 方法判断是否使用尾部斜杠匹配。如果启用了，那么例如 `/users` 这样的 URL 模式匹配也可以匹配到 `/users`。默认是 **false**。

如果需要启用该选项，目前我能想到的方法是：在某一个 `@Configuration` 配置类中绑定字段类型 `HandlerMapping`，使用构造器后置处理修改其属性。

```java
@Configuration
@AutoConfigureAfter(WebMvcAutoConfiguration.class)
public class HandlerMappingConfig {

    @Resource(name = "resourceHandlerMapping")
    HandlerMapping handlerMapping;
    
    @PostConstruct
    public void setSimpleUrlHandlerMapping() {
        ((SimpleUrlHandlerMapping) handlerMapping).setUseTrailingSlashMatch(true);
    }
}
```

由于 `SimpleUrlHandlerMapping` 是在 `WebMvcAutoConfiguration` 注入到容器中的，因此需要使用 `@AutoConfigureAfter` 确保自动配置的顺序，防止找到 bean。

由于 `SimpleUrlHandlerMapping` 是以 `HandlerMapping` 类型注入到容器中的，因此无法使用基于类型绑定的 `@Autowired`，只能使用 `@Resource` 根据 bean name 进行绑定。

