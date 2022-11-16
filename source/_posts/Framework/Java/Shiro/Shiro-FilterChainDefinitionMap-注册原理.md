---
title: Shiro FilterChainDefinitionMap 注册原理
date: 2022-07-14 14:28:34
categories:
- 框架
tags:
- Shiro
---
# Shiro FilterChainDefinitionMap 注册原理

在进行 `FilterChainDefinitionMap` 配置的时候，需要准备两个字符串，分别称之为 `antPath` 和 `definition`。以如下的配置为例：
```java
DefaultShiroFilterChainDefinition chainDefinition = new DefaultShiroFilterChainDefinition();
chainDefinition.addPathDefinition("/url", "authc, roles[admin,user], perms[file:edit]");
```

> 第一个字符串可以认为是路径（可以包含通配符），第二个字符串是过滤器链定义

对于 `FilterChainDefinitionMap` 中每个 filter Chain Definition 的处理都是在 `DefaultFilterChainManager` 进行的，主要关注如下方法：
```java
// 此处的 chainName 就是 antPath 
public void createChain(String chainName, String chainDefinition) {
    if (!StringUtils.hasText(chainName)) {
        throw new NullPointerException("chainName cannot be null or empty.");
    }
    if (!StringUtils.hasText(chainDefinition)) {
        throw new NullPointerException("chainDefinition cannot be null or empty.");
    }

    if (log.isDebugEnabled()) {
        log.debug("Creating chain [" + chainName + "] with global filters " + globalFilterNames + " and from String definition [" + chainDefinition + "]");
    }

    // 首先以此添加全局 filter，比如 InvalidRequestFilter
    if (!CollectionUtils.isEmpty(globalFilterNames)) {
        globalFilterNames.stream().forEach(filterName -> addToChain(chainName, filterName));
    }


    // 对值进行标记解析，以获得最后特定于过滤器的配置项
    // 这里以半角逗号(,) 作为分隔符，忽略两边空白符
    // 例如对于值：
    //     "authc, roles[admin,user], perms[file:edit]"
    // 最终的标记数组为：
    //     { "authc", "roles[admin,user]", "perms[file:edit]" }
    //
    String[] filterTokens = splitChainDefinition(chainDefinition);

    // 每个标记都是特定于每个过滤器的
    // 即，这些配置可能是过滤器约定好的，你需要熟悉这些用法
    // 譬如 roles[admin,user] 括号 [] 之间代表着角色
    //      perms[file:edit] 括号 [] 之间代表权限，权限又用 : 隔开，前者表示操作对象，后者表示操作类型
    // 剥离 name，提取括号 [] 之间的特定于过滤器的配置
    for (String token : filterTokens) {
        // 一定是一个包含 2 个元素的数组，第一个是 filter name，第二个 config 可能是 null
        // [ "authc", null ]
        // [ "roles", "admin,user" ]
        // [ "perms", "file:edit" ]
        String[] nameConfigPair = toNameConfigPair(token);

        // 现在，我们拥有过滤器名称，路径，以及特定于路径的配置（可能是 null，也就是没有配置）
        addToChain(chainName, nameConfigPair[0], nameConfigPair[1]);
    }
}
```


```java
public void addToChain(String chainName, String filterName, String chainSpecificFilterConfig) {
    if (!StringUtils.hasText(chainName)) {
        throw new IllegalArgumentException("chainName cannot be null or empty.");
    }
    Filter filter = getFilter(filterName);
    if (filter == null) {
        throw new IllegalArgumentException("There is no filter with name '" + filterName +
                "' to apply to chain [" + chainName + "] in the pool of available Filters.  Ensure a " +
                "filter with that name/path has first been registered with the addFilter method(s).");
    }

    // 这里主要就是把配置字符串，如：admin,user 按照半角逗号（,）分割
    // 将得到的 chainName 和 [admin, user] 放入过滤器中  Map<String, Object> appliedPaths 结构中
    // 之所以这样做，是因为对于不同的路径，可能会配置同一个过滤器的不同过滤规则
    // 比如： 学校列表学校管理员与区级管理员可访问，区列表仅区级管理员访问
    //      /school/list  roles[school_admin, area_admin]
    //      /area/list    roles[area_admin]
    applyChainConfig(chainName, filter, chainSpecificFilterConfig);

    // ensureChain 顾名思义，表示确保 chain 存在，如果不存在就新建一个
    NamedFilterList chain = ensureChain(chainName);
    chain.add(filter);
}
```