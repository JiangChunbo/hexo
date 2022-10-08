---
title: JWT 笔记
date: 2022-10-03 10:17:39
tags:
---
# 参考

https://jwt.io/introduction

# 什么是 JSON Web Token?

JSON Web Token(JWT) 是一个开放标准（[RFC 7519](https://www.rfc-editor.org/rfc/rfc7519.html)），它定义了一种紧凑和自包含的方式，用于以 JSON 对象哎各方之间安全地传输信息。此信息可以进行验证和信任，因为它是经过数字签名地。JWT 可以使用 secret 或者使用 RSA 或 ECDSA 地公钥/私钥对进行签名。

虽然可以对 JWT 进行加密，以便在各方之间提供保密性，但是我们将重点关注已签名的令牌。签名令牌可以验证其中包含的声明的完整性，而加密令牌可以向其他方隐藏这些声明。当使用公钥/私钥对对令牌进行签名时，该签名还证明只有持有私钥的一方才时对其进行签名的一方。

# JSON Web Token 结构是什么?

在其紧凑的形式中，JSON Web Token 由以点（`.`）分隔的三个部分组成，它们是:

- Header
- Payload
- Signature

因此，JWT 通常看起来像下面这样:

```bash
xxxxx.yyyyy.zzzzz
```

## Header

Header 通常由两部分组成：令牌（即 JWT）的类型，以及所使用的签名算法（如: HMAC SHA256 或者 RSA）。

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```
然后，对这个 JSON 进行 Base64Url 编码，形成 JWT 的第一部分。

## Payload

令牌的第二部分是有效载荷，其中包含声明。声明是关于实体（通常是用户）和其他数据的语句。有三种类型的声明：registered，public，以及 private 声明

- **Registered claims**: 这是一组预定义的声明，它们不是强制性的，而是推荐的，以提供一组有用的、可互操作的声明。其中一些是: iss（发行者），exp（过期时间），sub（主题），aud（手中）等其他。

> 注意，声明的名字只有三个字符，因为 JWT 旨在紧凑。

- **Public claims**: 使用 JWT 的人可以随意定义它们。但是为了避免冲突，应该在 IANA JSON Web Token 注册表中定义它们，或者将它们定义为包含抗冲突名称空间的 URI。

- **Private claims**: 这些是创建用于在同意使用它们的各方之间共享信息的自定义声明，既不是 registered 声明，也不是 public 声明


```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true
}
```

然后，对有效载荷进行 **Base64Url** 编码，形成 JSON Web 令牌的第二部分。

> 请注意，对于已经签名的令牌，这些信息虽然受到保护，不会被篡改，但任何人都可以阅读。除非加密，否则不要将机密信息放在 JWT 的有效载荷或头元素中。


## Signature

要创建签名部分，你必须获取编码的 header，编码的有效载荷，secret，表头中指定的算法，并对其签名。

例如，如果你想使用 HMAC SHA256 算法，签名将按以下方式创建: 

```java
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

该签名用于验证消息在执行过程中没有被更改，并且，对于使用私钥签名的令牌，它还可以验证 JWT 的发送方是否就是它所说的那个人。


# JSON Web Token 如何工作

在身份认证中，当用户使用凭据成功登陆时，将返回一个 JSON Web Token。由于令牌是凭证，因此必须非常小心地注意，防止出现安全问题。一般来说，令牌的保存时间不应超过所需时间。

由于缺乏安全性，也不应将敏感会话数据存储在浏览器存储中。

无论何时用户想要访问受保护的路由或资源，用户代理都应该发送 JWT，通常在 Authorization 头部使用 Bearer 模式。header 的内容应如下: 

```json
Authorization: Bearer <token>
```

在某些情况下，这可以是无状态授权机制。服务器的受保护路由将在 Authorization 头部检查有效的 JWT，如果存在有效的 JWT，则允许用户访问受保护的资源。如果 JWT 包含必要的数据，那么查询数据库以执行某些操作的需求可能会减少，尽管情况可能并非总是如此。

注意，如果通过 HTTP 头发送 JWT 令牌，应该尽量避免它们变得太大。有些服务器不接受超过 8KB 的报头。如果你试图在 JWT 令牌中嵌入太多信息（比如通过包含用户所有的权限），那么你可能需要另一种解决方案，比如 Auth0 Fine-Grined Authorization。

如果令牌是在 Authorization 头部发送的，那么跨域资源共享（CORS）就不是问题，因为它不适用 Cookie。