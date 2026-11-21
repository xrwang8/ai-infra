---
title: "docker ps 找不到这个 Pod，应该去哪里查？"
date: "2026-11-21"
description: "沿 kubelet、CRI 和 containerd 定位 Kubernetes 容器，区分运行时端点、镜像仓库与 cgroup 驱动。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "Kubernetes", "CRI", "containerd"]
---

Pod 在 Kubernetes 里已经是 Running，登录节点执行 `docker ps` 却找不到它。第一步先确认 kubelet 连的是哪个容器运行时。节点上即使装有 Docker CLI，也可能只是给人构建镜像用。

## 先定位 kubelet 实际连接的 runtime

先用 Kubernetes API 确认 Pod 所在节点和 `containerID`。然后在该节点上核对 kubelet 的 CRI 端点，使用指向同一端点的 `crictl` 查看 sandbox、容器和镜像。下面的命令依赖节点已经为 `crictl` 配置了正确端点。

```sh
kubectl get pod -n YOUR_NAMESPACE YOUR_POD -o wide
crictl info
crictl pods
crictl ps -a
```

在常见 Kubernetes 部署中，kubelet 经 CRI 调用 containerd 或其他兼容运行时。Kubernetes 从 1.24 起移除了内置 dockershim。若集群仍通过兼容适配运行 Docker Engine，以节点实际配置为准。registry 中的 OCI 镜像可以由不同运行时消费，但本地镜像列表、状态和事件都由当时接入的运行时维护。 [Kubernetes 容器运行时](https://kubernetes.io/docs/setup/production-environment/container-runtimes/) · [crictl 调试指南](https://kubernetes.io/docs/tasks/debug/debug-cluster/crictl/)

## 把失败分到创建的哪一步

如果 Pod 还在 Pending，先读调度或节点事件。到了运行时仍未启动，再查镜像准备、sandbox 网络、容器启动和探针。有 PID 时可以继续查进程与 cgroup。cgroup driver 是否一致，要看 kubelet 与运行时的实际设置，避免拿旧 Docker 环境的默认值推断现网。

用于扩容排查时，最好记录 PodScheduled、镜像准备开始和结束、容器启动、模型 Ready 及首次业务成功。这些时刻来自不同层。对齐同一个 Pod 的时间线后，才能知道该查新节点的镜像、容器运行时，还是模型初始化。 [双卡推理冷启动记录]({{< relref "posts/qwen-dual-gpu-cold-start/index.md" >}})

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
