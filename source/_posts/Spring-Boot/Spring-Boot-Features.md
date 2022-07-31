---
title: Spring Boot Features
date: 2022-07-07 17:49:07
categories:
- 框架
tags:
- Spring Boot
---

# Spring Boot Features

## [1. SpringApplication](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-spring-application)
`SpringApplication` 类提供了一种简单的方式来引导 Spring 应用程序从 main 方法中启动。在许多情况下，你可以委托给静态的 `SpringApplication.run` 方法，就像下面的例子：

```java
public static void main(String[] args) {
    SpringApplication.run(MySpringConfiguration.class, args);
}
```

当你的应用启动时，你应该会看到类似的输出：
```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::   v2.3.12.RELEASE

2019-04-31 13:09:54.117  INFO 56603 --- [           main] o.s.b.s.app.SampleApplication            : Starting SampleApplication v0.1.0 on mycomputer with PID 56603 (/apps/myapp.jar started by pwebb)
2019-04-31 13:09:54.166  INFO 56603 --- [           main] ationConfigServletWebServerApplicationContext : Refreshing org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext@6e5a8246: startup date [Wed Jul 31 00:08:16 PDT 2013]; root of context hierarchy
2019-04-01 13:09:56.912  INFO 41370 --- [           main] .t.TomcatServletWebServerFactory : Server initialized with port: 8080
2019-04-01 13:09:57.501  INFO 41370 --- [           main] o.s.b.s.app.SampleApplication            : Started SampleApplication in 2.992 seconds (JVM running for 3.658)
```
### [1.1. Startup Failure](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-startup-failure)

如果你的应用程序未能启动，则注册的 `FailureAnalyzers` 将有机会提供专用的错误消息以及具体操作来解决该问题。例如，如果你在端口 `8080` 启动一个 Web 项目，并且该端口已经被占用，你应该能看到类似以下消息的东西：

```log
***************************
APPLICATION FAILED TO START
***************************

Description:

Embedded servlet container failed to start. Port 8080 was already in use.

Action:

Identify and stop the process that's listening on port 8080 or configure this application to listen on another port.
```

> Spring Boot 提供了许多 `FailureAnalyzer` 实现，你可以添加自己的实现。

### [1.4. Customizing SpringApplication](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-customizing-spring-application)
如果 `SpringApplication` 的默认值不符合你的口味，那么你可以取而代之地创建一个本地实例并自定义它。例如，关闭 banner，你可以这样写：

```java
public static void main(String[] args) {
    SpringApplication app = new SpringApplication(MySpringConfiguration.class);
    app.setBannerMode(Banner.Mode.OFF);
    app.run(args);
}
```

> 传递给 `SpringApplication` 地构造器参数是用于 Spring Bean 地配置源。在大多数情况下，这些是 `@Configuration` 类地引用，但它们也可以引用 XML 配置或者应该扫描的包。


也可以使用 `application.properties` 文件来配置 `SpringApplication`。有关详细信息，参见 [Externalized Configuration](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-external-config)

### [1.5. Fluent Builder API](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-fluent-builder-api)

如果你需要构建 `ApplicationContext` 层次结构（具有父子关系的多上下文）或者你更喜欢用 "fluent" 构造器 API，你可以使用 `SpringApplicationBuilder`。

`SpringApplicationBuilder` 使你可以将多个方法连接在一起，并且包含 `parent` 和 `child` 方法，可以让你创建一个层次结构，如下示例所示：

```java
new SpringApplicationBuilder()
        .sources(Parent.class)
        .child(Application.class)
        .bannerMode(Banner.Mode.OFF)
        .run(args);
```

### [1.7. Application Events and Listeners](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-application-events-and-listeners)

除了通常的 Spring Framework 事件，例如 `ContextRefreshedEvent`，一个 `SpringApplication` 还会发送一些其他应用事件。

> 实际上，在 `ApplicationContext` 创建之前一些事件就已经被触发了，因此你无法以 `@Bean`的方式在上面注册监听器。你可以用 `SpringApplication.addListeners(...)` 方法或者 `SpringApplicationBuilder.listeners(...)` 方法注册它们。
> 如果你希望这些监听器自动注册，无论应用程序的创建方式如何，你可以添加一个 `META-INF/spring.factories` 文件到你的项目中，并通过使用 `org.springframework.context.ApplicationListener` key 引用你的监听器，如下示例所示：
> ```properties
> org.springframework.context.ApplicationListener=com.example.project.MyListener
> ```


### [1.8. Web Environment](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-web-environment)

`SpringApplication` 试图在你的立场上创建正确的 `ApplicationContext` 类型。决定一个 `WebApplicationType` 的算法如下：

- 如果 Spring MVC 存在，使用 `AnnotationConfigServletWebServerApplicationContext`
- 如果 Spring MVC 不存在，Spring WebFlux 存在，使用 `AnnotationConfigReactiveWebServerApplicationContext`
- 否则，使用 `AnnotationConfigApplicationContext`


这意味着，如果你在同一个应用使用 Spring MVC 以及来自 Spring WebFlux 的新 `WebClient`，默认使用 Spring MVC。你可以通过调用 `setWebApplicationType(WebApplicationType)` 轻易覆盖这一点。

### [1.9. Accessing Application Arguments](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-application-arguments)
如果你需要访问传递给 `SpringApplication.run(...)` 的参数，那么你可以注入一个 `org.springframework.boot.ApplicationArguments` 的 bean。`ApplicationArguments` 接口提供了对原始 String[] 参数的访问，以及被解析的 option 和 non-option 参数，正如下面这个例子：

```java
import org.springframework.boot.*;
import org.springframework.beans.factory.annotation.*;
import org.springframework.stereotype.*;

@Component
public class MyBean {
    @Autowired
    public MyBean(ApplicationArguments args) {
        boolean debug = args.containsOption("debug");
        List<String> files = args.getNonOptionArgs();
        // if run with "--debug logfile.txt" debug=true, files=["logfile.txt"]
    }
}
```
### [1.11. Application Exit](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-application-exit)

每个 `SpringApplication` 都会在 JVM 上注册一个关闭钩子，以确保 `ApplicationContext` 在退出时优雅地关闭。所有标准的 Spring 生命周期回调（例如 `DisposableBean` 接口或者 `@PreDestroy` 注解）都可以使用。

此外，如果希望当 `SpringApplication.exit()` 调用时返回特定的结束码，Bean 可以实现 `org.springframework.boot.ExitCodeGenerator` 接口。然后可以将此退出码传递给 `System.exit()`，以一个状态码的形式返回，如下示例所示：

```java
@SpringBootApplication
public class ExitCodeApplication {

    @Bean
    public ExitCodeGenerator exitCodeGenerator() {
        return () -> 42;
    }

    public static void main(String[] args) {
        System.exit(SpringApplication.exit(SpringApplication.run(ExitCodeApplication.class, args)));
    }

}
```


## [2. Externalized Configuration](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-external-config)
### [2.1. Configuring Random Values](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-external-config-random-values)

`RandomValuePropertySource` 可用于注入随机值（例如，secret 或者测试用例）。它可以产生 integer，long，uuid，或者 string，如下示例所示：

```properties
my.secret=${random.value}
my.number=${random.int}
my.bignumber=${random.long}
my.uuid=${random.uuid}
my.number.less.than.ten=${random.int(10)}
my.number.in.range=${random.int[1024,65536]}
```

### [2.3. Application Property Files](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-external-config-application-property-files)

`SpringApplication` 在以下位置的 `application.properties` 文件中加载属性，并将它们添加到 Spring `Environment`：

1. 当前目录的一个 `/config` 子目录
2. 当前文件夹
3. 类路径 `/config` 包
4. 类路径根


## [2.7. Using YAML Instead of Properties](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-external-config-yaml)

YAML 是 JSON 的超集，因此，也是一种指定层次配置数据的方便格式。`SpringApplication` 类自动支持 YAML 作为 properties 的替代品，只需要在类路径添加 SnakeYAML 库支持。

如果你使用 "starter"，SnakeYAML 将由 `spring-boot-starter` 自动提供。

### [2.7.1. Loading YAML](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-external-config-loading-yaml)

Spring Framework 提供了两个方便的类，可以用于加载 YAML 文档。`YamlPropertiesFactoryBean` 将 YAML 加载作为 `Properties`，而 `YamlMapFactoryBean` 将 YAML 加载作为 `Map`。

例如，考虑如下 YAML 文档：
```yml
environments:
    dev:
        url: https://dev.example.com
        name: Developer Setup
    prod:
        url: https://another.example.com
        name: My Cool App
```
前面的示例将转换为以下属性：

```properties
environments.dev.url=https://dev.example.com
environments.dev.name=Developer Setup
environments.prod.url=https://another.example.com
environments.prod.name=My Cool App
```

### [2.7.4. YAML Shortcomings](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-external-config-yaml-shortcomings)

YAML 文件不能使用 `@PropertySource` 注解加载。因此，以这种方式加载值，你需要使用 properties 文件。

在特定 profile 的 YAML 文件中使用多 YAML 文档语法可能导致不可预测的行为。例如，考虑如下文件配置：

```yml
server:
  port: 8000
---
spring:
  profiles: "!test"
  security:
    user:
      password: "secret"
```

如果你以参数 `--spring.profiles.active=dev` 运行应用，你可能期望 `security.user.password` 设置为 "secret"，但是事实并非如此。

嵌套文档将被过滤，因为主文件名为 `application-dev.yml`。它已经被认为是特定于配置文件的，并且嵌套的文档将会被忽略。

> 我们建议你不要将特定 profile 的 YAML 文件和多 YAML 文档混合。坚持使用其一。


## [2.8. Type-safe Configuration Properties](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-external-config-typesafe-configuration-properties)

### [2.8.1. JavaBean properties binding](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-external-config-java-bean-binding)

如下示例所示，可以绑定声明标准 JavaBean 属性：

```java
package com.example;

import java.net.InetAddress;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties("acme")
public class AcmeProperties {

    private boolean enabled;

    private InetAddress remoteAddress;

    private final Security security = new Security();

    public boolean isEnabled() { ... }

    public void setEnabled(boolean enabled) { ... }

    public InetAddress getRemoteAddress() { ... }

    public void setRemoteAddress(InetAddress remoteAddress) { ... }

    public Security getSecurity() { ... }

    public static class Security {

        private String username;

        private String password;

        private List<String> roles = new ArrayList<>(Collections.singleton("USER"));

        public String getUsername() { ... }

        public void setUsername(String username) { ... }

        public String getPassword() { ... }

        public void setPassword(String password) { ... }

        public List<String> getRoles() { ... }

        public void setRoles(List<String> roles) { ... }

    }
}
```

上面的 POJO 定义了以下属性：

- `acme.enabled`，具有默认值 `false`
- `acme.remote-address`，具有可以从 `String` 强转的类型
- `acme.security.username`，具有一个嵌套的 "security" 对象，其名称由属性名决定。特别注意，返回类型这里根本没有使用，有可能是 `SecurityProperties`
- `acme.security.password`
- `acme.security.roles` 带有默认为 `USER` 的 `String` 集合

一般需要提供一个无参构造器，并且 getter 和 setter 是强制地，除了一些情况：

- Map，只要被实例化过了，只需要一个 getter 无需 setter
- Collection 和 Array，可以使用索引或者逗号分隔来访问。后者需要 setter。始终建议添加 setter。如果自己初始化，确保它是 not immutable
- 嵌套属性无需 setter。如果希望默认构造器能够创建该嵌套属性地实例，需要 setter。

有人使用 Lombok，确保其不会为属性类生成任何特定地构造函数，因为容器会自动使用它来实例化对象。

只考虑标准 Java Bean，不支持静态属性绑定。


### [2.8.3. Enabling @ConfigurationProperties-annotated types](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-external-config-enabling)

Spring Boot 提供了绑定 `@ConfigurationProperties` 类型并将其注册为 Bean 的基础架构。你可以在一个一个的类上启用配置属性，或者启用配置属性扫描，这以类似于组件扫描的方式工作。


有时候，`@ConfigurationProperties` 可能不适合总是被扫描。例如，如果你正在开发自己的 auto-configuration，或者你需要有条件的启用它们。在这些情况下，使用 `@EnableConfiguratonProperties` 注解指定要处理的类列表。这可以在任何 `@Configuration` 类上完成，如下示例所示：

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(AcmeProperties.class)
public class MyConfiguration {
}
```

要使用配置属性扫描，请将 `@ConfigurationPropertiesScan` 注解添加到你的应用。通常，会将其添加到用 `@SpringBootApplication` 注解的主应用类中，但也可以添加到任何 `@Configuration` 类。默认情况下，将从声明注解的类所在的包开始扫描。如果你想定义特定的包扫描，你可以像下面示例这样做：

```java
@SpringBootApplication
@ConfigurationPropertiesScan({ "com.example.app", "org.acme.another" })
public class MyApplication {
}
```


> 当使用配置属性扫描或者 @EnableConfigurationProperties 注册 @ConfigurationProperties bean 时，bean 有一个约定名称：`<prefix>-<fqn>`，`<prefix>` 是 @ConfigurationProperties 的 prefix 属性，`<fqn>` 是 bean 的全限定名。如果注解没有提供任何前缀，则使用 bean 的完全限定名称。


## [3. Profiles](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-profiles)

Spring 配置文件提供了一种方式，可以分离应用配置的一部分，并使其仅在某些环境中可用。任何 `@Component`，`@Configuration`，或者 `@ConfigurationProperties` 都可以用 `@Profile` 标记以限制何时加载，如下示例所示：

```java
@Configuration(proxyBeanMethods = false)
@Profile("production")
public class ProductionConfiguration {

    // ...

}
```

你可以使用 `spring.profiles.active` `Environment` 属性来指定哪些配置文件处于激活状态。你可以以本章前面描述的任何方式指定属性。例如，你可以在你的 `application.properties` 中包含，如下示例所示：

```properties
spring.profiles.active=dev,hsqldb
```

你也可以使用以下开关在命令行上指定：`--spring.profiles.active=dev.hsqldb`


如果没有配置文件处于激活状态。则启用默认配置文件。默认配置文件的名称为 `default`，并且可以使用 `spring.profiles.default` `Environment` 属性进行调整，如下示例所示：

```properties
spring.profiles.default=none
```

## [4. Logging](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-logging)

Spring Boot 使用 Commons Logging 用于所有内部的日志记录，但是保持底层的日志实现开放。为 Java Util Logging，Log4J2，以及 Logback 提供了默认配置。在每种情况下，日志记录器是预先配置使用控制台输出，同事可选的文件输出也是可用的。

默认情况下，如果你使用 "starter"，则使用 Logback 进行日志记录。适当的 Logback 路由也会被包括进来，确保使用 Java Util Logging，Commons Logging，Log4J，或者 SLF4J 的独立库都能正常工作。

> Java 有很多可供选择的日志框架。如果上面的清单似乎令人困惑，但不用担心。通常，你无需更改日志依赖项，Spring Boot 默认项工作就好。

### [4.1. Log Format](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-logging-format)

Spring Boot 的默认日志输出类似于以下示例：

```log
2019-03-05 10:57:51.112  INFO 45469 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet Engine: Apache Tomcat/7.0.52
2019-03-05 10:57:51.253  INFO 45469 --- [ost-startStop-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2019-03-05 10:57:51.253  INFO 45469 --- [ost-startStop-1] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1358 ms
2019-03-05 10:57:51.698  INFO 45469 --- [ost-startStop-1] o.s.b.c.e.ServletRegistrationBean        : Mapping servlet: 'dispatcherServlet' to [/]
2019-03-05 10:57:51.702  INFO 45469 --- [ost-startStop-1] o.s.b.c.embedded.FilterRegistrationBean  : Mapping filter: 'hiddenHttpMethodFilter' to: [/*]
```

### [4.2. Console Output](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-logging-console-output)

默认的日志配置在写入时会将信息回显到控制台。默认地，`ERROR` 级别，`WARN` 级别，以及 `INFO` 级别的信息会被记录。你还可以通过使用 `--debug` 标记启动应用，开启 "debug" 模式。


### [4.3. File Output](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-logging-file-output)

默认情况下，Spring Boot 日志仅仅输出到控制台，并不会写到日志文件。如果你希望除了输出到控制台还能写到日志文件，你需要设置 `logging.file.name` 或者 `logging.file.path` 属性（例如，在你的 `application.properties`）。

下表一起展示了如何使用 `logging.*` 属性：

|`logging.file.name`|`logging.file.path`|Example|Description|
|(none)|(none)||仅控制台记录|
|特定文件|(none)|my.log|写到特定日志文件。名称可以是精确的位置或者相对于当前文件夹|
|(none)|Specific directory|/var/log|写 `spring.log` 到特定文件夹。名字可以是一个精确的位置或者相对于当前文件夹|

日志文件达到 10MB 时旋转，并且与控制台输出一样，默认记录了 `ERROR` 级别，`WARN` 级别，以及 `INFO` 级别的信息。可以使用 `logging.file.max-size` 属性修改大小限制。默认情况下，保留最后7天的轮转日志文件，除非设置 `logging.file.max-history`。可以使用 `logging.file.total-size-cap` 属性限制日志归档的总大小。当日志归档的总大小超过该阈值的时候，将删除备份。要在应用程序启动时强制执行日志归档清理，请使用 `logging.file.clean-history-on-start` 属性。

> 日志属性与实际日志基础架构无关。结果就是，Spring Boot 不会管理特定的配置键（例如，对于 Logback 的 `logback.configurationFile`）。

### [4.4. Log Levels](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-custom-log-levels)

所有支持的日志系统都可以通过使用 `logging.level.<logger-name>=<level>` 在 Spring `Environment` 中（例如，在 `application.properties`）设置日志级别，其中 `level` 是 TRACE，DEBUG，INFO，WARN，ERROR，FATAL，或者 OFF 之一。可以使用 `logging.level.root` 配置 `root` 记录器。

以下示例展示了 `application.properties` 中的可能的日志设置：

```properties
logging.level.root=warn
logging.level.org.springframework.web=debug
logging.level.org.hibernate=error
```

也可以使用环境变量设置日志级别。例如，`LOGGING_LEVEL_ORG_SPRINGFRAMEWORK_WEB=DEBUG` 将设置 `org.springframework.web` 为 `DEBUG`。


### [4.7. Custom Log Configuration](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-custom-log-configuration)

可以通过在类路径上包含适当的库来激活各种日志系统，并可以通过提供合适的配置文件进行进一步的定制化，配置文件可以在类路径的根或者由下面 Spring `Environment` 属性：`logging.config` 指定位置。

你可以使用 `org.springframework.boot.logging.LoggingSystem` 系统属性强制 Spring Boot 使用特定的日志系统。该值必须是 `LoggingSystem` 实现的完全限定类名。你也可以使用 `none` 值完全禁用 Spring Boot 的日志配置。

> 由于 logging 是在 `ApplicationContext` 创建之前初始化的，因此不可能从 `@Configuration` 文件中的 `@PropertySources` 控制日志。更改日志系统或者完全禁用唯一的方式是通过系统属性。


根据你的日志系统，加载下面的文件：

|Logging System|Customization|
|:-|:-|
|Logback|`logback-spring.xml`, `logback-spring.groovy`, `logback.xml`, 或者 `logback.groovy`|
|Log4j2|`log4j2-spring.xml`，或者 `log4j2.xml`|
|JDK (Java Util Logging)|`logging.properties`|


## [5. Internationalization](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-internationalization)

## [6. JSON](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-json)

## [7. Developing Web Applications](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-developing-web-applications)
### [7.1. The “Spring Web MVC Framework”](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-spring-mvc)
#### [7.1.1. Spring MVC Auto-configuration](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-spring-mvc-auto-configuration)
Spring Boot 为 Spring MVC 提供了自动配置 `WebMvcAutoConfiguration`。


- WebMvcAutoConfigurationAdapter


如果希望保持 Spring Boot MVC 的定制，并作出更多 MVC 自定义的话，只需要添加自己的 `WebMvcConfigurer`，并添加 `@Configuration` 类，而不要使用 `@EnableWebMvc`。

`@EnableWebMvc` 导致 Spring Boot 自动配置失效原因：`@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)`

如果向完全控制 Spring MVC，可以添加带有 `@EnableWebMvc` 的 `@Configuration`，或者添加自己的 @Configuration 的 `DelegatingWebMvcConfiguration`


### [7.1.2. HttpMessageConverters](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-spring-mvc-message-converters)
Spring MVC 使用 `HttpMessageConverter` 接口来转换 HTTP 请求和响应。开箱即用地包含一些合理的默认项。例如，可以自动地将对象转换为 JSON（通过使用 Jackson 库）或者 XML（通过使用 Jackson XML 扩展，如果可用，或者如果 Jackson XML 扩展不可用就是用 JAXB）。

如果需要添加 converter，可以使用 Spring Boot 的 HttpMessageConverters：
```java
@Configuration(proxyBeanMethods = false)
public class MyConfiguration {
    @Bean
    public HttpMessageConverters customConverters() {
        HttpMessageConverter<?> additional = ...
        HttpMessageConverter<?> another = ...
        return new HttpMessageConverters(additional, another);
    }
}
```

### [7.1.3. Custom JSON Serializers and Deserializers](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-json-components)

使用 @JsonComponent 自定义序列化和反序列化


&nbsp;
### [7.1.5. Static Content](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-spring-mvc-static-content)
默认地，Spring Boot 提供静态资源的路径（见 `ResourceProperties.CLASSPATH_RESOURCE_LOCATIONS`）：

- classpath:/static
- classpath:/public
- classpath:/resources
- classpath:/META-INF/resources
- servletContext 根路径

Spring Boot 使用 Spring MVC 的 `ResourceHttpRequestHandler` 处理静态资源，可以通过添加自己的 `WebMvcConfigurer` 并覆盖 `addResourceHandlers` 方法修改此行为。

&nbsp;
默认地，静态资源资源映射在 /**，可以使用如下配置调整：
```bash
# 增加一个固定前缀 resources
spring.mvc.static-path-pattern=/resources/**
```

&nbsp;
修改默认的静态资源搜寻路径，支持数组（上面描述的 4 个 classpath 路径会失效）：
```bash
# Servlet Context 的根路径会自动添加
# 注意末尾需要添加 /
spring.resources.static-locations=classpath:/res/
```


**官方提示** 如果应用程序预计打包为 jar，不要使用 src/main/webapp 目录。虽然这是一个通用标准，但只是在 war 包上，如果生成 jar，会被大多数构建工具忽略。


### [7.1.6. Welcome Page](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-spring-mvc-welcome-page)

Spring Boot 支持静态和模板的欢迎页面。Spring Boot 首先在配置静态内容位置中寻找 `index.html` 文件。如果一个都没找到，那么它会寻找一个 `index` 模板。如果找到了某一个模板，模板将会自动使用作为应用的欢迎页。


### [7.1.7. Custom Favicon](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-spring-mvc-favicon)
在配置的静态内容位置寻找 favicon.ico，如果存在自动作为应用的图标（不可以增加静态资源前缀）。

### [7.1.11. Error Handling](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-error-handling)

默认地，Spring Boot 提供一个 `/error` 的 mapping 以明智的方式来处理所有的错误，并且它被注册为一个 Servlet 容器“全局”错误页面。对于机器客户端，它会产生一个具有错误详情，HTTP 状态码，异常信息的 JSON 响应。对于浏览器客户端，有一个白页错误视图，以 HTML 的格式渲染相同的数据（要定制它，需要添加一个解析到 `error` 的视图）。

如果你要自定义默认的错误处理行为，则可以设置许多 `server.error` 属性。参见附录的 ["Server Properties"](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/appendix-application-properties.html#common-application-properties-server)


为了完全替换默认行为，你可以实现 `ErrorController` 并注册该类型的 Bean Definition，或者添加类型为 `ErrorAttributes` 的 Bean，以使用现存的机制，但是替换内容。



你还可以定义一个以 `@ControllerAdvice` 注解的类，来自定义 JSON 格式，用于特定的 Controller 或者异常类型，如下示例所示：
```java
@ControllerAdvice(basePackageClasses = AcmeController.class)
public class AcmeControllerAdvice extends ResponseEntityExceptionHandler {

    @ExceptionHandler(YourException.class)
    @ResponseBody
    ResponseEntity<?> handleControllerException(HttpServletRequest request, Throwable ex) {
        HttpStatus status = getStatus(request);
        return new ResponseEntity<>(new CustomErrorType(status.value(), ex.getMessage()), status);
    }

    private HttpStatus getStatus(HttpServletRequest request) {
        Integer statusCode = (Integer) request.getAttribute("javax.servlet.error.status_code");
        if (statusCode == null) {
            return HttpStatus.INTERNAL_SERVER_ERROR;
        }
        return HttpStatus.valueOf(statusCode);
    }

}
```


&nbsp;
- @ResponseStatus 注解的异常类

通过处理过程抛出特定的异常，该异常类被 @ResponseStatus 注解

&nbsp;
在 Spring 中，HandlerExceptionResolverComposite 也是一个 HandlerExceptionResolver，而且它是多个 HandlerExceptionResolver 的组合。默认地，它以如下的顺序包含多个 HandlerExceptionResolver：

(1) ExceptionHandlerExceptionResolver
(2) ResponseStatusExceptionResolver
(3) DefaultHandlerExceptionResolver，垫底处理，实现 Ordered 接口，并且 order = Ordered.LOWEST_PRECEDENCE



#### [Custom Error Pages](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-error-handling-custom-error-pages)
对于给定的 HTTP 状态码，自定义 HTML 错误页面，可以添加文件到 /error 路径。错误页面可以是静态 HTML 或者使用模板引擎。

页面名称应该是精确的状态码或一系列掩码，如：404.html, 5xx.html。


### [7.4. Embedded Servlet Container Support](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-embedded-container)

Spring Boot 包含对内嵌的 Tomcat，Jetty，以及 Undertow 服务器的支持。大多数开发者使用适当的 "starter" 获得完整的配置实例。默认的，内嵌的服务器在端口 8080 上监听 HTTP 请求。


#### [7.4.1. Servlets, Filters, and listeners](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-embedded-container-servlets-filters-listeners)

当使用内嵌的 Servlet 容器时，你可以通过使用 Spring Bean 或者通过扫描 Servlet 组件的方式注册来自于 Servlet 规范的 Servlet，Filter，以及所有的 Listener（例如 `HttpSessionListener`）

##### [Registering Servlets, Filters, and Listeners as Spring Beans](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-embedded-container-servlets-filters-listeners-beans)

任何作为一个 Spring Bean 的 `Servlet`，`Filter`，或者 servlet `*Listener` 实例都会注册到内嵌的容器中。如果你想在配置期间引用一个来自于你 `application.properties` 的值，这是极其方便的。

默认情况下，如果上下文仅包含一个 Servlet，则将其映射到 `/`。在多 Servlet Bean 场景下，Bean Name 用于路径前缀。过滤器映射到 `/*`。

如果基于约定的映射不够灵活，你可以使用 `ServletRegistrationBean`，`FilterRegistrationBean`，以及 `ServletListenerRegistrationBean` 类获得完全控制。

通常，保持 Filter Bean 无序是安全的。如果需要特定的 order，你应该用 `@Order` 注解 `Filter` 或者使其实现 `Ordered`。你无法通过用 `@Order` 注解其 bean 方法配置 `Filter` 的 order。如果你无法更改 Filter 类以添加 `@Order` 或者实现 `Ordered`，你必须定义为该 `Filter` 定义一个 `FilterRegistrationBean` 并使用 `setOrder(int)` 方法设置注册 Bean 的 order。避免配置在 `Ordered.HIGEST_PRECEDENCE` 读取请求体的过滤器，因为它可能与你应用的字符编码配置违背。如果 Servlet filter 包装请求，则应使用小于等于 `OrderedFilter.REQUEST_WRAPPER_FILTER_MAX_ORDER` 的 order 值。

#### [7.4.2. Servlet Context Initialization](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-embedded-container-context-initializer)

嵌入式 Servlet 容器不会直接执行 Servlet 3.0+ 的 `javax.servlet.ServletContainerInitializer` 接口或者是 Spring 的 `org.springframework.web.WebApplicationInitializer` 接口。这是一个故意的设计决定，旨在降低被设计在 war 中运行的第三方包可能破坏 Spring Boot 应用的风险。

如果你需要在 Spring Boot 应用中执行 Servlet Context 初始化，你应该注册一个实现了 `org.springframework.boot.web.servlet.ServletContextInitializer` 接口的 Bean。单个 `onStartup` 方法提供了对 `ServletContext` 的访问，并且如果有必要，可以轻松地用作现存的 `WebApplicationInitializer` 的适配器。


#### [Scanning for Servlets, Filters, and listeners](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-embedded-container-servlets-filters-listeners-scanning)

当使用嵌入式容器时，可以通过使用 `@ServletComponentScan` 自动注册被 `@WebServlet`，`@WebFilter`，以及 `@WebListener` 注解的类。

## [8. Graceful shutdown](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-graceful-shutdown)


## [11. Working with SQL Databases](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-sql)

Spring Framework 为使用 SQL 数据库提供了广泛支持，从使用 `JdbcTemplate` 直接进行 JDBC 访问，到完成 "object relational mapping" 技术，例如 Hibernate。Spring Data 提供了更多功能级别：从你的接口中直接创建 `Repository` 实现，并使用约定从你的方法名称中生成查询。

### [11.1. Configure a DataSource](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-configure-datasource)

Java 的 `javax.sql.DataSource` 接口提供了一种使用数据库连接的标准方法。传统上，一个 'DataSource' 使用 `URL` 以及一些凭证来建立数据库连接。


#### [11.1.1. Embedded Database Support](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-embedded-database-support)

通常，使用一个内存嵌入式数据库开发应用很方便。显然，内存数据库不提供持久化存储。当你的应用程序启动时，你需要填充你的数据库，并在应用结束时准备好丢掉数据。

Spring Boot 可以自动配置嵌入式 H2，HSQL，以及 Derby 数据库。你不需要提供任何连接 URL。你只需要对要使用的嵌入式数据库包含一个构建依赖。


#### [11.1.2. Connection to a Production Database](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-connect-to-production-database)

生产数据库连接也可以通过池化 `DataSource` 自动配置。Spring Boot 使用下面的算法选择特定的实现：

1. 我们更喜欢 HikariCP 的性能和并发。如果 HikariCP 可用，我们始终会选择它
2. 否则，如果 Tomcat 池化 `DataSource` 可用，我们将使用它
3. 如果 HikariCP 和 Tomcat 池化数据源都不可用，并且 Commons DBCP2 可用，我们会使用它


如果你使用 `spring-boot-starter-jdbc` 或者 `spring-boot-starter-data-jpa` "starter"，你会自动获得 `HikariCP` 的依赖。


DataSource 的配置由 `spring.datasource.*` 之中的外部配置属性控制。例如，你可以在 `application.properties` 中声明如下部分：

```properties
spring.datasource.url=jdbc:mysql://localhost/test
spring.datasource.username=dbuser
spring.datasource.password=dbpass
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

> 你应该至少通过设置 `spring.datasource.url` 属性来指定 URL。否则，Spring Boot 将尝试自动配置嵌入式数据库。

> 你通常不需要指定 `driver-class-name`，因为 Spring Boot 可以从 `url` 中推导得到大多数数据库。


### [11.2. Using JdbcTemplate](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-using-jdbc-template)

Spring 的 `JdbcTemplate` 和 `NamedParameterJdbcTemplate` 类是自动配置的，你可以将它们直接 `@Autowire` 进自己的 Bean，如下示例所示：

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

@Component
public class MyBean {

    private final JdbcTemplate jdbcTemplate;

    @Autowired
    public MyBean(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    // ...

}
```

> **作者的话** 上面的例子其实不必在构造器上加 `@Autowired`，因为这是 Spring 的默认行为


### [11.3. JPA and Spring Data JPA](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-jpa-and-spring-data)

Java Persistence API 是一种标准技术，可以让你映射对象到关系型数据库。`spring-boot-starter-data-jpa` POM 提供了一种快速入门的方法。它提供了如下几个关键依赖：

- Hibernate：最流行的 JPA 实现之一
- Spring Data JPA：帮你实现基于 JPA 的存储库
- Spring ORM：为 Spring Framework 提供核心 ORM 支持

#### [11.3.1. Entity Classes](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-entity-classes)

传统上，JPA 实体类在 `persistence.xml` 文件中指定。使用 Spring Boot，不需要此文件，而是使用 "实体扫描"。默认情况下，在你主应用类（注解了 `@EnableAutoConfiguration` 以及 `@SpringBootApplication` 的类）下的所有包都会被搜索。

使用 `@Entity`，`@Embeddable`，或者 `@MappedSuperclass` 注解的类都会被纳入。一个典型的实体类类似于以下示例：

```java
package com.example.myapp.domain;

import java.io.Serializable;
import javax.persistence.*;

@Entity
public class City implements Serializable {

    @Id
    @GeneratedValue
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private String state;

    // ... additional members, often include @OneToMany mappings

    protected City() {
        // no-args constructor required by JPA spec
        // this one is protected since it shouldn't be used directly
    }

    public City(String name, String state) {
        this.name = name;
        this.state = state;
    }

    public String getName() {
        return this.name;
    }

    public String getState() {
        return this.state;
    }

    // ... etc

}
```

#### [11.3.2. Spring Data JPA Repositories](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-spring-data-jpa-repositories)

Spring Data JPA repository 是一些接口，你可以定义其用于访问数据。JPA 查询从你的方法名称自动创建。例如，一个 `CityRepository` 接口可能声明了一个 `findAllByState(String state)` 方法，用于找到给定 state 的所有 city。

有关更复杂的查询，你可以使用 Spring Data 的 `Query` 注解来注解你的方法。

通常，Spring Data repository 从 `Repository` 或者 `CrudRepository` 接口扩展。如果你使用自动配置，将会从包含主配置类（被 `@EnableAutoConfiguration` 或者 `@SpringBootApplication` 注解的类）的包下面搜索 repository。

以下示例展示了一个典型的 Spring Data repository 接口的定义：

```java
package com.example.myapp.domain;

import org.springframework.data.domain.*;
import org.springframework.data.repository.*;

public interface CityRepository extends Repository<City, Long> {

    Page<City> findAll(Pageable pageable);

    City findByNameAndStateAllIgnoringCase(String name, String state);

}
```

Spring Data JPA repository 支持三种不同的自我引导模式：default，deferred，以及 lazy。要启动 deferred 或者 lazy 自启，请将分别将 `spring.data.jpa.repositories.bootstrap-mode` 属性设置为 `deferred` 或者 `lazy`。当使用 deferred 或者 lazy 引导模式时，自动配置的 `EntityManagerFactoryBuilder` 将会使用上下文的 `AsyncTaskExecutor`（如果有的话）作为引导的执行器。如果存在多个，将会使用名为 `applicationTaskExecutor` 的那个。


### [11.4. Spring Data JDBC](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-data-jdbc)


Spring Data 包含对 JDBC 支持的 repository，并将自动为 `CrudRepository` 上的方法生成 SQL。对于更高级的查询，提供了一个 `@Query` 注解。

当必要的依赖在类路径上时，Spring Boot 将自动配置 Spring Data 的 JDBC repository。使用仅仅一个依赖 `spring-boot-starter-data-jdbc`，就可以将它们添加到你的项目。如果有必要，你可以通过添加 `@EnableJdbcRepositories` 注解或者 `JdbcConfiguration` 子类到你的应用来控制 Spring Data JDBC 的配置。

### [11.5. Using H2’s Web Console](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-sql-h2-console)

H2 数据库提供了一个基于浏览器的控制台，Spring Boot 可以为你自动配置。当满足如下条件时，将自动配置控制台：

- 你正在开发一个基于 Servlet 的 Web 应用
- `com.h2database:h2` 在类路径
- 你正在使用 Spring Boot's developer tools


如果你没有使用 Spring Boot developer tools，但仍然希望使用 H2 的控制台，你可以配置 `spring.h2.console.enabled` 属性值为 `true`


#### [11.5.1. Changing the H2 Console’s Path](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-sql-h2-console-custom-path)

默认情况下，控制台在 `/h2-console` 可以获得。你可以使用 `spring.h2.console.path` 属性自定义控制台路径。

## [12. Working with NoSQL Technologies](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-nosql)



### [12.5. Elasticsearch](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-elasticsearch)




### [12.1. Redis](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-redis)


## [13. Caching](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-caching)

Spring Framework 为透明地将缓存添加到应用提供了支持。从其核心，抽象将缓存应用于方法，从而根据缓存中可用的信息减少了执行次数。缓存逻辑是透明地应用的，不会对调用者进行任何干扰。只要通过 `@EnableCaching` 注解启用缓存支持，Spring Boot 就会自动配置缓存基础架构。

简而言之，为了将缓存添加到服务的操作中，将相关的注解添加到其方法中，如下图所示：

```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Component;

@Component
public class MathService {

    @Cacheable("piDecimals")
    public int computePiDecimal(int i) {
        // ...
    }

}
```

该示例证明了在可能高成本的操作中使用缓存。在调用 `computePiDecimal`，抽象在 `piDecimals` 缓存中匹配 `i` 参数的条目。如果找到条目，则立即将缓存中的内容返回给调用者，并且该方法不会调用。否则，将调用方法，在返回该值之前更新缓存。

> 你还可以透明地使用标准 JSR-107(JCache) 注解（例如 `@CacheResult`）。但是，我们强烈建议你不要混合并匹配 Spring Cache 和 JCache 注解。


如果你没有添加任何特定的缓存库，Spring Boot 自动配置一个在内存的使用 concurrent map 的 simple provider。当需要缓存（例如在前面例子中的 `piDecimals`）时，该 provider 就会为你创建一个。实际上，并不建议将 simple provider 用于生产，但是它非常适合入门，以及确保你了解这些功能。当你决定要使用 cache provider 时，请确保阅读其文档，以找出如何配置应用程序使用的缓存。几乎所有的 provider 都要求你显式配置每个你在应用中使用的缓存。有些提供一种自定义默认缓存的方法，通过定义 `spring.cache.cache-names` 属性。

> 也可以透明地更新或从缓存中驱逐数据

### [13.1. Supported Cache Providers](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-caching-provider)

缓存抽象并不提供实际的存储，并依赖于由 `org.springframework.cache.Cache` 和 `org.springframework.cache.CacheManager` 接口实现的抽象。

如果你没有定义 `CacheManager` 类型的 Bean 或者名为 `cacheResolver` 的 `CacheResolver`，Spring Boot 会尝试检测以下 provider（按照指示的顺序）：

1. Generic
2. JCache (JSR-107)(EhCache 3, Hazelcast, Infinispan, and others)
3. EhCache 2.x
4. Hazelcast
5. Infinispan
6. Couchbase
7. Redis
8. Caffeine
9. Simple


#### [13.1.1. Generic](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-caching-provider-generic)

如果上下文定义了至少一个 `org.springframework.cache.Cache` Bean，则使用 Generic 缓存。将会创建一个 `CacheManager` 包裹所有该类型的 Bean。


#### [13.1.2. JCache (JSR-107)](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-caching-provider-jcache)

通过类路径包含一个 `javax.cache.spi.CachingProvider` 会自动引导 JCache，并且 `spring-boot-starter-cache` "starter" 提供 `JCacheCacheManager`。可以获得各种符合条件的库，Spring 为 Ehcache 3，Hazelcast 和 Infinispan 提供依赖管理。也可以添加任何其他符合条件的库。

有可能出现多个 provider，在这种情况下，provider 必须显式指定。即使 JSR-107 标准没有强制一种标准的方式定义配置文件的路径，Spring Boot 尽力容纳使用实现细节配置一个缓存，如下示例所示：

```properties
# Only necessary if more than one provider is present
spring.cache.jcache.provider=com.acme.MyCachingProvider
spring.cache.jcache.config=classpath:acme.xml
```

有两种方法可以自定义底层的 `javax.cache.cacheManager`:

- 可以通过启动时设置 `spring.cache.cache-names` 属性创建缓存。如果定义了自定义的 `javax.cache.configuration.Configuration`，则将用于定制化缓存。
- 使用 `CacheManager` 引用调用 `org.springframework.boot.autoconfigure.cache.JCacheManagerCustomizer` Bean，获得完整定制化。

#### [13.1.3. EhCache 2.x](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-caching-provider-ehcache2)

如果在类路径的根路径下能够找到名为 `ehcache.xml` 的文件，那么 EhCache 2.x 会被使用。如果找到 EhCache 2.x，那么由 `spring-boot-starter-cache` "starter" 提供的 `EhCacheCacheManager` 将用于引导 cache manager。也可以提供可替代的配置文件，如下示例所示：

```properties
spring.cache.ehcache.config=classpath:config/another-config.xml
```

#### [13.1.4. Hazelcast](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-caching-provider-hazelcast)

Spring Boot 对 Hazelcast 有一般支持。如果 `HazelcastInstance` 已经自动配置了，那么它将自动包裹进 `CacheManager`。


#### [13.1.5. Infinispan](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-caching-provider-infinispan)


#### [13.1.7. Redis](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-caching-provider-redis)

如果 Redis 可用且已配置，`RedisCacheManager` 就会自动配置。可以通过设置 `spring.cache.cache-names` 属性在启动时创建额外的缓存，以及通过使用 `spring.cache.redis.*` 属性配置缓存默认值。例如，以下配置创建了 `cache` 和 `cache2` 缓存，并具有 10 分钟的存活时间: 
```properties
spring.cache.cache-names=cache1,cache2
spring.cache.redis.time-to-live=600000
```

> **Tips** 需要依赖 `spring-boot-starter-data-redis` 才能引入 `RedisCacheManager`


> 默认情况下，添加了一个 key 前缀，这是为了，如果两个分开的缓存使用相同的 key，redis 没有重叠的 key，且无法返回无效值。我们强烈建议你在创建自己的 `RedisCacheManager` 时启用该设置。



## [21. Quartz Scheduler]()

## [26. Testing](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-testing)

Spring Boot 在测试你的应用程序时提供了许多工具和注解。由两个模块提供测试支持：`spring-boot-test` 包含核心项目，`spring-boot-test-autoconfigure` 包含测试的自动配置。

大多数开发者使用 `spring-boot-starter-test` "starter"，该测试不仅导入了 Spring Boot 测试模块，而且还导入了 JUnit Jupiter，AssertJ，Hamcrest，以及许多其他有用的库。

> starter 还引入了 vintage 引擎，因此你既可以运行 JUnit 4 也可以运行 JUnit 5 的测试。如果你已经将测试迁移到 JUnit 5，你应该排除 JUnit 4 的支持，如下示例所示：
> ```java
> <dependency>
>     <groupId>org.springframework.boot</groupId>
>     <artifactId>spring-boot-starter-test</> artifactId>
>     <scope>test</scope>
>     <exclusions>
>         <exclusion>
>             <groupId>org.junit.vintage</groupId>
>             <artifactId>junit-vintage-engine</> artifactId>
>         </exclusion>
>     </exclusions>
> </dependency>
> ```

### [26.1. Test Scope Dependencies](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-test-scope-dependencies)


## [29. Creating Your Own Auto-configuration](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-developing-auto-configuration)

### [29.1. Understanding Auto-configured Beans](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-understanding-auto-configured-beans)

表层之下 ，使用标准 `@Configuration` 类实现自动配置。当使用自动配置时，需要使用额外的 `@Conditional` 注解进行约束。通常，自动配置类使用 `@ConditionalOnClass` 以及 `@ConditionalOnMissingBean` 注解。这样可以确保仅当找到相关的类，以及未声明自己的 `@Configuration` 时才使用自动配置。

你可以浏览 `spring-boot-autoconfigure` 源码，查看 Spring 提供的 `@Configuration` 类（见 `META-INF/spring.factories` 文件）。

### [29.2. Locating Auto-configuration Candidates](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-locating-auto-configuration-candidates)

Spring Boot 检查你已发布的 jar 中是否存在 `META-INF/spring.factories` 文件。该文件应在 `EnableAutoConfiguration` key 下列出你的配置类，如下示例所示：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.mycorp.libx.autoconfigure.LibXAutoConfiguration,\
com.mycorp.libx.autoconfigure.LibXWebAutoConfiguration
```

## [29.3. Condition Annotations](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-condition-annotations)
### [29.3.1. Class Conditions](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-class-conditions)

- @ConditionalConClass
- @ConditionalOnMissingClass


### [29.3.2. Bean Conditions](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-bean-conditions)
- @ConditionalOnBean
- @ConditionalOnMissingBean


### [29.3.3. Property Conditions](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-property-conditions)
- @ConditionalOnProperty

### [29.3.4. Resource Conditions](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-resource-conditions)
- @ConditionalOnResource

### [29.3.5. Web Application Conditions](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-web-application-conditions)
- @ConditionalOnWebApplication
- @ConditionalOnNotWebApplication

### [29.3.6. SpEL Expression Conditions](https://docs.spring.io/spring-boot/docs/2.3.12.RELEASE/reference/html/spring-boot-features.html#boot-features-spel-conditions)

