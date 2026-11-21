---
title: "容器退出码 137，怎么判断是不是 cgroup OOM？"
date: "2026-11-21"
description: "从 Docker 状态和 cgroup v2 的 memory.current、memory.events 等证据判断内存压力和 OOM。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "cgroup v2", "内存", "故障排查"]
---

容器内存有几种计量。进程能看到自己的 RSS，内核在 cgroup 上又统计整组记账的内存，推理模型还有独立的设备显存。先知道每个数字在数谁，再谈“快到上限了”。容器退出时只有一个 137，也只能说明进程收到 SIGKILL 的一种常见表现，具体应由运行时状态和内核事件共同确认。

## 有哪些内存记在容器这组 {#先取组证据再看总量}

`memory.current` 统计 cgroup 及其子组当前记账的内存。它同时涵盖匿名页、文件页缓存及其他受控开销。一个看起来用量很高的容器，可能在频繁读取权重，内核保留了不少文件页。另一种情况是权重 staging 或 KV Offload 缓冲让匿名内存增长。二者对回收的敏感程度不同，应该看 `memory.stat` 的分项及当前活动。读取文件缓存后发现总量升高，仍需结合回收和业务停顿来解释。 [Linux cgroup v2 Memory](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory)

组限制按层级生效。某个子组还没有撑满自己的上限，也可能与同级组一起耗尽父组预算。准备推理服务时，可以把启动、权重加载、进入稳态和服务退出四阶段的主机内存与设备显存分别记录。GPU 留了余量的同时，主机内存仍可能出现压力。

## memory.high、memory.max 和内存保护

| 配置或指标 | 内核语义 | 读到变化后该怎么解释 |
|---|---|---|
| `memory.low` | 在相应条件下提供尽力而为的回收保护 | 多组竞争时，可按配套的上级预算理解保护 |
| `memory.high` | 超过后可能节流并加强回收的边界 | 往往先表现为分配慢和延迟，不一定马上结束进程 |
| `memory.max` | 组能配置的硬上限 | 回收不足时可能触发组内 OOM |
| `memory.events` | 记录 `high`、`max`、`oom`、`oom_kill` 等累积事件 | 取同一组的前后差值，先看回收压力，再看是否真被杀 |
| `memory.events.local` | 仅看当前组本地事件 | 用于区分组本身和子组里发生的事件 |

从 cgroup v1 的说明迁移时，不能机械把软限额的含义替换到 `memory.high`。v2 把保护、节流和硬上限分别表达，有明确的事件计数。容器的 swap 配置也可能影响达到压力点的表现，复现时要和内存上限一起记录。 [Docker 内存限制](https://docs.docker.com/engine/containers/resource_constraints/#memory) · [Linux cgroup v2 Memory](https://docs.kernel.org/admin-guide/cgroup-v2.html#memory)

## 一个读权重的进程怎样占用组内存

以读取权重分片为例。程序顺序读一个文件时，内核可能把读取过的数据保留为文件页缓存。进程把反序列化结果放进普通分配，又会消耗匿名内存。在传输到设备之前，可能同时保留输入页、CPU 上的临时副本和已分配的 GPU 张量。用文件大小直接当容器总用量，会漏掉同时存活的这几笔。只看设备曲线，也看不到由主机页回收带来的延迟。

用三个状态理解 `memory.events`。超过 `memory.high` 后组可能仍然运行，但分配路径承担更多回收相关工作。达到 `memory.max` 时，内核会尝试使使用量回到界限内。满足不了分配又触发组内 OOM 时，才可能选择任务结束。预测容量时，先按请求形状和加载路径标出内存峰值，再决定哪些缓存可回收、哪些状态必须常驻。线上读数以对应 cgroup 和实际事件为准。

## 三类证据怎样相互印证 {#gpu-还有余量时也要查主机内存}

第一类是容器状态。运行时仍保留这条记录时，用下面的方式读出结束时间，同时核对当时有没有部署停止或其他强制终止。

```sh
docker inspect -f 'OOMKilled={{.State.OOMKilled}} exit={{.State.ExitCode}} finished={{.State.FinishedAt}}' my-container
```

第二类是在目标进程仍存在时定位其 cgroup，保存组内统计和事件。组被回收后，历史文件可能就查不到了。第三类是节点内核日志，核对发生的是直接触发在限制组内的 OOM，还是节点整机压力导致的终止。

当 `high` 增加却没有结束进程，可以检查是否存在回收延迟。出现 `oom_kill` 后还要核对时间、进程和组。把内存数据和对应阶段放在一起，才能决定是降低主机缓存占用、改变 staging、调整资源设置，还是查其他终止动作。

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
