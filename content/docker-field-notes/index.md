---
title: "Docker 工程笔记"
description: "12 篇从工程问题出发的 Docker 学习笔记，覆盖容器进程、namespace、cgroup v2、网络、BuildKit、镜像交付和 Kubernetes 运行时。"
---

一个镜像从构建机走到线上，要经过制品选择、进程启动、资源分配和请求交付。出了问题，先确定卡在哪一段。这组笔记把 Docker 的主要机制拆成 12 个可以单独排查的问题，方便按现场现象选读。

## 运行与资源

- [docker run 之后，镜像怎么变成一个 Linux 进程？]({{< relref "posts/docker-run-container-process/index.md" >}})
- [同一进程为什么有两个 PID？用 namespace 看容器隔离]({{< relref "posts/linux-container-namespaces/index.md" >}})
- [Docker CPU 限额怎么查？从 cpu.max 追到 throttling]({{< relref "posts/docker-cgroup-v2-cpu-throttling/index.md" >}})
- [容器退出码 137，怎么判断是不是 cgroup OOM？]({{< relref "posts/docker-cgroup-v2-memory-oom/index.md" >}})

## 存储、网络和生命周期

- [容器停了文件还在，为什么重新 docker run 就没了？]({{< relref "posts/docker-overlayfs-writable-layer-volume/index.md" >}})
- [容器 80 端口有返回，为什么宿主访问还是失败？]({{< relref "posts/docker-port-mapping-bridge/index.md" >}})
- [docker stop 了，服务为什么没走完自己的退出逻辑？]({{< relref "posts/docker-pid1-graceful-shutdown/index.md" >}})

## 镜像构建和交付

- [只改了一行代码，为什么 Docker 又从头构建？]({{< relref "posts/docker-buildkit-context-cache/index.md" >}})
- [多阶段构建打包成功了，为什么最终镜像起不来？]({{< relref "posts/docker-multi-stage-runtime-image/index.md" >}})
- [同一个 Docker tag，为什么两台节点拉到的程序不一样？]({{< relref "posts/docker-tag-digest-multiarch/index.md" >}})

## 从 Docker 走到 Kubernetes

- [同节点容器能通，跨节点 Pod 为什么超时？]({{< relref "posts/docker-vxlan-cross-node-troubleshooting/index.md" >}})
- [docker ps 找不到这个 Pod，应该去哪里查？]({{< relref "posts/docker-kubernetes-containerd-cri/index.md" >}})

如果要补 Docker Engine 安装，按目标操作系统的当前官方步骤配置软件源。旧课件里的 `apt-key` 流程适合识别历史环境，部署时以当前发行版文档为准。 [Docker Engine 在 Ubuntu 上的安装说明](https://docs.docker.com/engine/install/ubuntu/)

系列的学习线索来自孟凡杰的《Docker 核心技术》课件。各篇独立组织工程场景、复现配方和配图，技术引用附在对应段落。
