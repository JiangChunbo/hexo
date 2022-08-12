---
title: OAuth 2.0 Client 设计
date: 2022-08-04 23:27:25
tags:
---


## Client


### 依赖

```xml
<!-- spring boot web starter -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.oltu.oauth2</groupId>
    <artifactId>org.apache.oltu.oauth2.client</artifactId>
    <version>1.0.2</version>
</dependency>
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-spring-boot-web-starter</artifactId>
    <version>1.9.1</version>
</dependency>
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.1.0</version>
</dependency>
```


### 实体

- AuthorizationServerConfig
接口，提供各个授权服务器的基本配置

- AuthorizationServerToken
接口，提供各个授权服务器的 token 访问

- AuthorizationServer
实体类，实现了 token 访问和基本信息访问


`AuthorizationServer.java` 用于表示该客户端接入的某一个授权服务器。

考虑到客户端可能接入多个授权服务器，因此需要维护客户端在不同授权服务器中的基础信息。

基础信息包括：

- name, 标识不同授权服务器
- client_id, 该授权服务器中的 client_id
- client_secret, 该授权服务器中 client_secret
- access_token_uri, 该授权服务器暴露的 token 获取端点
- user_info_uri, 该资源服务器暴露的 user_info 获取端点
- redirect_uri, 跳转地址
- ...

#### AuthorizationServerConfig

```java
package com.jiangchunbo.oauth2.client.entity;

public interface AuthorizationServerConfig {
    /**
     * 返回授权服务器的唯一标识，如 baidu、tencent
     *
     * @return name
     */
    String getName();

    /**
     * 授权服务器的 authorize 端点地址
     *
     * @return authorize_uri
     */
    String getAuthorizeUri();

    /**
     * 授权服务器的 scope
     *
     * @return scope
     */
    String getScope();

    /**
     * 授权地址额外的自定义参数
     *
     * @return params
     */
    String getCustomParams();

    /**
     * access_token 获取地址
     *
     * @return access_token_uri
     */
    String getAccessTokenUri();

    /**
     * 该授权服务器中的 client_id
     *
     * @return client_id
     */
    String getClientId();

    /**
     * 该授权服务器中 client_secret
     *
     * @return client_secret
     */
    String getClientSecret();

    /**
     * 跳转地址
     *
     * @return redirect_uri
     */
    String getRedirectUri();

    /**
     * 获取用户信息地址
     *
     * @return user_info_uri
     */
    String getUserInfoUri();
}
```

#### AuthorizationServerToken

```java
package com.jiangchunbo.oauth2.client.entity;

public interface AuthorizationServerToken {

    /**
     * 返回存储的 access token
     *
     * @return access token
     */
    String getAccessToken();

    /**
     * 返回存储的 refresh token
     *
     * @return refresh token
     */
    String getRefreshToken();

    /**
     * 返回存储的 access token 过期时间
     *
     * @return access token 过期时间
     */
    String getAccessTokenExpire();

    /**
     * 返回存储的 refresh token 过期时间
     *
     * @return refresh token 过期时间
     */
    String getRefreshTokenExpire();
}
```

#### AuthorizationServer

```java
public class AuthorizationServer implements AuthorizationServerConfig, AuthorizationServerToken {
    private Integer id;
    private String name;
    private String authorizeUri;
    private String scope;
    private String customParams;
    private String accessTokenUri;
    private String clientId;
    private String clientSecret;
    private String redirectUri;
    private String userInfoUri;
    private String accessToken;
    private String refreshToken;
    private String accessTokenExpire;
    private String refreshTokenExpire;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }


    @Override
    public String getCustomParams() {
        return customParams;
    }

    public void setCustomParams(String customParams) {
        this.customParams = customParams;
    }

    @Override
    public String getScope() {
        return scope;
    }

    public void setScope(String scope) {
        this.scope = scope;
    }

    public String getAccessTokenUri() {
        return accessTokenUri;
    }

    public void setAccessTokenUri(String accessTokenUri) {
        this.accessTokenUri = accessTokenUri;
    }

    public String getClientId() {
        return clientId;
    }

    public void setClientId(String clientId) {
        this.clientId = clientId;
    }

    public String getClientSecret() {
        return clientSecret;
    }

    public void setClientSecret(String clientSecret) {
        this.clientSecret = clientSecret;
    }

    @Override
    public String getRedirectUri() {
        return redirectUri;
    }

    public void setRedirectUri(String redirectUri) {
        this.redirectUri = redirectUri;
    }

    @Override
    public String getUserInfoUri() {
        return userInfoUri;
    }

    public void setUserInfoUri(String userInfoUri) {
        this.userInfoUri = userInfoUri;
    }

    public String getName() {
        return name;
    }

    public void setAuthorizeUri(String authorizeUri) {
        this.authorizeUri = authorizeUri;
    }

    @Override
    public String getAuthorizeUri() {
        return this.authorizeUri;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getAccessToken() {
        return accessToken;
    }

    public void setAccessToken(String accessToken) {
        this.accessToken = accessToken;
    }

    public String getRefreshToken() {
        return refreshToken;
    }

    public void setRefreshToken(String refreshToken) {
        this.refreshToken = refreshToken;
    }

    public String getAccessTokenExpire() {
        return accessTokenExpire;
    }

    public void setAccessTokenExpire(String accessTokenExpire) {
        this.accessTokenExpire = accessTokenExpire;
    }

    public String getRefreshTokenExpire() {
        return refreshTokenExpire;
    }

    public void setRefreshTokenExpire(String refreshTokenExpire) {
        this.refreshTokenExpire = refreshTokenExpire;
    }
}
```


### 授权服务器配置 Bean

创建 Bean 时调用 init 方法从数据库加载所有配置到内存

> 也可以考虑从配置文件加载

```java
@Component
public class AuthorizationServerConfigProperties extends ConcurrentHashMap<String, AuthorizationServerConfig> {

    @Resource
    AuthorizationServerMapper authorizationServerMapper;

    @PostConstruct
    public void init() {
        final List<AuthorizationServerConfig> configs = authorizationServerMapper.selectAllConfig();
        if (configs != null && !configs.isEmpty()) {
            for (AuthorizationServerConfig config : configs) {
                this.put(config.getName(), config);
            }
        }
    }
}
```

### DAO(Mapper)

```java
@Mapper
public interface AuthorizationServerMapper {

    /**
     * 依据授权服务器的标识符从数据库访问信息
     *
     * @param name 授权服务器标识符
     * @return 基本配置信息
     */
    AuthorizationServerConfig selectConfigByName(String name);

    /**
     * 从数据库取出所有的授权服务器配置信息
     *
     * @return 授权服务器配置信息列表
     */
    List<AuthorizationServerConfig> selectAllConfig();

    /**
     * 依据授权服务器的标识符从数据库取出信息
     *
     * @param name 授权服务器标识符
     * @return 授权服务器信息
     */
    AuthorizationServer selectByName(String name);

    /**
     * 根据主键 Id 更新存储的授权服务器 token 信息
     *
     * @param authorizationServer token 信息
     * @param id                  主键 id
     * @return 影响行数
     */
    Integer updateTokenById(@Param("authorizationServer") AuthorizationServer authorizationServer, @Param("id") Integer id);
}
```

```xml
<?xml version="1.0" encoding="utf-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.jiangchunbo.oauth2.client.mapper.AuthorizationServerMapper">
    <update id="updateTokenById">
        update `authorization_server`
        set `access_token`=#{authorizationServer.accessToken},
            `access_token_expire`=#{authorizationServer.accessTokenExpire},
            `refresh_token`=#{authorizationServer.refreshToken},
            `refresh_token_expire` = #{authorizationServer.refreshTokenExpire}
        where `id` = #{id}
    </update>
    <select id="selectConfigByName" resultType="com.jiangchunbo.oauth2.client.entity.AuthorizationServer">
        select *
        from `authorization_server`
        where `name` = #{name}
    </select>
    <select id="selectByName" resultType="com.jiangchunbo.oauth2.client.entity.AuthorizationServer">
        select *
        from `authorization_server`
        where `name` = #{name}
    </select>
    <select id="selectAllConfig" resultType="com.jiangchunbo.oauth2.client.entity.AuthorizationServer">
        select *
        from `authorization_server` where `is_del`=0
    </select>
</mapper>
```


### Service

```java
public interface AuthorizationServerService {

    /**
     * 更新 token 相关信息
     *
     * @param authorizationServer token 信息
     * @param id                  主键 id
     * @return 影响函数
     */
    Integer updateTokenById(AuthorizationServer authorizationServer, Integer id);

    /**
     * 使用 code 请求 access_token；如果缓存 token 未过期，则使用缓存 token
     *
     * @param token code token
     * @return access_token
     * @throws Exception 异常
     */
    String getAccessToken(OAuthCodeToken token) throws Exception;
}
```

```java
@Service
public class AuthorizationServerServiceImpl implements AuthorizationServerService {

    @Resource
    AuthorizationServerMapper authorizationServerMapper;


    @Override
    public Integer updateTokenById(AuthorizationServer authorizationServer, Integer id) {
        return authorizationServerMapper.updateTokenById(authorizationServer, id);
    }

    @Override
    public String getAccessToken(OAuthCodeToken token) throws Exception {
        final String name = token.getName();
        final AuthorizationServer server = authorizationServerMapper.selectByName(name);
        if (server == null) {
            throw new RuntimeException("未找到 " + name + " 相关的配置");
        }
        if (!StringUtils.isEmpty(server.getAccessToken()) && new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").parse(server.getAccessTokenExpire()).after(new Date())) {
            return server.getAccessToken();
        }

        final String code = token.getCode();
        final OAuthClientRequest accessTokenRequest = OAuthClientRequest.tokenLocation(server.getAccessTokenUri())
                .setGrantType(GrantType.AUTHORIZATION_CODE)
                .setClientId(server.getClientId())
                .setClientSecret(server.getClientSecret())
                .setCode(code)
                .setRedirectURI(server.getRedirectUri())
                .buildQueryMessage();
        OAuthClient oAuthClient = new OAuthClient(new URLConnectionClient());
        OAuthAccessTokenResponse oAuthResponse = oAuthClient.accessToken(accessTokenRequest, OAuth.HttpMethod.GET);
        String accessToken = oAuthResponse.getAccessToken();
        final String refreshToken = oAuthResponse.getRefreshToken();
        Long expiresIn = oAuthResponse.getExpiresIn();

        final String accessTokenExpire = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss")
                .format(new Date(System.currentTimeMillis() + expiresIn * 1000));

        final String refreshTokenExpire = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss")
                .format(new Date(System.currentTimeMillis() + 10L * 365 * 24 * 60 * 60 * 1000));

        final AuthorizationServer authorizationServer = new AuthorizationServer();
        authorizationServer.setAccessToken(accessToken);
        authorizationServer.setRefreshToken(refreshToken);
        authorizationServer.setAccessTokenExpire(accessTokenExpire);
        authorizationServer.setRefreshTokenExpire(refreshTokenExpire);
        this.updateTokenById(authorizationServer, server.getId());
        return accessToken;
    }
}
```



### Shiro 相关的准备

### Shiro 配置类

```java
@Configuration
@ConditionalOnProperty(name = "shiro.web.enabled", matchIfMissing = true)
public class ShiroConfiguration {

    @Bean
    public Realm baiduOAuthRealm() {
        final BaiduOAuthRealm baiduOAuthRealm = new BaiduOAuthRealm();
        baiduOAuthRealm.setAuthenticationTokenClass(OAuthCodeToken.class);
        return baiduOAuthRealm;
    }

    @Bean("oauth2")
    public OAuth2AuthenticationFilter oAuth2AuthenticationFilter() {
        return new OAuth2AuthenticationFilter();
    }

    @Bean
    public FilterRegistrationBean<OAuth2AuthenticationFilter> oAuth2AuthenticationFilterRegistrationBean() {
        final FilterRegistrationBean<OAuth2AuthenticationFilter> registrationBean = new FilterRegistrationBean<>();
        registrationBean.setFilter(oAuth2AuthenticationFilter());
        registrationBean.setEnabled(false);
        return registrationBean;
    }

    /**
     * 配置 chain definition
     *
     * @return chainDefinition
     */
    @Bean
    protected ShiroFilterChainDefinition shiroFilterChainDefinition() {
        DefaultShiroFilterChainDefinition chainDefinition = new DefaultShiroFilterChainDefinition();
        chainDefinition.addPathDefinition("/index.html", "oauth2");
        chainDefinition.addPathDefinition("/**", "anon");
        return chainDefinition;
    }
}
```

### AuthenticatingFilter

```java
public class OAuth2AuthenticationFilter extends AuthenticatingFilter {

    private final static String CODE_PARAM = "code";

    private final static String NAME_PARAM = "name";

    /**
     * 当 isAccessDenied 返回 true 时，回退到该方法，该方法一般会执行 login 逻辑
     *
     * @param request  请求
     * @param response 响应
     * @return 是否继续
     * @throws Exception executeLogin 抛出的一些异常
     */
    @Override
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
        Subject subject = getSubject(request, response);
        if (!subject.isAuthenticated() && !StringUtils.isEmpty(request.getParameter(CODE_PARAM))) {
            return executeLogin(request, response);
        }
        return true;
    }

    /**
     * 执行 login 逻辑的时候创建的 token
     *
     * @param request  请求
     * @param response 响应
     * @return 执行 login 的 token
     */
    @Override
    protected AuthenticationToken createToken(ServletRequest request, ServletResponse response) {
        String code = request.getParameter(CODE_PARAM);
        code = !StringUtils.isEmpty(code) ? code : "";
        final String name = request.getParameter(NAME_PARAM);
        return new OAuthCodeToken(name, code);
    }

    /**
     * 登录失败的逻辑
     *
     * @param token    token
     * @param e        异常信息
     * @param request  请求
     * @param response 响应
     * @return 是否继续过滤器
     */
    @Override
    protected boolean onLoginFailure(AuthenticationToken token, AuthenticationException e, ServletRequest request, ServletResponse response) {
        try {
            final String body = OAuthResponse.errorResponse(HttpServletResponse.SC_BAD_REQUEST)
                    .setError("错误")
                    .setErrorDescription(e.getMessage())
                    .buildJSONMessage().getBody();
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.setCharacterEncoding(StandardCharsets.UTF_8.name());
            response.getWriter().print(body);
            response.getWriter().flush();
        } catch (IOException | OAuthSystemException ex) {
            // TODO
            throw new RuntimeException(ex);
        }
        return true;
    }
}
```


### AuthenticationToken

存储 code 以及 name 标识授权服务器

```java
public class OAuthCodeToken implements AuthenticationToken {
    /**
     * 授权码
     */
    private final String code;

    /**
     * 每个授权服务器方的名字标识，如: baidu、tencent
     */
    private final String name;

    public OAuthCodeToken(String name, String code) {
        this.name = name;
        this.code = code;
    }


    @Override
    public Object getPrincipal() {
        return name;
    }

    @Override
    public Object getCredentials() {
        return code;
    }

    public String getName() {
        return name;
    }

    public String getCode() {
        return code;
    }
}
```


### Realm

`OAuthRealm` 提供基本的 OAuth2 获取用户名的流程

```java
public abstract class OAuthRealm extends AuthenticatingRealm {

    @Resource
    AuthorizationServerConfigProperties authorizationServerProperties;

    @Resource
    AuthorizationServerService authorizationServerService;

    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        // 确保 token 中的 credential 与 authentication info 的 credentials 一致，否则后面验证会出错
        OAuthCodeToken codeToken = (OAuthCodeToken) token;
        SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo();
        try {
            String username = extractUsername(codeToken);
            authenticationInfo.setPrincipals(new SimplePrincipalCollection(username, getName()));
            authenticationInfo.setCredentials(codeToken.getCode());
            return authenticationInfo;
        } catch (Exception e) {
            return authenticationInfo;
        }
    }

    protected abstract String extractUsername(OAuthCodeToken codeToken) throws Exception;
}
```

`BaiduOAuthRealm` 实现了 `OAuthRealm`，返回 netdisk_name 名称

```java
public class BaiduOAuthRealm extends OAuthRealm {
    @Override
    protected String extractUsername(OAuthCodeToken codeToken) throws Exception {
        OAuthClient oAuthClient = new OAuthClient(new URLConnectionClient());
        final String accessToken = authorizationServerService.getAccessToken(codeToken);
        final AuthorizationServerConfig serverConfig = authorizationServerProperties.get(codeToken.getName());
        try {
            // 保存 access_token 和 expires in
            OAuthClientRequest userInfoRequest = new OAuthBearerClientRequest(serverConfig.getUserInfoUri())
                    .setAccessToken(accessToken)
                    .buildQueryMessage();
            OAuthResourceResponse resourceResponse = oAuthClient.resource(userInfoRequest, OAuth.HttpMethod.GET, OAuthResourceResponse.class);
            final Map<String, Object> data = JSONUtils.parseJSON(resourceResponse.getBody());
            return data.get("netdisk_name").toString();
        } catch (OAuthSystemException | OAuthProblemException e) {
            throw new AuthenticationException(e.getMessage());
        }
    }
}
```