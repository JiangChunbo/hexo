---
title: Nginx 官方学习笔记
date: 2022-07-12 10:43:27
tags:
---

# Nginx 官方学习笔记



## Module 参考

### [ngx_http_core_module](https://nginx.org/en/docs/http/ngx_http_core_module.html)


> 语法：server_name name ...;
> 默认：server_name "";
> 上下文：server

设置虚拟主机名，例如：

```nginx
server {
    server_name example.com www.example.com;
}
```

第一个名字成为主服务名。

服务名可以包含一个星号（"*"）替换名字的第一个部分或者最后一个部分：

```nginx
server {
    server_name example.com *.example.com www.example.*;
}
```

这样的名称称为通配符名。

上面提到的前两个名称可以合并为一个：

```nginx
server {
    server_name .example.com;
}
```

也可以在服务名称中使用正则表达式，并在名字前面使用波浪号（"~"）：

```nginx
server {
    server_name www.example.com ~^www\d+\.example\.com$;
}
```

正则表达式中有名捕获会创建变量（0.8.25），之后可以在其他指令中使用：

```nginx
server {
    server_name ~^(www\.)?(?<domain>.+)$;

    location / {
        root /sites/$domain;
    }
}

server {
    server_name _;

    location / {
        root /sites/default;
    }
}
```

允许该服务在没有 "Host" 头部字段的情况下为给定的 address:port 对处理请求。

> 在 0.8.48 之前，默认使用机器的主机名。




### [ngx_http_log_module](https://nginx.org/en/docs/http/ngx_http_log_module.html)

`ngx_http_log_module` 模块以指定格式写入请求日志

#### Example Configuration

```css
log_format compression '$remote_addr - $remote_user [$time_local] '
                       '"$request" $status $bytes_sent '
                       '"$http_referer" "$http_user_agent" "$gzip_ratio"';

access_log /spool/logs/nginx-access.log compression buffer=32k;
```

#### Directives


##### access_log

语法：
```css
access_log [path [buffer=size] [gzip[=level]] [flush=time] [if=condition]];
access_log off;
```
默认值:
```css
access_log logs/access.log combined;
```
上下文：

```css
http, server, location, if in location, limit_except
```

为缓冲日志写入设置 path，format，以及配置。可以在同一配置级别上指定几个日志。可以在第一个参数中指定 "`syslog:`" 前缀来配置日志记录到 syslog。特殊值 `off` 取消当前级别上所有的 access_log 指令。如果没有指定 format，那么就会使用预定义的 "combined"。

如果使用 `buffer` 或者 `gzip` 参数，那么写入日志将会缓冲。

当开启缓冲时，以下几种时机将会把数据写入文件：

- 如果下一行日志放不下缓冲区；
- 如果缓冲数据比 `flush` 参数指定的还要旧；
- 当工作进程重新打开日志文件，或者工作进程关闭


如果使用 `gzip` 参数，则在写入文件之前压缩缓冲区的数据。压缩的级别可以设置在 1（最快，压缩率低）到 9 （最慢，压缩率高）之间。默认情况下，缓冲区大小为 64K 字节，压缩级别为 1。由于数据在原子块中压缩，日志文件可以通过 "`zcat`" 解压缩或读取。

例如：

```bash
access_log /path/to/log.gz combined gzip flush=5m;
```

文件路径可以包含变量（0.7.6+），但是此类日志有一些约束：

- 由 worker process 使用的用户凭据应该有权限使用此类日志在文件夹创建文件
- 不可以使用缓冲写
- 为每个日志写入打开并关闭文件。但是，由于可以将频繁使用的文件描述符存储到缓存中，在由 open_log_file_cache 指令的 valid 参数指定的时间内，旧文件的写入可以继续。
- 在每个日志写入过程中，请检查请求的 root 目录是否存在，如果不存在，日志不会创建。因此，指定 root 和 access_log 以相同的配置级别是一个好主意：

```css
server {
    root       /spool/vhost/data/$host;
    access_log /spool/vhost/logs/$host;
    ...
}
```

`if` 参数可以开启有条件的日志记录。如果 `condition` 等于 0 或者一个空字符串，则不会记录请求。在下面的示例中，请求码为 2xx 以及 3xx 不会被记录：

```css
map $status $loggable {
    ~^[23]  0;
    default 1;
}

access_log /path/to/access.log combined if=$loggable;
```

##### log_format

日志格式可以包含一些通用的变量，以及一些仅仅存在于日志写入时的变量：

$bytes_sent 客户端发送的字节数

$connection 连接序列号

$connection_requests 通过一个连接发起的请求的当前序号

$msec 日志写入的时间，以秒为单位，精确到毫秒

$pipe 如果请求是管道，则是 "p"，否则为 "."

$request_length 请求长度（包含请求行，头部，以及请求体）

$request_time 请求处理时间，以秒为单位，精确到毫秒，从客户端读取第一个字节，并在最后一个字节发送给客户端之后写入日志所用时间

$status 响应状态码

$time_iso8601 以 ISO 8601 标准格式的本地时间。类似 `2022-07-12T15:05:16+08:00`

$time_local 以通用日志格式的本地时间。类似 `12/Jul/2022:15:05:16 +0800`


配置始终包含预定义的 "`combined`" 格式：

```css
log_format combined '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent"';
```


##### open_log_file_cache

定义一个缓冲，该缓冲存储了名称包含变量的频繁使用的日志文件描述符。该指令具有如下参数：

max

inactive

min_uses

valid

设置应该检查仍然存在相同名称的文件的时间，默认情况下，为 60 秒

off
禁用缓存