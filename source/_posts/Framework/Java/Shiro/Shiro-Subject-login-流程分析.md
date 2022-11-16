---
title: Shiro Subject login 流程分析
date: 2022-07-14 15:23:58
categories:
- 框架
tags:
- Shiro
---
# Subject login 流程分析


通常，在执行登录之前，我们必须拥有一个 `Subject` 对象，可能是从 `SecurityUtils` 类中获取：

```java
Subject subject = SecurityUtils.getSubject();
```

Shiro 框架中 `Subject` 实现类 `DelegatingSubject`，顾名思义，委托中的 `Subject`，该类本身不做 login 操作，而是将 login 操作委托给 `SecurityManager`

Subject.login() 的方法声明如下，需要传入一个 `AuthenticationToken`：
```java
void login(AuthenticationToken token) throws AuthenticationException;
```

具体实现代码如下：
```java
public void login(AuthenticationToken token) throws AuthenticationException {
    clearRunAsIdentitiesInternal();
    // 委托给 SecurityManager 执行
    Subject subject = securityManager.login(this, token);

    PrincipalCollection principals;

    String host = null;

    if (subject instanceof DelegatingSubject) {
        DelegatingSubject delegating = (DelegatingSubject) subject;
        //we have to do this in case there are assumed identities - we don't want to lose the 'real' principals:
        principals = delegating.principals;
        host = delegating.host;
    } else {
        principals = subject.getPrincipals();
    }

    if (principals == null || principals.isEmpty()) {
        String msg = "Principals returned from securityManager.login( token ) returned a null or " +
                "empty value.  This value must be non null and populated with one or more elements.";
        throw new IllegalStateException(msg);
    }
    this.principals = principals;
    this.authenticated = true;
    if (token instanceof HostAuthenticationToken) {
        host = ((HostAuthenticationToken) token).getHost();
    }
    if (host != null) {
        this.host = host;
    }
    Session session = subject.getSession(false);
    if (session != null) {
        this.session = decorate(session);
    } else {
        this.session = null;
    }
}
```

`DefaultSecurityManager` 的 login 方法如下，其中 authenticate 是执行认证的关键方法：

```java
public Subject login(Subject subject, AuthenticationToken token) throws AuthenticationException {
    AuthenticationInfo info;
    try {
        info = authenticate(token);
    } catch (AuthenticationException ae) {
        try {
            onFailedLogin(token, ae, subject);
        } catch (Exception e) {
            if (log.isInfoEnabled()) {
                log.info("onFailedLogin method threw an " +
                        "exception.  Logging and propagating original AuthenticationException.", e);
            }
        }
        throw ae; //propagate
    }

    Subject loggedIn = createSubject(token, info, subject);

    onSuccessfulLogin(token, info, loggedIn);

    return loggedIn;
}
```

`SecurityManager` 将认证的方法委托给了内部认证器 `Authenticator`：

```java
public AuthenticationInfo authenticate(AuthenticationToken token) throws AuthenticationException {
    return this.authenticator.authenticate(token);
}
```

`AbstractAuthenticator` 认证方法 `authenticate()` 代码如下，其中 `doAuthenticate()` 是认证的关键方法：

```java
public final AuthenticationInfo authenticate(AuthenticationToken token) throws AuthenticationException {

    if (token == null) {
        throw new IllegalArgumentException("Method argument (authentication token) cannot be null.");
    }

    log.trace("Authentication attempt received for token [{}]", token);

    AuthenticationInfo info;
    try {
        info = doAuthenticate(token);
        if (info == null) {
            String msg = "No account information found for authentication token [" + token + "] by this " +
                    "Authenticator instance.  Please check that it is configured correctly.";
            throw new AuthenticationException(msg);
        }
    } catch (Throwable t) {
        AuthenticationException ae = null;
        if (t instanceof AuthenticationException) {
            ae = (AuthenticationException) t;
        }
        if (ae == null) {
            //Exception thrown was not an expected AuthenticationException.  Therefore it is probably a little more
            //severe or unexpected.  So, wrap in an AuthenticationException, log to warn, and propagate:
            String msg = "Authentication failed for token submission [" + token + "].  Possible unexpected " +
                    "error? (Typical or expected login exceptions should extend from AuthenticationException).";
            ae = new AuthenticationException(msg, t);
            if (log.isWarnEnabled())
                log.warn(msg, t);
        }
        try {
            notifyFailure(token, ae);
        } catch (Throwable t2) {
            if (log.isWarnEnabled()) {
                String msg = "Unable to send notification for failed authentication attempt - listener error?.  " +
                        "Please check your AuthenticationListener implementation(s).  Logging sending exception " +
                        "and propagating original AuthenticationException instead...";
                log.warn(msg, t2);
            }
        }


        throw ae;
    }

    log.debug("Authentication successful for token [{}].  Returned account [{}]", token, info);

    notifySuccess(token, info);

    return info;
}
```

通常，我们可以认为 Shiro 框架中的认证器就是 `ModularRealmAuthenticator`，因为没有其他实现类了，其 `doAuthenticate()` 方法如下：

```java
protected AuthenticationInfo doAuthenticate(AuthenticationToken authenticationToken) throws AuthenticationException {
    assertRealmsConfigured();
    Collection<Realm> realms = getRealms();
    if (realms.size() == 1) {
        // 只有 1 个 Realm
        return doSingleRealmAuthentication(realms.iterator().next(), authenticationToken);
    } else {
        // 具有多个 Realm
        return doMultiRealmAuthentication(realms, authenticationToken);
    }
}
```

> 如果配置了多个 `Realm`，会使用到认证策略 `AuthenticationStrategy`，认证策略也有很多种，其中的 `AllSuccessfulStrategy` 要求所有的 `Realm` 都必须认证成功，并且会合并所有的 `AuthenticationInfo` 中的 `PrincipalCollection` 形成 `MutablePrincipalCollection`，凭证 `credential` 也会合并为集合。