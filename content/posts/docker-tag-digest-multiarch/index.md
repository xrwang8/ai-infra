---
title: "同一个 Docker tag，为什么两台节点拉到的程序不一样？"
date: "2026-11-21"
description: "区分 Docker tag、OCI index digest、单平台 manifest 和 Git 版本，设计可复核的镜像交付记录。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "OCI", "多架构镜像", "镜像仓库"]
---

同一个仓库名字可以对应一个平台的镜像，也可以对应一组镜像。理解镜像交付要从 registry 里存了什么开始。一个多平台制品通常先由镜像索引列出候选。客户端按当前或指定的平台选出一份 manifest，再根据它声明的配置和层摘要获取内容。tag、index digest、单平台 manifest digest 和本地 image ID，分别出现在不同环节。 [OCI Image Index](https://github.com/opencontainers/image-spec/blob/main/image-index.md) · [OCI Image Manifest](https://github.com/opencontainers/image-spec/blob/main/manifest.md)

## 仓库引用如何解析到一份字节内容 {#先知道自己在固定哪一层}

| 记下的值 | 它指向什么 | 排查时要配套保存什么 |
|---|---|---|
| Git 提交 | 构建使用的源码状态 | 构建参数、基础镜像和锁定依赖 |
| Registry tag | 某个仓库内的名字，能否改变由仓库策略决定 | 解析它的时间与结果 |
| Index digest | 一份带平台清单的 OCI 镜像索引 | 目标 OS、架构与变体 |
| 单平台 manifest digest | 指定平台的配置和层清单 | 运行节点实际选到的制品 |
| Docker 本地 image ID | 经典存储使用配置摘要，containerd image store 可以显示目标 manifest 或 index 的摘要 | Docker 版本、镜像存储后端和目标平台 |

Manifest 指向配置和有顺序的层。索引再指向这些不同平台的 manifest。因此固定索引 digest，适合让多类节点引用同一份平台集合。两台不同架构的节点仍会选择属于各自平台的制品。固定单平台 manifest digest，则适合明确部署环境的单平台交付。

Docker CLI 的本地 `IMAGE ID` 也要按存储后端解释。以 Engine 29.10.0 为例，经典存储的镜像身份来自配置内容。containerd image store 的列表和 inspect 则从已存镜像的目标描述符取 ID。这个目标可能是索引，也可能是单平台 manifest。两台架构不同的机器即使显示相同的索引 ID，仍要分别核对实际选到的 manifest。 [经典镜像格式的 ImageID](https://github.com/moby/docker-image-spec/blob/main/spec.md#imageid) · [Moby 29.10.0 的列表实现](https://github.com/moby/moby/blob/docker-v29.10.0/daemon/containerd/image_list.go#L285) · [Moby 29.10.0 的 inspect 实现](https://github.com/moby/moby/blob/docker-v29.10.0/daemon/containerd/image_inspect.go#L95)

用 `docker buildx imagetools inspect <仓库>/<镜像>:<tag>` 可以查看远端索引和平台。发布记录里我会同时保留索引 digest、支持的平台和各平台的 manifest。比如 amd64 及 arm64 都通过了构建验证，但只有 amd64 做过真实推理，就在交付记录中分别标状态。 [Docker 多平台镜像](https://docs.docker.com/build/building/multi-platform/) · [imagetools inspect](https://docs.docker.com/reference/cli/docker/buildx/imagetools/inspect/)

## 固定了制品还要复核运行环境

源码提交固定，仍可能因为构建输入或依赖变化，生成不同的镜像。镜像 digest 固定，也仍可能因为运行参数、挂载、宿主驱动不同而出现不同表现。把两者结合起来才能追溯。对 AI Infra 还要记录模型权重来源、revision、设备类型和运行时配置。代码镜像与模型制品通常分开交付。

假设生产部署刚从 A 换到 B，记录只有 `api:stable`。发生问题时，先拿节点实际运行的制品记录，再找仓库里确切的 A 和 B，避免重新解析当前的 stable 就以为能复原历史。回退应先确认旧 digest 在仓库仍可获取、架构匹配，业务状态及模型版本能兼容。只回退标签，无法替代这些检查。

## 内容寻址如何带来层复用

假设同一个版本要构建模型 API 和定时批处理两个镜像，它们可以各有 manifest，但引用相同 digest 的基础镜像层。节点或 registry 已拥有的内容可以按摘要复用；改动一层时，需要更新引用它的 manifest，前面的未改内容仍可以保持自己的身份。反过来，更新索引里的一个平台版本，索引 digest 本身也会变化，即使其他架构的 manifest 保持原来那一份。

要区分摘要在哪个环节生成。压缩层 blob 的 digest 是分发时用来验证对应内容的标识。镜像配置的 rootfs diff IDs 记录与解包层相关的标识。用仓库的 index digest 和节点的本地 image ID 直接比较没有对齐对象。上线只需要从实际 manifest 查回配置与分层，再把这个链条记在发布记录里。 [OCI Image Configuration](https://github.com/opencontainers/image-spec/blob/main/config.md)

## 镜像备份、容器导出和数据备份 {#推送与回滚都拿制品身份验收}

- `docker image save` 和 `docker image load` 处理镜像归档，可用于传递相应镜像内容和标签。
- `docker container export` 导出容器文件系统视图。它不导出外接的数据卷，也不等同于带原镜像层历史和配置的分发。
- 数据卷应按自己的备份、恢复和保留策略处理。模型权重从外部制品拉取时，应检查恢复后真正读到的内容版本。

在线分发还需要考虑 registry 的认证、传输和制品留存。把一个 HTTP 开放地址当作生产镜像仓库之前，应先验证谁能读、谁能写、如何保留已知可回退的制品。 [OCI 分发规范](https://github.com/opencontainers/distribution-spec/blob/main/spec.md) · [docker image save](https://docs.docker.com/reference/cli/docker/image/save/) · [docker container export](https://docs.docker.com/reference/cli/docker/container/export/)

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
