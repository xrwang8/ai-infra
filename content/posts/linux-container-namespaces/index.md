---
title: "同一进程为什么有两个 PID？用 namespace 看容器隔离"
date: "2026-11-21"
description: "用 Linux 上的容器 PID 与 /proc namespace 标识，解释 PID、mount 和 network 隔离分别改变什么。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "Linux", "Namespace"]
---

同一个 Linux 进程在容器里能叫 PID 1，在 Engine 所在节点上可能叫 PID 36210。变化的是进程位于哪个观察视图。**namespace 让一类内核资源对不同进程呈现不同的视图。** 一个进程可以共享另一个进程的网络视图，同时拥有独立的挂载和 PID 视图。

## 六类视图和几处容易混淆的边界 {#逐个核对资源视图}

| namespace | 控制的视图 | 用什么观察 |
|---|---|---|
| PID | 进程 ID 和进程可见范围 | `/proc/<pid>/status` 与 `ns/pid` |
| Mount | 目录树中的挂载关系 | `/proc/<pid>/mountinfo` |
| Network | 网络设备、路由表、端口等 | 同一个网络视图里的 `ip addr`、`ip route` 和 `ss` |
| UTS | 主机名和 NIS 域名 | `hostname` |
| User | UID、GID 的映射以及相关权限 | `uid_map`、`gid_map` |
| IPC | 消息队列、共享内存等 IPC 资源 | 相应的 IPC 对象与 `ns/ipc` |

还有 cgroup namespace 和 time namespace 等类型。cgroup namespace 改变进程看到的组路径，CPU 和内存限额则由该进程实际所属的 cgroup 施加。多种隔离手段同时使用，才组成应用实际看到的容器环境。 [Linux namespaces 手册](https://man7.org/linux/man-pages/man7/namespaces.7.html)

拿挂载视图举例。两个容器可能都打开 `/data/weights`，运行时可以分别把两块存储挂到同名路径。它们也可以显式共享一个卷。路径表面相同，实际指向需要顺着挂载配置检查。对网络同理。默认隔离时，两套网络视图可以各有自己的 8080 监听。共享同一个 net namespace 时，它们用同一组端口，可能发生冲突。

## 内核如何让进程进入一个视图

创建进程、拆开当前共享视图、加入已有视图，是三类不同操作。`clone` 可以在创建子进程时要求新 namespace，`unshare` 用于让进程与原来共享的资源视图分开，`setns` 可以让线程根据 namespace 文件描述符加入目标视图。工具 `unshare` 和 `nsenter` 是操作这些内核接口的用户态入口。PID namespace 有特殊语义，通过某些接口切换时，新视图用于之后创建的子进程，调用者正在使用的 PID 不会被重新编号。 [clone(2)](https://man7.org/linux/man-pages/man2/clone.2.html) · [unshare(2)](https://man7.org/linux/man-pages/man2/unshare.2.html) · [setns(2)](https://man7.org/linux/man-pages/man2/setns.2.html)

从宿主视图看，祖先 PID namespace 可以看到后代里的进程。容器里的 PID 1 在其视图内是该组的初始进程。这里也牵涉孤儿子进程、信号和退出语义，应用设计入口进程时要考虑这些责任。 [Linux PID namespaces](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html)

## 两个 namespace 组合时会发生什么

PID 视图与挂载视图是一个好例子。程序执行 `ps` 时通常通过 `/proc` 枚举进程。只改变新子进程所在的 PID namespace，如果还在读旧视图里的 procfs，得到的列表就容易令人困惑。一般需要给相应视图挂载匹配的 procfs，再解释看到的 PID。容器运行时把这些准备动作做在应用启动之前。 [PID namespaces 与 procfs](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html)

独立 mount namespace 还需要考虑挂载传播。共享挂载可以把挂载和卸载事件传播给关联的视图，private 挂载不传播，slave 挂载可接收上游变化而不向反方向传播。挂载发生在什么视图，决定了别处能不能看到，并不等于挂载目标的数据都被复制了一份。Linux 的 namespace 可通过 `/proc/<pid>/ns/` 下的句柄访问。一条引用还存在时，一些 namespace 的生命周期可以长于原进程。 [Mount namespaces](https://man7.org/linux/man-pages/man7/mount_namespaces.7.html) · [namespace 的生命周期](https://man7.org/linux/man-pages/man7/namespaces.7.html)

## 同一进程的两个编号怎么对上 {#在同一台-linux-上对两个编号}

在 Docker Engine 所在的 Linux 上，可以用下面这个配方核对。桌面客户端连接远端 Engine 时，宿主侧 `/proc` 也要到远端去看。

```sh
demo=$(docker run -d --rm --network none busybox:1.36 sleep 60) || exit
trap 'docker stop -t 1 "$demo" >/dev/null 2>&1 || true' EXIT

engine_pid=$(docker inspect -f '{{.State.Pid}}' "$demo")
printf 'Engine node PID: %s\n' "$engine_pid"
readlink "/proc/$engine_pid/ns/pid"
docker exec "$demo" readlink /proc/1/ns/pid
docker exec "$demo" grep '^Pid:' /proc/1/status
```

两个 `readlink` 定位到同一进程，应看到同一个 PID namespace 标识。最后一行用容器内的编号查看 PID。掌握这种比较之后，`ns/net` 和 `ns/mnt` 也能用来核对，具体条件取决于运行时是否让容器加入同一个资源视图。

namespace 说明了资源如何呈现。资源用量、权限和系统调用范围还要结合 cgroup、capabilities、seccomp 等机制理解。把它们分开，才能知道改网络视图、扩大 CPU 预算和调整应用权限分别会影响什么。

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
