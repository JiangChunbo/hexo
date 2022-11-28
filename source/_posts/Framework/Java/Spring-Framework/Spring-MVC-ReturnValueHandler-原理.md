---
title: Spring MVC ReturnValueHandler 原理
date: 2022-07-30 17:28:13
tags:
- Spring Framework
---


# Spring MVC ReturnValueHandler 原理

本文章中提及的 ReturnValueHandler 实际为 `HandlerMethodReturnValueHandler` 类型。


该 `ReturnValueHandler` 会在调用 handler 方法的时候发挥作用，其中通过以下代码注入：

```java
// 注入类型为 HandlerMethodReturnValueHandlerComposite，是一个具有许多 ReturnValueHandler 的类型
invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
```


## Handler Method 处理框架

以下是 handler 方法被调用的整体流程: 

```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
    // 调用 handler，获得返回值
    Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
    setResponseStatus(webRequest);

    if (returnValue == null) {
        if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
            disableContentCachingIfNecessary(webRequest);
            mavContainer.setRequestHandled(true);
            return;
        }
    }
    else if (StringUtils.hasText(getResponseStatusReason())) {
        // 与 ResponseStatus 相关
        mavContainer.setRequestHandled(true);
        return;
    }

    mavContainer.setRequestHandled(false);
    Assert.state(this.returnValueHandlers != null, "No return value handlers");
    try {
        // 处理返回值
        this.returnValueHandlers.handleReturnValue(
                returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
    }
    catch (Exception ex) {
        if (logger.isTraceEnabled()) {
            logger.trace(formatErrorForReturnValue(returnValue), ex);
        }
        throw ex;
    }
}
```

## Return Value 处理框架

若要处理返回值，需要找到合适的返回值处理器，然后调用其 `handleReturnValue` 方法进行处理，以下是处理返回值的整体流程代码清单:

```java
// HandlerMethodReturnValueHandlerComposite
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
        ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

    HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
    if (handler == null) {
        throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
    }
    handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
}
```


## 判断 Return Value 是否支持


关于如何判断一个 `ReturnValueHandler` 是否支持这种返回值，主要是通过自定义实现的方法 `supportsReturnType` 完成，以下是挑选合适 `ReturnValueHandler` 的代码清单：



```java
private HandlerMethodReturnValueHandler selectHandler(@Nullable Object value, MethodParameter returnType) {
    boolean isAsyncValue = isAsyncReturnValue(value, returnType);
    for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
        if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
            continue;
        }
        if (handler.supportsReturnType(returnType)) {
            return handler;
        }
    }
    return null;
}
```

> 是否支持该返回值，具体逻辑根据不同 `ReturnValueHandler` 有所不同，但一般来说都是通过类型判断


应该注意到，`selectHandler` 会首先调用 `isAsyncReturnValue` 判断是否是一个异步返回值，但默认情况下，始终返回 false，因为并不存在 `AsyncHandlerMethodReturnValueHandler` 实例，`isAsyncReturnValue` 逻辑如下:

```java
private boolean isAsyncReturnValue(@Nullable Object value, MethodParameter returnType) {
    for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
        if (handler instanceof AsyncHandlerMethodReturnValueHandler &&
                ((AsyncHandlerMethodReturnValueHandler) handler).isAsyncReturnValue(value, returnType)) {
            return true;
        }
    }
    return false;
}
```


## Spring MVC 支持的返回值

- `ModelAndView` 对象