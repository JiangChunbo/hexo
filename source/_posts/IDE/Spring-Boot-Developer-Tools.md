---
title: Spring Boot Developer Tools
date: 2022-10-09 08:36:17
tags:
---

# 参考引用

关于新版本 2021 `compiler.automake.allow.when.app.running` 自动构建选项消失，如何热部署 Spring Boot

https://youtrack.jetbrains.com/issue/IDEA-274903

https://blog.csdn.net/qq_19007335/article/details/124069635


Spring Boot dev tools 使用
https://www.cnblogs.com/farajmujey/p/14344148.html

# Developer Tools
## 依赖 

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>
</dependency>
```

## 默认属性

参考 [DevToolsPropertyDefaultsPostProcessor.java](https://github.com/spring-projects/spring-boot/blob/v2.1.0.RELEASE/spring-boot-project/spring-boot-devtools/src/main/java/org/springframework/boot/devtools/env/DevToolsPropertyDefaultsPostProcessor.java)


如果你需要禁用 devtools 提供的默认属性，请设置 `spring.devtools.add-properties=false`

## 原理

devtools 会监视 classpath 下面的资源，当 classpath 下的文件发生了变化，devtools 会重启服务。

> 由于 IDEA 保存并不会同步到运行时编译目录，因此需要手动构建。Eclipse 则不必。


devtools 具有独立类加载器。


# IDEA 的使用

由于 IDEA 在保存源码的时候并不会像 Eclipse 那样立即同步到**编译输出目录**，因此需要设置 `On 'Update' action` 选项:

<img src="https://img-blog.csdnimg.cn/3c96ff245b454b73845cfdb0c27b306a.png">

> 比如，选择 Update resources，那么你修改 resources 文件，IDEA 就会同步改动到 target；选择 Update classes and resources，那么你修改 java 文件和 resources 文件就会同步到 target。

不过，默认情况下，IDEA 并不会允许**运行时**重新构建**编译输出目录**，因此你需要手动开启:


<img src="https://img-blog.csdnimg.cn/69d37bbe104348968d675ae1d6f5e129.png">

> 上图使用的是 2022.2.2 UE 版本，旧版本可能有所区别，比如要从 Registry... 中开启。