---
title: MyBatis Generator Eclipse 插件使用
date: 2022-11-18 17:28:49
tags:
---

# 参考


# Eclipse 插件

MBG 不会合并 Java 文件，他可以覆盖已经存在的文件或者保存新生成的文件为一个不同的唯一的名字。 您可以手动合并这些更改。当您使用Eclipse 插件时, MBG 可以自动合并 Java 文件.

Eclipse Market 下载 MyBatis Generator，一般版本跟随 MyBatis Generator。

安装完插件之后：

- 你可以右键新建 `MyBatis Generator Configuration File`。
- 你可以右键 `generatorConfig.xml` > `Run As` > `1 Run MyBatis Generator` 执行