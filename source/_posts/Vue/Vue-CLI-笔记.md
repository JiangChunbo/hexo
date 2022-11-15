---
title: Vue CLI 笔记
date: 2022-11-15 22:31:18
tags:
- Vue
---
# 参考引用

[官方文档](https://cli.vuejs.org/zh/guide/installation.html)

# 1. 安装
> 首先你需要具有 npm



```bash
# 查看版本
vue --version

# Vue CLI 2+
npm install -g vue-cli

# Vue CLI 3+
npm install -g @vue/cli
```

# 2. 创建项目

```bash
# Vue CLI 2+
vue init webpack project-name

# Vue CLI 3+
vue create my-project
```

# [vue.config.js](https://cli.vuejs.org/zh/config/#vue-config-js)

## 跨域 proxy
- 将 `/api/user` 代理到 `http://localhost:3000/api/user`

```js
module.exports = {
  devServer: {
    proxy: {
      '/api': 'http://localhost:3000'
    }
  }
};
```

- 将 `/api/user` 代理到 `http://localhost:3000/user`

适用于不同接口的处理，如 api、file
```js
module.exports = {
  devServer: {
    proxy: {
      '/api': {
        target: 'http://localhost:3000',
        pathRewrite: {'^/api' : ''}
      }
    }
  }
};
```

