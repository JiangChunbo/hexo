---
title: Vue 打包部署脚本
date: 2022-07-18 08:50:05
categories:
- 前端
tags:
- Vue
---
# Vue 打包部署脚本

## 环境准备
```bash
npm install --save-dev scp2 cross-env ora@4.1.1 chalk
```

## 配置文件
```js
const scpClient = require('scp2');      // 基于ssh2的纯javascript安全复制程序
const ora = require('ora');             // 一个优雅的 Node.js 终端加载动画效果
const chalk = require('chalk');         // 字体颜色插件

const spinner = ora('正在发布到 ' + process.env.NODE_ENV + ' 服务器...\n');
spinner.start();
const SERVER_LIST = {
    'development': {
        hostname: '0.0.0.0',            // ip
        port: 22,                       // 端口
        username: 'root',               // 登录服务器的账号
        password: 'xxx',                // 登录服务器的账号
        path: '/opt/www'                // 发布至静态服务器的项目路径
    },
    'production': {
        hostname: '0.0.0.0',      // ip
        port: 22,                       // 端口
        username: 'root',               // 登录服务器的账号
        password: 'xxx',                // 登录服务器的账号
        path: '/opt/www'                // 发布至静态服务器的项目路径
    },
};

// 如果是公钥认证在这里创建私钥文件
const path = 'C:\\Users\\Administrator\\.ssh\\id_rsa'
const privateKey = require('fs').readFileSync(path).toString()

const server = SERVER_LIST[process.env.NODE_ENV];
scpClient.scp(
    'dist/',
    {
        host: server.hostname,
        port: server.port,
        username: server.username,
        password: server.password,
        privateKey,
        path: server.path
    },
    function (err) {
        spinner.stop();
        if (err) {
            console.log(chalk.red('发布失败.\n'));
            throw err;
        } else {
            console.log(chalk.green('Success! 成功发布到' + process.env.NODE_ENV + '服务器! \n'));
        }
    }
);

```


**package.json**

```js
"scripts": {
  "build": "vue-cli-service build",
  "deploy": "npm run build && cross-env NODE_ENV=development node ./deploy"
},
```
