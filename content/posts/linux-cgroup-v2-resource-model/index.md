---
title: "Linux cgroup v2 的资源模型：层级、controller 与容器限额"
date: 2026-11-21
description: "从进程归组和 controller 下发讲透 cgroup v2。对照 v1 接口，理解 pids、CPU、内存、I/O 和 cpuset 如何影响 Docker 容器。"
categories: ["Docker 工程笔记"]
tags: ["Linux", "Docker", "cgroup v2", "资源隔离"]
---

同样启动四个 worker，有的容器只会把请求处理慢，有的却在创建线程时失败。两种情况都可能是资源受限，但要查的文件不同。CPU controller 管运行时间，pids controller 管任务数，memory controller 管主机内存记账。控制器作用于进程所在的组，还受父组约束。学 Docker 的资源控制，先要把**进程属于哪一组、谁把资源管理权交给它、每种控制器怎样实施限制**这三件事连起来。

## cgroup 把进程组织成一棵资源树

namespace 决定一组进程从哪个视图看资源。cgroup 则需要回答这些进程的用量记到哪里，以及运行时为整组划了多少预算。一个进程加入 cgroup 后，它之后创建的子进程通常也进入同一组。这里的资源要求可以独立于应用自身的进程树。多进程的推理服务，模型 worker、HTTP 处理线程和辅助进程也可以共同消费一份组预算。 [Linux cgroup v2 的进程归组](https://docs.kernel.org/admin-guide/cgroup-v2.html#organizing-processes-and-threads)

Linux 将 cgroup 层级通过虚拟文件系统暴露给用户态。目录代表组，`cgroup.procs` 展示本组有哪些进程，controller 暴露与资源相关的参数和统计。读取这些文件相当于查看内核当前状态。比如 `memory.current` 不是一行上次运行保存的日志，而是该组此刻的记账量。

| 要回答的问题 | 去哪里看 | 常见误读 |
|---|---|---|
| 进程实际属于哪一组 | `/proc/<pid>/cgroup` | 用容器名猜路径，忽略实际 daemon、驱动和 namespace |
| 一组进程有哪些 | `cgroup.procs` | 把本组 PID 列表当作整个子树的任务总数 |
| controller 能否下发 | `cgroup.controllers`、`cgroup.subtree_control` | 认为能看到 controller 就意味着它已经分配到下一层 |
| 子树是否还有活进程 | `cgroup.events` 的 `populated` | 父组没进程就以为其全部后代都空了 |
| 是否触发资源限制 | 各 controller 的计数和事件 | 用一次总值替代问题窗口内的增量 |

进程结束后，cgroup 目录可能由管理器回收。若需要留取一次容器资源事件，应在相应组仍存在时采集，并记下容器身份和观察窗口。 [Linux cgroup v2 核心接口](https://docs.kernel.org/admin-guide/cgroup-v2.html#core-interface-files)

## v2 统一层级以后，多了哪项组织约定

v1 可以把 CPU、memory 等 controller 挂在不同层级，同一个进程在各层级分别归组。一些 controller 也可联合挂载。`/proc/<pid>/cgroup` 可能出现多行。v2 的 controller 共享统一层级，通常可以用 `0::/某个组` 找到进程在该层级的位置。这简化了成员关系，资源文件的单位和语义仍各由 controller 定义。过渡环境可能同时挂载 v1 与 v2，要按节点实际挂载判断某项控制到底在哪一边。 [Linux cgroup v1](https://docs.kernel.org/admin-guide/cgroup-v1/index.html) · [Linux cgroup v2](https://docs.kernel.org/admin-guide/cgroup-v2.html)

假设节点有以下分组。

```text
推理服务这一层
├── 在线推理组
│   ├── 模型进程
│   └── HTTP 进程
└── 离线处理组
    └── 批处理进程
```

在上层的 `cgroup.controllers` 中，可以查看这层获准继续分配哪些资源。上层的 `cgroup.subtree_control` 记录实际给直属子组启用了哪些 controller。例如父组启用 CPU controller 后，子组才能有相应的 CPU 资源控制接口。创建子目录、移动进程和给子树启用 controller 是三个动作。某个文件不存在，应先核对挂载、内核及祖先是否具备并下发该 controller，不宜立即认为文件损坏。

还有一个重要的结构约束。常见非根 domain cgroup 中，准备将 domain controller 向子组分发时，应先把直接运行的进程放到叶子组。这样父节点负责向下分配，叶子节点负责承载任务。根组和 threaded 子树有各自的规则。线上通常由 systemd 和容器运行时管理这些层级，用管理器对应的配置接口来读取和调整更清楚。 [Linux controller 下发与层级约束](https://docs.kernel.org/admin-guide/cgroup-v2.html#controlling-controllers) · [systemd 资源控制](https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html)

## pids 数的是任务，一个进程能用掉多个名额

一个进程本身可以有很多线程。Linux 的 pids controller 计的是内核任务，线程也占用名额。因此应用进程没有变多，线程池扩大却可能碰到 `pids.max`。 `pids.current` 显示当前组及其后代的任务数，`pids.events` 可用于看限制是否被触发。达到有效上限以后可能无法再成功 `fork` 或 `clone`，即使同一节点还有空闲 CPU 和内存。 [Linux pids controller](https://docs.kernel.org/admin-guide/cgroup-v2.html#pid)

用两个子组算一遍。父组最多允许 48 个任务，在线与离线各自允许最多 40 个。在线已经用了 40 个。假设父组内没有其他任务和额外约束，离线当前有 8 个时，组总量已经是 48。离线要创建下一条线程时会受到父组限制。只看到离线自己的 40 名额，会误以为还能继续创建 32 个任务。

`pids.current` 也可能大于当前 `pids.max`。例如管理员把限额调到低于已有任务数，超限状态可以维持，现有任务不会被这个动作自动杀掉。接下来创建新任务时仍会执行有效限制。遇到 `EAGAIN` 时，可以把应用尝试创建线程的时刻、目标组的 pids 事件和父组当前量一起核对。这个错误码还可能来自其他系统约束，所以要保留这条证据链。 [pids controller 的组织操作与创建边界](https://docs.kernel.org/admin-guide/cgroup-v2.html#pid-interface-files)

下面这个对照只在启用了 pids controller 的本地临时容器中运行。它让一个 Python 进程申请若干线程。所有线程在同一个事件上等待，因此能在释放它们之前观察任务数和限制计数。

```sh
docker run --rm -i --network none --cgroupns=private \
  --pids-limit=8 python:3.12-alpine python - <<'PY'
from pathlib import Path
from threading import Event, Thread

release = Event()
threads = []
pids_events = Path("/sys/fs/cgroup/pids.events")
print("before:", pids_events.read_text().strip())
try:
    for _ in range(12):
        worker = Thread(target=release.wait, args=(10,))
        worker.start()
        threads.append(worker)
except RuntimeError as exc:
    print("thread creation failed:", exc)
finally:
    print("worker threads:", len(threads))
    print("pids.current:", Path("/sys/fs/cgroup/pids.current").read_text().strip())
    print("after:", pids_events.read_text().strip())
    release.set()
    for worker in threads:
        worker.join()
PY
```

如果这一组被限制在八个任务，并且只有一个主线程先运行，就预计至多再创建七个同时存活的 worker。判据是创建线程失败时，`pids.current` 接近有效上限且对应事件计数增加。线程释放后当前任务数回落，累计事件仍可记录。这个例子的关键在于它始终只有一个 Python 应用进程，线程却改变了 cgroup 的任务数。

## CPU、内存、I/O 和 cpuset 控制哪些不同维度

| controller | 主要接口 | 核心语义 | 学习时要分开的量 |
|---|---|---|---|
| CPU | `cpu.max`、`cpu.weight`、`cpu.stat` | 周期预算、争用份额和使用统计 | 最多能用多少、争用时怎么分、实际用了多少 |
| Memory | `memory.low`、`memory.high`、`memory.max`、`memory.events` | 回收保护、节流边界、硬上限和事件 | 瞬时用量与事件增量、文件页与匿名页、压力与 OOM |
| I/O | `io.max`、`io.weight`、`io.stat` | 按设备线索进行带宽或操作数限制、权重和统计 | 哪个块设备、读或写、每秒字节还是操作数 |
| Cpuset | `cpuset.cpus.effective`、`cpuset.mems.effective` | 任务允许运行的 CPU、允许使用的内存节点 | 目标配置、祖先约束及最终有效范围 |
| Pids | `pids.max`、`pids.current`、`pids.events` | 子树能够创建的任务数及限制记录 | 进程数、线程数与父组的总容量 |

CPU 的权重主要在争用时起作用。一个组权重大，可能在两个都需要 CPU 的同级组中取得更多时间，但仍受周期预算限制。比如两个子组分别配置 1.5 CPU，父组总预算只有 2 CPU。在两边都饱和时，不能据此承诺两边各拿 1.5 CPU。Cpuset 另外约束可运行的 CPU 和内存节点。即使几个线程可以在一组 CPU 上运行，也仍要遵守总时间预算。 [CPU 预算与节流详解]({{< relref "posts/docker-cgroup-v2-cpu-throttling/index.md" >}})

内存也有几类走向。达到 `memory.high` 会加强回收并可能使分配变慢。达到硬上限后，如果可用回收不足，才可能走到组内 OOM。评估推理服务时分别记模型权重读取引起的文件页、CPU 计划使用的缓冲和设备显存。Device memory 不能直接从 Linux 组内存数值反推。 [内存回收、上限与 OOM]({{< relref "posts/docker-cgroup-v2-memory-oom/index.md" >}})

I/O 要先找到读写实际落入的设备。同一进程写两个挂载时，背后可能是两块不同设备，也可能经过同一块设备的多个层。看 `io.max` 时应按设备号和限制字段区分。使用 cgroup v2 设备权限时，也不要去找 v1 的 `devices.allow` 文件。v2 的设备访问控制使用 cgroup BPF，通常由运行时按配置落实。常见 controller 的功能范围、统计与前提可以继续对照内核文档。 [Linux cgroup v2 资源控制器](https://docs.kernel.org/admin-guide/cgroup-v2.html#controllers)

## v1 字段迁移不能只改名字

| 从旧教程里看到的接口 | v2 中如何理解 |
|---|---|
| `cpu.cfs_quota_us` 与 `cpu.cfs_period_us` | `cpu.max` 中写同一行的配额和周期，`max` 表示不设置此级配额上限 |
| `cpu.shares` | 用 `cpu.weight` 表达相对权重，数值范围和默认值要按 v2 重新核对 |
| `cpuacct.usage` | 查 `cpu.stat` 的 `usage_usec`，注意旧接口常见为纳秒，新字段是微秒 |
| `memory.usage_in_bytes` | 读取 `memory.current`，结合层级与统计口径解释 |
| `memory.soft_limit_in_bytes` | 先决定需要回收保护还是回收阈值，分别评估 `memory.low` 或 `memory.high` |
| `memory.limit_in_bytes` | 使用 v2 的 `memory.max`，并独立观察相关事件与 swap 条件 |
| `blkio.*` | 找 `io.*` 对应的统计或限制，并按实际块设备配置 |
| freezer controller | v2 用核心接口 `cgroup.freeze` 表达冻结状态 |

一些工具看到 `/sys/fs/cgroup` 下没有 `cpu.cfs_quota_us` 就直接退出。遇到这种情况，先查当前真正使用的版本，再决定是升级脚本、适配两套接口还是迁移部署。路径、默认值、单位和资源语义都要一起校对。 [Linux cgroup v1 接口](https://docs.kernel.org/admin-guide/cgroup-v1/index.html) · [Linux cgroup v2 接口](https://docs.kernel.org/admin-guide/cgroup-v2.html)

## 从容器参数只读核对到内核文件

在本机 Linux、采用完整统一挂载且 Docker Engine 也运行在这台机器上的环境，可以用一次性容器复习整个路径。示例创建一个 `sleep` 进程以便采样，容器资源预算由 Docker 配置，脚本只读取内核状态。

```sh
test "$(docker info --format '{{.CgroupVersion}}')" = 2 || exit 1
test "$(findmnt -n -o FSROOT -T /sys/fs/cgroup)" = / || exit 1

demo=$(docker run -d --rm --network none   --cpus=0.5 --memory=128m --pids-limit=64 busybox:1.36 sleep 60) || exit
trap 'docker stop -t 1 "$demo" >/dev/null 2>&1 || true' EXIT

pid=$(docker inspect -f '{{.State.Pid}}' "$demo")
relative=$(sed -n 's/^0:://p' "/proc/$pid/cgroup") || exit
case "$relative" in /?*) ;; *) exit 1 ;; esac
group="/sys/fs/cgroup$relative"

printf 'cgroup: %s\n' "$relative"
for f in cpu.max cpu.stat memory.max memory.current memory.events pids.max pids.current; do
  if test -r "$group/$f"; then
    printf '\n%s\n' "$f"
    cat "$group/$f"
  fi
done
```

建组的具体路径会随 systemd、cgroupfs 驱动、daemon 和委派方式变化。按已知 PID 查比记某个模板更可靠。应能看到这里指定的组限制和当前用量。运行时采用常见默认周期时，半个 CPU 对应 `50000 100000`。128 MiB 对应 134217728 字节。pids 的上限为 64，当前任务仍然很少。这个例子观察配置生效的位置，没有让容器达到 CPU、内存或 pids 上限。远端或桌面虚拟机环境要到 Engine 所在 Linux 读对应的 `/proc` 和 cgroup 挂载。 [Docker 运行时指标](https://docs.docker.com/engine/containers/runmetrics/) · [Docker 资源约束](https://docs.docker.com/engine/containers/resource_constraints/)

## 把这套模型带到 Kubernetes

资源声明到内核执行之间，还有 kubelet、本地 CRI 以及 cgroup 管理器。CPU 和内存请求会进入调度与资源分配等逻辑，实际运行上限应从目标组和祖先的参数读出来。网络和设备身份还各有自己的配置。理解完这一层，再看 Kubernetes 节点为什么使用不同的 slice、scope 或组路径，就能把 API 对象与 Linux 状态分开核对。 [Kubernetes 容器资源管理](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) · [Docker 与 Kubernetes 的运行时入口]({{< relref "posts/docker-kubernetes-containerd-cri/index.md" >}})

下次遇到资源问题，先定位进程和 cgroup，再确认控制器是否有效、各级上限怎么叠加，最后读取本次窗口中的使用量和事件。这个顺序可以复用到任务数、CPU、内存和 I/O。

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
