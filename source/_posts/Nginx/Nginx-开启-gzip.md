---
title: Nginx 开启 gzip
date: 2022-11-29 14:39:53
tags:
---


**GZIP 指令说明**
|指令|含义|默认值|备注|
|:---|:---|:---|:---|
|[gzip](http://nginx.org/en/docs/http/ngx_http_gzip_module.html#gzip)| 打开或关闭 gzip| gzip off|
|[gzip_buffers](http://nginx.org/en/docs/http/ngx_http_gzip_module.html#gzip_buffers)|用于压缩相应的缓冲区数量和大小|默认 gzip_buffers 32 4k \| 16 8k，与平台有关||
|[gzip_comp_level](http://nginx.org/en/docs/http/ngx_http_gzip_module.html#gzip_comp_level)|压缩等级，可选值 1 ~ 9|1|不是越大越好，越大会消耗 CPU，效果不成正比|
|[gzip_disable regex](http://nginx.org/en/docs/http/ngx_http_gzip_module.html#gzip_disable)|使用指定的正则匹配需要压缩的 User-Agent 请求|无|默认匹配一切 User-Agent|
|[gzip_http_version](http://nginx.org/en/docs/http/ngx_http_gzip_module.html#gzip_http_version)|压缩响应所需的请求最低 HTTP 版本|1.1||
|[gzip_min_length](http://nginx.org/en/docs/http/ngx_http_gzip_module.html#gzip_min_length)|设置被 gzip 压缩的响应的最小长度，由 Content-Length 决定，单位 K|20||
|[gzip_types](http://nginx.org/en/docs/http/ngx_http_gzip_module.html#gzip_types)|除了 text/html 之外，还需要压缩的 MIME 类型|text/html|text/html 总是被压缩|


**配置示例**

```css
# 开启gzip，关闭用off
gzip on;

# 是否在http header中添加Vary: Accept-Encoding，建议开启
gzip_vary on;

# gzip 压缩级别，1-9，数字越大压缩的越好，也越占用CPU时间，推荐6
gzip_comp_level 6;

# 设置压缩所需要的缓冲区大小 
gzip_buffers 16 8k;

# 设置gzip压缩针对的HTTP协议版本
gzip_http_version 1.1;

# 选择压缩的文件类型，其值可以在 mime.types 文件中找到。
gzip_types text/plain text/css application/json application/javascript


# 启用gzip压缩的最小文件，小于设置值的文件将不会压缩
gzip_min_length 1k;

# gzip_proxied any;
```


**如何查看 gzip 是否开启？**

浏览器打开开发者工具，查看响应头是否有 `Content-Encoding: gzip`