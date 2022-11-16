---
title: Shiro DefaultFilter 使用笔记
date: 2022-07-14 11:17:34
categories: 
- 框架
tags:
- Shiro
---


# Shiro DefaultFilter 使用笔记

## anon

AnonymousFilter

匿名过滤器，请求 `onPreHandle()` 直接通过。



## authc

FormAuthenticationFilter

onAccessDenied 方法分 loginUrl 处理和非 loginUrl 处理：
(1) 当请求是 loginUrl 时，其中又根据是否为 POST 请求判断是否是 login 页面请求还是，login 提交。
(2) 当请求是非 loginUrl 时，向 session 存储了一个属性 shiroSavedRequest， 然后跳转到登录页面。



## authcBasic

BasicHttpAuthenticationFilter

onAccessDenied 逻辑：
1. 通过请求头 Authorization 判断是否为 login 请求，如果是，执行 executeLogin 逻辑，获取 username, password 构造 AuthenticationToken 进行 Realm 认证。
2. 如果不是 login 请求，或者登录失败，发送质询


## invalidRequest

InvalidRequestFilter

默认全局过滤器，过滤一些非法请求