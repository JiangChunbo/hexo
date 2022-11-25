---
title: Struts2 loadPackage 过程
date: 2022-11-25 14:46:56
tags:
- Struts2
---


项目启动后解析web.xml文件，会解析到配置的StrutsPrepareAndExecuteFilter的过滤器: 

```xml
<filter>
    <filter-name>struts2</filter-name>
    <filter-class>org.apache.struts2.dispatcher.ng.filter.StrutsPrepareAndExecuteFilter</filter-class>
</filter>
```


web容器一启动，就会初始化核心过滤器StrutsPrepareAndExecuteFilter，并执行初始化方法，初始化方法如下：

```java
public void init(FilterConfig filterConfig) throws ServletException {
    InitOperations init = new InitOperations();
    Dispatcher dispatcher = null;
    try {
        FilterHostConfig config = new FilterHostConfig(filterConfig);
        init.initLogging(config);
        dispatcher = init.initDispatcher(config);
        init.initStaticContentLoader(config, dispatcher);

        prepare = new PrepareOperations(dispatcher);
        execute = new ExecuteOperations(dispatcher);
        this.excludedPatterns = init.buildExcludedPatternsList(dispatcher);

        postInit(dispatcher, filterConfig);
    } finally {
        if (dispatcher != null) {
            dispatcher.cleanUpAfterInit();
        }
        init.cleanup();
    }
}
```
关键方法:

initDispatcher


```java
public void init() {

    if (configurationManager == null) {
        configurationManager = createConfigurationManager(DefaultBeanSelectionProvider.DEFAULT_BEAN_NAME);
    }

    try {
        init_FileManager();
        init_DefaultProperties(); // [1]
        init_TraditionalXmlConfigurations(); // [2]
        init_LegacyStrutsProperties(); // [3]
        init_CustomConfigurationProviders(); // [5]
        init_FilterInitParameters() ; // [6]
        init_AliasStandardObjects() ; // [7]

        Container container = init_PreloadConfiguration();
        container.inject(this);
        init_CheckWebLogicWorkaround(container);

        if (!dispatcherListeners.isEmpty()) {
            for (DispatcherListener l : dispatcherListeners) {
                l.dispatcherInitialized(this);
            }
        }
        errorHandler.init(servletContext);

    } catch (Exception ex) {
        if (LOG.isErrorEnabled())
            LOG.error("Dispatcher initialization failed", ex);
        throw new StrutsException(ex);
    }
}
```


### init_PreloadConfiguration