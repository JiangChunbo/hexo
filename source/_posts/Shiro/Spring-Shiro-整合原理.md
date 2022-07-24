---
title: Spring Shiro 整合原理
date: 2022-07-23 19:59:35
tags:
---

# Spring Shiro 整合原理
## SpringShiroFilter

`SpringShiroFilter` 是 Shiro 整合 Spring Web 提供的一个 `Filter`，通过将其配置到 Servlet 容器的过滤器链中参与处理。


1. 包装 Request 和 Response，使它们由原来的 HttpServlet 系列包装（装饰）为 ShiroHttpServletRequest

> 装饰器设计模式，扩展了一些功能

2. 创建 Subject，传递给接下来的过滤器（通过 ThreadLocal）；

如果没有定义任何 `FilterChainDefinitionMap`，那 Shiro 也会把 Request 交给它默认的过滤器过滤。

3. 寻找合适的 `FilterChain`，如果找到则转交给该 `FilterChain` 过滤

4. 更新 SessionLastAccessTime（native sessions）


以下是 `SpringShiroFilter` 的代码清单：

```java
protected void doFilterInternal(ServletRequest servletRequest, ServletResponse servletResponse, final FilterChain chain)
        throws ServletException, IOException {

    Throwable t = null;

    try {
        // 装饰原生 request response
        final ServletRequest request = prepareServletRequest(servletRequest, servletResponse, chain);
        final ServletResponse response = prepareServletResponse(request, servletResponse, chain);

        final Subject subject = createSubject(request, response);

        //noinspection unchecked
        // 通过 Callable 进行增强，底层会将 subject、securityManager 都存到 ThreadLocal
        subject.execute(new Callable() {
            public Object call() throws Exception {
                // 更新 session last access time
                updateSessionLastAccessTime(request, response);
                // 执行链
                executeChain(request, response, chain);
                return null;
            }
        });
    } catch (ExecutionException ex) {
        t = ex.getCause();
    } catch (Throwable throwable) {
        t = throwable;
    }

    if (t != null) {
        if (t instanceof ServletException) {
            throw (ServletException) t;
        }
        if (t instanceof IOException) {
            throw (IOException) t;
        }
        //otherwise it's not one of the two exceptions expected by the filter method signature - wrap it in one:
        String msg = "Filtered request failed.";
        throw new ServletException(msg, t);
    }
}
```


从上述代码可以看到，subject 对象并不是直接调用相关的方法，而是通过向 execute 方法传递一个 `Callable`，该 `Callable` 实际会被 Shiro 的 `SubjectCallable` 包装起来，可以认为是一种装饰器模式。`SubjectCallable` 调用如下：

```java
public V call() throws Exception {
    try {
        // threadState 实际上是 SubjectThreadState
        // bind 方法底层会将 subject, securityManager 都绑定到一个 Map 的 ThreadLocal
        threadState.bind();

        // 实际调用被装饰的 callable
        return doCall(this.callable);
    } finally {
        threadState.restore();
    }
}

protected V doCall(Callable<V> target) throws Exception {
    return target.call();
}
```

Subject 执行 executeChain 的代码清单如下：

```java
protected void executeChain(ServletRequest request, ServletResponse response, FilterChain origChain) throws IOException, ServletException {
    // 寻找合适的 FilterChain
    // 可能是 Tomcat 原生（未找到），也可能是 Shiro 自己链
    FilterChain chain = getExecutionChain(request, response, origChain);
    chain.doFilter(request, response);
}
```

以下是通过 requestURI 获取 FilterChain 的方法：



```java
public FilterChain getChain(ServletRequest request, ServletResponse response, FilterChain originalChain) {
    FilterChainManager filterChainManager = getFilterChainManager();
    if (!filterChainManager.hasChains()) {
        return null;
    }

    final String requestURI = getPathWithinApplication(request);
    final String requestURINoTrailingSlash = removeTrailingSlash(requestURI);

    //the 'chain names' in this implementation are actually path patterns defined by the user.  We just use them
    //as the chain name for the FilterChainManager's requirements
    for (String pathPattern : filterChainManager.getChainNames()) {
        // If the path does match, then pass on to the subclass implementation for specific checks:
        if (pathMatches(pathPattern, requestURI)) {
            if (log.isTraceEnabled()) {
                log.trace("Matched path pattern [{}] for requestURI [{}].  " +
                        "Utilizing corresponding filter chain...", pathPattern, Encode.forHtml(requestURI));
            }
            return filterChainManager.proxy(originalChain, pathPattern);
        } else {

            // in spring web, the requestURI "/resource/menus" ---- "resource/menus/" bose can access the resource
            // but the pathPattern match "/resource/menus" can not match "resource/menus/"
            // user can use requestURI + "/" to simply bypassed chain filter, to bypassed shiro protect

            // 在 Spring Web 中，requestURI "/resource/menus" ---- "resource/menus/" 都可以访问资源
            // 但是 pathPattern 匹配 "/resource/menus" 不能匹配 "resource/menus/"
            // 用户可能使用 requestURI + "/" 来简单跳过 chain filter，以避开 shiro 的保护

            // 因此，这里去除了末尾的 "/"
            pathPattern = removeTrailingSlash(pathPattern);

            // 逻辑类似上面
            if (pathMatches(pathPattern, requestURINoTrailingSlash)) {
                if (log.isTraceEnabled()) {
                    log.trace("Matched path pattern [{}] for requestURI [{}].  " +
                                "Utilizing corresponding filter chain...", pathPattern, Encode.forHtml(requestURINoTrailingSlash));
                }
                return filterChainManager.proxy(originalChain, pathPattern);
            }
        }
    }

    return null;
}
```