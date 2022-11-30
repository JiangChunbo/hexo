---
title: Nginx 安装
date: 2022-11-30 10:35:05
tags:
- Nginx
---

1. 进入官网 [http://nginx.org/en/download.html](http://nginx.org/en/download.html) 下载 tar.gz 文件
2. 安装相关依赖
```bash
yum -y --setopt=protected_multilib=false  install make zlib zlib-devel gcc-c++ libtool openssl openssl-devel
```
3. 进入目录之后执行 ./configure

|选项|描述|
|:---|:---|
|--prefix=/opt/nginx/|可以指定安装路径|
|--with-config-file-path=/etc|设置 php.ini 的搜索路径。默认为 PREFIX/lib|
|--enable-fpm|Enable building of the fpm SAPI executable|
|--enable-inline-optimization||
|--enable-soap|Enable SOAP support|
|--enable-ftp|Enable FTP support|
|--enable-mbstring|Enable multibyte string support|
|--enable-zip|Include Zip read/write support|
|--enable-json||
|--enable-pdo||
|--enable-fileinfo||
|--enable-filter||

4. 使用 make && make install 编译安装

configure 可能出现的错误：
> ./configure: error: the HTTP rewrite module requires the PCRE library.
You can either disable the module by using --without-http_rewrite_module
option, or install the PCRE library into the system, or build the PCRE library
statically from the source with nginx by using --with-pcre=<path> option.

解决方式: `yum -y install pcre-devel`
