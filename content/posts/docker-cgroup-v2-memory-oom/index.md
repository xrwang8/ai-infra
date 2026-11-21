---
title: "容器退出码 137，怎么判断是不是 cgroup OOM？"
date: "2026-11-21"
description: "从 Docker 状态和 cgroup v2 的 memory.current、memory.events 等证据判断内存压力和 OOM。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "cgroup v2", "内存", "故障排查"]
---

容器收到 SIGKILL 后，运行状态常会留下 137。内存耗尽、停止超时以后被强制杀掉，运维主动终止，都可能得到这个退出码。一次复盘需要把容器状态、cgroup 事件和发生时间对上。

## 先取组证据，再看总量

在 Docker 容器记录还保留时，可以读状态。

```sh
docker inspect -f 'OOMKilled={{.State.OOMKilled}} exit={{.State.ExitCode}} finished={{.State.FinishedAt}}' my-container
```

在目标进程仍运行时，按 `/proc/<pid>/cgroup` 找到它的 v2 cgroup。在 Engine 所在的 Linux 上检查这些文件。

| 文件 | 复盘时用来回答什么 |
|---|---|
| `memory.current` | 组及其子组当下记账的内存 |
| `memory.stat` | 匿名页、文件页等分别占了多少 |
| `memory.high` | 超过后可能遇到回收与节流的阈值 |
| `memory.max` | 无法继续回收时可能触发组内 OOM 的上限 |
| `memory.events` | `high`、`max`、`oom`、`oom_kill` 等发生了多少次 |
| `memory.events.local` | 需要区分本组与子组事件时的本组计数 |

采集要早于容器删除或 cgroup 回收。运行时的 `OOMKilled` 和内核的事件是两类证据，优先对准确时间和对应容器。宿主发生全局 OOM 时，还要检查节点内核日志，区分组内达到上限和节点内存耗尽。 [Linux cgroup v2 Memory 接口](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory) · [Docker 内存约束](https://docs.docker.com/engine/containers/resource_constraints/#memory)

## GPU 还有余量时也要查主机内存

推理容器可能同时持有 CPU 上的权重 staging、文件页缓存、分词对象和 KV Offload 缓冲。GPU 指标讲的是设备显存，`memory.current` 讲的是 Linux cgroup 内存。把这两组用量各自画出来，再按启动、载入、推理和退出的阶段对齐。

若没有退出却出现延迟毛刺，可以先对 `memory.events` 的 `high` 增量和回收活动。真要复现内存不足，使用一次性小容器做受控分配，保留起止快照和退出状态。历史记录只有 137 时，先保留“收到 SIGKILL”的结论，再补原因。

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
