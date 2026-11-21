---
title: "只改了一行代码，为什么 Docker 又从头构建？"
date: "2026-11-21"
description: "通过构建上下文、Dockerfile 位置和 BuildKit 缓存依赖，定位一次源代码变更打断了哪一步。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "BuildKit", "Dockerfile"]
images: ["images/cover.png"]
---

改一段服务代码，CI 却重新下载整套依赖。我会先把构建日志切换成 `--progress=plain`，找最早没有出现 `CACHED` 的一步。最常见的线索是，在下载依赖之前就把整个项目放进了构建输入。

![稳定装配件与可替换模块的原创概念插画](images/cover.png)

## 先确定谁被送进构建

在仓库根目录执行 `docker build -f ci/release.Dockerfile app/`，Dockerfile 来自 `ci/`，上下文却是最后的 `app/`。普通 `COPY` 的路径从上下文找，`.dockerignore` 也要按这份上下文确认。有没有文件可以复制进镜像，和 Dockerfile 自己放在哪是两件事。

排除大型缓存、训练数据、构建产物和版本控制目录，可以减少上下文和意外失效。敏感值走 BuildKit secret mount，源目录里也应对不需要的文件设置排除。 [Docker 构建上下文](https://docs.docker.com/build/concepts/context/) · [BuildKit 构建秘密](https://docs.docker.com/build/building/secrets/)

引入文件时按来源选指令。上下文里的锁文件和源码用普通 `COPY`，编译产物用 `COPY --from` 从指定阶段取。远程 URL 属于 `ADD` 支持的来源之一，长期依赖还要考虑版本与校验。 [Dockerfile COPY](https://docs.docker.com/reference/dockerfile/#copy) · [Dockerfile ADD](https://docs.docker.com/reference/dockerfile/#add)

## 做一次只有两步的对照

在空的临时目录准备 `deps.lock` 和 `site.txt`，任意写两段文本。Dockerfile 用这三个指令，避免引入包仓库波动。

```dockerfile
FROM scratch
COPY deps.lock /deps.lock
COPY site.txt /site.txt
```

运行 `docker build --progress=plain -t cache-notes:demo .` 两次，中间只改 `site.txt`。预期第二次仍可复用前一条 `COPY`，后一条会重新执行。再只改 `deps.lock`，对比两步的状态。这是同一阶段里的依赖关系。BuildKit 的独立阶段还可以有自己的缓存分支。 [Docker Build 缓存](https://docs.docker.com/build/cache/) · [缓存失效规则](https://docs.docker.com/build/cache/invalidation/)

回到 CI，先复制锁文件、安装依赖，再复制业务源码。跨 CI 节点要复用缓存，则另外看 builder 位置与缓存的导出、导入。用日志标清哪些步骤命中以及耗时花在哪，避免把缓存命中、镜像层拉取和总构建时长混成一个指标。

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
