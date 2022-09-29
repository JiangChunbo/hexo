---
title: GitLab CI/CD 笔记
date: 2022-09-25 22:53:25
tags:
---

# 参考

https://docs.gitlab.com/ee/ci/yaml/#keywords


# 安装

## GitLab Runner


```bash
docker run --rm -v /srv/gitlab-runner/config:/etc/gitlab-runner gitlab/gitlab-runner register \
    --non-interactive \
    --executor "docker" \
    --docker-image alpine:latest \
    --url "http://192.168.59.59" \
    --registration-token "FNc7eS247zHfjLgugiw4" \
    --description "first-register-runner" \
    --tag-list "test-cicd,dockercicd" \
    --run-untagged="true" \
    --locked="true" \
    --access-level="not_protected"
```


如果出现主机名无法解析的情况，编辑 `/etc/gitlab-runner/config.toml`，在 `[runners]` 节点下面的 `[runners.docker]` 增加配置:

```
extra_hosts = ["主机名:IP"]
```


## Runner Token 类型

- shared

Admin > Overview > Runners

- Group

Groups > Settins > CI/CD

- project

项目 > Settins > CI/CD



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



### <a id="cache" href="https://docs.gitlab.com/ee/ci/yaml/#cache">cache</a>

使用 `cache` 指定要在 job 之间缓存的文件和目录的列表。只能使用本地工作副本中的路径.


#### cache:paths

### <a id="image" href="https://docs.gitlab.com/ee/ci/yaml/#image">image</a>

使用 `image` 指定 job 运行所在的 Docker 镜像


### <a id="only/except" href="https://docs.gitlab.com/ee/ci/yaml/#only--except">only / except</span>

### <span id="retry">retry</span>


### <span id="tags" href="https://docs.gitlab.com/ee/ci/yaml/#tags">tags</span>

使用 `tags` 可以从项目可用的所有 runner 中选择特定的运行程序

当你注册运行 runner 时，你可以指定 runner 的 `tags`，例如 `ruby`，`postgres`，或者 `development`。要选择并运行一个 job，runner 必须为作业中列出的每个标记分配一个 runner。


### <span id="when" href="https://docs.gitlab.com/ee/ci/yaml/#when">when</span>
