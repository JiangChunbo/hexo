---
title: Nginx 开启 gzip
date: 2022-11-29 14:39:53
tags:
---



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