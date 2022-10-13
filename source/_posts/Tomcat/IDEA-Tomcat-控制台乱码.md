---
title: IDEA Tomcat 控制台乱码
date: 2022-10-12 15:14:59
tags:
---


# 环境


```bash
D:\apache-tomcat-9.0.65\bin>catalina version
Using CATALINA_BASE:   "D:\apache-tomcat-9.0.65"
Using CATALINA_HOME:   "D:\apache-tomcat-9.0.65"
Using CATALINA_TMPDIR: "D:\apache-tomcat-9.0.65\temp"
Using JRE_HOME:        "C:\Program Files\Java\jdk1.8.0_341"
Using CLASSPATH:       "D:\apache-tomcat-9.0.65\bin\bootstrap.jar;D:\apache-tomcat-9.0.65\bin\tomcat-juli.jar"
Using CATALINA_OPTS:   ""
Server version: Apache Tomcat/9.0.65
Server built:   Jul 14 2022 12:28:53 UTC
Server number:  9.0.65.0
OS Name:        Windows 11
OS Version:     10.0
Architecture:   amd64
JVM Version:    1.8.0_341-b10
JVM Vendor:     Oracle Corporation
```


# Tomcat 日志配置

配置文件是 `conf/logging.properties`。



```properties
############################################################
# Handler specific properties.
# Describes specific configuration info for Handlers.
############################################################

1catalina.org.apache.juli.AsyncFileHandler.level = FINE
1catalina.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
1catalina.org.apache.juli.AsyncFileHandler.prefix = catalina.
1catalina.org.apache.juli.AsyncFileHandler.maxDays = 90
1catalina.org.apache.juli.AsyncFileHandler.encoding = GBK

2localhost.org.apache.juli.AsyncFileHandler.level = FINE
2localhost.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
2localhost.org.apache.juli.AsyncFileHandler.prefix = localhost.
2localhost.org.apache.juli.AsyncFileHandler.maxDays = 90
2localhost.org.apache.juli.AsyncFileHandler.encoding = GBK

3manager.org.apache.juli.AsyncFileHandler.level = FINE
3manager.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
3manager.org.apache.juli.AsyncFileHandler.prefix = manager.
3manager.org.apache.juli.AsyncFileHandler.maxDays = 90
3manager.org.apache.juli.AsyncFileHandler.encoding = GBK

4host-manager.org.apache.juli.AsyncFileHandler.level = FINE
4host-manager.org.apache.juli.AsyncFileHandler.directory = ${catalina.base}/logs
4host-manager.org.apache.juli.AsyncFileHandler.prefix = host-manager.
4host-manager.org.apache.juli.AsyncFileHandler.maxDays = 90
4host-manager.org.apache.juli.AsyncFileHandler.encoding = GBK

java.util.logging.ConsoleHandler.level = FINE
java.util.logging.ConsoleHandler.formatter = org.apache.juli.OneLineFormatter
java.util.logging.ConsoleHandler.encoding = GBK
```

- **1catalina**: 程序的输出，tomcat 的日志输出等
- **2localhost**: 程序异常没有被捕获的时候抛出的地方
- **3manager**: manager 项目专用
- **4host-manager**: manager 项目专用


> 以上这几个配置初始可能是 **UTF-8**，而 Windows 控制台的编码通常是 **GBK**，为了正常显示，建议全部修改为 **GBK**。

