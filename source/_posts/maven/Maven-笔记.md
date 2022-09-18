---
title: Maven 笔记
date: 2022-09-08 00:06:31
tags:
---

# Maven 笔记

## Maven 是什么
[官方介绍 What is  Maven?](https://maven.apache.org/what-is-maven.html)

Maven 是 Apache 软件基金会唯一维护的一款【自动化构建工具】。专注于 Java 平台的【项目构建】和【项目管理】


## Inheritance

Maven 为构建管理带来的强大的额外特性就是项目继承 (Inheritance) 的概念。


对于父工程和聚合 (多模块) 工程，`packaging` 类型必须是 `pom`。这些类型定义了绑定到一组生命周期阶段的目标。例如，如果 packaging 是 `jar`, 那么 `package` 阶段就会执行 `jar:jar` 目标。


请注意 `relativePath` 元素。它不是必需的，但是它可以作为一个指示器，告诉 Maven 在搜索本地以及远程仓库之前，首先搜索该工程的父工程给定的路径。

> 个人感觉 `relativePath` 使用场景很小


## 构建项目的主要环节
- clean：清理，删除以前的编译结果
- compile：编译，将 Java 源程序编译为字节码
- test：对项目进行测试
- package：将项目进行打包，以 jar 或 war 格式。
- install：将打包的结果 jar 或者 war 安装到本地仓库。
- deploy：将打包的结果部署到远程仓库或者 war 包部署到服务器上运行



## Maven 常用命令
`mvn -version / mvn -v` 显示版本信息
`mvn clean` 清理
`mvn compile` 编译
`mvn test` 编译并测试
`mvn package` 生成 target 目录，编译、测试代码，生成测试报告，生成 jar/war
`mvn site` 生成项目相关信息的网站
`mvn clean compile` 清理之后编译
`mvn clean package` 清理之后打包
`mvn clean install` 清理之后安装
`mvn clean deploy` 清理之后发布




## 依赖

### 依赖的范围

依赖范围 `<scope>` 控制哪些依赖在哪些 classpath 中可用，哪些依赖包含在一个应用中。

【compile】默认的范围；编译范围依赖在所有的 classpath 中可用，同时它们也会被打包。

【provided】provided 依赖只有在当 JDK 或者一个容器已提供该依赖之后才使用。例如， 如果你开发了一个web 应用，你可能在编译 classpath 中需要可用的 Servlet API 来编译一个 servlet，但是你不会想要在打包好的 WAR 中包含这个Servlet API；这个Servlet API JAR 由你的应用服务器或者servlet 容器提供。已提供范围的依赖在编译classpath （不是运行时）可用。它们不是传递性的，也不会被打包。

或者还有 lombok 的使用，打包完毕之后便不再需要包含这个 API。

【runtime】 runtime 依赖在运行和测试系统的时候需要，但在编译的时候不需要。比如，你可能在编译的时候只需要JDBC API JAR，而只有在运行的时候才需要JDBC 驱动实现。


【test】 test范围依赖 在一般的编译和运行时都不需要，它们只有在测试编译和测试运行阶段可用。

【system】 system范围依赖与provided类似，但是你必须显式的提供一个对于本地系统中JAR文件的路径。这么做是为了允许基于本地对象编译，而这些对象是系统类库的一部分。这样的构建应该是一直可用的，Maven 也不会在仓库中去寻找它。如果你将一个依赖范围设置成系统范围，你必须同时提供一个systemPath元素。注意该范围是不推荐使用的（建议尽量去从公共或定制的 Maven 仓库中引用依赖）。


【import】 import 仅支持在 `<dependencyManagement>` 中的类型依赖项上。该选项可以用来解决 Maven 的单继承。

通常，我们会使用 spring-boot-starter-parent 作为项目的父工程，因为其中进行了很多版本管理，同时我们也希望引入其他版本管理的工程，如 spring-cloud、spring-cloud-alibaba。
```xml
<dependencyManagement>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Hoxton.SR1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
</dependencyManagement>
```


### `<version>`
- `<parent>` 标签下的 `<version>`

Snapshot版本代表不稳定、尚处于开发中的版本  Release版本则代表稳定的版本
如果 deploy 到远程服务器 如果是 release 只能 deploy 一次，以后部署的话，就会报错冲突，因为是稳定版
但是如果是 snapshot 的话，你可以 deploy 多次，每一次都会冲掉原来的版本，表示不稳定

- `<dependency>` 下的 `<version>`

显式设置依赖的版本


### `<dependencyManagement >`

Maven 使用 dependencyManagement 元素来提供一种管理**依赖版本**的方式。通常会在一个组织或项目的最顶层的父 POM 中看到。

使用 pom.xml 中的 `<dependencyManagement > `能让所有在子项目中引用一个依赖而不用显式地写 `<version>`，Maven 会沿着父子层次向上找，直到找到一个拥有 dependencyManagement 元素的项目，然后它就会使用这个版本号。

dependencyManagement 只是声明依赖，并不引入依赖，因此子项目需要显式进行依赖引入。


### `<optional>`

默认为 false，optional 标签 设置为true 可以防止依赖继承到 jar 包，比如 spring-boot-devtools



