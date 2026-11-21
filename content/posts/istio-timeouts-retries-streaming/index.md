---
title: "Istio 超时与重试怎么一起算？从一次尝试到流式交付"
date: "2026-11-21"
description: "区分总期限、单次尝试与空闲计时，再看 SSE、gRPC 和重试成本。"
categories: ["Istio 流量管理"]
tags: ["Istio", "超时", "重试", "流式推理"]
---

同样是“2 秒超时”，连接没建立、上游迟迟没回响应头、SSE 中途停止输出，发生在不同阶段。把一个值放进 VirtualService 只解决其中一层预算。要读懂一次 504，得先知道哪个时钟先到期，以及代理在到期时是否已经向客户端开始交付响应。

## 三种不同的时钟

| 配置位置 | 计时对象 | 典型问题 |
| --- | --- | --- |
| 上游集群的 connect timeout | 上游连接的建立 | 地址或端口正确，但连接迟迟没有建立 |
| 路由的 request timeout、per-try timeout | 上游整次响应与单次尝试的期限 | 连接已经就绪，后端响应太慢 |
| HTTP stream idle timeout | 当前流长时间没有活动 | SSE 或 gRPC 流建立后一直没有数据 |

Envoy 的 route timeout 在收到完整下游请求后开始，等待的是完整的上游响应，因此流式响应也会受有限的路由总期限影响。stream idle timeout 关心光标多久没前进，连接池的 idle timeout 则关心连接上是否还有活动请求或流。只调整后者不能解决前者。应用自己的 deadline 和外层入口的期限还会独立生效。 [Envoy 超时机制](https://www.envoyproxy.io/docs/envoy/latest/faq/configuration/timeouts)

## 带重试的 2 秒如何分配

给 `catalog` 的普通读请求设置总期限 2 s、每次尝试 600 ms，额外允许两次重试，并按示例只对可重试的 `5xx` 分支试验。`attempts: 2` 的上界是一次初始调用和两次重试。如果三次尝试都因为迟迟未开始交付响应而耗尽 600 ms，2 s 只给退避、调度和其他等待留下 200 ms。Envoy 的单次期限只在向下游开始交付响应之前生效，响应一旦开始，流能存活多久由总期限、空闲时间等其他时钟决定。总预算提前用尽、返回了不会重试的结果、或代理无法重新发送请求时，实际尝试次数都可能更少。

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: catalog-read-budget
  namespace: mesh-study
spec:
  hosts: [catalog.mesh-study.svc.cluster.local]
  http:
    - name: read
      match:
        - method:
            exact: GET
          uri:
            prefix: /items/
      timeout: 2s
      retries:
        attempts: 2
        perTryTimeout: 600ms
        retryOn: "5xx"
      route:
        - destination:
            host: catalog.mesh-study.svc.cluster.local
    - name: other
      retries:
        attempts: 0
      route:
        - destination:
            host: catalog.mesh-study.svc.cluster.local
```

在 1.32.0 的 Istio 路由转换里，`setTimeout` 先给 route 设置无总超时，再使用显式写的值；走直接 Envoy API 时遇到的默认值不能套回 Istio。路由省略 `retries` 时，`applyHTTPRouteDestination` 查 mesh 默认重试策略。需要为某条路由明确禁用重试时，使用 `attempts: 0`。`ConvertPolicy` 再把次数、`retryOn`、单次时限等变成 Envoy RetryPolicy。Gateway API 上游 v1.7 的 HTTPRoute retry 字段虽进入 Standard channel，具体重试能力仍需按 Extended Support 表检验。Istio 1.32.0 测试的 Gateway API 版本为 v1.6.1，不能用上游字段存在推断当前安装必然实现。 [Istio 超时及重试 API](https://istio.io/latest/docs/reference/config/networking/virtual-service/)，[路由源码](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/networking/core/route/route.go)，[重试转换源码](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/networking/core/route/retry/retry.go)，[Gateway API 重试指南](https://gateway-api.sigs.k8s.io/guides/user-guides/http-retries/)

一次上游失败也可能消耗多个层次的时间。假设第一次调用在 200 ms 得到 503，随后退避 30 ms，第二次在 600 ms 的单次期限处结束，则第三次最早在约 830 ms 之后开始，还要给下一次退避和转发开销留预算。如果外层调用者只给 1 s，即使内部代理还认为自己有 2 s 总预算，用户也可能先看到外层超时。这个对照需要分别记下每次尝试的起止、取消传播和成功的尝试。只看终局的一个状态码，无法恢复这个时间账本。

在读接口上调试时，可以把总期限显式收紧到小于三次单次期限的和，再用固定延迟的上游观察尝试次数。这样的变化比把所有 timeout 一次调大更容易验证上下游预算是否按预期传播。

## 谁重试，谁承担成本

以三个代理或客户端调用层都各允许两次重试为例，若错误在每一层都满足各自重试条件，理论上最里面的服务可能收到 `3 × 3 × 3 = 27` 次尝试，其中多数没有改善原依赖故障。配置时应先确定哪个层对这次依赖拥有最佳上下文，再对该层给预算，并考虑退避、并发重试容量与熔断。应用返回了超时也可能已经提交写操作。只有带幂等语义、幂等键或有明确定义的清理机制时，重试写请求才有依据。

对生成模型，HTTP 200 送到用户以后，后续响应流不能靠代理悄悄重新开始来修补。SSE 要单独测收到响应头、首个事件和完整交付的时间，并观察取消是否传到模型后端。gRPC 还需理解请求携带的期限和流类型。一次流式测试最好分成三个对照：首包前故障、事件间隔过长和总时长超限，并记录客户端收到的最后一段内容。只有在同一条路径知道客户端、入口、服务代理和推理服务各自的预算，才能解释是哪一层做出的结束决定。

[Istio 流量管理专题]({{< relref "istio-traffic-learning/index.md" >}})

前置阅读 · [Istio 故障注入验证了哪一层？延迟、终止与真实后端故障]({{< relref "posts/istio-fault-injection/index.md" >}})

继续阅读 · [Istio 连接池、熔断和异常实例摘除分别做什么？]({{< relref "posts/istio-connection-pool-outlier-detection/index.md" >}})
