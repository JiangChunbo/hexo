---
title: OAuth2 设计
date: 2022-08-03 20:02:33
tags:
---


# OAuth2 设计

## 暴露的端点

- GET /oauth/authorize
该端点采用 GET 请求，主要是让用户能够跳转到授权页面。

- GET /oauth/confirm_access
提供用户确认授权的视图

- POST /oauth/authorize
用户确认授权