---
title: "docker ps 找不到这个 Pod，应该去哪里查？"
date: "2026-11-21"
description: "沿 kubelet、CRI 和 containerd 定位 Kubernetes 容器，区分运行时端点、镜像仓库与 cgroup 驱动。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "Kubernetes", "CRI", "containerd"]
---

Docker 和 Kubernetes 都能让 Linux 进程跑在容器里，但它们的管理入口不同。Docker CLI 调到 Docker Engine。Kubernetes 节点上，kubelet 用 CRI 管理 Pod sandbox、镜像与容器，再由配置好的运行时执行。节点装了 `docker` 命令，并不能说明 kubelet 也管理着同一套容器。

## OCI、CRI 和具体运行时的位置 {#先定位-kubelet-实际连接的-runtime}

| 层 | 作用 | 学习和排查时对应的对象 |
|---|---|---|
| Kubernetes API | 声明 Pod 应该跑什么并呈现状态 | Pod spec、已经调度的节点与事件 |
| kubelet / CRI | 将 Pod 所需的管理操作交给节点运行时 | kubelet 的 CRI socket、sandbox、容器及镜像服务 |
| containerd、CRI-O 等 | 管理创建环境、镜像与生命周期 | 运行时端点、事件及其实现 |
| OCI runtime | 按运行配置启动进程 | 对应的 root filesystem、资源和底层执行状态 |
| Linux 内核 | 执行进程并提供隔离和资源控制 | PID、namespace、cgroup、挂载与网络 |

CRI 面向 kubelet 的管理调用，OCI 描述镜像和低层运行约定。两套接口位于不同位置。Kubernetes 从 1.24 起移除了内置 dockershim，依赖通过 CRI 接入运行时。需要运行 Docker Engine 的场景仍可使用兼容适配，判断节点实际行为要读它当前连接的端点。 [Kubernetes 容器运行时](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) · [OCI Runtime](https://github.com/opencontainers/runtime-spec/blob/main/runtime.md)

## 一个 Pod 从计划走到进程

在典型非 hostNetwork 的 Linux Pod 中，运行时先准备 Pod sandbox，并配合 CNI 处理网络。之后各应用容器加入所需的共享环境。容器仍可以有各自的启动命令、资源配置、文件系统视图和进程。镜像层、Pod 网络和容器状态因此可以分别发生问题。Pod 已经选中节点而容器没有启动时，先判断 kubelet 走到了镜像、sandbox、创建进程还是探针阶段。

用 Docker 构建好了 OCI 兼容镜像，可以通过 registry 交给 containerd 等运行时消费。Docker 本机镜像列表一般属于当前 Docker Engine 管理的视图。Kubernetes 节点是否有它、凭什么凭据拉它、拉到了哪个平台和 digest，由节点实际运行时和 Pod 配置决定。构建机器上能 `docker run` 只覆盖那套环境。

## 同一镜像与同一 Pod 能力是两回事

CRI 把镜像服务与运行时服务作为管理接口。镜像层和配置可以由不同软件理解，Pod 的共享网络、挂载来源和资源约束却是在节点创建过程中实现。Pod 还可以用 RuntimeClass 选择运行时处理程序，例如某类更隔离的执行环境。一个支持相同 OCI 镜像格式的环境，可能提供不同的设备、内核语义和调试方法。 [Kubernetes RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/)

运行时也会管理 cgroup 的组织方式。kubelet 和运行时分别认哪种 cgroup driver，要与该集群版本及节点配置一起确认。更换 runtime、节点镜像或 cgroup driver 时，除了工作负载能启动，还应复核资源请求、限制和相应指标。云原生交付里真正需要保存的，是声明到执行的对应关系，而不仅是一条容器启动命令。

## 对齐三层状态 {#把失败分到创建的哪一步}

```sh
kubectl get pod -n YOUR_NAMESPACE YOUR_POD -o wide
crictl info
crictl pods
crictl ps -a
```

先用 Kubernetes API 找对节点和容器身份，再在目标节点将 `crictl` 指到 kubelet 实际使用的 CRI 端点。这个配方假定工具已正确配置。仅看 `docker ps` 或去不相干的节点读状态，很容易把没有查到误读成应用没有运行。 [crictl 调试指南](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)

这条关系也能用于学习资源控制。kubelet 把容器资源约束交给运行时落实，cgroup driver 和具体组路径仍应在节点上确认。把 PodScheduled、镜像准备、进程创建和业务就绪分开保存，应用和平台可以各自对自己的阶段负责。 [双卡模型冷启动的时间线]({{< relref "posts/qwen-dual-gpu-cold-start/index.md" >}})

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
