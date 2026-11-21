---
title: "Docker 工程笔记"
description: "从容器运行模型入手的 Docker 学习笔记，覆盖 Linux namespace、cgroup v2、存储、网络、BuildKit、镜像交付和 Kubernetes 运行时。"
---

学 Docker 可以沿着同一个镜像从构建走到运行的过程。先理解镜像、容器和进程的关系，再分清 namespace 的资源视图、cgroup 的资源预算、存储层的数据生命周期和网络的通信路径。理解运行模型之后，再学怎样用 Dockerfile 交付可复现的制品，最后接到 Kubernetes 的 CRI 和跨节点网络。

## 先理解容器怎么运行

- [docker run 之后，镜像怎么变成一个 Linux 进程？]({{< relref "posts/docker-run-container-process/index.md" >}})
- [同一进程为什么有两个 PID？用 namespace 看容器隔离]({{< relref "posts/linux-container-namespaces/index.md" >}})
## 再掌握 Linux 资源控制

先学层级、controller 与 v1/v2 接口的关系，再分别看 CPU 的时间预算和内存的回收边界。

- [Linux cgroup v2 的资源模型：层级、controller 与容器限额]({{< relref "posts/linux-cgroup-v2-resource-model/index.md" >}})
- [Docker CPU 限额怎么查？从 cpu.max 追到 throttling]({{< relref "posts/docker-cgroup-v2-cpu-throttling/index.md" >}})
- [容器退出码 137，怎么判断是不是 cgroup OOM？]({{< relref "posts/docker-cgroup-v2-memory-oom/index.md" >}})

## 存储、网络和生命周期

- [Docker 存储驱动详解：镜像层、写时复制与后端取舍]({{< relref "posts/docker-overlayfs-writable-layer-volume/index.md" >}})
- [容器 80 端口有返回，为什么宿主访问还是失败？]({{< relref "posts/docker-port-mapping-bridge/index.md" >}})
- [docker stop 了，服务为什么没走完自己的退出逻辑？]({{< relref "posts/docker-pid1-graceful-shutdown/index.md" >}})

## 镜像构建和交付

- [只改了一行代码，为什么 Docker 又从头构建？]({{< relref "posts/docker-buildkit-context-cache/index.md" >}})
- [多阶段构建打包成功了，为什么最终镜像起不来？]({{< relref "posts/docker-multi-stage-runtime-image/index.md" >}})
- [同一个 Docker tag，为什么两台节点拉到的程序不一样？]({{< relref "posts/docker-tag-digest-multiarch/index.md" >}})

## 从 Docker 走到 Kubernetes

- [同节点容器能通，跨节点 Pod 为什么超时？]({{< relref "posts/docker-vxlan-cross-node-troubleshooting/index.md" >}})
- [docker ps 找不到这个 Pod，应该去哪里查？]({{< relref "posts/docker-kubernetes-containerd-cri/index.md" >}})
