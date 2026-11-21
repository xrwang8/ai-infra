---
title: "Docker CPU 限额怎么查？从 cpu.max 追到 throttling"
date: "2026-11-21"
description: "用 cgroup v2 的 cpu.max 和 cpu.stat 区分 CPU 配额、权重与绑核，解释推理服务尾延迟的排查顺序。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "Linux", "cgroup v2", "CPU"]
---

Docker 配置里可以限制 CPU 时间、设置竞争权重，也可以指定任务在哪些 CPU 上运行。三个配置常被统称为“给容器分核”，实际分别控制时间预算、资源竞争和可运行范围。理解这三项，就能解释为什么一台看起来很空闲的机器，某个推理容器仍然在等 CPU。

## cgroup 是分层预算 {#先查到真正生效的组}

Linux cgroup 把进程分到组里，用 controller 记录和限制不同资源。cgroup v2 使用统一的树结构。子组可以有自己的设置，同时也受上级分配约束。同一个服务有多个 worker 时，配置给服务组的预算由这一组 worker 共同消耗。 [Linux cgroup v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)

| 想控制的量 | v2 里的主要接口 | 适合理解的问题 |
|---|---|---|
| 最多能用多少 CPU 时间 | `cpu.max` | 服务能在一个周期内消耗多少时间 |
| 争用时拿到多少份额 | `cpu.weight` | 多个同级组都在抢 CPU 时怎么分 |
| 能在哪些 CPU 上跑 | `cpuset.cpus.effective` | 实际可用的 CPU 集合 |
| 用掉多少、是否被限流 | `cpu.stat` | 在指定窗口里的 CPU 使用和节流计数 |

Docker 的 `--cpus` 主要交易的是带宽预算。即使允许在线程迁移到多个 CPU 的范围内运行，也可能受到同一份周期配额限制。`--cpuset-cpus` 影响可以在哪些 CPU 上运行，两者也可以同时配置。相对权重在竞争时才有意义。节点有空余时，权重较低的组仍可能获得更多时间。 [Docker CPU 约束](https://docs.docker.com/engine/containers/resource_constraints/#cpu)

## 两个线程怎样用完半个 CPU 的预算

如果 `cpu.max` 读到 `50000 100000`，基础预算就是每个 100 ms 周期共用 50 ms CPU 时间。在 `cpu.max.burst=0`、上级没有更紧限制的算例里，两个线程若能同时各忙 25 ms，就可能在约 25 ms 墙钟时间内用完预算。周期余下的时间仍有工作时，需要等额度恢复。开启 `cpu.max.burst` 后应另外考虑允许短时消费的余额。实际触发节流的时刻还受调度、层级和线程是否一直可运行影响。

对 v1 手册，需要把 `cpu.cfs_quota_us` 和 `cpu.cfs_period_us` 合在 v2 的 `cpu.max` 概念里读。公平调度决定许多可运行任务的先后，cgroup 带宽控制又对组总消耗有约束。Linux 从 6.6 开始向 EEVDF 过渡，阅读内核调度实现时要按版本看。 [Linux CPU Controller](https://docs.kernel.org/admin-guide/cgroup-v2.html#cpu) · [Linux EEVDF](https://docs.kernel.org/scheduler/sched-eevdf.html)

## 算出平均利用量，还需要看时间分布

假设两组各处理相同的一批输入。甲服务所有工作均匀分到多个周期，乙服务会在一条批处理启动时突然并行解码、分词或组织传输。两组在五秒内总用量可以一样，乙服务却可能在其中几个周期反复撞上预算。这种差异能解释平均 CPU 使用率正常时尾延迟仍变坏，但还需要在同一采样窗口看到节流和请求时间的变化。

例如在 120 个已观察周期里，有 40 个发生节流。可说这段约三分之一的计时周期触发过节流。不能把它换算成“所有请求延迟加了三分之一”，因为发生节流的时长和请求当时的状态还未知。Kubernetes 的 CPU request 另外参与调度等决策，limit 是运行限制的一部分。节点选择、请求量、CPU 配额、可运行集合各有自己的语义。 [Kubernetes 容器资源管理](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

## 如何理解两份 cpu.stat {#只消耗两秒-cpu再比一次计数}

在启用了相应 controller 的 v2 环境，可按这个配方读取一次前后差值。

```sh
docker run --rm --network none --cgroupns=private --cpus=0.5 busybox:1.36 sh -c '
  cat /sys/fs/cgroup/cpu.max
  cat /sys/fs/cgroup/cpu.stat
  (while :; do :; done) &
  worker=$!
  sleep 2
  kill "$worker"
  wait "$worker" 2>/dev/null || true
  cat /sys/fs/cgroup/cpu.stat
'
```

`nr_periods` 统计计时周期，`nr_throttled` 统计触发节流的周期数。`throttled_usec` 是组级累计量。要判断某段业务是否受限，先在相同 cgroup、相同窗口取差值，再把分词、批次准备、请求等待和 GPU 空转放到相同时间轴。`throttled_usec` 本身无法告诉你哪条请求等了几毫秒。若目标组看起来没有限流，也要查父 cgroup 与实际绑核范围。

前置阅读可先理解 [cgroup v2 的层级与 controller]({{< relref "posts/linux-cgroup-v2-resource-model/index.md" >}})，再回到这一类资源怎么限制和记账。

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
