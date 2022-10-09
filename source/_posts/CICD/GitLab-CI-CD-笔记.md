---
title: GitLab CI/CD 笔记
date: 2022-09-25 22:53:25
tags:
- CI/CD
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
    --url "http://192.168.101.101" \
    --registration-token "FNc7eS247zHfjLgugiw4" \
    --description "gitlab-runner" \
    --tag-list "docker-ci-cd" \
    --run-untagged="true" \
    --locked="true" \
    --access-level="not_protected"
```



如果出现执行器 `executor=docker` 主机名无法解析的情况，编辑 `/etc/gitlab-runner/config.toml`，在 `[runners.docker]` 增加配置:

```
extra_hosts = ["主机名:IP"]
```


- 查询 Runner

```bash
gitlab-runner list
```


- 删除 Runner

如果在 GitLab 中删除了 Runner，还需要在 GitLab Runner 中继续删除:

```bash
# 删除所有的，跳过激活状态的
gitlab-runner verify --delete
# 删除指定 name
gitlab-runner verify --delete --name xxx
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


### <a id="after_script" href="https://docs.gitlab.com/ee/ci/yaml/#after_script">after_script</a>

使用 `after_script` 定义在每个 job 之后运行的命令组，包括失败的 job。

支持 CI/CD 变量。

**`after_script` 实例**:

```yml
job:
  script:
    - echo "An example script section."
  after_script:
    - echo "Execute this command after the `script` section completes."
```

**更多细节**:
在 `after_script` 中指定的脚本在新的 shell 中执行，与任何 `before_script` 或者 `script` 命令分开。因此，它们:

- 将当前工作目录设置回默认值（根据定义 runner 如何处理 Git 请求的变量）

- 无法访问 `before_script` 或者 `script` 命令所做出的更改，包括:

    - 在 `script` 脚本中导出的命令别名以及变量

    - 工作树之外的更改（取决于 runner 执行器），比如由 `before_script` 或者 `script` 脚本安装的软件。

- 有一个单独的超时时间，硬编码为 5 分钟

- 不影响 job 的退出码。如果 `script` 成功，`after_script` 超时或者失败，那么 job 以 0 作为退出码（job 成功）。

如果作业超时或被取消，则不执行 `after_script` 命令。


### <a id="allow_failure" href="https://docs.gitlab.com/ee/ci/yaml/#allow_failure">allow_failure</a>

使用 `allow_failure` 确定作业失败时管道是否应该继续运行。

- 若要让管道继续运行后续作业，请使用 `allow_failure: true`。
- 若要停止管道运行后续作业，请使用 `allow_failure: false`。

### <a id="before_script" href="https://docs.gitlab.com/ee/ci/yaml/#before_script">before_script</a>

使用 `before_script` 定义一个命令组，这个命令组应该在每个 job 的 `script` 命令之前运行，但是在 artifacts 恢复之后。



支持 CI/CD 变量。

**`before_script` 的例子**:

```yml
job:
  before_script:
    - echo "Execute this command before any 'script:' commands."
  script:
    - echo "This command executes after the job's 'before_script' commands."
```

**更多细节**:

- 你在 `before_script` 中指定的脚本与你在主要的 `script` 中指定的任何脚本连接在一起。组合起来的脚本在一个 shell 中一起执行。

- 在顶层而非 `default` 域使用 `before_script`，这是不建议的

### <a id="cache" href="https://docs.gitlab.com/ee/ci/yaml/#cache">cache</a>

使用 `cache` 指定要在 job 之间缓存的文件和目录的列表。只能使用本地工作副本中的路径.


#### cache:paths

### <a id="image" href="https://docs.gitlab.com/ee/ci/yaml/#image">image</a>

使用 `image` 指定 job 运行所在的 Docker 镜像


### <a id="only/except" href="https://docs.gitlab.com/ee/ci/yaml/#only--except">only\/except</a>

### <a id="retry">retry</a>


### <a id="tags" href="https://docs.gitlab.com/ee/ci/yaml/#tags">tags</a>

使用 `tags` 可以从项目可用的所有 Runner 中选择特定的运行程序

当你注册运行 Runner 时，你可以指定 Runner 的 `tags`，例如 `ruby`，`postgres`，或者 `development`。要选择并运行一个 job，Runner 必须为作业中列出的每个标记分配一个 runner。

**`tags` 的例子**:

```yml
job:
  tags:
    - ruby
    - postgres
```


### <a id="when" href="https://docs.gitlab.com/ee/ci/yaml/#when">when</a>

使用 `when` 配置作业运行时的条件。如果未在作业中定义，则默认值为 `on_success`。


**可能的输入**:

- `on_success`（默认值）: 只有在早期阶段的所有作业都成功或者具有 `allow_failure: true` 时才运行。
- `manual`: 只有在手动触发时才运行作业。
- `always`: 不管作业在早期状态如何，都运行作业。也可以在 `workflow:rules` 中使用。
- `on_failure`: 只有在早期阶段中至少有一个作业失败时才运行该作业。
- `delayed`: 在指定的期限内延迟作业的执行。
- `never`: 不运行此作业。只能在 `rules` 部分或者 `workflow: rules` 中使用。