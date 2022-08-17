---
title: Spring 请求参数处理实践
date: 2022-08-17 15:37:26
tags:
---


场景描述：请求 Content Type 为 `application/x-www-form-urlencoded`，即参数以键值对形式传递，其中有简单类型，如：数字，字符串，也有 JSON 类型，需要绑定到 Spring 的模型中，完成校验。

参数格式参考如下，为了便于显示，进行了换行: 

```bash
lesson_date=2022-08-08
&lesson_period_id=1
&subject_id=1
&grade_id=1
&class_id=1
&teacher_id=1
&student_list=[{"x":0,"y":0,"speak_num":0,"distract_num":0,"is_special":0}]
```