---
title: "容器停了文件还在，为什么重新 docker run 就没了？"
date: "2026-11-21"
description: "从容器可写层、OverlayFS copy-up 和挂载位置解释重启与重建的数据生命周期。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "OverlayFS", "存储"]
images: ["images/cover.png"]
---

有一个服务把结果放进 `/var/app/results`。重启同一个容器时文件正常，重新创建容器后整个目录却是空的。先别从应用日志猜删除动作。判断这是同一个容器重新启动，还是从同一镜像创建了一个新容器，再检查结果目录到底写在哪一层。

![分层文件视图与独立持久化存储的原创概念插画](images/cover.png)

## 镜像层和写入属于不同生命周期

镜像层提供初始文件，容器有自己的可写层。使用 OverlayFS 的相关驱动时，读取可以来自下层。当运行过程第一次修改一个来自下层的文件，通常需要先做 copy-up，把它移到可写层再修改。大文件的小改动也可能产生明显写入成本。

`docker stop` 停止现有容器，后续 `docker start` 仍用它的可写层。重新 `docker run` 会创建新的容器及可写层。Volume 与 bind mount 把数据落到各自的挂载来源，生命周期要按挂载来算。 [Docker 存储驱动](https://docs.docker.com/engine/storage/drivers/) · [OverlayFS copy-up](https://docs.docker.com/engine/storage/drivers/overlayfs-driver/)

## 复盘先记目录落在哪

| 写入位置 | 重新创建时的检查 |
|---|---|
| 容器可写层 | 容器 ID 换了就重新检查文件是否还在 |
| 命名卷 | 新容器是否重新挂载同一个卷 |
| Bind mount | Engine 节点路径、权限和容器目标路径是否一致 |
| tmpfs | 停止后按临时数据处理 |

可以先按这个方法做一个小对照。在一次性容器向普通目录和显式 bind mount 各写同样的标志，再用同镜像启动一个新容器。预期挂载目录能读到同一个标志，普通路径用的是新可写层。对生产检查 `docker inspect` 的 `Mounts`，也要留意挂载会遮盖同一路径上原本的镜像文件。

Dockerfile 里的 `VOLUME` 声明了挂载点。部署时仍要给数据目录安排明确的挂载来源。构建阶段若在 `VOLUME` 之后写同一路径，Legacy builder 会丢弃这些改动，BuildKit 会保留。遇到历史镜像和当前构建行为不同，先核对 builder，再检查容器实际挂载。 [Dockerfile VOLUME](https://docs.docker.com/reference/dockerfile/#volume)

Docker Engine 29.0 起的新装环境默认使用 containerd image store。已安装环境可能仍用经典驱动，具体以机器配置为准。查存储时先记版本和实际后端，再决定用哪套目录及工具取证。 [Docker 存储后端说明](https://docs.docker.com/engine/storage/drivers/)

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
