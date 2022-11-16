---
title: Shiro BasicHttpAuthenticationFilter 认证流程
date: 2022-07-08 08:51:17
categories:
- 框架
tags:
- Shiro
- Security
---


# Shiro BasicHttpAuthenticationFilter 认证流程分析

以 Shiro 1.8.0 为例子，其中新增了一些新的组件，譬如 `HttpAuthenticationFilter`
之前使用的 1.4.1 并不存在 `HttpAuthenticationFilter` 组件

## 1. BasicHttpAuthenticationFilter 层次结构

要分析 `BasicHttpAuthenticationFilter` 的认证流程，其实也是分析该过滤器（`doFilter`）的执行流程，而该过滤器的继承层次有一定复杂度，因此先了解一下其继承结构：

<img src="https://cannedbread-1302516612.cos.ap-shanghai.myqcloud.com/BasicHttpAuthenticationFilter.png" width="500">


## 2. OncePerRequestFilter.doFilter

从继承关系可以看到 `BasicHttpAuthenticationFilter` 继承自抽象类 `OncePerRequestFilter`。

> `OncePerRequestFilter` 的字面意思是：Once Per Request，即每个请求只执行一次，它巧妙的设计使得无论添加多少个过滤器，都只执行一次。

如下是 `OncePerRequestFilter` 的 `doFilter` 方法：

```java
@Override
public final void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
        throws ServletException, IOException {

    if (!(request instanceof HttpServletRequest) || !(response instanceof HttpServletResponse)) {
        throw new ServletException("OncePerRequestFilter just supports HTTP requests");
    }
    HttpServletRequest httpRequest = (HttpServletRequest) request;
    HttpServletResponse httpResponse = (HttpServletResponse) response;

    String alreadyFilteredAttributeName = getAlreadyFilteredAttributeName();
    boolean hasAlreadyFilteredAttribute = request.getAttribute(alreadyFilteredAttributeName) != null;

    if (hasAlreadyFilteredAttribute || skipDispatch(httpRequest) || shouldNotFilter(httpRequest)) {
        // 继续执行，不调用此过滤器
        filterChain.doFilter(request, response);
    }
    else {
        // 调用该过滤器
        request.setAttribute(alreadyFilteredAttributeName, Boolean.TRUE);
        try {
            doFilterInternal(httpRequest, httpResponse, filterChain);
        }
        finally {
            // 为本次请求删除 "已经过滤" 的请求属性，能够释放一些空间
            request.removeAttribute(alreadyFilteredAttributeName);
        }
    }
}
```


## 3. AdviceFilter.doFilterInternal
可以知道 `OncePerRequestFilter` 已经实现了 `doFilter`，而且知道，真正的处理逻辑在 `doFilterInternal()` 方法中。然而，`doFilterInternal()` 方法是在子类 `AdviceFilter` 实现的，源码如下：

```java
public void doFilterInternal(ServletRequest request, ServletResponse response, FilterChain chain)
    throws ServletException, IOException {

    Exception exception = null;

    try {
        // 前置处理
        boolean continueChain = preHandle(request, response);
        if (log.isTraceEnabled()) {
            log.trace("Invoked preHandle method.  Continuing chain?: [" + continueChain + "]");
        }

        // 是否继续
        if (continueChain) {
            executeChain(request, response, chain);
        }

        // 后置处理
        postHandle(request, response);
        if (log.isTraceEnabled()) {
            log.trace("Successfully invoked postHandle method");
        }

    } catch (Exception e) {
        exception = e;
    } finally {
        cleanup(request, response, exception);
    }
}
```



## 4. PathMatchingFilter.preHandle

下面先看 preHandle，`PathMatchingFilter` 已经实现了 preHandle()。PathMatchingFilter，顾名思义，路径匹配过滤器，它的作用就是来根据路径匹配结果，调用相应过滤器（没匹配上的直接 return true，即继续执行过滤器链）。


> path 匹配是通过 `FilterChainDefinitionMap` 注册的，比如设置了 “/login”, “anon”，那么如果本次请求的地址也是 /login，则会匹配上。


```java
protected boolean preHandle(ServletRequest request, ServletResponse response) throws Exception {

    if (this.appliedPaths == null || this.appliedPaths.isEmpty()) {
        if (log.isTraceEnabled()) {
            log.trace("appliedPaths property is null or empty.  This Filter will passthrough immediately.");
        }
        return true;
    }

    for (String path : this.appliedPaths.keySet()) {
        // If the path does match, then pass on to the subclass implementation for
        // specific checks
        // (first match 'wins'):
        if (pathsMatch(path, request)) {
            log.trace("Current requestURI matches pattern '{}'.  Determining filter chain execution...", path);
            Object config = this.appliedPaths.get(path);
            return isFilterChainContinued(request, response, path, config);
        }
    }

    // no path matched, allow the request to go through:
    return true;
}
```
> 上述代码中的 config 对象具有非常高的灵活性，在 `BasicHttpAuthenticationFilter` 的流程中，你可以进行一些 HTTP Method、permissive 的特殊配置，这些设计都是内嵌在过滤器中的：
> ```java
> // 表示对于 /** 的 GET POST 请求都需要经过 Basic 验证
> chainDefinition.addPathDefinition("/**", "authcBasic[get,post]");
> // 表示对于 /permissive 开头的请求都会放行
> chainDefinition.addPathDefinition("/permissive/**", "authcBasic[permissive]");
> ``` 

如果 path 匹配成功，则会先执行 isFilterChainContinued()，isFilterChainContinued() 方法也是在 PathMatchingFilter 实现的。它的作用就是判断过滤器是否可用，如果可用就继续执行；否则，跳过，return true。

```java
private boolean isFilterChainContinued(ServletRequest request, ServletResponse response,
    String path, Object pathConfig) throws Exception {

    if (isEnabled(request, response, path, pathConfig)) { // isEnabled check added in 1.2
        if (log.isTraceEnabled()) {
            log.trace("Filter '{}' is enabled for the current request under path '{}' with config [{}].  " +
                "Delegating to subclass implementation for 'onPreHandle' check.",
                new Object[] { getName(), path, pathConfig });
        }
//The filter is enabled for this specific request, so delegate to subclass implementations
//so they can decide if the request should continue through the chain or not:
        return onPreHandle(request, response, pathConfig);
    }

    if (log.isTraceEnabled()) {
        log.trace("Filter '{}' is disabled for the current request under path '{}' with config [{}].  " +
            "The next element in the FilterChain will be called immediately.",
            new Object[] { getName(), path, pathConfig });
    }
//This filter is disabled for this specific request,
//return 'true' immediately to indicate that the filter will not process the request
//and let the request/response to continue through the filter chain:
    return true;
}
```


isEnabled 方法本质上是判断 enabled 是否为 true。其实几乎所有的过滤器都可以执行，因此 enabled 默认为 true，除非人为的去设置它的值：


## 5. AccessControlFilter.onPreHandle

从 `preHandle()` 走下来的，这里之所以起名为 `onPreHandle()`，是因为这才是真正的执行逻辑，之前的种种都是可以看作判断。

`onPreHandle()` 在 `PathMatchingFilter` 的子类 `AccessControlFilter` 有了新的实现，它的返回值依赖两个方法 isAccessAllowed()、onAccessDenied()。


```java
public boolean onPreHandle(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
    return isAccessAllowed(request, response, mappedValue) || onAccessDenied(request, response, mappedValue);
}
```
上述逻辑可以转换为：

```java
public boolean onPreHandle(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
    if (isAccessAllowed(request, response, mappedValue)) {
        return true;
    }
    return onAccessDenied(request, response, mappedValue);
}
```

## 6. HttpAuthenticationFilter.isAccessAllowed

当执行 AccessControlFilter.onPreHandle 会首先判断 `isAccessAllowed`

```java
protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
    HttpServletRequest httpRequest = WebUtils.toHttp(request);
    String httpMethod = httpRequest.getMethod();

    // Check whether the current request's method requires authentication.
    // If no methods have been configured, then all of them require auth,
    // otherwise only the declared ones need authentication.


    // 此处的 mappedValue 其实就是上述的 config 对象，本质上是一个 String[]
    // 这里可以校验一些 HTTP Method 是否需要认证
    // 这也就是为什么你可以设计为 authBasic[get,post]
    Set<String> methods = httpMethodsFromOptions((String[]) mappedValue);
    boolean authcRequired = methods.size() == 0;
    for (String m : methods) {
        if (httpMethod.toUpperCase(Locale.ENGLISH).equals(m)) {
            // 需要认证
            authcRequired = true;
            break;
        }
    }

    if (authcRequired) {
        return super.isAccessAllowed(request, response, mappedValue);
    } else {
        return true;
    }
}
```
上述校验 HTTP Method 的方法对 RESTful 风格非常有效：
实际上，该方法先做了一个 HTTP Method 的比对，自定义 FilterChainDefinitionMap 的时候，可以设置一批 HTTP method 是需要认证的，比如：
如果当前使用 RESTful 风格请求。现有 [PUT] /project 用于更新，[GET] /project 用于获取全部数据，这两个请求 URL 都是一样的，但如何让 GET 请求通过，PUT 请求需要授权呢？答案就是使用 HTTP Method 方法过滤。
配置 /project = authcBasic[PUT]
那么，访问 /project 的时候，GET 方法是不用认证的。
所以现在知道，即使没有写 GET，依然也会走 BasicHttpAuthenticationFilter，只是认证直接跳过（return true）。
因此，如果 HTTP Method 属于这一类 Method，那么就调用了 super.isAccessAllowed 进行判断。


## 7. AuthenticationFilter.isAccessAllowed
下面继续观察 super.isAccessAllowed() 方法到底做了什么？

首先，在继承链上，离 `HttpAuthenticationFilter` 最近的 `AuthenticatingFilter` 也实现了 isAccessAllowed() 方法，如下所示：

```java
protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
    return super.isAccessAllowed(request, response, mappedValue) ||
            (!isLoginRequest(request, response) && isPermissive(mappedValue));
}
```


注意到，该方法也使用了 super 去调用父类方法，找到最近的有实现方法的父类 AuthenticationFilter，方法如下：

```java
protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
    Subject subject = getSubject(request, response);
    return subject.isAuthenticated();
}
```

获取 Subject，然后调用 isAuthenticated() 判断是否已经认证过了。

> 作用：判断是否认证过了，通俗来说，就是登陆了没。

如果 isAccessAllowed 返回 false，表示不允许访问，那么需要继续判断：

```java
!isLoginRequest(request, response) && isPermissive(mappedValue)
```
> `isLoginRequest()` 的判断，在本 BasicHttpAuthenticationFilter 的案例中，根据请求头判断
> `isPermissive()` 的判断是根据你是否在 chainDefinition 配置 [permissive]


## 8. BasicHttpAuthenticationFilter.isLoginRequest

```java
protected final boolean isLoginRequest(ServletRequest request, ServletResponse response) {
    return this.isLoginAttempt(request, response);
}
protected boolean isLoginAttempt(ServletRequest request, ServletResponse response) {
    String authzHeader = getAuthzHeader(request);
    // 判断头部 Authorization 是否以 BASIC 开头
    return authzHeader != null && isLoginAttempt(authzHeader);
}
```

## 9. BasicHttpAuthenticationFilter.onAccessDenied

回到 AccessControlFilter.onPreHandle 第二个处理逻辑 —— `onAccessDenied`

该方法就是 isAccessAllowed 返回 false 之后执行的，即访问拒绝的逻辑。

`BasicHttpAuthenticationFilter` 实现了自己的 onAccessDenied：

```java
protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
    boolean loggedIn = false; // false by default or we wouldn't be in this method
    // 判断是否是 Basic 特定请求头
    if (isLoginAttempt(request, response)) {
        loggedIn = executeLogin(request, response);
    }
    if (!loggedIn) {
        // 发送质询
        sendChallenge(request, response);
    }
    return loggedIn;
}
```


## 10. AuthenticatingFilter.executeLogin

BasicHttpAuthenticationFilter 是没有实现 executeLogin() 的，因此将调用父类 AuthenticatingFilter 的 executeLogin() 方法。

```java
protected boolean executeLogin(ServletRequest request, ServletResponse response) throws Exception {
    AuthenticationToken token = createToken(request, response);
    if (token == null) {
        String msg = "createToken method implementation returned null. A valid non-null AuthenticationToken " +
            "must be created in order to execute a login attempt.";
        throw new IllegalStateException(msg);
    }
    try {
        Subject subject = getSubject(request, response);
        subject.login(token);
        return onLoginSuccess(token, subject, request, response);
    } catch (AuthenticationException e) {
        return onLoginFailure(token, e, request, response);
    }
}
```
createToken()，该方法又是 BasicHttpAuthenticationFilter 来实现的，其实也就是从 Authorization 的 Request Header 提取base64 编码的用户名和密码，然后解析，最终会实例化 UsernamePasswordToken。

```java
protected AuthenticationToken createToken(ServletRequest request, ServletResponse response) {
    String authorizationHeader = getAuthzHeader(request);
    if (authorizationHeader == null || authorizationHeader.length() == 0) {
        // Create an empty authentication token since there is no
        // Authorization header.
        return createToken("", "", request, response);
    }

    log.debug("Attempting to execute login with auth header");

    String[] prinCred = getPrincipalsAndCredentials(authorizationHeader, request);
    if (prinCred == null || prinCred.length < 2) {
        // Create an authentication token with an empty password,
        // since one hasn't been provided in the request.
        String username = prinCred == null || prinCred.length == 0 ? "" : prinCred[0];
        return createToken(username, "", request, response);
    }

    String username = prinCred[0];
    String password = prinCred[1];

    return createToken(username, password, request, response);
}
```


在 createToken 之后，会 getSubject，执行 login()。后面会委托给 SecurityManager.login 方法，在 securityManager 中会对 token 进行验证，本质上就是调用 Realm 方法验证，如果验证过程中没有异常抛出，则顺利执行，



​如果认证过程没有异常抛出，最终会走到 onLoginSuccess()，如果有异常抛出则执行 onLoginFailure()。

> 一般也就是在 Realm 的执行逻辑中抛出异常。