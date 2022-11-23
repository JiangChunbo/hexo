---
title: MyBatis Generator Eclipse 插件使用
date: 2022-11-18 17:28:49
tags:
---

# 参考

http://mybatis.org/generator/generatedobjects/javamodel.html

http://mybatis.org/generator/generatedobjects/javaclient.html

# Eclipse 插件

> 对于 java model 和 java client，在不使用 Eclipse MBG 插件的时候，不会合并这些 Java 文件。 这也是官方所告知的。

Eclipse Market 下载 MyBatis Generator，一般版本跟随 MyBatis Generator。

安装完插件之后：

- 你可以右键新建 `MyBatis Generator Configuration File`。
- 你可以右键 `generatorConfig.xml` > `Run As` > `1 Run MyBatis Generator` 执行


# 注意点

## Classpath 配置问题

如果你使用了 `<properties>` 配置，如下所示:

```xml
<properties resource="generatorConfig.properties" />
```

那么你要确保类路径存在 `generatorConfig.properties`，否则会提示找不到。此时，你需要将相关的类路径项目引入 `Run As` > `Run Configurations...`


配置了 JDBC，那么也要在 `Run As` > `Run Configurations...` 将 JDBC 的 jar 包添加到 classpath。


## targetProject 问题

使用 Eclipse 插件时，targetProject 不能配置成绝对路径，否则会报路径找不到。