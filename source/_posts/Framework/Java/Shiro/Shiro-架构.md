---
title: Shiro 架构
date: 2022-07-24 09:34:39
tags:
---


# Shiro 架构


## [Apache Shiro 架构](https://shiro.apache.org/architecture.html)

Apache Shiro 的设计目的就是，通过直观化且易于使用来简化应用程序安全性。Shiro 的核心设计模型化了大多数人考虑应用安全性的方式 —— 在某人（或某物）与应用程序交互的上下文中。

软件应用通常基于用户故事设计的。即，你通常会根据用户将会（或者应该）如何与软件交互来设计用户接口或者服务 API。例如，你可能会说，"如果与我的软件交互的用户已经登陆了，我将向他们展示一个按钮，他们可以点机查看其账户信息。如果他们未登录，我将显示一个注册按钮"。

该实例语句标识，应用程序主要是为了满足用户需求和需要而大量编写的。即使“用户”是另一个软件系统，不是一个人类，你仍然可以编码，根据谁（或者什么东西）目前正在与你的软件进行交互反映行为。


### 高级概述

<img src="https://shiro.apache.org/images/ShiroBasicArchitecture.png">

- **Subject**: 正如我们在教程中提及的，`Subject` 本质上是当前正在执行用户的安全性特定“视图”。尽管词语“用户”通常表示一个人，但是 `Subject` 可以是一个人，但它也可以表示第三方服务，守护账户，cron 作业，或者任何类似的东西 —— 基本任何当前与软件交互的东西。

`Subject` 实例都会绑定到（且需要）一个 `SecurityManager`。当你和一个 `Subject` 交互时，这些交互会转换为特定于 subject 的与 `SecurityManager` 的交互。

- **SecurityManager**: `SecurityManager` 是 Shiro 架构的核心。它主要是一个 "伞" 对象，可以协调其管理的组件，确保他们能顺序运行。


我们将在稍后详细讨论 `SecurityManager`，但是重要的是要意识到，当你与 `Subject` 进行互动时，实际上，幕后的 `SecurityManager` 为任何 `Subject` 安全操作做了所有繁重的工作。这反映在上面的基本流程图中。

- **Realms**: Realm 充当 Shiro 与你应用程序安全性数据之间的“桥梁”或者连接器。


从这个意义上讲，一个 Realm 本质上是一种特定于安全性的 DAO: 它封装了数据源的连接细节，并使得 Shiro 在需要时获得相关数据。当配置 Shiro 时，你必须至少指定一个用于认证以及（或者）授权的 Realm。`SecurityManager` 可以配置多个 Realm，但至少需要配置一个。

### [详细架构](https://shiro.apache.org/architecture.html )

![请添加图片描述](https://img-blog.csdnimg.cn/66b4aa43259d4065b55e1a631f72918c.png)

- Subject (`org.apache.shiro.subject.Subject`) 当前正在与软件交互的，特定安全的实体视图（用户，第三方服务，cron 任务等）
- SecurityManager (`org.apache.shiro.mgt.SecurityManager`) 如上所述，`SecurityManager` 是 Shiro 架构的核心。它主要是一个 "伞" 对象，可以协调其管理的组件，确保他们能顺序运行。它还管理每个应用用户的视图，因此它知道每个用户如何执行安全操作。
- Authenticator (`org.apache.shiro.authc.Authenticator`) `Authenticator` 是负责执行和响应用户的认证（登录）请求的组件。当一个用户尝试登录，`Authenticator` 就会执行该逻辑。`Authenticator` 知道如何与一个或多个存储相关用户/账户信息的 `Realms` 协作。用从这些 `Realms` 获得的数据验证用户身份，保证用户的确如他们所说。
- Authorizer (`org.apache.shiro.authz.Authorizer`) `Authorizer` 是负责确定用户在应用程序中访问控制的组件。归根结底地说，这是一种机制，判断是否用户允许做某事。像 `Authenticator` 一样，`Authorizer` 也知道如何与多个后端数据源协作，访问角色和权限信息。`Authorizer` 使用这些信息，确定是否用户允许执行给定的操作。
- SessionManager (`org.apache.shiro.session.mgt.SessionManager`) `SessionManager`  知道如何创建和管理用户 `Session` 生命周期，为所有环境中的用户提供强大的会话体验。这是安全框架的世界独有的功能，Shiro 有能力本地化地管理任何环境下的用户会话，即使没有可用的 Web/Servlet 或者 EJB 容器。默认情况下，Shiro 将使用现有的会话机制，如果可用，（例如 Servlet Container），但是如果没有，例如在一个独立应用或者非 Web 环境下，将使用其内建的企业会话管理，提供相同的编程体验。`SessionDAO` 允许使用任何数据源持久化 Session。
- CacheManager (`org.apache.shiro.cache.CacheManager`)
- Cryptography (org.apache.shiro.crypto.*) 
- Realms (`org.apache.shiro.realm.Realm`) 如上所述，Realms 充当 Shiro 和应用安全数据之间的 "桥梁" 或者 "连接器"。当实际需要与安全相关数据（就像用户账户）进行交互，执行认证（login）以及授权（访问控制）的时候，Shiro 会从一个或多个为应用配置的 Realms 中查找许多相关的数据。你可以根据需要配置尽可能多的 `Realms`（通常每个数据源配一个），Shiro 在认证和授权的时候，根据需要协调它们。


### [Apache Shiro Web Support](https://shiro.apache.org/web.html)



### Default Filters




### Session Management
#### Servlet Container Sessions
在 Web 环境下，Shiro 的默认会话管理器 `SessionManager` 的实现是 `ServletContainerSessionManager`。这是一个非常简易的实现，将所有会话管理的职责（包括会话集群，如果 Servlet 容器支持）委托给运行时的 Servlet 容器。它本质上是 Shiro 会话 API 到 Servlet 容器的桥梁，几乎没做什么。


#### Native Sessions
如果你希望会话配置设置以及集群可以在 Servlet 容器上移植，或者你想控制特定的会话/集群功能，你可以启用 Shiro 的本地会话管理。

这里 "Native" 一词意味着，将会使用 Shiro 自己的企业会话管理实现，支持所有 `Subject` 以及 `HttpServletRequest` 会话，并完全绕过 Servlet 容器。但是，请放心，Shiro 直接实现了相关的 Servlet 规范部分，因此任何现有的 web/http 相关代码都可以按照预期工作，并且也不需要知道 Shiro 正在透明地管理会话。

##### `DefaultWebSessionManager`