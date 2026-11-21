---
title: "只改了一行代码，为什么 Docker 又从头构建？"
date: "2026-11-21"
description: "通过构建上下文、Dockerfile 位置和 BuildKit 缓存依赖，定位一次源代码变更打断了哪一步。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "BuildKit", "Dockerfile"]
---

Dockerfile 看起来是一串命令。构建时，builder 实际要先确定输入，按依赖关系执行步骤，再决定哪些结果可以复用。理解这三件事，就能看懂为什么只改了一份应用源码，前面的依赖安装有时无需运行，有时整段又做了一遍。

## 构建上下文定义了输入范围 {#先确定谁被送进构建}

在仓库根目录执行 `docker build -f ci/release.Dockerfile app/`，Dockerfile 路径是 `ci/release.Dockerfile`，上下文是最后一个参数 `app/`。普通 `COPY src/ /app/src/` 要到上下文里找 `src/`，和 Dockerfile 放在哪个目录分开。远程 Git、压缩包和其他命名上下文也能成为输入，要按本次命令和 builder 配置确认。 [Docker 构建上下文](https://docs.docker.com/build/concepts/context/)

`.dockerignore` 影响构建器能看到哪些本地文件。模型权重缓存、临时文件和构建输出如果根本不该进入发布制品，应约束在明确的上下文之外或显式排除。构建需要访问私有仓库时，可透过 BuildKit secret mount 传递短期凭据，避免将其作为普通构建文件复制。 [构建秘密](https://docs.docker.com/build/building/secrets/)

几条常用指令也可按输入来源记。普通 `COPY` 从上下文取本地文件。`COPY --from` 可以从另一个阶段或指定来源取产物。`ADD` 另外支持远程来源以及特定压缩包场景。对远程输入，依赖可复现时还需固定版本与校验。 [Dockerfile 指令](https://docs.docker.com/reference/dockerfile/)

## 缓存的复用范围跟着依赖走 {#做一次只有两步的对照}

步骤的缓存决策与指令、输入、父步骤等有关。文件内容发生变化，依赖它的复制步骤以及后续相依的步骤就可能要重做。独立构建阶段有自己的依赖分支。一次 `RUN` 通过网络读包仓库时，上游数据发生变化也未必会自动导致原来的缓存失效。更新依赖应有可见的输入变化或明确的刷新方式。 [BuildKit 缓存失效](https://docs.docker.com/build/cache/invalidation/)

```dockerfile
FROM scratch
COPY deps.lock /deps.lock
COPY site.txt /site.txt
```

在只有这两个输入文件的临时目录，构建一次后只改 `site.txt`，再用 `--progress=plain` 构建同一个目标。按该依赖关系，第一条 `COPY` 可复用，后一条需要重做。改动 `deps.lock` 后，再看哪些步骤受到上游变化影响。前提是两次使用同一个可读到先前缓存的 builder。

## 为什么缓存命中也会带来旧依赖

例如依赖安装步骤含 `RUN apt-get update`，缓存只复用这一步以前的结果。远端仓库更新了，镜像构建器不会因为这条命令的网络响应发生变化，就自动知道结果应该重跑。将更新索引和安装软件包放进同一个合理的步骤，可避免复用旧索引但重跑安装。要从同一 Dockerfile 生成可追溯镜像，还要安排基础镜像和依赖的版本更新，写清楚什么时候应刷新缓存。 [Docker 构建最佳实践](https://docs.docker.com/build/building/best-practices/)

再看分阶段的例子。镜像编译和静态文档生成可以是两条独立分支。最终目标没有依赖文档分支时，BuildKit 只需执行它真正依赖的阶段。把所有东西做成一个大步骤，可以避免留下过多中间文件，也可能缩小可独立复用的范围。优化时按输入边界划步骤，比只数 Dockerfile 里有几条 `RUN` 更能解释命中结果。 [BuildKit 的阶段依赖](https://docs.docker.com/build/building/multi-stage/#differences-between-legacy-builder-and-buildkit)

## 三类复用分别帮了什么

| 方式 | 复用的对象 | 常见用途 |
|---|---|---|
| 步骤结果缓存 | 一次已知输入的执行结果 | 稳定的安装步骤可跳过 |
| cache mount | 供构建工具使用的缓存目录 | 步骤必须重跑时，包管理器还可复用已下载的内容 |
| 外部缓存 | 导出到仓库或其他后端的构建缓存 | 跨 CI 执行器重复使用构建结果 |

例如构建 Python 推理服务时，可以先处理锁定依赖，再复制频繁变更的应用代码。有些 CI 节点完全重新创建，还需要验证缓存是否被正确导入。观察时分别记录上下文大小、最早失效步骤、构建总时长与最终镜像体积。每个指标回答的问题不同。 [优化构建缓存](https://docs.docker.com/build/cache/optimize/)

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
