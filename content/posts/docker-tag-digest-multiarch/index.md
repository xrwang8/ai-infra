---
title: "同一个 Docker tag，为什么两台节点拉到的程序不一样？"
date: "2026-11-21"
description: "区分 Docker tag、OCI index digest、单平台 manifest 和 Git 版本，设计可复核的镜像交付记录。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "OCI", "多架构镜像", "镜像仓库"]
---

两台机器都写 `api:stable`，启动日志里的应用版本却对不上。查这种问题，我会先记录解析到的镜像 digest 和节点架构。标签可能在两次拉取之间被更新。即使标签没有变，arm64 和 amd64 也可能从同一个镜像索引选到不同的平台制品。

## 先知道自己在固定哪一层

| 身份 | 发布记录里应该写什么 |
|---|---|
| Git 提交 | 对应的源码版本 |
| Registry tag | 查询时用的名字，可受仓库标签策略影响 |
| OCI index digest | 带平台索引的制品身份 |
| 单平台 manifest digest | 某个 OS/架构实际选到的制品 |

用 `docker buildx imagetools inspect <仓库>/<镜像>:<tag>` 可以查看仓库引用和平台列表。若部署目标支持多种平台，固定索引 digest，保留平台各自解析到的 manifest。节点故障时能据此确认它拉到了哪个构建。单一平台也可按需求直接固定对应 manifest digest。 [OCI 镜像规范](https://github.com/opencontainers/image-spec/blob/main/manifest.md) · [Docker 多平台构建](https://docs.docker.com/build/building/multi-platform/)

## 推送与回滚都拿制品身份验收

我会把源码提交、不可变 digest、平台和依赖版本一起放进上线记录。之后比较两台节点时，先看 digest 是否一致，再查应用启动参数、挂载和运行时。拿一个可变标签证明从未换过制品，会漏掉这层变化。

离线交付时，`docker image save` 处理镜像归档，可通过 `docker image load` 恢复。容器文件系统导出属于另一条路径，外接卷里的数据需单独备份。 [docker image save](https://docs.docker.com/reference/cli/docker/image/save/) · [docker container export](https://docs.docker.com/reference/cli/docker/container/export/)

回滚时也按同样的表核对。镜像由仓库制品决定，模型权重还可能由另一份 revision 或制品地址决定。再确认预先验证过的平台和驱动环境，避免把恢复镜像标签当作恢复了整个服务。

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
