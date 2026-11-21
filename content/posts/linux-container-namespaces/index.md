---
title: "同一进程为什么有两个 PID？用 namespace 看容器隔离"
date: "2026-11-21"
description: "用 Linux 上的容器 PID 与 /proc namespace 标识，解释 PID、mount 和 network 隔离分别改变什么。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "Linux", "Namespace"]
---

容器里 `ps` 看到 PID 1，Docker Engine 节点上却有一个更大的 PID。排故障时拿错 PID，打开的可能是另一个进程的 `/proc`。先从 Docker 拿到进程在 Engine 所在 Linux 的 PID，再按那个编号读进程信息。

## 逐个核对资源视图

Linux namespace 按资源类别生效。可以一组进程共享网络，同时各自使用不同的挂载视图。要判断隔离是否成立，直接比较相应的 namespace 标识。

| 视图 | 程序通常能观察到的差异 |
|---|---|
| PID | 可见的进程集合及编号 |
| Mount | 执行 `mountinfo` 检查时能看到的挂载 |
| Network | 接口、路由、端口和协议栈 |
| UTS | 主机名 |
| User | UID、GID 与权限的映射 |
| IPC | 共享内存、消息队列等资源的可见性 |

PID namespace 有父子层次。祖先 namespace 能看到后代里的进程，并为它提供祖先视图下的编号。CPU 配额、内存上限则要查相应的 cgroup。权限边界还要结合身份、capabilities 和 seccomp 等配置看。 [Linux namespaces 手册](https://man7.org/linux/man-pages/man7/namespaces.7.html)

## 在同一台 Linux 上对两个编号

这段在 Docker Engine 所在的 Linux 执行。macOS 的 `/proc` 不能用于复核远端或虚拟机进程。

```sh
demo=$(docker run -d --rm --network none busybox:1.36 sleep 60)
trap 'docker stop -t 1 "$demo" >/dev/null 2>&1 || true' EXIT

engine_pid=$(docker inspect -f '{{.State.Pid}}' "$demo")
printf 'Engine node PID: %s\n' "$engine_pid"
readlink "/proc/$engine_pid/ns/pid"
docker exec "$demo" readlink /proc/1/ns/pid
docker exec "$demo" cat /proc/1/status | grep '^Pid:'
```

两个 `readlink` 应指向同一个 PID namespace，因为它们查询的是同一进程。最后一行从容器的视角读到 PID 1。继续查网络时，改为对照 `ns/net`、接口与路由。查同名路径的文件为什么不同，就看 `ns/mnt` 和挂载列表。用哪个 namespace 的工具，取决于具体出问题的资源。

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
