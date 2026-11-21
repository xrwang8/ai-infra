---
title: "Docker 存储驱动详解：镜像层、写时复制与后端取舍"
date: "2026-11-21"
description: "比较 AUFS、OverlayFS、Device Mapper、Btrfs 和 ZFS 的写时复制与版本支持，理解 Docker 可写层、containerd snapshotter 和数据卷的关系。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "OverlayFS", "存储"]
---

一个镜像启动十个容器时，十个容器都能看到 `/usr/bin` 下的文件，其中任何一个还可以写自己的 `/tmp`。要解释这个现象，需要拆清三件事。**镜像层如何复用，某个容器修改旧文件时数据写到哪里，以及挂载进容器的数据由谁管理。** 换存储后端会改变前两项的实现，数据卷仍有自己的生命周期。

## 从镜像层到容器的合并视图 {#镜像层和写入属于不同生命周期}

OCI 镜像的文件系统层按顺序描述新增、变更和删除。运行时准备 root filesystem 时，需要把这些层呈现为一个目录树，然后给每个容器准备自己的可写空间。相同镜像层可以共用。两个容器各自修改同一路径，写入仍各进各的可写层。 [OCI 镜像层规范](https://github.com/opencontainers/image-spec/blob/main/layer.md) · [Docker 存储驱动](https://docs.docker.com/engine/storage/drivers/)

以 Linux OverlayFS 为例，`lowerdir` 可以包含多层只读目录，`upperdir` 存放可写内容，`merged` 是应用打开的合并视图。写入时的中间操作还需要满足同一文件系统等条件的 `workdir`。同一名字在上下层都出现时，上层内容优先。要理解层的数量，应该区分**一层可写上层**和**可以叠加多个的只读下层**。 [Linux OverlayFS 文档](https://docs.kernel.org/filesystems/overlayfs.html)

假设镜像里有一个 1 GiB 文件，容器第一次以写入方式修改其中几字节。在典型 OverlayFS 数据 copy-up 路径里，要先为这个容器准备一份上层文件，再进行修改。第一次改动的写入工作可能远大于应用实际修改的字节。上层文件已经存在后，后续写入可以直接作用于它。删除下层文件则会在上层留下隐藏该路径的标记，下层镜像里的文件仍然存在。这些机制也解释了为什么在后续构建层里删除一个大文件，前面镜像层已经增加的字节通常还在。 [Linux OverlayFS copy-up](https://docs.kernel.org/filesystems/overlayfs.html) · [Docker overlay2 行为](https://docs.docker.com/engine/storage/drivers/overlayfs-driver/)

## 文件被合并后，磁盘并不会按容器各拷一套

共享只读下层时，多个仅读同一文件的进程可以利用相同的下层数据和缓存。发生 copy-up 后，则要区分正在访问原先下层还是已经访问每个容器自己的上层文件。用两份容器视图里都看到的文件大小相加，不能直接得出磁盘实际消耗。镜像层、可写层、单独的卷和日志还可能各自占空间。 [Docker overlay2 页缓存与写入行为](https://docs.docker.com/engine/storage/drivers/overlayfs-driver/)

仍以那个大文件为例。三个容器只读可共用下层。同一文件若在两个容器中各发生一次需要复制数据的写入，就可能出现两份额外的上层副本。若工作负载要持续修改大文件，还有一种路径是事先由应用或制品流程把需要可写的内容放入显式数据卷，再按卷所在文件系统的语义写入。这里保留同一个原始镜像，分别比较首次修改、后续写入和重新创建时的数据状态，就能看清存储后端与挂载的分工。

## 五种后端为什么会有不同的取舍

比较存储后端时，可以先问写时复制发生在文件视图、文件系统快照还是块设备。接着看底层文件系统、缓存复用和运维要求。历史驱动适合理解旧节点和迁移，当前部署要以 Engine 版本和实际配置为准。

| 后端 | 写入机制 | 适合观察的优势与代价 | 当前 Docker Engine 口径 |
|---|---|---|---|
| AUFS | 叠加多条目录分支，在可写分支处理改动 | 早期 Docker 借它复用镜像层，运行依赖额外的内核支持 | Engine 24.0 移除了 `aufs`，主要出现在遗留节点 |
| OverlayFS | 合并只读下层和可写上层，通过 copy-up 处理下层文件的运行时写入 | 共用下层数据便于部署大量相同镜像，大文件首次写入要留意复制成本。经典 `overlay2` 在 XFS 上还要求 `ftype=1` | 经典驱动叫 `overlay2`，早期名为 `overlay` 的驱动在 24.0 移除 |
| Device Mapper | 通过 thin provisioning 和块级快照给镜像与容器提供块设备 | 改写时按块处理，避免走文件级 copy-up，运维还要管理 thin pool 的数据与元数据容量 | Engine 25.0 移除了 Docker 的 `devicemapper` 驱动 |
| Btrfs | 使用文件系统子卷与快照，修改时复用未变化的数据块 | 原生快照可以快速创建派生视图，需要合适的 Btrfs 后端，并观察写放大、碎片与元数据开销 | 仍有经典驱动文档，按分发版和存储前提核对 |
| ZFS | 以数据集的快照和克隆提供写时复制视图 | 利用 ZFS 的快照管理能力，运维涉及存储池、缓存和内存预算 | 仍有经典驱动文档，依赖已准备好的 ZFS 存储池 |

比较时先固定工作负载，比如首次修改镜像中一个大文件、写入一个全新的文件、启动大量同镜像容器。分别测首次写入耗时、磁盘增长和读写表现。不同场景涉及不同操作，单个速度数字不能代表整个驱动。 [Docker 驱动移除记录](https://docs.docker.com/engine/deprecated/) · [Device Mapper 历史驱动](https://docs.docker.com/engine/storage/drivers/device-mapper-driver/) · [Btrfs 驱动](https://docs.docker.com/engine/storage/drivers/btrfs-driver/) · [ZFS 驱动](https://docs.docker.com/engine/storage/drivers/zfs-driver/)

## Docker 的 overlay2 和 containerd 的 overlayfs

`overlay2` 是 Docker 经典存储驱动名，`overlayfs` 也可以指 containerd 的 snapshotter。二者都使用 Linux 的叠加文件系统能力，但镜像元数据、快照管理方式和排查目录不同。**从 Docker Engine 29.0 起，常规新装默认使用 containerd image store。** 由旧版本升级的节点可能继续使用经典驱动。配置了 `userns-remap` 的 Daemon 属于例外，不能使用 Docker 的 containerd image store。可先读 `docker info` 中的 Storage Driver 和 Driver Status。 [Docker containerd image store](https://docs.docker.com/engine/storage/containerd/)

切换 image store 是一次存储迁移。原后端的镜像和容器可能仍留在磁盘，而新的存储视图看不到它们。上线前应该分别安排镜像重新导入或拉取，以及对卷数据的备份与恢复演练。

## 把数据放回它应该有的生命周期 {#复盘先记目录落在哪}

同一容器停止再启动时，原可写层仍在。重新 `docker run` 创建新容器，会有新的可写层。适合留下的输出和缓存，需要按用途选择显式挂载。

| 位置 | 适合管理什么 | 随容器重新创建时要核对什么 |
|---|---|---|
| 容器可写层 | 临时结果、可重新生成的文件 | 新容器的可写层与原来分开 |
| 命名卷 | 由 Docker 管理的持久数据 | 新容器是否挂回同一个卷 |
| Bind mount | 已有宿主目录、团队指定的模型缓存 | Engine 所在机器的实际路径、权限和生命周期 |
| tmpfs | 随生命周期消失的数据 | 重启后重新准备 |

Dockerfile 的 `VOLUME` 声明挂载点，使用方仍需明确卷的来源。构建阶段在该声明之后写同一目录时，Legacy builder 与 BuildKit 对改动的处理也不同。合并视图、实际挂载和卷的身份，现场按 `docker inspect` 的信息来对。对推理容器，模型权重如果要跨 Pod 保存，优先按制品和持久化策略来设计。运行日志和模型缓存属于什么生命周期，应先写在部署约定里。 [Docker Volume](https://docs.docker.com/engine/storage/volumes/) · [Dockerfile VOLUME](https://docs.docker.com/reference/dockerfile/#volume)

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
