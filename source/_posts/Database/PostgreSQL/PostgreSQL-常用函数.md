---
title: PostgreSQL 常用函数
date: 2022-11-07 11:47:17
tags:
---

- to_char



- split_part

text=“name.cn” split_part(text,’.’,1) 结果： name
text=“name.cn” split_part(text,’.’,2) 结果： cn
text=“name.cn.com” split_part(text,’.’,3) 结果： com

split_part(field, '-', length(replace(field, '-', '--')) - length(field) +1)