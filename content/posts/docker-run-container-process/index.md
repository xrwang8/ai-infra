---
title: "docker run 之后，镜像怎么变成一个 Linux 进程？"
date: "2026-11-21"
description: "沿 Docker Engine、containerd 和 OCI runtime 看容器创建、进程退出与 root filesystem 的关系。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "OCI", "容器运行时"]
---

Docker 镜像、容器、进程经常被叫成同一个东西。先把时间轴铺开。镜像先存在，Docker 据此创建一个容器，接着才在 Linux 内核上启动进程。一个镜像可以对应多个容器，一个还没启动的容器也有 ID。理解这三个对象，后面学 namespace、cgroup、镜像层和容器网络时就知道各在配置谁。

## 镜像保存文件，也保存启动意图 {#从三份状态开始查}

镜像由文件系统层和配置组成。层提供程序与依赖的文件内容。配置可以给出入口命令、默认参数、工作目录、环境变量和运行用户。仓库里给镜像加的标签用于查找制品，下载之后的镜像仍然可以只是静态数据。 [OCI Image Manifest](https://github.com/opencontainers/image-spec/blob/main/manifest.md) · [OCI Image Configuration](https://github.com/opencontainers/image-spec/blob/main/config.md)

创建容器时，Engine 读取这些默认值，再与本次运行参数合并。相同镜像启动两个容器，可以选不同的命令、挂载和资源限额。容器级设置也有自己的状态。写了一份文件通常进入该容器的可写层；同镜像的第二个容器有另外一层。 [Docker 容器运行指南](https://docs.docker.com/engine/containers/run/)

## CLI、Engine、containerd 和 OCI runtime 的分工

| 层 | 负责什么 | 不同环境要核对的地方 |
|---|---|---|
| Docker CLI | 把用户操作转换为 Engine API 请求 | `docker context` 决定请求送往哪里 |
| Docker Engine | 解释容器创建参数，协调镜像、网络、卷和容器管理 | 它可能在本机、远端 Linux 或桌面工具的虚拟机 |
| containerd | 在常见 Engine 部署里管理容器执行生命周期，配合组件启动低层运行时 | 单独部署 containerd 时，入口与本地状态不必和 Docker 共用 |
| OCI runtime | 按运行配置建立容器环境，并启动目标程序 | 典型 Linux 运行时为 runc，实际实现可替换 |
| Linux 内核 | 运行和调度进程，提供命名空间、cgroup、挂载与网络等机制 | 多个 Linux 容器通常共享运行它们的这个内核 |

这是一条典型的链路。关键在于区分管理请求和真正执行进程的地方。运行规范描述 OCI bundle 里的 root filesystem 与配置，内核负责执行配置后的程序。容器应用看到的是准备好的用户态根目录，内核仍来自它所在的 Linux 环境。macOS 上的 Linux 容器通常运行在 Linux 虚拟机内，所以 Docker CLI 所在系统和进程所在内核可以不同。 [OCI Runtime 生命周期](https://github.com/opencontainers/runtime-spec/blob/main/runtime.md) · [Docker Engine](https://docs.docker.com/engine/)

OCI 三套规范可以沿这个过程区分。Image 描述内容怎样组织，Distribution 描述这些内容如何经仓库传输，Runtime 描述准备后的文件系统和配置如何变成正在运行的容器。跨环境使用同一镜像还要匹配 OS、CPU 架构和依赖的内核能力。需要 GPU 时，另检查节点驱动和设备的运行时注入。 [OCI 规范项目](https://opencontainers.org/about/overview/) · [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/main/spec.md)

## 下载、解包和创建各产生什么状态

拉取镜像时，客户端先解析 tag 或 digest，选择目标平台，再获取该 manifest 引用的配置与层。多台容器反复使用同一基础镜像时，有机会复用已有内容。层的压缩文件已经在本地，也可能仍需要解包和准备文件系统。因而“镜像没再下载”和“已经能立即执行”是两个观察点。像推理服务这样有大制品的场景，还要区分容器镜像的层与应用另外下载的模型权重。 [OCI 镜像清单](https://github.com/opencontainers/image-spec/blob/main/manifest.md) · [模型服务冷启动时间线]({{< relref "posts/qwen-dual-gpu-cold-start/index.md" >}})

以 containerd 的运行时模型为例，镜像引用、解包得到的快照、容器记录和 task 有各自的状态。OCI runtime 负责创建执行环境与启动，长期运行、传递 I/O、报告退出等工作还由相应的 shim 和上层组件配合完成。看到管理进程重启时，容器是否受到影响，要按运行时和 Engine 的配置确认。这个分工也使我们能把四类耗时拆开记录：制品取得、文件系统准备、主进程启动、应用首次可以交付。 [containerd Runtime v2](https://github.com/containerd/containerd/blob/main/core/runtime/v2/README.md)

## 用三步看 created 和 exited {#给退出留一条可查的记录}

一条 `docker run` 可以从用户视角拆成创建和启动。下面用故意以 7 结束的小程序观察这两个阶段。

```sh
demo=$(docker create --network none busybox:1.36 sh -c 'echo app-started; exit 7') || exit
trap 'docker rm "$demo" >/dev/null 2>&1 || true' EXIT

docker inspect -f '{{.State.Status}}' "$demo"
docker start "$demo" >/dev/null
docker wait "$demo"
docker inspect -f 'state={{.State.Status}} exit={{.State.ExitCode}}' "$demo"
docker logs "$demo"
```

预期先看到 `created`，最后看到 `exited`、退出码 7 和 `app-started`。镜像仍在，容器记录也还在，主进程已经结束。重新 `docker start` 会使用同一个容器的配置和可写层。重新 `docker run` 会创建一个新容器。文件能否继续看到，由它是写进原可写层还是外部挂载决定。 [容器可写层与存储后端]({{< relref "posts/docker-overlayfs-writable-layer-volume/index.md" >}})

## 容器和应用边界怎样配合

把服务装进容器改变的是交付方式。单体应用可以这样发布，多个微服务也可以分别交付。是否拆分服务，要考虑部署频率、独立扩容、跨服务接口以及网络故障，而不是先规定每种部署必须有多少个容器。一个容器在进程层面也可以有子进程，需要明确信号和退出时的责任。 [PID 1 与容器终止]({{< relref "posts/docker-pid1-graceful-shutdown/index.md" >}})

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
