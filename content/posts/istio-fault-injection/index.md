---
title: "Istio 故障注入验证了哪一层？延迟、终止与真实后端故障"
date: "2026-11-21"
description: "拆解 HTTP delay 和 abort 的作用位置、匹配条件与可复核的对照。"
categories: ["Istio 流量管理"]
tags: ["Istio", "故障注入", "VirtualService"]
---

给上游加 250 ms 延迟和让 HTTP 代理立刻返回 503，是两种不同的故障。延迟可以暴露调用方的期限、并行聚合方式和队列增长。终止可以检验上层是否正确处理错误。两者都在匹配到规则的代理处注入，像“后端的第一个线程卡住了”或“服务写了一半就断开”这样的故障，需要不同的实验设计。

## 规则决定哪些请求看到故障

下面用 `x-lab-case` 区分两组测试窗口。假设调用者的 Sidecar 能识别 `catalog:8080` 为 HTTP，且目标服务正常响应 `GET /items/42`。故障比例设为 100% 是为了让两种测试请求有确定的分支。未带指定 Header 的请求走最后一条基线规则。这是一份用于隔离练习环境的完整 VirtualService；与本系列其他 `catalog` 路由分开使用。

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: catalog-fault
  namespace: mesh-study
spec:
  hosts: [catalog.mesh-study.svc.cluster.local]
  http:
    - name: delay
      match:
        - headers:
            x-lab-case:
              exact: slow
      fault:
        delay:
          fixedDelay: 250ms
          percentage:
            value: 100
      route:
        - destination:
            host: catalog.mesh-study.svc.cluster.local
    - name: abort
      match:
        - headers:
            x-lab-case:
              exact: abort
      fault:
        abort:
          httpStatus: 503
          percentage:
            value: 100
      route:
        - destination:
            host: catalog.mesh-study.svc.cluster.local
    - name: baseline
      route:
        - destination:
            host: catalog.mesh-study.svc.cluster.local
```

延迟组中，在客户端和代理期限允许、且不存在其他失败时，代理等约 250 ms 后才发往目标。用户所见耗时还包含连接、请求传输和后端执行。终止组的 503 由故障过滤器直接生成，正常情况下后端不接到这次请求。访问日志里的代理来源、目标服务接收计数和应用日志联合起来，可以区别这次 503 的生成位置。只看业务应用的错误率可能看到正常，因为请求在抵达应用前就结束。对于代理侧的未注入样本，也要把真实上游 503 与本地注入区分开来。 [Istio 故障注入任务](https://istio.io/latest/docs/tasks/traffic-management/fault-injection/)

## 为什么故障注入与路由重试要分开看

Istio 的 API 参考明确说明，同一客户端 HTTP 路由配置了 fault 时，不能把 timeout 或 retry 当成在该 route 上共同生效的实验组合。Istio 1.32.0 的 `TranslateFault` 把 `delay`、`abort` 及比例翻译为 Envoy HTTP fault filter 使用的配置。过滤器位于正常主路由向上游发送之前，这也解释了为什么一个尚未发往后端的请求可以直接得到 503。想验证故障发生在业务后端时路由是否重试，可以让隔离的练习后端返回 503 或超时，再观察一条只配置 timeout 和 retries 的客户端路由。 [VirtualService 约束](https://istio.io/latest/docs/reference/config/networking/virtual-service/)，[固定版本的故障转换源码](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/networking/core/route/route.go)

## 两层预算叠加时怎么解释

考虑 reader 并行调用 catalog 与价格服务的情况。聚合接口总期限为 300 ms，正常的 catalog 自身需要 80 ms。只对 catalog 的一跳加 250 ms 时，该分支的估算完成时间会到 330 ms。即使价格服务在 50 ms 完成，聚合层也需要按自己设计的语义决定提前失败、部分返回还是降级。若把 250 ms 加在入口，两个分支可能都还未启动，实验测到的是另一件事。每次只把变化放进一个调用环节，并记录耗时起点、下游期限和应用是否实际开始处理。

插入固定延迟还会增加处在等待中的并发。假设连续每秒有 120 条请求、只对其中四分之一增加 250 ms，在其他瓶颈不存在且进入速率稳定的近似下，平均额外有 `120 × 0.25 × 0.25 = 7.5` 条请求在等待。该推算用于解释为什么分配相同错误率与延迟比例会产生不同的压力。

| 要验证的现象 | 让哪个环节发生变化 | 应该核对的证据 |
| --- | --- | --- |
| 前端只给整个调用 200 ms | 在代理给一次调用延迟 250 ms，观察客户端 | 客户端期限、代理结束时间、后端是否收到 |
| 应用返回 503 会不会触发重试 | 用练习后端返回预定错误 | 总耗时、上游实际尝试次数、每次结果 |
| SSE 已经开始输出后断开 | 让测试后端在输出若干事件后关闭当前流 | 是否已有响应头、客户端已收事件数、取消及上游清理 |

故障注入百分比与灰度权重的含义也不同。前者决定一条已经选定的路由有多大概率施加某个故障，后者负责挑选上游。一次实验中可先把触发条件收窄，检查无 Header 基线与单一故障分支，再扩大样本观察真实的尾延迟。这样才能知道变化来自哪个环节。

[Istio 流量管理专题]({{< relref "istio-traffic-learning/index.md" >}})

前置阅读 · [Istio 流量镜像复制了什么？主请求、影子请求和副作用]({{< relref "posts/istio-traffic-mirroring/index.md" >}})

继续阅读 · [Istio 超时与重试怎么一起算？从一次尝试到流式交付]({{< relref "posts/istio-timeouts-retries-streaming/index.md" >}})
