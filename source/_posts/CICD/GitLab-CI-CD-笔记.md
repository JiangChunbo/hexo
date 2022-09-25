---
title: GitLab CI/CD 笔记
date: 2022-09-25 22:53:25
tags:
---

# 参考

https://docs.gitlab.com/ee/ci/yaml/#keywords



## Keywords

使用 job 关键字配置 `job`:

|关键字|描述|
|:-|:-|
|retry|在失败的时候，job 何时重试，以及重试多少次|
|`script`|runner 执行的 shell 脚本|
|`stage`|定义任务的 stage|

## Global keywords



### <span id="stages">stages</span>

在 job 中使用 `stage` 可以将 job 配置在特定的阶段运行

如果 `.gitlab-ci.yml` 没有定义 `stages`，默认的 pipeline 阶段是:

> 如果你自定义了 `stages`，那么默认阶段配置不再生效

- `.pre`
- `build`
- `test`
- `deploy`
- `.post`

`stages` 中项目的顺序定义了 job 的执行顺序:

- jobs 在同一阶段并行运行
- 在前面的阶段一系列任务成功完成，下一个阶段的任务才会执行


如果管道只包含 `.pre` 或者 `.post` 阶段，并不会运行。


### script

除了触发器 job 之外，其他所有的 job 都需要一个 `script` 关键字


## Job keywords




### <a id="image" href="https://docs.gitlab.com/ee/ci/yaml/#image">image</a>

使用 `image` 指定 job 运行所在的 Docker 镜像


### <a id="only/except" href="https://docs.gitlab.com/ee/ci/yaml/#only--except">only / except</span>

### <span id="retry">retry</span>


### <span id="tags">tags</span>


### <span id="when" href="https://docs.gitlab.com/ee/ci/yaml/#when">when</span>
