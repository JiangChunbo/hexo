---
title: Struts2 StrutsPrepareAndExecuteFilter doFilter 过程
date: 2022-11-25 16:14:32
tags:
- Struts2
---

当访问某个 URL 时，会进入 `StrutsPrepareAndExecuteFilter` 的 `doFilter` 方法:

```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {

    HttpServletRequest request = (HttpServletRequest) req;
    HttpServletResponse response = (HttpServletResponse) res;

    try {
        if (excludedPatterns != null && prepare.isUrlExcluded(request, excludedPatterns)) {
            chain.doFilter(request, response);
        } else {
            prepare.setEncodingAndLocale(request, response);
            prepare.createActionContext(request, response);
            prepare.assignDispatcherToThread();
            request = prepare.wrapRequest(request);
            ActionMapping mapping = prepare.findActionMapping(request, response, true);
            if (mapping == null) {
                boolean handled = execute.executeStaticResourceRequest(request, response);
                if (!handled) {
                    chain.doFilter(request, response);
                }
            } else {
                execute.executeAction(request, response, mapping);
            }
        }
    } finally {
        prepare.cleanupRequest(request);
    }
}
```


### setEncodingAndLocale

```java
prepare.setEncodingAndLocale(request, response);
```

该方法只是简单的设置了 encoding 和 locale

### createActionContext

```java
prepare.createActionContext(request, response)
```

这是action上下文的创建，ActionContext是一个容器，这个容器主要存储request、session、application、parameters等相关信 息. ActionContext是一个线程的本地变量，这意味着不同的action之间不会共享ActionContext，所以也不用考虑线程安全问 题。

### wrapRequest
```java
request = prepare.wrapRequest(request)
```

```java
public HttpServletRequest wrapRequest(HttpServletRequest oldRequest) throws ServletException {
    HttpServletRequest request = oldRequest;
    try {
        // Wrap request first, just in case it is multipart/form-data
        // parameters might not be accessible through before encoding (ww-1278)
        request = dispatcher.wrapRequest(request);
        ServletActionContext.setRequest(request);
    } catch (IOException e) {
        throw new ServletException("Could not wrap servlet request with MultipartRequestWrapper!", e);
    }
    return request;
}
```

```java
public HttpServletRequest wrapRequest(HttpServletRequest request) throws IOException {
    // don't wrap more than once
    if (request instanceof StrutsRequestWrapper) {
        return request;
    }

    String content_type = request.getContentType();
    if (content_type != null && content_type.contains("multipart/form-data")) {
        MultiPartRequest mpr = getMultiPartRequest();
        LocaleProvider provider = getContainer().getInstance(LocaleProvider.class);
        request = new MultiPartRequestWrapper(mpr, request, getSaveDir(), provider, disableRequestAttributeValueStackLookup);
    } else {
        request = new StrutsRequestWrapper(request, disableRequestAttributeValueStackLookup);
    }

    return request;
}
```

此次包装根据请求内容的类型不同，返回不同的对象，如果为multipart/form-data类型，则返回MultiPartRequestWrapper类型的对象，该对象服务于文件上传，否则返回StrutsRequestWrapper类型的对象，MultiPartRequestWrapper是StrutsRequestWrapper的子类，而这两个类都是HttpServletRequest接口的实现。


### findActionMapping

```java
ActionMapping mapping = prepare.findActionMapping(request, response, true)
```
包装request后，通过ActionMapper的getMapping()方法得到请求的Action，Action的配置信息存储在ActionMapping对象中，如StrutsPrepareAndExecuteFilter的doFilter方法中第16行：ActionMapping mapping = prepare.findActionMapping(request, response, true);我们找到prepare对象的findActionMapping方法：