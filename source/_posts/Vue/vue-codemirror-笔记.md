---
title: vue-codemirror 笔记
date: 2022-11-15 23:11:12
tags:
---

# 参考引用

https://www.npmjs.com/package/vue-codemirror

https://github.com/surmon-china/vue-codemirror/tree/v4.0.6

# 安装

默认下载是最新版本只支持Vue3

```bash
# 支持 Vue2
npm install vue-codemirror@4.0.6
```


# 使用

main.js 导入

```js
import VueCodemirror from 'vue-codemirror'
// import base style
import 'codemirror/lib/codemirror.css'

Vue.use(VueCodemirror)
```




## options

- autoCloseBrackets

能够自动输入另一个右括号

```js
import "codemirror/addon/edit/closebrackets.js";
```

- foldGutter

是否折叠


- gutters

翻译是沟，其实就是显示行号的那一列。类型数组，按照定义的顺序显示。如:

```js
gutters: ['CodeMirror-foldgutter', 'CodeMirror-linenumbers', 'CodeMirror-lint-markers'],
```

将会按照折叠箭头、行号、提示的顺序显示。


- indentUnit

缩进空格量。默认: 2

- lineNumbers

是否显示行号。true | false


- lineWrapping

宽度不够是否换行。如果为 false，codemirror 将会显示滚动条。

- lint

是否错误提示。

- mode

支持的语言: https://codemirror.net/5/mode/

设置 codemirror 的语言

application/json


- matchBrackets

当光标在括号周围时，能够高亮匹配的两个括号

```js
import "codemirror/addon/edit/matchbrackets.js";
```

- theme

主题，需要引入相应的文件，如:

```js
import 'codemirror/theme/xq-light.css'
```



# 主题

可以从以下两个链接查看默认提供的主题。

https://codemirror.net/demo/theme.html

https://www.codenong.com/cs106429772/


# 问题

##### 修改字体?

默认是 monospace

找到 codemirror.css 中 

```css
.CodeMirror {
    /* Set height, width, borders, and global font properties here */
    font-family: monospace;
    height: 300px;
    color: black;
}
```

然后添加自定义字体:

```css
.CodeMirror {
    /* Set height, width, borders, and global font properties here */
    font-family: Consolas, monospace;
    height: 300px;
    color: black;
}
```


#### 代码折叠

```js
// 支持各种代码折叠
import "codemirror/addon/fold/foldgutter.css";
import "codemirror/addon/fold/foldcode.js";
import "codemirror/addon/fold/foldgutter.js";
import "codemirror/addon/fold/brace-fold.js";
import "codemirror/addon/fold/comment-fold.js";
```