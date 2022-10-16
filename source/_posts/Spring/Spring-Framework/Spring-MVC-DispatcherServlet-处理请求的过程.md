---
title: Spring MVC DispatcherServlet 处理请求的过程
date: 2022-07-22 22:43:01
tags:
- Spring Framework
---


# Spring MVC DispatcherServlet 处理请求的过程


## 代码清单

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            // 检查是否为 Multipart，底层根据 Content-Type
            // 这里看似 check（检查），其实还有转换（包装）请求，返回 MultipartHttpServletRequest
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);

            // Determine handler for the current request.
            // 确定当前请求的 handler
            // 此处本质上返回的是 handler 的包装 HandlerExecutionChain
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                noHandlerFound(processedRequest, response);
                return;
            }

            // Determine handler adapter for the current request.
            // 确定当前请求的 handler 适配器
            // 因为 handler 是个 Object，它不可能具有处理请求的方法，需要适配
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // Process last-modified header, if supported by the handler.
            String method = request.getMethod();
            boolean isGet = HttpMethod.GET.matches(method);
            if (isGet || HttpMethod.HEAD.matches(method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                    return;
                }
            }

            // 前置处理，应用一些拦截器，如果返回 false，则不再处理，或者抛出异常被 catch
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // Actually invoke the handler.
            // 实际调用 handler
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }

            applyDefaultViewName(processedRequest, mv);
            // 后置处理，应用一些拦截器
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (Exception ex) {
            dispatchException = ex;
        }
        catch (Throwable err) {
            // As of 4.3, we're processing Errors thrown from handler methods as well,
            // making them available for @ExceptionHandler methods and other scenarios.
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }
        // 包括调用 HandlerInterceptor.afterCompletion
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
    catch (Throwable err) {
        triggerAfterCompletion(processedRequest, response, mappedHandler,
                new NestedServletException("Handler processing failed", err));
    }
    finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            // Instead of postHandle and afterCompletion
            if (mappedHandler != null) {
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        }
        else {
            // Clean up any resources used by a multipart request.
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
}
```


## 具体流程


### getHandler

找到合适的 `HandlerMapping` ，把请求映射为 `HandlerExecutionChain` 对象（其中包含一个类型为 `Object` 的 handler 对象）

默认情况下，Spring 提供了 5 个默认的 `HandlerMapping`，分别是：

- `RequestMappingHandlerMapping`
- `BeanNameUrlHandlerMapping`
- `RouterFunctionMapping`
- `SimpleUrlHandlerMapping`
- `WelcomePageHandlerMapping`

比较常用的就是 `RequestMappingHandlerMapping`，一般用于 `@RequestMapping` 注解；`SimpleUrlHandlerMapping`，一般用于映射静态资源。


### getHandlerAdapter

找到 handler 的适配器。

由于 handler 没有一个统一的接口，因此需要寻找合适的适配器。

> 即使 handler 设计一个统一的接口，在统一处理的时候也要写大量的 if else 根据具体的实现类进行逻辑匹配。