---
title: "Docker CPU 限额怎么查？从 cpu.max 追到 throttling"
date: "2026-11-21"
description: "用 cgroup v2 的 cpu.max 和 cpu.stat 区分 CPU 配额、权重与绑核，解释推理服务尾延迟的排查顺序。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "Linux", "cgroup v2", "CPU"]
---

GPU 没吃满，推理服务的 P99 却上去了。先别急着加卡。分词、请求排队和响应序列化仍要吃 CPU。如果服务的线程在某个计时周期里提前用完配额，后半段会等下一周期。整台机器的 CPU 利用率可能看不出这次等待。

## 先查到真正生效的组

用 `docker info --format '{{.CgroupVersion}}'` 确定版本，从目标进程的 `/proc/<pid>/cgroup` 找组。v2 用 `cpu.max` 表示周期预算。第一列是配额，第二列是周期，单位都是微秒。若读到 `50000 100000`，这个组每个 100 ms 周期最多使用 50 ms CPU 时间，相当于持续消耗半个逻辑 CPU 的预算。多个线程可以并行用完这笔时间。

还要查两个独立的量。`cpu.weight` 用于争用时分配份额。`cpuset.cpus.effective` 说明任务能在哪些 CPU 上运行。父 cgroup 仍可能把整个子树限制得更紧。分别记录这三项，才能区分配额不足、抢 CPU 和绑核范围过窄。 [Linux cgroup v2 CPU 接口](https://docs.kernel.org/admin-guide/cgroup-v2.html#cpu) · [Docker CPU 约束](https://docs.docker.com/engine/containers/resource_constraints/)

## 只消耗两秒 CPU，再比一次计数

下面是 cgroup v2 上的复现配方。循环限制在独立容器里，外层要保留限额和窗口。

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

预期配额生效时，`nr_throttled` 和 `throttled_usec` 在两份快照间增加。前者是发生限流的周期数，后者记录限流累计时间。都要取同一窗口的增量，`throttled_usec` 不能直接当某个请求的等待时间。

测业务时固定请求形状和到达速率，再把 `cpu.stat` 增量跟应用首段和排队一起画。只调整 CPU 限额做一次对照，还能看限流减少后延迟是否改善。旧手册里的 `cpu.cfs_quota_us` 属于 v1 接口。内核从 6.6 开始引入 EEVDF 过渡，读新的调度代码时也应先确认内核版本。 [Linux EEVDF 文档](https://docs.kernel.org/scheduler/sched-eevdf.html)

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
