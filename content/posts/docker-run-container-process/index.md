---
title: "docker run 之后，镜像怎么变成一个 Linux 进程？"
date: "2026-11-21"
description: "沿 Docker Engine、containerd 和 OCI runtime 看容器创建、进程退出与 root filesystem 的关系。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "OCI", "容器运行时"]
images: ["images/cover.png"]
---

有个服务执行 `docker run` 后立刻退出。先看镜像能不能拉下来，只能确认制品在不在。真正需要对的是容器状态、启动命令和退出码。镜像已经在节点上，进程仍可能因为入口路径写错、没有执行权限，或者读不到配置而起不来。

![从工程制品到独立运行装置的原创概念插画](images/cover.png)

## 从三份状态开始查

- **镜像**负责保存文件系统层与启动配置。镜像标识可以相同，运行参数和挂载仍各有一份。
- **容器**由运行时为这次启动准备。它有自己的运行配置、文件系统视图、进程状态和生命周期。
- **进程**最终在某个 Linux 内核上执行。Linux 容器共享运行它们的内核。macOS 上运行 Linux 容器时，查的是虚拟机内的 Linux 进程。

Docker 常见 Linux 调用链中，CLI 把创建和启动等请求交给 Docker Engine，Engine 使用 containerd 管理容器生命周期，再由 OCI runtime 按配置准备命名空间、资源约束和进程。镜像内容、运行参数和日志的位置各不相同，排障时要先确定卡在哪一步。 [Docker 容器运行指南](https://docs.docker.com/engine/containers/run/) · [OCI Runtime 生命周期](https://github.com/opencontainers/runtime-spec/blob/main/runtime.md)

OCI 的三个规范也正好对上三段。Image 描述制品如何组织，Distribution 定义仓库如何分发，Runtime 负责接下运行配置并管理执行。一个单体应用也可以用这条链交付。什么时候拆成多个服务，要看它们是否需要独立发布或扩容，还要考虑多出来的网络依赖。 [OCI 镜像规范](https://github.com/opencontainers/image-spec/blob/main/manifest.md) · [OCI 分发规范](https://github.com/opencontainers/distribution-spec/blob/main/spec.md)

## 给退出留一条可查的记录

用独立的测试容器，把创建与启动分开。这个示例的 shell 会主动用 7 退出，适合核对“启动过但进程结束”和“还没启动”的区别。

```sh
demo=$(docker create --network none busybox:1.36 sh -c 'echo app-started; exit 7')
trap 'docker rm "$demo" >/dev/null 2>&1 || true' EXIT

docker inspect -f '{{.State.Status}}' "$demo"
docker start "$demo" >/dev/null
docker wait "$demo"
docker inspect -f 'state={{.State.Status}} exit={{.State.ExitCode}}' "$demo"
docker logs "$demo"
```

按这个脚本的逻辑，状态会从 `created` 走到 `exited`，`docker wait` 输出 7，日志出现 `app-started`。这证明主进程运行到了 shell 那一行。实际服务如果看不到自己的启动日志，还可以核对有效的 `Entrypoint`、`Cmd` 和运行时返回的错误。

容器正运行时，用 `docker inspect`、`docker top`、`docker exec` 分别看配置、进程列表和容器内视图。排查服务是否可接流量，再加一次请求验证。一个进程已经启动，只对应生命周期中的一个节点。

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
