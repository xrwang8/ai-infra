---
title: "Istio 故障注入验证了哪一层？延迟、终止与真实后端故障"
date: "2026-11-21"
lastmod: "2026-11-22"
description: "拆解 HTTP delay 和 abort 的作用位置、匹配条件与可复核的对照。"
categories: ["Istio 流量管理"]
tags: ["Istio", "故障注入", "VirtualService"]
---

给上游加 250 ms 延迟和让 HTTP 代理立刻返回 503，是两种不同的故障。延迟可以暴露调用方的期限、并行聚合方式和队列增长。终止可以检验上层是否正确处理错误。两者都在匹配到规则的代理处注入，像“后端的第一个线程卡住了”或“服务写了一半就断开”这样的故障，需要不同的实验设计。

本文以 Istio 1.32.0 的 Sidecar 出站为例。`mesh-study` 内的 reader 是带 Sidecar 的 curl 客户端，catalog Service 以 HTTP 8080 提供健康的 `GET /items/42`。只有专用实验请求携带试验 Header。

## 规则决定哪些请求看到故障

下面用 `x-lab-case` 区分两组测试窗口。假设调用者的 Sidecar 能识别 `catalog:8080` 为 HTTP，且目标服务正常响应 `GET /items/42`。故障比例设为 100% 是为了让两种测试请求有确定的分支。未带指定 Header 的请求走最后一条基线规则。把这份 VirtualService 用在隔离练习环境，同一主机上只配置这组试验路由。

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

## Envoy 故障过滤器怎样暂停请求

在 Istio 的 `TranslateRoute` 里，路由下的 `fault` 不会被放进上游 cluster，而是先用 `TranslateFault` 编码，再写入 route 的 `typedPerFilterConfig`。Envoy HTTP filter chain 中需要有 fault filter 来消费它。对固定延迟，Istio 把时长及概率映射进 envoy 的 `HTTPFault.delay`；对 abort，把 HTTP 或 gRPC 状态及概率写入 `HTTPFault.abort`。所以查看路由时只关注 `route.cluster` 会漏掉真正执行的配置。 [Istio 路由过滤器配置](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/networking/core/route/route.go#L550-L589) · [Fault API 转换](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/networking/core/route/route.go#L1462-L1513)

Envoy 的 `FaultFilter::decodeHeaders` 在正常 router 发往上游之前执行。它选择本路由有效配置，满足条件时调用 `maybeSetupDelay`，给 dispatcher 注册定时器并暂停处理。定时器到期，`postDelayInjection` 可继续处理，也可以根据同条规则的 abort 设定直接结束请求。没有延迟而命中 abort 时，走 `abortWithStatus` 直接生成本地响应。这个过程占用一个等待中的流和定时器，线程可以继续处理其他流。合用 delay 与 abort 时，可以先延迟后失败；把它们放在两条不同的 match 分支时，则每个请求只采用先匹配的一条。 [Envoy fault filter 请求头阶段](https://github.com/envoyproxy/envoy/blob/56e0748e3deaeb8af7efcbf2f97329bb15539767/source/extensions/filters/http/fault/fault_filter.cc#L105-L195) · [定时器到期与本地响应](https://github.com/envoyproxy/envoy/blob/56e0748e3deaeb8af7efcbf2f97329bb15539767/source/extensions/filters/http/fault/fault_filter.cc#L433-L496)

在同时配置两种非零概率时，单个请求存在四种结果：正常、仅延迟、仅失败、延迟后失败。如果用相互独立的抽样做概念估算，30% 延迟、20% abort 会给出约 6% 的共同命中份额。真实环境还会受具体匹配、代理配置和活跃故障上限影响，延迟和终止计数应以代理实际观测为准。默认的验证方式是先分别做 100% 的小批量对照，再组合比例。

## 让客户端状态、代理和后端同时有证据

源端 Envoy 的本地回复可以带 `fault_filter_abort` 细节和代表故障注入的 Response Flag，延迟路径也有对应 flag 与统计。如果两个分支都返回 503，只看 HTTP code 不够。给各组同样的 URL、受控输入和超过 250 ms 的客户端总时间，记录状态与耗时；应用端核对 abort 对照中的实验请求是否到达。

```sh
kubectl -n mesh-study exec deploy/reader -c reader -- sh -c '
  url=http://catalog.mesh-study.svc.cluster.local:8080/items/42
  for case in baseline slow abort; do
    curl -sS --max-time 2 -o /dev/null \
      --header "x-lab-case: $case" \
      --write-out "$case status=%{http_code} start=%{time_starttransfer} total=%{time_total}\n" \
      "$url"
  done
'
```

命中 slow 时，网络连接和后端执行还会贡献时间，所以重点看相对基线的偏移；abort 比对后端 `GET /items/42` 的正常请求计数。若客户端本身的 deadline 设得更短，可能先看到客户端错误并提前关闭连接。此时不能从客户端状态直接证明故障定时器已经到期。

## 为什么端到端超时会先在另一个代理出现

假设 reader 等整个操作 300 ms，catalog 分支进入源 Sidecar 后被故障过滤器暂停 250 ms。此时外层 ingress 的总时间可能已经开始，catalog 上游 cluster 还没有一个进行中的应用请求。延迟结束后，端到端预算仅剩之前扣除开销后的部分。客户端重试整次请求时，又会重新经过 delay 的 match，这会在外层放大开销。因此教学习题里要给故障注入、应用超时、调用方重试不同的触发组，保持一个变量变化。

可再对照同样的 503 来自应用和来自代理两种情形。应用 503 表示已经走过正常 Router 和上游连接；故障注入的本地 503 发生在当前 route 的正常上游转发之前。上一级代理只看到下一级返回的状态码，仍可能按自己的路由重试，除非也在那一级关闭或约束了重试。API 对同一路由的限制不能直接推广为全链路没有重试。 [虚拟服务的 fault 与 retries 字段](https://istio.io/latest/docs/reference/config/networking/virtual-service/)

## 两层预算叠加时怎么解释

考虑 reader 并行调用 catalog 与价格服务的情况。聚合接口总期限为 300 ms，正常的 catalog 自身需要 80 ms。只对 catalog 的一跳加 250 ms 时，该分支的估算完成时间会到 330 ms。即使价格服务在 50 ms 完成，聚合层也需要按自己设计的语义决定提前失败、部分返回还是降级。若把 250 ms 加在入口，两个分支可能都还未启动，实验测到的是另一件事。每次只把变化放进一个调用环节，并记录耗时起点、下游期限和应用是否实际开始处理。

插入固定延迟还会增加处在等待中的并发。假设连续每秒有 120 条请求、只对其中四分之一增加 250 ms，在其他瓶颈不存在且进入速率稳定的近似下，平均额外有 `120 × 0.25 × 0.25 = 7.5` 条请求在等待。该推算用于解释为什么分配相同错误率与延迟比例会产生不同的压力。

| 要验证的现象 | 让哪个环节发生变化 | 应该核对的证据 |
| --- | --- | --- |
| 前端只给整个调用 200 ms | 在代理给一次调用延迟 250 ms，观察客户端 | 客户端期限、代理结束时间、后端是否收到 |
| 应用返回 503 会不会触发重试 | 用练习后端返回预定错误 | 总耗时、上游实际尝试次数、每次结果 |
| SSE 已经开始输出后断开 | 让测试后端在输出若干事件后关闭当前流 | 是否已有响应头、客户端已收事件数、取消及上游清理 |

故障注入百分比与灰度权重的含义也不同。前者决定一条已经选定的路由有多大概率施加某个故障，后者负责挑选上游。一次实验中可先把触发条件收窄，检查无 Header 基线与单一故障分支，再扩大样本观察真实的尾延迟。这样才能知道变化来自哪个环节。

---

收录于 [Istio 流量管理]({{< relref "istio-traffic-learning/index.md" >}})。
