---
title: Struts2 编写 action
date: 2022-11-25 14:10:05
tags:
---

# 0. 参考引用

https://struts.apache.org/getting-started/coding-actions

# Introduction

编写 struts2 Action 包含几个部分:

1. 将一个 action 映射到一个类
2. 将一个结果映射到一个视图
3. 在 Action 类中编写控制器逻辑

**Action Mapping**

```xml
<action name="hello" class="org.apache.struts.helloworld.action.HelloWorldAction" method="execute">
    <result name="success">/HelloWorld.jsp</result>
</action>
```
上面的 Action 映射还指定，如果类 `HelloWorldAction` 的 `execute` 方法返回 `success`，则将视图 `HelloWorld.jsp` 返回给浏览器。


# Struts 2 Action Classes

Action 类在 MVC 模式中充当控制器。Action 类响应用户操作，执行业务逻辑（或者调用其他类执行业务逻辑），然后返回一个结果，告诉 Struts 要呈现什么视图。

Struts2 的 Action 类通常扩展了由 Struts 框架提供的 `ActionSupport` 类。类 `ActionSupport` 为最常见（例如执行、输入）提供默认实现，还实现了几个有用的 Struts2 接口。当你的 Action 类扩展 ActionSupport 类时，你的类可以覆盖默认的实现或继承它们。


**Method execute of HelloWorldAction**

```java
public String execute() throws Exception {
    messageStore = new MessageStore() ;

    helloCount++;

    return SUCCESS;
}
```


**Processing Form Input In The Action Class**

Action 类最常见的职责之一是处理表单上的用户输入，然后将处理结果提供给视图页面。为了说明这一职责，让我们假设在视图页面 `HelloWorld.jsp` 上，我们希望显示一个个人的 hello，例如: "Hello Sturts User Bruce"。

在使用 Struts 2 标签示例应用程序中，我们向 index.jsp 添加了一个 Struts 2 表单。

**Struts 2 Form Tags**
```xml
<s:form action="hello">
    <s:textfield name="userName" label="Your name" />
    <s:submit value="Submit" />
</s:form>
```

注意，Struts 2 文本字段标记的 name 属性的值，是 userName。当用户单击上述表单的提交按钮时，将执行 action hello(hello.action)。表单字段值将被发布到 Struts 2 Action 类（HelloWorldAction）。Action 类可以自动接收这些表单字段值，前提是它有一个与表单字段 name 值匹配的 public set 方法。

因此，要使 `HelloWorldAction` 类自动接收 userName 值，它必须有一个 public 方法 setUserName。

