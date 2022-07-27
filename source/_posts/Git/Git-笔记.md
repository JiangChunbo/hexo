---
title: Git 笔记
date: 2022-07-20 20:29:47
tags:
---
# 官方网址
---
[https://git-scm.com/](https://git-scm.com/)

[https://git-scm.com/book/zh/v2/](https://git-scm.com/book/zh/v2/)



# 使用前的准备
[配置用户信息](https://git-scm.com/book/zh/v2/%E8%B5%B7%E6%AD%A5-%E5%88%9D%E6%AC%A1%E8%BF%90%E8%A1%8C-Git-%E5%89%8D%E7%9A%84%E9%85%8D%E7%BD%AE)

项目配置优先生效，如果没有项目配置，则使用全局配置。


# 服务器 Git
[裸仓库搭建](https://git-scm.com/book/zh/v2/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-%E5%9C%A8%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E6%90%AD%E5%BB%BA-Git)

&nbsp;
# 常用命令
## Getting and Creating Projects
### [clone](https://git-scm.com/docs/git-clone)

`git clone <repository>`
克隆仓库。默认克隆全部分支。


`git clone -b <branch> <repository> <dirname>`
指定克隆某个分支。如果只需要一个分支，可以使用此命令，减少 clone 时间。

执行了这条命令之后自动在当前文件夹下初始化本地库且克隆了远程库，提供了一个名为 origin 的指向远程库的别名。

注意：
(1) windows 的凭据管理器会记住你的登录信息并不再需要登录，因此在同一机器上想要达到不同角色 push 的效果时需注意。
(2) 通过添加 -b 参数可以指定克隆的分支。long style: --branch
(3) dirname 表示生成的本地文件夹名称，默认为仓库名



## Branching and Merging
### [branch](https://git-scm.com/docs/git-branch)
```bash
git branch [--color[=<when>] | --no-color] [--show-current]
	[-v [--abbrev=<n> | --no-abbrev]]
	[--column[=<options>] | --no-column] [--sort=<key>]
	[--merged [<commit>]] [--no-merged [<commit>]]
	[--contains [<commit>]] [--no-contains [<commit>]]
	[--points-at <object>] [--format=<format>]
	[(-r | --remotes) | (-a | --all)]
	[--list] [<pattern>…​]
```

可以简写为 `git branch`，列出本地存在的分支。当前分支会以绿色高亮并带 * 前缀。short style: `git branch -l`
![在这里插入图片描述](https://img-blog.csdnimg.cn/e62d35e42b064b82a88638d37e972731.png)
### `git branch --remote`
列出远程分支，short style: `git branch -r`
![在这里插入图片描述](https://img-blog.csdnimg.cn/7b4a04429ffd47eb9ba204c7adabdf67.png)
### `git branch --all`

列出所有分支，short style: `git branch -a`
![在这里插入图片描述](https://img-blog.csdnimg.cn/eed9d23a5e35469fbe35f83a2a1d9cbc.png)
### `git branch --list <pattern>`

使用 shell 通配符列出 branch，pattern 可以传入多个，它们之间是 "或" 关系，即匹配任意一个即可。
![在这里插入图片描述](https://img-blog.csdnimg.cn/64b84c70d2a1432ba931eaa2b92ba3b9.png)

### `git branch -d <branch>`
删除分支。或者：`git branch -D <branch>`


### `git branch -m <oldbranch> <newbranch>`
重命名分支。如果 <newbranch> 已经存在，需要使用 -M 强制。



### `git branch --set-upstream-to=<远程主机>/<远程分支> <本地分支>`



&nbsp;
## remote
管理被跟踪仓库的设置

- `git remote [-v | --verbose]`

查看所有的别名。

-v | --verbose 表示显示更详细内容。




- `git remote add <别名> <地址>`

给远程库添加别名。

- `git remote rename <old> <new>`

将名为 `<old>`  的 remote 重命名为 `<new>`。更新所有远程跟踪分支和配置设置。


- `git remote remove <name>`

删除名为 `<name>` 的 remote。所有 跟踪的 remote 分支和配置设置都会被删除。


&nbsp;
## checkout
### `git checkout <branch>`
切换分支。

情况一 <branch> 是本地已存在的分支。立即切换，保留工作树的改动。

情况二 <branch> 本地不存在，远程不存在。从远程拉取分支，并切换，保留工作树的改动。

情况三 <branch> 本地和远程都不存在。git 会将 <branch> 识别为一个本地文件或文件夹，此时 git 会丢弃该文件当前工作树上所有的改动（危险！！！）。
用法：`git checkout .` 可以以当前所在目录为根，递归丢弃当前文件夹下所有改动，对于 PHP 线上调试还原比较有用。


### `git checkout -b <branch>`
等价于 `git branch <branch>` + `git checkout <branch>`，但如果 <branch> 已经存在，则会报错，因为 `git branch <branch>` 不允许创建重名分支。



### `git checkout -B <branch>`
情况一 如果 <branch> 不存在，则创建并切换。
情况二 如果 <branch> 存在，则将 <branch> 重置为当前分支的状态，切换，工作树保留。（危险！！！）



## rm
### `git rm --cached <file>`
删除文件缓存。可以将已经被 git 跟踪，但从此不想被跟踪的文件进行缓存删除。

### `git rm -r --cached <dir>`
删除文件夹缓存。可以将已经被 git 跟踪，但从此不想被跟踪的文件进行缓存删除。-r 代表递归，用于文件夹。


## Administration


### `clean`

删除所有未跟踪的文件
```bash
git clean -d -f
```

## stash
### `git stash`
创建储藏


### `git stash list`
查看贮藏

### `git stash apply`
使用贮藏。默认使用最近的一个，如果要指定最近的第二个，则 `git stash apply stash@{2}`


# 凭证存储
[官方文档](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E5%87%AD%E8%AF%81%E5%AD%98%E5%82%A8)

凭证存储涉及到免密登录

# IDE 的使用
Eclipse 如何给没有 git 版本控制的项目添加版本控制？


# Github 访问慢
https://blog.csdn.net/qq_45921491/article/details/115977828



