---
title: PHPStorm XDebug 调试笔记
date: 2022-09-23 13:54:37
tags:
- PHP
- Debug
---


# PHPStorm

## 1. 配置准备

> **tips** 可能你的 php 已经下载了 XDebug，并且在 `php.ini` 有所配置，但是版本可能并不是恰当的。建议按照下面的步骤重新下载。


下载 Xdebug 插件, 访问[XDebug 下载地址](https://xdebug.org/wizard)将控制台命令 `php -i` 输入进去就能提供适合的 XDebug 版本:

> 除了控制台输出，其他输出也可以，具体看官网的指引


<img src="https://img-blog.csdnimg.cn/1d950edc3b83480abea7b3b4be9eeff9.png">


> **补充** 怎么快速粘贴 `php -i` 的输出内容, 可以考虑重定向输出到文件，比如:
> ```text
> C:\Users\94508>php -i > Desktop\phpinfo.txt
> ```
> 然后全选文本复制即可




将下载的文件放到特定目录，比如: `D:\wamp64\bin\php\php7.1.33\zend_ext`


配置 php.ini 的 XDebug 路径，需要找到 `[xdebug]` 节点:

```ini
[xdebug]
zend_extension ="d:/wamp64/bin/php/php7.1.33/zend_ext/php_xdebug-2.9.8-7.1-vc14-x86_64.dll"
xdebug.remote_enable = on
xdebug.profiler_enable = off
xdebug.profiler_enable_trigger = Off
xdebug.profiler_output_name = cachegrind.out.%t.%p
xdebug.profiler_output_dir ="d:/wamp64/tmp"
xdebug.show_local_vars=0
```

## 2. PHP 脚本调试

这种调试一般不用配置就可以进行脚本调试


之后就可以在 PHPStorm 通过 debug 按钮启动调试了

<img src="https://img-blog.csdnimg.cn/10095383fff44c50ab79054214a43a8f.png">

> 这种脚本调试与是否 Start Listening for PHP Debug Connections 无关（就是那个📞按钮）



## 3. 客户端调试

这种调试方法也就是用 Postman 此类工具配合 PHPStorm 调试


在 `phpForApache.ini` 文件配置端口号 `xdebug.remote_port`:

> - 注意别改错了，不是改 `php.ini`;
> - 这个端口随便配置，只要没有被占用即可

```ini
[xdebug]
zend_extension ="d:/wamp64/bin/php/php7.1.33/zend_ext/php_xdebug-2.9.8-7.1-vc14-x86_64.dll"
xdebug.remote_enable = on
xdebug.profiler_enable = off
xdebug.profiler_enable_trigger = Off
xdebug.profiler_output_name = cachegrind.out.%t.%p
xdebug.profiler_output_dir ="d:/wamp64/tmp"
xdebug.show_local_vars=0

xdebug.remote_port = 9696
```

在 PHPStorm 配置 Xdebug 的端口, 此处的端口必须与 `xdebug.remote_port` 一致:

<img src="https://img-blog.csdnimg.cn/5d0ae2e36e13449ebe0b28873bd3d15a.png">


之后就可以调试了，注意开启📞按钮，PHPStorm 才可以监听

<img src="https://img-blog.csdnimg.cn/ff0cb7f2386b4999a79926802ff67b09.png">