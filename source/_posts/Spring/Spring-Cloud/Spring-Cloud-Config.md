---
title: Spring Cloud Config
date: 2022-07-17 21:34:41
categories:
- 框架
tags:
- Spring Cloud
---

# [Spring Cloud Config](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/)
## [1. Quick Start](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_quick_start)
> 该章是官网的一个体验案例

首先，启动服务，如下：
```bash
$ cd spring-cloud-config-server
$ ../mvnw spring-boot:run
```
服务是一个 Spring Boot 程序，你也可以从 IDE 运行。

接下来，试验一下客户端，如下：
```bash
$ curl localhost:8888/foo/development
{"name":"foo","label":"master","propertySources":[
  {"name":"https://github.com/scratches/config-repo/foo-development.properties","source":{"bar":"spam"}},
  {"name":"https://github.com/scratches/config-repo/foo.properties","source":{"foo":"bar"}}
]}
```

定位属性资源的默认策略是去克隆一个 git 存储库（位于 `spring.cloud.config.server.git.uri`），并使用它去实例化一个迷你的 `SpringApplication`。

HTTP 服务以下格式的资源：
```text
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```

### [1.1. Client Side Usage](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_client_side_usage)
为了在应用程序中使用这些功能，你可以将其构建为依赖于 `spring-cloud-config-client` 的Spring Boot 项目。最简便的方法是使用 Spring Boot 启动器 `org.springframework.cloud:spring-cloud-starter-config`。对于 maven 用户以及 Gradle 和 Spring CLI 用户的 Spring IO 版本管理属性文件，也有一个父 POM 和 BOM（spring-cloud-starter-parent）。

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```


现在，你可以创建一个标准的 Spring Boot 应用，就像下面的 HTTP 服务：
```java
@SpringBootApplication
@RestController
public class Application {
    @RequestMapping("/")
    public String home() {
        return "Hello World!";
    }
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

当这个 HTTP 服务启动的时候，它会从默认的监听于本地端口 8888 配置服务（如果启动了）获取外部配置。如果想修改默认行为，你可以修改 bootstrap.properties 中的配置服务的位置，如下所示：

```bash
spring.cloud.config.uri: http://myconfigserver.com
```

默认地，如果应用名称没有设置，则会使用 `application`。如果要修改默认行为，可以使用 `spring.application.name` 进行修改：

```yml
spring.application.name: myapp
```

> - 设置属性 `${spring.application.name}` 不要使用保留字 `application-` 作为应用名前缀，防止无法解析出正确的资源。



## [2. Spring Cloud Config Server](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_spring_cloud_config_server)
Spring Cloud Config Server 提供了一个用于外部配置的 HTTP 资源 API。通过使用 `@EnableConfigServer` 注解，服务就能嵌入到 Spring Boot 应用中。因此，下面的应用就是一个配置服务：

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServer {
  public static void main(String[] args) {
    SpringApplication.run(ConfigServer.class, args);
  }
}
```

和所有的 Spring Boot 应用一样，它默认运行在 8080 端口，但你可以将其切换到约定好的 8888 端口。最简单的方式是，通过配置 `spring.config.name=configserver` 来启动应用，这同时也设置了默认的存储库类型。

> 注意，这种配置方式的依据是 Config Server jar 包下的 configserver.yml 文件。实际并没有作用，引用的是 github 上面的样本地址。

另一种方式是使用你自己的 `application.properties`，如下所示：

**application.properties**
```yml
server.port: 8888
spring.cloud.config.server.git.uri: file://${user.home}/config-repo
```

在这里，`${user.home}/config-repo` 是一个包含 YAML 以及属性文件的 git 存储库。

> 在 Windows 上，如果 git 存储库是一个绝对驱动的前缀，你需要再加一个 "/"，例如：`file:///${user.home}/config-repo)`



### [2.1. Environment Repository](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_environment_repository)
在什么地方存储配置服务的配置数据？管理此行为的策略是 `EnvironmentRepository`，服务 `Environment` 对象。这个 `Environment` 是 Spring Environment 的浅拷贝（包括 `propertySources` 作为主要功能）。`Environment` 资源是三个变量的参数化：

- `{application}`，映射到 `spring.application.name`
- `{profile}`，在客户端映射到 `spring.profiles.active`
- `{label}`

存储库的实现通常表现得像一个 Spring Boot 程序，它从 `spring.config.name` 等于 `{application}` 以及 `spring.profiles.active` 等于 `{profiles}` 中加载配置文件。配置文件的优先规则也与常规的 Spring Boot 程序相同：激活的配置文件优先于默认值，如果又多个配置文件，则选择最后一个（类似向 Map 添加条目）。


如果存储库是基于文件的，那么服务器将从 application.yml 和 foo.yml 中创建一个 `Environment`。如果 YAML 文件在它们内部有指向 Spring 配置文件的文档，那么会使用更高的优先级。如果有特定的配置 YAML 文件，那么这些文件也以比默认值更高的优先级而使用。高优先级转换为在 `Environment` 中提前列出的 `PropertySource`。


#### [2.1.1. Git Backend](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_git_backend)

默认的 `EnvironmentRepository` 的实现使用 Git 后端，这对于管理升级和物理环境，以及对于跟踪变化非常方便。要更改存储库的位置，你可以在 Config Server（例如在 `application.yml` 中）中设置 `spring.cloud.config.server.git.uri` 配置属性。如果你用一个 `file:` 前缀进行设置，则应从本地存储库工作，以便于你可以在没有服务器的情况下快速启动。但是，在这种情况下，服务直接在本地存储库上操作而无需克隆（无论是否是裸仓库都无关紧要，因为 Config Server 永远不会更改 "remote" 存储库）。为了扩展 Config Server 并使其高度可用，你需要将所有服务实例指向相同的存储库，因此只有共享文件系统才能起作用。甚至在这种情况下，最好将 `ssh:` 协议用于共享文件系统存储库，以便于服务可以克隆它，并将本地工作副本作为缓存。

##### [Skipping SSL Certificate Validation](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_skipping_ssl_certificate_validation)

通过将 `git.skipSslValidation` 属性设置为 `true`（默认为 `false`），可以禁用配置服务器对 Git 服务器的 SSSL 证书校验：
```yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://example.com/my/repo
          skipSslValidation: true
```

##### [Setting HTTP Connection Timeout](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_setting_http_connection_timeout)


#### [2.1.3. File System Backend](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_file_system_backend)
Config Server 中拥有一个 `native` profile，该配置不使用 Git，而是从本地类路径或者文件系加载配置文件。为了使用本地配置，以 `spring.profiles.active=native` 启动 Config Server。

搜索路径可以包含占位符 `{application}`, `{profile}`, `{label}`。通过这种方式，你可以在路径中分离目录并选择一个对你有意义的策略。

如果你没有在搜索路径中使用占位符，存储库还会将 `{label}` 参数追加到存储路径末尾，因此属性文件会从每个搜索路径，以及有一个 label 后缀的子目录去加载。因此，没有占位符的默认行为与在搜索路径末尾添加 `/{label}/` 相同。举个例子，`file:/tmp/config` 与 `file:/tmp/config,file:/tmp/config/{label}` 相同。这个行为可以通过设置 `spring.cloud.config.server.native.addLabelLocations=false` 从而禁用。

> - 默认添加 label 后缀的行为很单纯，例如，你设置了 `spring.cloud.config.server.native.search-locations=file:///f:/profiles/application`，同时 label 参数为 dev，那么还会搜索的路径就是 `file:///f:/profiles/applicationdev`，并不会帮你自动添加路径分隔符 `/`，至少在 2.2.8.RELEASE 测试是如此。



#### [2.1.6. Sharing Configuration With All Applications](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_sharing_configuration_with_all_applications)


##### [2.1.6.1. File Based Repositories](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#spring-cloud-config-server-file-based-repositories)
使用基于文件的存储库，在所有客户端应用之间共享文件名为 `application*` 的资源（`application.properties`, `application.yml`, `application-*.properties` 等）。你可以使用具有这些文件名的资源来进行全局默认配置，并且根据需要让它们被应用特定的文件覆盖。

属性覆盖功能也可以用于设置全局默认，应用程序允许在本地覆盖它们。

> 使用 native 配置文件（本地文件系统后端），你应该使用不属于服务自己的配置的指定搜索路径。否则，位于默认搜索路径中的 `application*` 资源会被移除，因为它们是服务的一部分。



#### [2.1.7. JDBC Backend](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_jdbc_backend)
Spring Cloud Config 服务支持 JDBC 作为配置属性的后端。你可以通过添加 `spring-jdbc` 到类路径，并使用 `jdbc` 配置，或者添加 `JdbcEnvironmentRepository` 该 bean 来启用此功能。

你可以通过设置 `spring.cloud.config.server.jdbc.enabled=false` 来禁用 `JdbcEnvironmentRepository` 的自动配置。

> 至少 Spring Cloud Config 2.2.8.RELEASE 开始支持 enabled 属性

数据库需要有一个名为 `PROPERTIES` 的表，列为 `APPLICATION`, `PROFILE`, `LABEL`, `KEY`, `VALUE`。所有的字段都是 Java 的 String 类型，因此你可以定义为 `VARCHAR`。属性值表现与它们来自 Spring Boot 属性文件 `{application}-{profile}.properties` 相同，包括所有的编码与解码，这些稍后会进行处理（即不会直接再存储库实现中）。

> 默认的 SQL 为 `SELECT KEY, VALUE from PROPERTIES where APPLICATION=? and PROFILE=? and LABEL=?`，但这对于 MySQL 并不管用，因为 KEY 为关键字，应当被反引号包裹，否则在执行过程中报错。


#### [2.1.8. Redis Backend](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_redis_backend)


#### [2.1.11 Composite Environment Repositories](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#composite-environment-repositories)
从多个环境存储库提取配置数据。


### [2.2.  Health Indicator](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_health_indicator)
配置服务器附带一个健康指示器，用于检查配置的 `EnvironmentRepository` 是否正常工作。默认地，它会请求 `EnvironmentRepository` 一个名为 `app` 的应用，`default` 的配置，由 `EnvironmentRepository` 实现提供的默认标签。


通过 `health.config.enabled=false`，你可以禁用健康指示器。


### [2.3. Security](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_security)
你可以以任何对你有意义的方法保护 Config Server（从物理网络安全到 OAuth2 持票人令牌），Spring Security 和 Spring Boot 提供了许多安全性的功能。

为了使用默认的 Spring Boot 配置 HTTP Basic 安全，需要包含 Spring Security 到类路径。默认具有一个名为 `user` 的用户名和随机生成的密码。实践中，随机密码并没有太大用处，推荐配置密码并加密。

> - 需要包含 `spring-boot-starter-security` 依赖，以使用 Spring 的自动配置化的 HTTP Basic 安全
> - 通过设置 `spring.security.user.password` 配置密码
> - 客户端注意设置用户名和密码



### [2.4. Encryption and Decryption](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_encryption_and_decryption)
> 为了使用加密和解密功能，旧版本 JDK 需要下载完全的 [JCD](https://www.oracle.com/java/technologies/javase-jce-all-downloads.html)

如果远程属性资源包含加密内容（以 `{cipher}` 开头），则先解密再通过 HTTP 发送。该设置的优点是：当属性值 "静止" 时，不需要以纯文本方式展示。如果值无法被解密，将会从属性源中删除它，并添加一个额外的有相同键的属性，但是具有 `invalid` 前缀，值意味着不适用。这主要是为了加密文本用作密码，有可能意外泄漏。

### [2.5. Key Management](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_key_management)
Config Server 可以使用对称加密或者非对称加密（RSA 密钥对）。选择不对称加密在安全性方面更优越，但使用对称密钥通常更方便，因为它是在 `bootstrap.properties` 中配置的单个属性值。

要配置对称密钥，你需要设置 `encryt.key` 为密钥字符串（或者使用 `ENCRYPT_KEY` 环境变量，可以脱离纯文本配置文件）。


### [2.6. Creating a Key Store for Testing](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_creating_a_key_store_for_testing)
使用 JDK 自带的 `keytool` 工具创建密钥库。

```bash
$ keytool -genkeypair -alias mytestkey -keyalg RSA \
  -dname "CN=Web Server,OU=Unit,O=Organization,L=City,S=State,C=US" \
  -keypass changeme -keystore server.jks -storepass letmein
```

将生成的 `server.jks` 文件放到类路径下，然后在 `bootstrap.yml` 进行配置：
```yml
encrypt:
  keyStore:
    location: classpath:/server.jks # keystore 文件存储路径
    password: letmein # storepass 密钥仓库
    alias: mytestkey # 密钥对别名
    secret: changeme # keypass 用来保护所生成密钥对中的私钥
```

### [2.7. Using Multiple Keys and Key Rotation](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_using_multiple_keys_and_key_rotation)



### [2.8. Serving Encrypted Properties](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_serving_encrypted_properties)
有时候，你往客户端自行解密配置，而不是在配置中心解密完毕再传送过来。在这种情况下，如果你提供了 `encrypt.*` 相关配置来定位 key，你还是可以具有 `/encrypt` 和 `/decrypt` 端点，但是你需要在 `boostrap.[yml|properties]` 设置 `spring.cloud.config.server.encrypt.enabled=false` 来显式关闭传出属性的解密功能。如果你不关心端点，那么如果你没有配置 key 或者 enabled 标志，就能起作用了。


## [3. Serving Alternative Formats](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_serving_alternative_formats)
来自环境端点的默认 JSON 格式是完美被 Spring 应用消费的，因为它直接映射到 `Environment` 抽象上。如果你愿意，你也可以通过增加一个后缀（".yml", ".yaml" 或者 ".properties"）以 YAML 或者 Java 属性消费相同的数据。这对于那些不关心 JSON 端点的结构，或者额外元数据的应用来消费是非常有用的（例如，未使用 Spring 的应用可能会受益于此方法的简单性）。


## [4. Serving Plain Text](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_serving_plain_text)
不使用 `Environment` 抽象，你的应用可能需要对其环境量身定制的通用普通文本配置文件。Config Server 通过一个位于 `/{application}/{profile}/{label}/{path}` 的额外端点提供这些，其中，`application`，`profile`，`label` 与常规的环境端点有相同含义，但是 `path` 是一个文件名的路径（例如 log.xml）。此端点的源文件以与环境端点相同的方式定位。相同的搜索路径被用于 properties 和 YAML 文件。但是，仅返回第一个被匹配的资源，而不是聚合所有资源。


在资源被定位之后，以常规格式的占位符（`${...}`）会被使用提供的 application name，profile，label 解析。以这种方式，资源端点与环境端点紧密集成。


## [5. Embedding the Config Server](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_embedding_the_config_server)
配置服务最好以独立应用运行。但是，如果你需要，你也可以将其嵌入到另一个应用中。使用 `@EnableConfigServer` 注解，一个可选的属性 `spring.cloud.config.server.bootstrap` 在这种情况下是有用的。这是一个标记，指示该服务是否应该从它自己的远程存储库配置自己。默认地，该标记是关闭的，因为它可以延迟启动。但是，当嵌入另一个应用中时，将与任何其他应用程序以一样的方式启动是有意义的。将 `spring.cloud.config.server.bootstrap` 设置为 `true` 时，还必须使用符合环境存储库配置。


## [6. Push Notifications and Spring Cloud Bus](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_push_notifications_and_spring_cloud_bus)
许多源代码存储库提供者（如 Github，Gitlab，Gitea, Gitee, Gogs, 或者 Bitbucket）通过 webhook 通知你存储库中的更改。你可以通过提供者的用户接口以一个 URL 和一组你感兴趣的事件配置 webhook。如果你添加了 `spring-cloud-config-monitor` 依赖，并且在你的配置中心激活了 Spring Cloud Bus，那么 `/monitor` 端点会被启用。

当 webhook 被激活时，配置服务会针对它认为可能已经更改的应用程序发送 `RefreshRemoteApplicationEvent` 。变更检测是策略化的。但是，默认地，它会寻找与应用程序名称匹配的文件中地变更。


## [7. Spring Cloud Config Client](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_spring_cloud_config_client)
Spring Boot 应用可以立即使用 Spring Config 服务。


### [7.1. Config First Bootstrap](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#config-first-bootstrap)
类路径上拥有 Spring Cloud Config Client 的应用程序，默认的行为是：当配置客户端启动时，它会绑定到 Config Server（通过 `spring.cloud.config.uri` 引导配置属性），并使用远程属性源初始化 Spring 的 `Environment`。

此行为的最终结果是，所有希望消费 Config Server 的客户端需要一个 `bootstrap.yml`，其中需要在 `spring.cloud.config.uri` 中配置好服务地址（默认是 `http://localhost:8888`）。


### [7.2. Discovery First Bootstrap](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#discovery-first-bootstrap)


### [7.3. Config Client Fail Fast](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#config-client-fail-fast)


### [7.4. Config Client Retry](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#config-client-retry)

### [7.5. Locating Remote Configuration Resources](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_locating_remote_configuration_resources)

Config Service 从 `/{application}/{profile}/{label}` 供应属性源，在客户端应用中**默认**的绑定如下：

- "application"=`${spring.application.name}`
- "profile"=`${spring.profiles.active}`
- "label"="master"


你可以通过设置 `spring.cloud.config.*` 来覆盖它们（此处 `*` 表示 `name`, `profile`, `label`）。

`label` 对于回滚到之前的配置版本比较有用，使用默认的 Config Server 实现，它可以是 git label，分支名，commit ID。

`label` 也可以用逗号分隔的列表表示，在这种情况下，列表中的项目会逐个尝试，直至成功（即只有一个有效）。当工作在功能分支上时，此行为可能比较有用，例如，你可能希望将 `label` 与你的分支对齐，但使其可选，在这种情况下，你可以使用 `spring.cloud.config.label=myfeature,develop`。



### [7.6. Specifying Multiple Urls for the Config Server](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_specifying_multiple_urls_for_the_config_server)


### [7.7. Configuring Timeouts](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_configuring_timeouts)
配置超时阈值：

- 读取超时：`spring.cloud.config.request-read-timeout`
- 连接超时：`spring.cloud.config.request-connect-timeout`



### [7.8. Security](https://docs.spring.io/spring-cloud-config/docs/2.2.8.RELEASE/reference/html/#_security_2)
如果你使用 HTTP Basic 安全验证，客户端需要知晓密码（如果不是默认的，还需要用户名）。你可以通过配置服务的 URI 指定用户名和密码，或者通过 `username` 和 `password` 属性：

```yml
spring:
  cloud:
    config:
     uri: https://user:secret@myconfig.mycompany.com
```

```yml
spring:
  cloud:
    config:
     uri: https://myconfig.mycompany.com
     username: user
     password: secret
```

> `spring.cloud.config.username` 和 `spring.cloud.config.password` 会覆盖 URI 里的值


