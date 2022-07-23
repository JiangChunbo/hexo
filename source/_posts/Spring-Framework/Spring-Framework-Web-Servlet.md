---
title: Spring Framework Web Servlet
date: 2022-05-09 21:35:20
categories:
- 框架
tags:
- Spring Framework
---



# [Web on Servlet Stack](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html)

## [1. Spring Web MVC](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc)

Spring Web MVC 是基于 Servlet API 的原始 Web 框架，从很早的时候就包含在 Spring Framework 之中。正式名称 —— Spring Web MVC，来源于它的源模块名称（[spring-webmvc](https://github.com/spring-projects/spring-framework/tree/master/spring-webmvc)），但通常称之为 "Spring MVC"。

与 Spring Web MVC 平行，Spring Framework 5.0 引入了一个响应式栈 web 框架，其名称为 "Spring WebFlux"，也是基于其源模块（[spring-webflux](https://github.com/spring-projects/spring-framework/tree/master/spring-webflux)）。本节涵盖 Spring Web MVC。[下一节](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web-reactive.html#spring-web-reactive)涵盖 Spring WebFlux。

有关基线信息以及与 Servlet 容器和 Java EE 版本范围的兼容性，请参见 Spring Framework [Wiki](https://github.com/spring-projects/spring-framework/wiki/Spring-Framework-Versions)。


### [1.1. DispatcherServlet](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-servlet)

[WebFlux](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web-reactive.html#webflux-dispatcher-handler)

与许多其他 Web 框架一样，Spring MVC 围绕着前端控制器模式设计，其中首要的 `Servlet` —— `DispatcherServlet` 为请求处理提供了一组共享的算法，而实际的工作则由可配置的代理组件执行。这种模型灵活，并且支持各种工作流。

> **作者的话** 简单来说，`DispatcherServlet` 为所有请求提供统一的处理流程（`doDispatch`），但是细节部分由各个组件实现，而且并不是每个组件在处理每个请求时都会发挥作用。

正如任何 `Servlet` 一样，`DispatcherServlet` 也需要根据 Servlet 规范进行声明以及映射（路径），可以通过 Java 配置或者在 `web.xml` 中声明以及映射。反过来，`DispatcherServlet` 使用 Spring 配置，去发现它所需要的委托组件，比如用于请求映射，视图解析，异常处理的组件，等等。

下面是有关于 Java 配置注册以及初始化 `DispatcherServlet` 的示例，这会被 Servlet 容器自动检测（参见 [Servlet Config](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-container-config)）：

```java
public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletContext) {

        // Load Spring web application configuration
        AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
        context.register(AppConfig.class);

        // Create and register the DispatcherServlet
        DispatcherServlet servlet = new DispatcherServlet(context);
        ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
        registration.setLoadOnStartup(1);
        registration.addMapping("/app/*");
    }
}
```

> 除了直接使用 ServletContext API，你还可以继承 `AbstractAnnotationConfigDispatcherServletInitializer` 并覆盖特定的方法（见 [Context Hierarchy](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-servlet-context-hierarchy) 下面的例子）。

> **作者的话** 不必太关心这里，只有基于外部容器部署的时候才会加载 `WebApplicationInitializer`


以下示例是 `web.xml` 配置注册以及初始化 `DispatcherServlet`：
```xml
<web-app>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/app-context.xml</param-value>
    </context-param>

    <servlet>
        <servlet-name>app</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value></param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>app</servlet-name>
        <url-pattern>/app/*</url-pattern>
    </servlet-mapping>

</web-app>
```

> Spring Boot 遵循一种不同的初始化顺序。Spring Boot 并不是直接挂载到 Servlet 容器的生命周期，而是使用 Spring 配置来引导自身以及内嵌的 Servlet 容器。`Filter` 和 `Servlet` 的声明以 Spring 配置的方式被检测到，并注册到 Servlet 容器。

#### [1.1.1. Context Hierarchy](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-container-config)

`DispatcherServlet` 期望一个 `WebApplicationContext`（继承于简单 `ApplicationContext`） 用于自己的配置。`WebApplicationContext` 持有 `ServletContext` 的引用，并且持有相关的 `Servlet` 的引用。`WebApplicationContext` 也会绑定到 `ServletContext`，以便于应用程序在需要的时候可以通过 `RequestContextUtiles` 的静态方法找到 `WebApplicationContext`。

> **作者的话** 上面这句话简单来看就是 `WebApplicationContext` 和 `ServletContext` 相互引用。`ServletContext` 绑定 `WebApplicationContext` 方式是设置为请求的属性。

对于许多应用程序而言，具有单个的 `WebApplicationContext` 是简易且足够用了。也有可能，有一种上下文结构，其中，根 `WebApplicationContext` 被多个 `DispatcherServlet` 实例共享（或者其他 `Servlet`），每个 `Servlet` 又有自己的子 `WebApplicationContext` 配置。有关上下文层次结构功能的更多信息，请参见 [ Additional Capabilities of the `ApplicationContext`](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/core.html#context-introduction)


root `WebApplicationContext` 通常包含基于架构 Bean，例如数据仓库和业务服务，它们需要在多个 `Servlet` 实例上共享。这些 Bean 被有效继承，并且可以在特定于 Servlet 的子 `WebApplicationContext` 中覆盖，这通常包含给定的本地 `Servlet` Bean。下图展示了这种关系：

<img src="https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/images/mvc-context-hierarchy.png">

下面的示例配置了一个 `WebApplicationContext` 层次结构：

```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[] { RootConfig.class };
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { App1Config.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/app1/*" };
    }
}
```

> 如果不需要应用程序上下文的层次结构，应用程序可以通过 `getRootConfigClasses()` 返回所有的配置，并且在 `getServletConfigClasses()` 返回 `null`。

下面示例展示了 `web.xml` 的等价物：

```xml
<web-app>

    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/root-context.xml</param-value>
    </context-param>

    <servlet>
        <servlet-name>app1</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/app1-context.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>app1</servlet-name>
        <url-pattern>/app1/*</url-pattern>
    </servlet-mapping>

</web-app>
```

> 如果应用程序不需要上下文层次结构，则可以仅仅配置 "root" 上下文，并保持 `contextConfigLocation` Servlet 参数为空。

#### [1.1.2. Special Bean Types](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-servlet-special-bean-types)

`DispatcherServlet` 委托特殊的 Bean 来处理请求并给予合适的响应。通过“特殊 Bean”，我们意思是实现了框架约定的由 Spring 管理的 `Object`。这些通常带有内置的约定，但是你可以定制化它们的属性并扩展或更换它们。

下面的表格列出了由 `DispatcherServlet` 检测的特殊 Bean：

|Bean Type|描述|
|:---|:---|
|`HandlerMapping`|将请求映射到 handler，以及前置处理拦截器和后置处理拦截器的列表。该映射基于某些规则，其中的细节随着 `HandlerMapping` 的实现有所不同。<br><br>两个主要的 `HandlerMapping` 的实现类：<br>(1) `RequestMappingHandlerMapping `，它支持 `@RequestMapping` 注解方法<br>(2) `SimpleUrlHandlerMapping`，它维护 URI 路径模式到 handler 的显式注册|
|`HandlerAdapter`|帮助 `DispatcherServlet` 调用映射到请求的 handler，不管 handler 实际是如何调用。例如，调用注解式控制器需要解析注解。`HandlerAdapter` 的主要目的是防止 `DispatcherServlet` 受此类细节的影响|
|`HandlerExceptionResolver`|解析异常的策略，可能会将它们映射到 handler，html 错误视图，或者其他目标|
|`ViewResolver`|从 handler 返回的基于字符串的逻辑视图名称解析为要呈现给响应的实际视图|
|`LocaleResolver`,[LocaleContextResolver](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-timezone)|解析客户端正在使用的 `Locale` 以及可能的时区，以便能提供国际化的视图。参见 [Locale](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-localeresolver)|
|`ThemeResolver`|解析你的 Web 应用程序可以使用的主题|
|`MultipartResolver`|在某些 multipart 解析库的帮助下，用于解析 multi-part 请求（例如，浏览器表单文件上传）。请参阅 [Multipart Resolver](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-multipart)|
|`FlashMapManager`||



#### [1.1.3. Web MVC Config](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-servlet-config)

应用程序可以声明一些在 [Special Bean Types](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-servlet-special-bean-types) 列出的基础架构的 Bean，需要这些 Bean 来处理请求。`DispatcherServlet` 会为每个特殊 Bean 检查 `WebApplicationContext`。如果没有匹配的 Bean 类型，则会回退到使用 `DispatcherServlet.properties` 列出的默认 bean。

大多数情况下，MVC Config 是最佳出发点。它以 Java 或者 XML 的方式声明了需要的 bean，并且提供了更高级配置回调 API 用于自定义。

> **作者的话** MVC Config 是官方术语，代表 Spring MVC 提供的默认配置，可以是注解式启用，也可以是 XML 方式启用，具体来说就是 `@EnableWebMvc` 和 `<mvc:annotation-driven/>`


> Spring Boot 依赖于 MVC Java Config 配置 Spring MVC，以及提供了许多额外的便利选项


#### [1.1.4. Servlet Config](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-container-config)

> **作者的话** 对于 Spring Boot 来说，`DispatcherServlet` 是自动配置的，而且 Servlet 容器也是在 `ApplicationContext` 进行 `refresh` 的过程中内嵌启动的，所以这一节可以简单了解

在 Servlet 3.0+ 环境下，你可以选择使用编程方式作为一种替代方案进行 Servlet 容器配置，或者你也可以与 `web.xml` 方式组合使用。如下是注册一个 `DispatcherServlet` 的例子：

```java
import org.springframework.web.WebApplicationInitializer;

public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext container) {
        XmlWebApplicationContext appContext = new XmlWebApplicationContext();
        appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");

        ServletRegistration.Dynamic registration = container.addServlet("dispatcher", new DispatcherServlet(appContext));
        registration.setLoadOnStartup(1);
        registration.addMapping("/");
    }
}
```

`WebApplicationInitializer` 是由 Spring MVC 提供的接口，可以确保检测到你的实现类并用于初始化任何 Servlet 3 容器。`WebApplicationInitializer` 的一个实现类（抽象基类）是 `AbstractDispatcherServletInitializer`，你可以通过覆盖父类方法指定 Servlet 映射以及 `DispatcherServlet` 配置的路径使得注册 `DispatcherServlet` 更加容易。

应用程序使用基于 Java 的 Spring 配置如下示例所示，这也是官方建议的方式：

```java
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return null;
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { MyWebConfig.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```

如果使用基于 XML 的 Spring 配置，则应该直接从 `AbstractDispatcherServletInitializer` 继承下来，如下示例所示：
```java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    @Override
    protected WebApplicationContext createRootApplicationContext() {
        return null;
    }

    @Override
    protected WebApplicationContext createServletApplicationContext() {
        XmlWebApplicationContext cxt = new XmlWebApplicationContext();
        cxt.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");
        return cxt;
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```

`AbstractDispatcherServletInitializer` 还提供了一种添加 `Filter` 实例，并将其自动映射到 `DispatcherServlet` 的简便方法，如下示例所示：



```java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    // ...

    @Override
    protected Filter[] getServletFilters() {
        return new Filter[] {
            new HiddenHttpMethodFilter(), new CharacterEncodingFilter() };
    }
}
```

添加的每个过滤器都会基于其具体类型有一个默认名字，并自动映射到 `DispatcherServlet`。

> **作者的话** 将 `Filter` 映射到 `DispatcherServlet` 的方式是通过 Servlet API `javax.servlet.FilterRegistration#addMappingForServletNames` 实现的


`AbstractDispatcherServletInitializer` 的 protected 方法 `isAsyncSupported` 提供了一个地方可以在 `DispatcherServlet` 上启用异步支持，并且所有过滤器都会映射到它上面。默认地，这个标志位是 `true`


最后，如果你需要进一步自定义 `DispatcherServlet` 本身，你可以直接覆盖 `createDispatcherServlet` 方法。


#### [1.1.5. Processing](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-servlet-sequence)

`DispatcherServlet` 处理过程如下：

- 搜索 `WebApplicationContext` 并将其绑定到 Request 的属性上，处理过程中控制器和其他元素可能会使用到。默认绑定到 `DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE` key 上。

- 将 locale resolver 绑定到 Request 的属性上，以便于当处理请求的过程中让其他元素解析本地化使用。如果你不需要本地化解析，你就不必使用 locale resolver。

- 将主题解析器绑定到请求，以使视图之类的元素确定用何种主题。如果你不使用主题，可以忽略它。

- 如果你指定了一个 multipart 文件解析器，将会检查请求的 multiparts。如果找到了 multiparts，请求会被包装成 `MultipartHttpServletRequest`，用于过程后期其他元素的处理。

- 搜索合适的 handler。如果找到了一个 handler，与该 handler 有关的执行链（前置处理器、后置处理器、控制器）将会运行，用于准备 model 的渲染。或者，对于注解式控制器，响应可以直接渲染（在 `HandlerAdapter`）而不是返回一个视图。

- 如果返回的是 model，视图就会被渲染。如果没有返回 model（可能由于前置处理器或者后置处理器拦截了请求，可能是因为安全问题），就不会渲染视图，因为请求已经完成了。


声明在 `WebApplicationContext` 的 `HandlerExceptionResolver` bean 用于解析请求处理过程中抛出的异常。允许自定义异常处理器的逻辑用于处理异常。



`DispatcherServlet` 还支持返回 `last-modification-date`，正如 Servlet API 指定的。确定特定请求的最后一次修改日期很简单：`DispatcherServlet` 寻找合适的 handler mapping，并检测被找到的 handler 是否实现了 `LastModified` 接口。如果是，接口 `LastModified` 的方法 `long getLastModified(request)` 的值就会返回给客户端。

> **作者的话** 注解式 Controller 想要实现最后一次修改日期需要做一些特殊操作，因为 `RequestMappingHandlerAdapter` 的 `getLastModified` 总是返回 -1，但源码方法注解也给了我们指引，需要在 handler 中调用 `WebRequest#checkNotModified(long)`，如果返回值是 true，则返回 `null`

你可以通过在 `web.xml` 文件中添加 Servlet 初始化参数到 Servlet 声明中来自定义一个自己 `DispatcherServlet`。以下列表列出了支持的参数：

|参数|解释|
|:-|:-|
|`contextClass`|实现了 `ConfigurableWebApplicationContext` 的类，将由此 Servlet 进行实例化与本地配置。默认地，使用 `XmlWebApplicationContext`。|
|`contextConfigLocation`|传递给上下文实例的字符串，标识从哪里寻找上下文。|
|`namespace`|`WebApplicationContext`的命名空间。默认是 `[servlet-name]-servlet`|
|`throwExceptionIfNoHandlerFound`|当找不到请求的 handler 时，是否抛出异常|

#### [1.1.6. Interception](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-handlermapping-interceptor)

所有的 `HandlerMapping` 实现类都支持 handler 拦截器，这很有用，比如你希望将某些特定的功能应用于某些请求，举个例子，检查 principal。

拦截器必须实现 `org.springframework.web.servlet.HandlerInterceptor` 接口，它提供了三个方法，应该可以足够灵活地执行各种前置处理以及后置处理操作。

- `preHandle(..)`: 在实际的 handler 运行之前
- `postHandle(..)`: 在实际的 handler 运行之后
- `afterCompletion(..)`: 在完整的请求结束之后


`preHandle(..)` 方法返回一个 boolean 值。你可以使用此方法 break 或者 continue 执行链的执行。当方法返回 `true`，handler 执行链就会继续；当方法返回 `false`，`DispatcherServlet` 会认为拦截器本身已经处理了请求（例如，渲染了恰当的视图），并且不会继续执行其他拦截器以及实际的 handler。


请注意，当具有 `@ResponseBody` 以及 `ResponseEntity` 的时候，`postHandle` 没什么用，因为在 `postHandle` 之前响应就会在 `HandlerAdapter` 中写入并提交了。这意味着，对响应进行任何更改为时已晚，例如添加额外的 header 都是无用的。对于这种情况，你可以实现 `ResponseBodyAdvice`，并将它声明为一个 `Controller Advice` bean，或者，你也可以直接在 `RequestMappingHandlerAdapter` 上直接配置。

#### [1.1.7. Exceptions](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-exceptionhandlers)

如果请求 mapping 期间或者请求 handler（如 @Controller）抛出异常，则 DispatcherServlet 将委托给 HandlerExceptionResolver 链解析异常，并提供可替代的处理，这是典型的错误响应。

> **作者的话** 其实就是 try catch 模式，catch 中处理异常

以下表格列出了可用的 `HandlerExceptionResolver` 实现类：

|`HandlerExceptionResolver`|描述|
|:-|:-|
|`SimpleMappingExceptionResolver`|异常类名于错误视图名之间的映射。|
|`DefaultHandlerExceptionResolver`|解析 Spring MVC 的异常，并将它们映射到 HTTP 状态码|
|`ResponseStatusExceptionResolver`|用 `@ResponseStatus` 注解解析异常，并根据注解中的值将其映射到 HTTP 状态码|
|`ExceptionHandlerExceptionResolver`|通过调用 `@Controller` 或者 `@ControllerAdvice` 中的 `@ExceptionHandler` 方法解析异常|

`HandlerExceptionResolver` 的约定规定它可以返回：

- 一个指向错误视图的 `ModelAndView`
- 如果在解析器处理了异常，则是一个空的 `ModelAndView`
- 如果异常仍然未解析，则返回 null，后面的解析器继续尝试，如果最后仍然未解析，可以冒泡到 Servlet 容器

MVC Config 自动地为默认的 Spring MVC 异常，`@ResponseStatus` 注解异常，以及 `@ExceptionHandler` 方法的支持声明了内置的解析器。你可以自定义或者替换它们。

##### Chain of Resolvers
可以通过 HandlerExceptionResolver 在 Spring 配置中声明多个 bean 并根据需要设置他们的 order 属性。order 越高，异常处理器位置越靠后（越晚处理）。


HandlerExceptionResolver 约定可以返回如下：

- ModelAndView 跳转到错误页面
- 如果在解析器里处理了异常，则可以返回一个空的 ModelAndView 
- 如果异常仍未处理，则返回 null，供后续解析器继续尝试，如果异常一直存在，允许冒泡到 Servlet 容器。

##### Container Error Page
如果没有任何 `HandlerExceptionResolver` 处理异常，那么就会传播出去，或者如果响应状态为错误状态（即 4xx, 5xx），那么 Servlet 容器会在 HTML 中渲染默认的错误页面。要自定义容器的默认错误页面，可以在 `web.xml` 声明一个错误页面映射，如下示例所示：

```xml
<error-page>
    <location>/error</location>
</error-page>
```

给定之前的例子，当异常向上冒泡或者响应具有一个错误状态，Servlet 容器会将错误派发到配置的 URL（例如，`/error`）。接着，由 `DispatcherServlet` 处理，可能将其映射到一个 `@Controller`，其实现可以返回一个带有 model 的错误视图名，或者返回一个 JSON 响应，如下示例所示：

```java
@RestController
public class ErrorController {

    @RequestMapping(path = "/error")
    public Map<String, Object> handle(HttpServletRequest request) {
        Map<String, Object> map = new HashMap<String, Object>();
        map.put("status", request.getAttribute("javax.servlet.error.status_code"));
        map.put("reason", request.getAttribute("javax.servlet.error.message"));
        return map;
    }
}
```

#### [1.1.11. Multipart Resolver](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-multipart)

`org.springframework.web.multipart.MultipartResolver` 是解析包含文件上传的 multipart 请求的策略接口。

- 其中一个实现是基于 Commons FileUpload 框架
- 另一个是基于 Servlet 3.0 multipart 请求解析



##### [Apache Commons FileUpload](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-multipart-resolver-commons)

为了使用 Apache 的 Commons FileUpload：

- 配置一个 `CommonsMultipartResolver`，名称任意，如：`multipartResolver`
- 添加 `commons-fileupload` 依赖


##### [Servlet 3.0](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-multipart-resolver-standard)



### [1.2. Filters](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#filters)


#### [1.2.1. Form Data](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#filters-http-put)

浏览器只能通过 HTTP GET 或者 HTTP POST 提交表单数据，但是，非浏览器的客户端（比如 Postman）还可以使用 HTTP PUT, PATCH, DELETE 等。Servlet API 规定：`ServletRequest.getParameter*()` 系列方法仅仅支持访问 HTTP POST 的表单字段。

`spring-web` 模块提供了 `FormContentFilter`，拦截 Content Type 为 `application/x-www-form-urlencoded` 的 HTTP PUT, PATCH, DELETE 请求，读取请求体的表单数据，然后将 `ServletRequest` 进行一层包装，接着就可以通过 `ServletRequest.getParameter*()` 系列方法直接访问表单数据了。


#### [1.2.2. Forwarded Headers](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#filters-forwarded-headers)

当请求通过代理（例如，负载均衡器）时，主机，端口号，以及 schema 可能会更改，这使得形成一个指向用户正确的主机，端口和 schema 的连接比较困难。

RFC 7239 定义了 `Forwarded` HTTP 头部，代理可以用它来提供关于原始请求的信息。还有一些其他的非标准头部，包括 `X-Forwarded-Host`, `X-Forwarded-Port`, `X-Forwarded-Proto`, `X-Forwared-Ssl`, 以及 `X-Forwarded-Prefix`。

`ForwardedHeaderFilter` 是一个 Servlet 过滤器，它可以修改请求：a) 基于`Forwarded` 头部的主机，端口号，以及 schema；b) 删除这些头部，防止影响后期。该过滤器依赖于包装请求，因此，必须排在其他过滤器（如 `RequestContextFilter`）之前，它应该与修改后的，而不是原始请求一起使用。

由于应用程序并不知道 header 是否由代理（这是预期的），还是由恶意客户端添加的，因此有必要对于 `Forwared` 头部做一些安全考虑。这就是为什么在信任边界上（最外层）的代理应该做一层配置，去删除那些从外部传来的不信任的 `Forwarded` 头部。你还可以给 `ForwardedHeaderFilter` 配置 `removeOnly=true`，这种情况下，它会删除不使用这些头部。


#### [1.2.4. CORS](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#filters-cors)


Spring MVC 通过控制器的注解，为 CORS 配置提供了细粒度的支持。但是，当你和 Spring Security 一起使用时，建议依赖于内置的 `CorsFilter`，它必须排在 Spring Security 的过滤器链之前。


### [1.3. Annotated Controllers](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-controller)

Spring MVC 提供了一个基于注解的编程模型，其中 `@Controller` 和 `@RestController` 组件用注解来表示请求的映射、请求的输入、异常的处理等。

被注解的控制器具有灵活的方法签名（即方法名、参数个性化定制），不必扩展基类也不必实现特定接口。

以下例子展示了一个由注解定义的控制器：

```java
@Controller
public class HelloController {

    @GetMapping("/hello")
    public String handle(Model model) {
        model.addAttribute("message", "Hello World!");
        return "index";
    }
}
```

在上面的例子中，该方法接受 `Model` 并将视图名称以 `String` 形式返回。

#### [1.3.1. Declaration](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-ann-controller)

你可以在 Servlet 的 `WebApplicationContext` 中使用标准的 Spring Bean definition 来定义控制器 bean。`@Controller` 注解允许自动检测，这与 Spring 通用支持检测类路径中的 `@Component` 类并自动注册 bean definition 是一致的。`@Controller` 还充当注解类的刻板印象，表示其作为 Web 组件。

> 刻板印象是译词，表示该注解具有特定语义。


#### [1.3.2. Request Mapping](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-ann-requestmapping)

你可以使用 `@RequestMapping` 注解将请求映射到控制器方法。该注解有各种属性去映射，如：URL、HTTP 方法、请求参数、请求头、媒体类型。你可以在类级别使用它来表示共享映射或者在特定方法级别上使用来缩小到特定的端点映射。

还有一些 `@RequestMapping` 的特定快捷变体：

- `@GetMapping`
- `@PostMapping`
- `@PutMapping`
- `@DeleteMapping`
- `@PatchMapping`

快捷变体是自定义注解，因为，大多数控制器方法应该映射到特定的 HTTP 方法，而不是使用 `@RequestMapping`。在类级别，仍然需要 `@RequestMapping` 来表示共享映射。

> 默认地，`@RequestMapping` 与所有 HTTP 方法适配。


##### URI patterns

|<div style="width: 120px">Pattern</div>|Description|Example|
|-|-|-|
|`?`|匹配一个字符|`"/pages/t?st.html"`<br>匹配 `"/pages/test.html"` <br> `"/pages/t3st.html"`|
|`*`|匹配零个或多个字符|`"/resources/*.png"`<br>匹配 `"/resources/file.png"`<br><br>`"/projects/*/versions"`<br>匹配 `"/projects/spring/versions"`，<br>但不匹配 `"/projects/spring/boot/versions"`|
|`**`|匹配零个或多个路径段，直到路径结束|`"/resources/**"`<br>匹配 `"/resources"` <br> `"/resources/file.png"` <br> `"/resources/images/file.png"`|
|`{name}`|匹配一条路径段，并将其捕获为名为 "name" 的变量|`"/projects/{project}/versions"`<br>匹配 `"/projects/spring/versions"`，并捕获 `project=spring`|
|`{name:[a-z]+}`|匹配正则表达式 `"[a-z]+"` 作为名为 "name" 的路径变量|`"/projects/{project:[a-z]+}/versions"`<br>匹配 `"/projects/spring/versions"`，<br>但不匹配 `"/projects/spring1/versions"`|


#### [1.3.3. Handler Methods](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-methods)

`@RequestMapping` 处理器方法具有灵活的签名，可以从一系列支持的控制器方法参数和返回值中进行选择。

##### [Method Arguments](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-arguments)

|Controller method argument|Description|
|-|-|
|`WebRequest`,`NativeWebRequest`|无需直接使用 Servlet API，对请求参数、请求属性、会话属性的通用访问|
|`javax.servlet.ServletRequest`, `javax.servlet.ServletResponse`|选择任何特定的请求或相应类型，例如，`ServletRequest`, `HttpServletRequest`, 或者 Spring 的 `MultipartRequest`, `MultipartHttpServletRequest`|
|`javax.servlet.http.HttpSession`|强制 Session 必须存在。因此，该参数不可能是 null。注意，Session 访问不是线程安全的。如果允许多个请求可以同时访问会话，考虑设置 `RequestMappingHandlerAdapter` 实例的 `synchronizeOnSession` 标志位为 tue|
|`javax.servlet.http.PushBuilder`||
|`java.security.Principal`||
|`HttpMethod`|请求的 HTTP 方法|
|`java.util.Locale`|当前请求的语言环境，由可用的最准确的 `LocaleResolver` 决定（配置的 `LocaleResolver` 或者 `LocaleContextResolver`|
|`java.util.TimeZone` + `java.time.ZoneId`||
|`java.io.InputStream`,`java.io.Reader`|通过 Servlet API 访问暴露的原始请求体|
|`java.io.OutputStream`,`java.io.Writer`|通过 Servlet API 访问暴露的原始响应体|
|`@PathVariable`|用于访问 URI 模板变量|
|`@MatrixVariable`||
|`@RequestParam`|用于访问 Servlet 请求参数，包括 multipart 文件|
|`@RequestHeader`|用于访问请求头|
|`@CookieValue`|用于访问 cookie|
|`@RequestBody`|用于访问 HTTP 请求体|
|`HttpEntity<B>`||
|`@RequestPart`|用于访问 `multipart/form-data` 请求中的 part|
|`java.util.Map`, `org.springframework.ui.Model`, `org.springframework.ui.ModelMap`||
|`RedirectAttributes`||
|`@ModelAttribute`||
|`Errors`,`BindingResult`||
|`SessionStatus` + class-level `@SessionAttributes`||
|`UriComponentsBuilder`||
|`@SessionAttribute`||
|`@RequestAttribute`||
|Any other argument||


##### [Return Values](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-return-types)

|Controller method return value|Description|
|-|-|
|`@ResponseBody`|返回值通过 `HttpMessageConverter` 实现类进行转换，并写入响应中|
|`HttpEntity<B>`,`ResponseEntity<B>`|指定完整的响应（包括 HTTP 响应头以及响应体），将通过 `HttpMessageConverter` 实现类转换，并写入响应|
|`HttpHeaders`|用于返回一个只有响应头，没有响应体的 response|
|`String`|视图名...|
|`View`||

##### [Type Conversion](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-typeconversion)
如果 Controller 方法参数声明为 String 以外的其他类型，则某些带有 `@RequestParam`、`@RequestHeader`、`@PathVariable` 等注解的参数可能需要类型转换。

对于上述情况，将根据配置的转换器自动进行类型转换。默认地，支持一些简单的类型，如 `int`, `long`, `Date` 等。也可由通过 `WebDataBinder` 或者通过使用 `FormattingConversionService` 注册 `Formatters` 来自定义类型转换。


##### [@ModelAttribute](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-modelattrib-method-args)

在方法参数上使用 @ModelAttribute 注解来访问 model 中的属性。model 属性还涵盖来自 HTTP Servlet 请求参数的值，其名称与字段名称匹配，这称为数据绑定，让你不必处理解析和转化单个 query 参数以及 form 字段。以下例子展示了如何做：

```java
@PostMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute Pet pet) {
    // method logic...
}
```

> **官方提示** `@ModelAttribute` 是可选的。默认地，任何非简单类型（由 `BeanUtils#isSimpleProperty` 决定），且未被任何其他参数解析器解析的参数，都会被当作使用了 `@ModelAttribute` 注解进行解析。<br>
>  这是因为 `HandlerMethodArgumentResolverComposite#argumentResolvers`（这是一个 List） 包含了两个 `ServletModelAttributeMethodProcessor`，一个在中间，一个在最后，不同点在于属性 `annotationNotRequired`，位于中间的 `annotationNotRequired=false`，即必须有 `@ModelAttribute` 注解；位于最后的 `annotationNotRequired=true`，即没有被 `@ModelAttribute` 注解，所以，在搜索 Processor 进行处理的过程中，如果一直没有参数解析器可以解析，就会使用最后的 `ServletModelAttributeMethodProcessor` 进行解析。

**注意点：**
(1) 对简单参数不进行任何注解，可以认为与 `@RequestParam(required = false)` 等效，而且如果只是添加了 `@RequestParam` 注解，缺乏该参数反而会报错。

(2) @ModelAttribute 可以注解在方法上，先于 Controller 方法执行。

##### [@SessionAttributes](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-sessionattributes)

`@SessionAttributes` 用于在请求之间的 HTTP Servlet 会话中存储 model 属性。

这是一个类级别注解，该注解声明指定控制器使用的会话属性。

通常列出了应该透明地存储于 session 中的 model 属性的名称或者 mode 属性的类型，以供后续请求访问。、

以下示例使用 `@SessionAttributes` 注解：
```java
@Controller
@SessionAttributes("pet")   
public class EditPetForm {
    // ...
}
```
在第一次请求中，当具有名称 `pet` 的 model 属性添加到 Model 时，它会自动升级并保存到 HTTP Servlet Session 中。它会一直存储在那里，直到另一个控制器方法使用 `SessionStatus` 方法参数去清除存储，如下示例所示：

```java
@Controller
@SessionAttributes("pet") 
public class EditPetForm {

    // ...

    @PostMapping("/pets/{id}")
    public String handle(Pet pet, BindingResult errors, SessionStatus status) {
        if (errors.hasErrors) {
            // ...
        }
        status.setComplete(); 
        // ...
    }
}
```

##### [@SessionAttribute](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-sessionattribute)

如果你需要访问全局管理的预先存在的 session 属性（即，控制器之外，例如，通过过滤器产生），可能存在也可能不存在，你可以在方法参数上使用 `@SessionAttribute` 注解，正如下面的示例所示：

```java
@RequestMapping("/")
public String handle(@SessionAttribute User user) { 
    // ...
}
```

对于需要添加或者删除会话属性的案例，请考虑注入 `org.springframework.web.context.request.WebRequest` 或者 `javax.servlet.http.HttpSession` 到控制器方法。

对于要作为控制器工作流程的一部分在会话中临时存储 model 属性，请考虑使用 `@SessionAttributes`。

##### [@RequestAttribute](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-ann-requestattrib)
类似于 `@SessionAttribute`，你可以使用 `@RequestAttribute` 注解去访问事先已经存在的 Request 属性（例如，通过 `Filter` 或者 `HandlerInterceptor` 注入的属性）：

```java
@GetMapping("/")
public String handle(@RequestAttribute Client client) { 
    // ...
}
```

##### [Multipart](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-multipart-forms)

启用了 `MultipartResolver` 之后，使用 `multipart/form-data` 的 POST 请求内容会被解析并且可以作为一般的请求参数访问。

> Spring Boot 会通过 `MultipartAutoConfiguration` 自动注入 `MultipartResolver`

下面的示例访问了一个一般的表单字段以及一个上传的文件：

```java
@Controller
public class FileUploadController {

    @PostMapping("/form")
    public String handleFormUpload(@RequestParam("name") String name,
            @RequestParam("file") MultipartFile file) {

        if (!file.isEmpty()) {
            byte[] bytes = file.getBytes();
            // store the bytes somewhere
            return "redirect:uploadSuccess";
        }
        return "redirect:uploadFailure";
    }
}
```

将参数类型声明为 `List<MultipartFile>` 允许解析相同参数名的多个文件。

当 `@RequestParam` 没有指定注解参数名，且被声明为 `Map<String, MultipartFile>` 或者 `MultiValueMap<String, MultipartFile>`，，那么对于每个给定参数名的 multipart 文件都会填充到 map 中。

> **作者的话** 这种用法请勿设置参数名，否则会发生转化错误；而且这种用法无法收集多个相同参数名的 multipart 文件。


> **官方** 如果使用了 Servlet 3.0 multipart 解析，你还可以声明 `javax.servlet.http.Part` 作为方法参数或者集合的 Value 类型，以取代 Spring 的 `MultipartFile`

你还可以将 multipart 内容作为绑定到命令对象的数据的一部分。例如，前面示例中的表单字段和文件可以是表单对象上的字段，如下示例所示：

> **作者的话** 命令对象：command object，可以理解为复合对象

```java
class MyForm {

    private String name;

    private MultipartFile file;

    // ...
}

@Controller
public class FileUploadController {

    @PostMapping("/form")
    public String handleFormUpload(MyForm form, BindingResult errors) {
        if (!form.getFile().isEmpty()) {
            byte[] bytes = form.getFile().getBytes();
            // store the bytes somewhere
            return "redirect:uploadSuccess";
        }
        return "redirect:uploadFailure";
    }
}
```

在 RESTful 服务方案中，也可以从非浏览器的客户端提交 multipart 请求。如下示例展示了一个带有 JSON 的文件：

```text
POST /someUrl
Content-Type: multipart/mixed

--edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
Content-Disposition: form-data; name="meta-data"
Content-Type: application/json; charset=UTF-8
Content-Transfer-Encoding: 8bit

{
    "name": "value"
}
--edt7Tfrdusa7r3lNQc79vXuhIIMlatb7PQg7Vp
Content-Disposition: form-data; name="file-data"; filename="file.properties"
Content-Type: text/xml
Content-Transfer-Encoding: 8bit
... File Data ...
```

你可以用 `@RequestParam` 以 `String`  的方式访问 "meta-data" 部分，但是有可能你想将他从 JSON 反序列化（类似于 `@RequestBody`）。使用 `@RequestPart` 注解可以访问被 `HttpMessageConverter` 转换之后的 multipart：

```java
@PostMapping("/")
public String handle(@RequestPart("meta-data") MetaData metadata,
        @RequestPart("file-data") MultipartFile file) {
    // ...
}
```

> **作者的话** 这种情况下需要特殊设置某个文件的 Content-Type 为 application/json


#### [1.3.4. Model](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-ann-modelattrib-methods)

你可以使用 `@ModelAttribute` 注解：

- `@RequestMapping` 方法上的一个方法参数上，用于创建或者访问一个来自 model 的 `Object`，并通过 `WebDataBinder` 将它绑定到请求

- 作为 `@Controller` 或者 `@ControllerAdvice` 类中方法级注解，帮助在任何 `@RequestMapping` 方法调用之前初始化 model

- 用于 `@RequestMapping` 方法，标记其返回值是 model 属性


本节讨论 `@ModelAttribute` 方法，前面列表中的第 2 项。一个控制器可以有任意数量的 `@ModelAttribute` 方法。在同一个控制器的 `@RequestMapping` 方法之前，所有此类的方法都会被调用。`@ModelAttribute` 方法也可以通过 `@ControllerAdvice` 在控制器之间共享。详见...

`@ModelAttribute` 具有灵活的方法签名。他们支持许多与 `@RequestMapping` 方法相同的参数，除了 `@ModelAttribute` 本身，或者与请求体相关的任何内容。

以下示例展示了 `@ModelAttribute` 方法：

```java
@ModelAttribute
public void populateModel(@RequestParam String number, Model model) {
    model.addAttribute(accountRepository.findAccount(number));
    // add more ...
}
```

以下示例添加了仅仅 1 个属性：

```java
@ModelAttribute
public Account addAccount(@RequestParam String number) {
    return accountRepository.findAccount(number);
}
```

你看也可以在 `@RequestMapping` 方法上使用 `@ModelAttribute` 作为方法级注解，在这种情况下，`@RequestMapping` 方法的返回值会被解释为 model 属性。这通常不需要，因为这是 HTML 控制器的默认行为，除非返回值是 `String`，会被解释成视图名字。`@ModelAttribute` 也可以自定义 model 属性名，如下示例所示：
```java
@GetMapping("/accounts/{id}")
@ModelAttribute("myAccount")
public Account handle() {
    // ...
    return account;
}
```

#### [1.3.5. DataBinder](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-ann-initbinder)

`@Controller` 和 `@ControllerAdvice` 类可以具有一些 `@InitBinder` 方法，用来初始化 `WebDataBinder` 实例，而这些方法又可以：

- 绑定请求参数（即，表单或者查询数据）到 model 对象
- 将基于字符串的请求值（例如请求参数，路径变量，头，cookie等）转换为控制器方法参数的目标类型
- 当渲染 HTML 形式时，将 model 对象值以 `String` 值格式化


`@InitBinder` 方法可以注册特定于控制器的 `java.bean.PropertyEditor` 或者 Spring `Converter` 以及 `Formatter` 组件。此外，你也可以使用 MVC Config 在全局共享的 `FormattingConversionService` 注册 `Converter` 以及 `Formatter`。

`@InitBinder` 方法支持许多跟 `@RequestMapping` 一样支持的参数，除了 `@ModelAttribute` （command object）参数。通常，这些方法以一个 `WebDataBinder` 参数（用于注册）声明，并返回一个 `void`。下面展示了一个例子：

```java
@Controller
public class FormController {

    @InitBinder 
    public void initBinder(WebDataBinder binder) {
        SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");
        dateFormat.setLenient(false);
        binder.registerCustomEditor(Date.class, new CustomDateEditor(dateFormat, false));
    }

    // ...
}
```

另外，当你通过共享的 `FormattingConversionService` 使用基于 `Formatter` 的设置时，你可以再使用相同的方式注册特定于控制器的 `Formatter` 实现，如下示例所示：

```java
@Controller
public class FormController {

    @InitBinder 
    protected void initBinder(WebDataBinder binder) {
        binder.addCustomFormatter(new DateFormatter("yyyy-MM-dd"));
    }

    // ...
}
```
#### [1.3.6. Exceptions](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-ann-exceptionhandler)

`@Controller` 和 `@ControllerAdvice` 类可以拥有 `@ExceptionHandler` 方法来处理来自控制器方法的异常，如下示例所示：

```java
@Controller
public class SimpleController {

    // ...

    @ExceptionHandler
    public ResponseEntity<String> handle(IOException ex) {
        // ...
    }
}
```

上述例子中，异常可能与顶级异常（top-level）相匹配（即，直接抛出的 `IOException`），或者与在顶级包装器异常的直接 cause 相匹配（例如，`IOException` 被包裹在 `IllegalStateException`）。

> **作者的话** 顶级异常（top-level）可以理解为最外层异常，通常是一个模糊异常，并不具体，另外异常有一个 cause，通常表示具体异常，这里也可以匹配到进行处理。

对于匹配的异常类型，最好将目标异常声明为方法参数，如前面的示例所示。当多个异常方法匹配时，根异常匹配通常比 cause 异常匹配优先级更高。更具体地，使用 `ExceptionDepthComparator` 基于异常类型的深度来对异常进行排序。

另外，注解声明可能会缩小匹配的异常类型范围，如下示例所示：

```java
@ExceptionHandler({FileSystemException.class, RemoteException.class})
public ResponseEntity<String> handle(IOException ex) {
    // ...
}
```

你甚至可以使用特定的异常类型数组，以及一个非常通用的参数签名，如以下示例所示：

```java
@ExceptionHandler({FileSystemException.class, RemoteException.class})
public ResponseEntity<String> handle(Exception ex) {
    // ...
}
```

> 根异常匹配和 cause 异常匹配可能令人觉得很惊讶。
> 在前面展示的 `IOException` 变量中，该方法通常以实际的 `FileSystemException` 或者 `RemoteException` 实例作为参数，因为它们两个都从 `IOException` 继承。但是，如果在包装器异常中的任何这样的相匹配的异常被传播出来，它们本身也是一个 `IOException`，则传递的异常实例是包装器异常。
> **作者的话** 也就是需要通过 `ex.getCause()` 才能拿到具体异常
> 这种行为在 `handle(Exception)` 变量中更简单。在包装方案下，总是以包装器异常调用异常处理方法，在这种情况下，通过 `ex.getCause()` 找到实际匹配的异常。仅当作为顶级异常抛出的时候，传递的异常才是实际的 `FileSystemException` 或者 `RemoteException`。
> **作者的话** 不过，你调用 `ex.getCause()` 拿到的也是具体异常，因为 `this.cause=this`

我们通常建议你在参数签名中尽可能具体，可以降低 root 异常类型和 cause 异常类型之间不匹配的可能性。考虑将多匹配方法分解为单个 `@ExceptionHandler` 方法，每个方法都通过其签名匹配单个特定异常类型。

> **作者的话** 官方就是建议你将每个具体的异常都写一个 `@ExceptionHandler`

在多个 `@ControllerAdvice` 安排下，建议你在 `@ControllerAdvice` 上声明你的主要 root 异常映射，该映射使用相关的 order 优先使用。虽然 root 异常匹配优先于 cause，但是这是在给定的 `@Controller` 或者 `@ControllerAdvice` 类方法中定义。这意味着，在较高优先级 `@ControllerAdvice` bean 上的 cause 匹配优先于任何较低级的 `@ControllerAdvice` bean。

##### [Method Arguments](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-ann-exceptionhandler-args)

`@ExceptionHandler` 方法支持以下参数：

|方法参数|描述|
|-|-|
|||

##### [Return Values](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-ann-exceptionhandler-return-values)



#### [1.3.7. Controller Advice](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc-ann-controller-advice)

通常，`@ExceptionHandler`，`@InitBinder`，以及 `@ModelAttribute` 方法应用于 `@Controller` 类（或者类继承层次），将这些注解声明在里面。如果你希望这样的方法在全局范围内更多地应用（跨控制器），则可以在 `@ControllerAdvice` 或者 `@RestControllerAdvice` 注解类中声明它们。

`@ControllerAdvice` 使用 `@Component` 注解，这意味着这样的类可以通过组件扫描（component scanning）注册为 Spring bean。`@RestControllerAdvice` 是一个由 `@ControllerAdvice` 和 `@ResponseBody` 注解的组合注解，本质上意味着，`@ExceptionHandler` 方法是通过消息转换（以及视图解析或者模板解析）渲染到响应体的。

在启动时，用于 `@RequestMapping` 和 `@ExceptionHandler` 方法的基础设施类会检测注解有 `@ControllerAdvice` 的 Spring bean，然后在运行时应用它们的方法。 全局的 `@ExceptionHandler` 方法（来自 `@ControllerAdvice`）应用于本地方法之后（来自 `@Controller`）。相比之下，全局的 `@ModelAttribute` 和 `@InitBinder` 方法应用于本地方法之前。

默认地，`ControllerAdvice` 方法应用于每个请求（即，所有控制器），但是你可以通过注解上的属性，将其缩小到控制器的子集，如以下示例所示：

```java
// Target all Controllers annotated with @RestController
@ControllerAdvice(annotations = RestController.class)
public class ExampleAdvice1 {}

// Target all Controllers within specific packages
@ControllerAdvice("org.example.controllers")
public class ExampleAdvice2 {}

// Target all Controllers assignable to specific classes
@ControllerAdvice(assignableTypes = {ControllerInterface.class, AbstractController.class})
public class ExampleAdvice3 {}
```


全局异常处理 @RestControllerAdvice
```java
@RestControllerAdvice
public class GlobalExceptionHandler {
	@ExceptionHandler(value = AuthenticationException.class)
	public ResponseDTO AuthenticationExceptionHandler(AuthenticationException e) {
		return ResponseDTO.failure(e.getMessage());	
	}
}
```

*** ControllerAdvice 处理 Filter 抛出的异常

类比，Sturts 拦截器组，第一个就是 exception 拦截器。本质上是借助过滤器栈，将异常处理的过滤器放在第一个位置。

定义 ExceptionFilter，将捕捉的异常交给异常处理的 Controller。其他的过滤器不用处理异常，直接 throw 即可。
请务必调用 setOrder 方法，保持 order 值最大，这样过滤就能排在第一个。
```java
@Component
@Order(Integer.MIN_VALUE)
public class ExceptionFilter extends HttpFilter {
    @Override
    protected void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
        try {
            chain.doFilter(request, response);
        } catch (IOException e) {
            request.setAttribute("ioException", e);
            request.getRequestDispatcher("/error").forward(request, response);
        } catch (ServletException e) {
            request.setAttribute("servletException", e);
            request.getRequestDispatcher("/error").forward(request, response);
        }
    }
}
```

定义抛出异常的 Mapping
```java
@PostMapping("/error")
public ResponseVO throwException(HttpServletRequest request) throws Exception {
    throw (Exception) request.getAttribute("exception");
}
```


### [1.4. Functional Endpoints](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#webmvc-fn)


#### [1.4.1. Overview](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#webmvc-fn-overview)

在 WebMvc.fn 中，使用 `HandlerFunction` 处理 HTTP 请求：这是一个接受 `ServerRequest` 并返回 `ServerResponse` 的函数。请求和响应都有固定的约定，提供了 JDK8 友好的 HTTP 请求和响应访问。`HandlerFunction` 等价于基于注解的编程模型中的 `@RequestMapping` 方法体。

传入的请求会使用 `RouterFunction` 路由到一个 handler function：这是一个接收 `ServerRequest` 并返回 一个 Optional 包裹的 `HandlerFunction`（`Optional<HandlerFunction>`） 的函数。当路由器函数匹配到时，就会返回一个 handler function；否则是一个空的 Optional。`RouterFunction` 等价于 `@RequestMapping` 注解，但有一个主要的区别，路由函数不仅仅提供数据，还提供行为。

`RouterFunctions.route()` 提供了一个路由器构建器，可以比较容易地创建路由器，如以下示例所示：

```java
import static org.springframework.http.MediaType.APPLICATION_JSON;
import static org.springframework.web.servlet.function.RequestPredicates.*;
import static org.springframework.web.servlet.function.RouterFunctions.route;

PersonRepository repository = ...
PersonHandler handler = new PersonHandler(repository);

RouterFunction<ServerResponse> route = route()
    .GET("/person/{id}", accept(APPLICATION_JSON), handler::getPerson)
    .GET("/person", accept(APPLICATION_JSON), handler::listPeople)
    .POST("/person", handler::createPerson)
    .build();


public class PersonHandler {

    // ...

    public ServerResponse listPeople(ServerRequest request) {
        // ...
    }

    public ServerResponse createPerson(ServerRequest request) {
        // ...
    }

    public ServerResponse getPerson(ServerRequest request) {
        // ...
    }
}
```

> **作者的话** 上述代码不符合 Java 语法规范，只是作为一种示例参考，具体注入路由器需要使用 bean 注入

如果将 `RouterFunction` 注册为 bean，例如在 `@Configuration` 类中暴露它，它会被 servlet 自动检测，如 Running a Server 所述。

#### [1.4.2. HandlerFunction](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#webmvc-fn-handler-functions)

`ServerRequest` 和 `ServerResponse` 是不可变接口，可提供 JDK8 友好访问 HTTP 请求和响应，包括 headers，body，method，status code。

##### [ServerRequest](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#webmvc-fn-request)

`ServerRequest` 提供了对 HTTP method，URI，Headers，Query Parameters，而通过 `body` 方法访问 Body。

以下示例将请求体提取为一个 `String`：

```java
String string = request.body(String.class);
```


以下示例将 Body 提取为一个 `List<Person>`，其中 `Person` 对象是从 JSON 或者 XML 此类的序列化形式解吗得到的：

```java
List<Person> people = request.body(new ParameterizedTypeReference<List<Person>>() {});
```

以下示例展示了如何访问参数：

```java
MultiValueMap<String, String> params = request.params();
```

##### [ServerResponse](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#webmvc-fn-response)

`ServerResponse` 提供了对 HTTP response 的访问，并且由于它是不变的，因此你可以使用一个 `build` 方法来创建它。你可以通过构建器设置响应状态，添加响应头，或者提供响应体。以下示例创建了一个具有 JSON 内容的 200（OK）响应：

```java
Person person = ...
ServerResponse.ok().contentType(MediaType.APPLICATION_JSON).body(person);
```

以下示例展示了如何使用 `Location` 头以及无响应体构建一个 201（CREATED）响应：
```java
URI location = ...
ServerResponse.created(location).build();
```

##### [Handler Classes](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#webmvc-fn-handler-classes)

我可以用 lambda 方式书写一个 handler 函数，如以下示例所示：

```java
HandlerFunction<ServerResponse> helloWorld =
  request -> ServerResponse.ok().body("Hello World");
```

这很方便，但是在应用程序中我们需要多个函数，并且多个内联的 lambda 可能变得混乱。因此 ，将相关的 handler 函数一起分组到一个 handler 类是很有用的，这与基于注解的应用中，具有跟 `@Controller` 相似的角色。举个例子，下面的类暴露了响应式 `Person` 仓库：

```java
import static org.springframework.http.MediaType.APPLICATION_JSON;
import static org.springframework.web.reactive.function.server.ServerResponse.ok;

public class PersonHandler {

    private final PersonRepository repository;

    public PersonHandler(PersonRepository repository) {
        this.repository = repository;
    }

    public ServerResponse listPeople(ServerRequest request) { 
        List<Person> people = repository.allPeople();
        return ok().contentType(APPLICATION_JSON).body(people);
    }

    public ServerResponse createPerson(ServerRequest request) throws Exception { 
        Person person = request.body(Person.class);
        repository.savePerson(person);
        return ok().build();
    }

    public ServerResponse getPerson(ServerRequest request) { 
        int personId = Integer.parseInt(request.pathVariable("id"));
        Person person = repository.getPerson(personId);
        if (person != null) {
            return ok().contentType(APPLICATION_JSON).body(person);
        }
        else {
            return ServerResponse.notFound().build();
        }
    }

}
```

#### [1.4.3. RouterFunction](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#webmvc-fn-router-functions)

路由器函数用于将请求路由到相关的 `HandlerFunction`。通常你不用自己编写路由器函数，而是使用 `RouterFunctions` 工具类上的一个方法创建它。`RouterFunctions.route()` （无参）为你提供了一个流利的构建器用于创建路由器函数，而 `RouterFunctions.route(RequestPredicate, HandlerFunction)` 提供了一个直接创建路由器的方法。

#### [1.4.4. Running a Server](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#webmvc-fn-running)

通常，你可以通过 MVC Config 在基于 `DispatcherHandler` 的设置中运行路由函数，这种设置使用 Spring Configuration 声明处理请求所需的组件。MVC Java 配置声明了以下的一些基础设施组件用以支持功能性端点：

- RouterFunctionMapping
- HandlerFunctionAdapter

前面的组件使功能性端点兼容到 `DispatcherServlet` 请求处理生命周期里，并且（可能）与注解控制器并行，如果有任何声明。这也是 Spring Boot Web starter 启用功能性端点的方式。

以下示例展示了 WebFlux Java 配置：

```java
@Configuration
@EnableMvc
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public RouterFunction<?> routerFunctionA() {
        // ...
    }

    @Bean
    public RouterFunction<?> routerFunctionB() {
        // ...
    }

    // ...

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        // configure message conversion...
    }

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        // configure CORS...
    }

    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        // configure view resolution for HTML rendering...
    }
}
```

#### [1.4.5. Filtering Handler Functions](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#webmvc-fn-handler-filter-function)

你可以在路由函数构建器使用 `before`，`after` 或者 `filter` 方法来过滤 handler 函数。使用注解，你可以通过使用 `@ControllerAdvice`，`ServletFilter` 或者两者都用，实现类似的功能。过滤将会应用到构建器构建的路由中。这意味着在嵌套路由中定义的过滤其不适用于顶级路由。例如，考虑如下示例：

```java
RouterFunction<ServerResponse> route = route()
    .path("/person", b1 -> b1
        .nest(accept(APPLICATION_JSON), b2 -> b2
            .GET("/{id}", handler::getPerson)
            .GET("", handler::listPeople)
            .before(request -> ServerRequest.from(request) 
                .header("X-RequestHeader", "Value")
                .build()))
        .POST("/person", handler::createPerson))
    .after((request, response) -> logResponse(response)) 
    .build();
```

### [1.5. URI Links](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-uri-building)

#### [1.5.1. UriComponents](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#web-uricomponents)

`UriComponentBuilder` 有助于从带有变量的 URI 模板中构建 URI。如下示例所示：

```java
UriComponents uriComponents = UriComponentsBuilder
        .fromUriString("https://example.com/hotels/{hotel}")  
        .queryParam("q", "{q}")  
        .encode() 
        .build(); 

URI uri = uriComponents.expand("Westin", "123").toUri();  
```

> 前面的示例可以合并到一个链中，并使用 `buildAndExpand` 缩短，如下示例所示：

```java
URI uri = UriComponentsBuilder
        .fromUriString("https://example.com/hotels/{hotel}")
        .queryParam("q", "{q}")
        .encode()
        .buildAndExpand("Westin", "123")
        .toUri();
```

你可以直接进入 URI（表示编码）进一步缩短，如下示例所示：

```java
URI uri = UriComponentsBuilder
        .fromUriString("https://example.com/hotels/{hotel}")
        .queryParam("q", "{q}")
        .build("Westin", "123");
```
你可以使用完整的 URI 模板进一步缩短，如下示例所示：
```java
URI uri = UriComponentsBuilder
        .fromUriString("https://example.com/hotels/{hotel}?q={q}")
        .build("Westin", "123");
```

#### [1.5.2. UriBuilder](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#web-uribuilder)

### [1.6. Asynchronous Requests](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-ann-async)

Spring MVC 集成了 Servlet 3.0 异步请求处理：

- 在控制器方法中，`DeferredResult` 和 `Callable` 返回值，并为单个异步返回值提供了基本支持。
- 控制器可以流传输多个值，包括 SSE 和 原始数据
- 控制器可以使用 reactive 客户端，并为响应处理返回 reactive types


#### [1.6.1. DeferredResult](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-ann-async-deferredresult)

一旦在 Servlet 容器中启用了异步处理功能，控制器方法就可以用 `DeferredResult` 包装任何支持的控制器方法的返回值，如下示例所示：

```java
@GetMapping("/quotes")
@ResponseBody
public DeferredResult<String> quotes() {
    DeferredResult<String> deferredResult = new DeferredResult<String>();
    // Save the deferredResult somewhere..
    return deferredResult;
}

// From some other thread...
deferredResult.setResult(result);
```


#### [1.6.3. Processing](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-ann-async-processing)

这是 Servlet 异步请求处理非常简洁的概述：

- 通过调用 `request.startAsync()`，`ServletRequest` 可以放到异步模型中。这样做的主要影响就是，Servlet（以及任何过滤器）可以退出，但是响应会一直打开用于后面处理完成。

- 调用 `request.startAsync()` 会返回 `AsyncContext`，你可以用它来对异步处理做进一步控制。例如，它提供了 `dispatch` 方法，该方法类似于 Servlet API 的 forward，除了它可以让应用程序在 Servlet 容器线程上恢复请求处理。

- `ServletRequest` 提供了对当前 `DispatcherType` 的访问，你可以用它区别处理初始化请求，异步调度，forward，以及其他调度类型。

`DeferredResult` 处理工作如下：

- 控制器返回一个 `DeferredResult`，并将其保存到一些内存中的可访问的队列或者列表中。

- Spring MVC 调用 `request.startAsync()`

- 同时，`DispatcherServlet` 以及所有配置的过滤器都退出请求处理线程，但 response 仍然保持打开

- 应用程序从某个线程设置 `DeferredResult`，然后 Spring MVC 将请求调度回 Servlet 容器。

- 再次调用 `DispatcherServlet`，并以异步产生的返回值进行处理。


### [1.7. CORS](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-cors)
参考链接：

https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS

#### [1.7.1. Introduction](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-cors-intro)

出于安全原因，浏览器禁止 AJAX 调用当前源之外的资源。举个例子，你可以将你的银行账户放在一个标签中，而 evil.com 在另一个标签中。来自 evil.com 的脚本不应通过你的凭据向你的银行 API 发出 AJAX 请求，例如从你的账户提取资金。

跨域资源共享是 W3C 的规范，大多数浏览器都实现了这种规范，可以让你指定哪种类型的跨域请求已获得授权，而不是使用基于 IFRAME 或者 JSONP 的不太安全且不太强大的解决方案。

#### [1.7.2. Processing](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-cors-processing)

CORS 规范区分预请求、简单请求、实际请求。想要学习更多 CORS，参考[这篇文章](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)

Spring MVC `HandlerMapping` 实现类提供了内置的对 CORS 的支持。在成功将请求映射到一个 handler 之后，`HandlerMapping` 实现类就会检查给定请求以及 handler 的 CORS 配置，然后采取更进一步的措施。预请求直接处理，但是，简单请求和实际的 CORS 请求会被拦截，校验，并设置需要的 CORS 响应头。


为了启用跨域请求，你需要有一个显式的 CORS 配置。如果找不到合适的 CORS 配置，就会拒绝预请求。如果没有添加 CORS 头部到简单请求、实际 CORS 请求响应中，那么，浏览器就会拒绝它们。

> **作者的话** 可能有人会认为跨域是浏览器无法发起请求，其实不是，只是拒绝接收 response。


每个 `HandlerMapping` 都可以单独地配置基于模式的 URL `CorsConfiguration` 映射。在大多数情况下，应用使用 MVC Java 配置或者 XML 命名空间去声明映射，不过这会导致一个传递给所有 `HandlerMapping` 实例的全局映射。


你可以将位于 `HandlerMapping` 级别的全局 CORS 配置与更加细粒度的 handler 级别的 CORS 配置相结合。举个例子，注解式 Controller 可以使用类级别或者方法级别的 `@CrossOrigin` 注解（其他 handler 可以实现 `CorsConfigurationSource`）

#### [1.7.3. @CrossOrigin](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-cors-controller)

`@CrossOrigin` 注解在注解式控制器的方法上，可以启用跨域请求，如下示例所示：

```java
@RestController
@RequestMapping("/account")
public class AccountController {

    @CrossOrigin
    @GetMapping("/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
```

默认地，@CrossOrigin 允许：

- 所有 origin
- 所有 header
- 所有 HTTP 方法

默认情况下，`allowedCredentials` 不启用，因为那会建立一个信任级别，会暴露敏感的用户特定的信息（例如 cookie 和 CSRF 令牌），应该只能在适当的情况下使用。

@CrossOrigin 可以用于：
- Controller 的方法，开启 Controller 方法级别的跨域请求
- Controller 类，由所有方法继承

maxAge 默认 30 分钟

`@CrossOrigin` 也支持类级别，这会被所有方法继承，如下示例所示：

```java
@CrossOrigin(origins = "https://domain2.com", maxAge = 3600)
@RestController
@RequestMapping("/account")
public class AccountController {

    @GetMapping("/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
```

你可以在类级别和方法级别上同时使用 `@CrossOrigin`，如下示例所示：

```java
@CrossOrigin(maxAge = 3600)
@RestController
@RequestMapping("/account")
public class AccountController {

    @CrossOrigin("https://domain2.com")
    @GetMapping("/{id}")
    public Account retrieve(@PathVariable Long id) {
        // ...
    }

    @DeleteMapping("/{id}")
    public void remove(@PathVariable Long id) {
        // ...
    }
}
```


#### [1.7.4. Global Configuration](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-cors-global)
除了细粒度的控制外，也可以通过实现 `WebMvcConfigurer.addCorsMappings` 定义全局 CORS 配置，参考代码见官网。

具体原理可见 `DefaultCorsProcessor`。


#### [1.7.5. CORS Filter](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-cors-filter)
可以通过内置的 CorsFilter 增加 CORS 支持。


### [1.8. Web Security](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-web-security)

Spring Security 工程为保护 web 应用免受恶意漏洞提供了支持。清参阅 Spring Security 参考文档，包括：

- [Spring MVC Security](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#mvc)
- [Spring MVC Test Support](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#test-mockmvc)
- [CSRF protection](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#csrf)
- [Security Response Headers](https://docs.spring.io/spring-security/site/docs/current/reference/html5/#headers)

### [1.9. HTTP Caching](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-caching)


### [1.11. MVC Config](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-config)

MVC Java 配置和 MVC XML 命名空间配置提供了默认的配置，这适合大多数应用程序，如果你觉得不够，也提供了配置 API 供你自定义。

#### [1.11.1. Enable MVC Configuration](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-config-enable)

在 Java 配置中，你可以使用 `@EnableMvc` 注解来启用 MVC 配置，如下示例所示：

```java
@Configuration
@EnableWebMvc
public class WebConfig {
}
```

> **作者的话** Spring Boot 无需且尽量不要使用 `@EnableWebMvc`，这会覆盖 Spring Boot 的默认配置。

在 XML 配置中，你可以使用 `<mvc:annotation-driven>` 元素，以启用 MVC 配置，如下示例所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/mvc
        https://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <mvc:annotation-driven/>

</beans>
```

上面的示例会注册许多 Spring MVC 的基础 bean，并且适配类路径上的可用依赖（例如，载体转换器，JSON，XML，以及其他什么）

#### [1.11.2. MVC Config API](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-config-customize)


在 Java 配置中，你可以实现 `WebMvcConfigurer` 示例，如下示例所示：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    // Implement configuration methods...
}
```

在 XML 下，你可以检查属性，以及 `<mvc:annotation-driven/>` 的子元素。你可以浏览 [Spring MVC XML schema](https://schema.spring.io/mvc/spring-mvc.xsd) 或者使用 IDE 的代码编译功能，找到那些可用的属性以及子元素。

#### [1.11.3. Type Conversion](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-config-conversion)

默认地，Spring MVC 安装了各种数字和日期的格式化器（formatter），并且支持 `@NumberFormat` 和 `@DateTimeFormat` 对字段进行自定义。

如果要在 Java 配置中注册自定义的格式化器和转换器，使用以下方式：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addFormatters(FormatterRegistry registry) {
        // ...
    }
}
```

> **作者的话** 其实就是实现 `WebMvcConfigurer` 的 `addFormatters` 方法


#### [1.11.4. Validation](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-config-validation)

默认地，如果 Bean Validation 存在于类路径（例如，Hibernate Validator），就会注册一个 `LocalValidatorFactoryBean` 作为全局的 Validator，供控制器方法参数上的 `@Valid` 以及 `Validated` 使用

>  **作者的话** 判断是否存在 Bean Validation 框架的方法是注解查找是否存在 `javax.validation.Validator`

在 Java 配置中，你可以自定义全局 `Validator` 示例，如下示例所示：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public Validator getValidator() {
        // ...
    }
}
```

#### [1.11.5. Interceptors](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-config-interceptors)

在 Java 配置中，你可以注册拦截器，以应用于收到的请求，如下示例所示：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LocaleChangeInterceptor());
        registry.addInterceptor(new ThemeChangeInterceptor()).addPathPatterns("/**").excludePathPatterns("/admin/**");
        registry.addInterceptor(new SecurityInterceptor()).addPathPatterns("/secure/*");
    }
}
```


#### [1.11.6. Content Types](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-config-content-negotiation)




#### [1.11.7. Message Converters](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-config-message-converters)

你可以通过覆盖 `configureMessageConverters()`，以 Java 配置的方式自定义 `HttpMessageConverter`（这会取代 Spring MVC 创建的默认转换器），或者通过覆盖 `extendMessageConverters()`（这可以自定义默认的转换器或者添加额外的转换器）

以下示例使用自定义的 `ObjectMapper` 添加了 XML 和 Jackson JSON 转换器，取代默认的转换器：

```java
@Configuration
@EnableWebMvc
public class WebConfiguration implements WebMvcConfigurer {

    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder()
                .indentOutput(true)
                .dateFormat(new SimpleDateFormat("yyyy-MM-dd"))
                .modulesToInstall(new ParameterNamesModule());
        converters.add(new MappingJackson2HttpMessageConverter(builder.build()));
        converters.add(new MappingJackson2XmlHttpMessageConverter(builder.createXmlMapper(true).build()));
    }
}
```


#### [1.11.10. Static Resources](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-config-static-resources)

这里提供了一种简便的方法用于提供静态资源服务，是基于 `Resource` 的位置列表的。

在以下示例中，给定一个以 `/resources` 开头的请求，相对路径用于查找和提供位于 web 应用的根目录下的 `/public` 或者类路径 `/static` 下的静态资源

如果需要以 /resources 为前缀，根据其后的相对路径寻找 Web 应用程序根目录下的 /public 资源或 类路径 /static 下的静态资源，资源设置一年到期，还会评估 Last-Modified 头部，如果存在，返回 304，则可以按如下配置：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/resources/**")
            .addResourceLocations("/public", "classpath:/static/")
            .setCachePeriod(31556926);
    }
}
```

**注意**
(1) 配置类路径静态资源时，末尾必须有 /，不可缺省，否则 404

#### [1.11.11. Default Servlet](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-default-servlet-handler)

Spring MVC 允许将 DispatcherServlet 映射到 `/`（因此，这覆盖了容器默认的 Servlet 映射），但是，Spring MVC 仍然允许容器默认的 Servlet 处理静态资源请求。Spring MVC 配置有一个 `DefaultServletHttpRequestHandler`，使用 `/**` 的 URL 映射，相对于其他 URL 映射的具有最低优先级（Integer.MAX_VALUE）。

> **作者的话** 在 Spring Boot 中，你不启用，是不会将此 `HandlerMapping` 添加的。并且，启用了也不会发生作用，因为请求会被 `SimpleUrlHandlerMapping` 中的 `/**` 匹配到。

该 handler 将所有请求转发到默认的 Servlet。因此，它必须保持在所有其他 URL `HandlerMapping` 的最后。如果你使用 `mvc:annotation-driven`，就是这种情况。另外，如果你设置了自定义的 `HandlerMapping` 实例，请确保将其 `order` 属性设置低于 `DefaultServletHttpRequestHandler`（Integer.MAX_VALUE）

如下示例显示了如何通过使用默认设置启用该功能：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }
}
```


#### [1.11.12. Path Matching](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/spring-framework-reference/web.html#mvc-config-path-matching)

你可以自定义与 URL 的路径匹配和处理有关的可选项。有关各个选项的详细信息，参见 [`PathMatchConfigurer`](https://docs.spring.io/spring-framework/docs/5.2.17.RELEASE/javadoc-api/org/springframework/web/servlet/config/annotation/PathMatchConfigurer.html) javadoc。

以下示例显示了如何在 Java 配置中自定义路径匹配：

```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void configurePathMatch(PathMatchConfigurer configurer) {
        configurer
            .setUseTrailingSlashMatch(false)
            .setUseRegisteredSuffixPatternMatch(true)
            .setPathMatcher(antPathMatcher())
            .setUrlPathHelper(urlPathHelper())
            .addPathPrefix("/api", HandlerTypePredicate.forAnnotation(RestController.class));
    }

    @Bean
    public UrlPathHelper urlPathHelper() {
        //...
    }

    @Bean
    public PathMatcher antPathMatcher() {
        //...
    }
}
```


