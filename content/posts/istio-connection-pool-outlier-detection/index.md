---
title: "Istio 连接池、熔断和异常实例摘除分别做什么？"
date: "2026-11-21"
description: "沿 Envoy 上游集群理解 DestinationRule 的连接池、容量保护和异常实例摘除。"
categories: ["Istio 流量管理"]
tags: ["Istio", "DestinationRule", "Envoy", "负载均衡"]
---

连接池的任务是复用或建立连接；熔断给这些连接和并发请求设置成本上限；异常摘除降低一个持续出错端点被继续选择的机会。两个 Pod 都健康但请求把代理的等待队列填满，和只剩下一个 Pod 一直返回 500，都可能让客户端看到 503。要分辨它们，应该看容量统计与实例级证据。

## 从同一个上游集群看连接与请求

Envoy 的上游集群描述一组可供选择的端点以及与其通信的策略。路由先选中这个集群，负载均衡再挑端点，连接池尝试将请求交给可用连接。HTTP/1.1 的活跃请求对连接的占用和 HTTP/2 多路复用不同。把连接数直接当作所有协议下的并发请求上限，会低估一条 HTTP/2 连接承载的流数。等待连接可用的 pending 请求、已向上游发出的 active 请求和额外 retries 各自消耗不同的容量。 [Envoy circuit breaking](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/circuit_breaking)

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: catalog-capacity
  namespace: mesh-study
spec:
  host: catalog.mesh-study.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        connectTimeout: 250ms
        maxConnections: 20
      http:
        http1MaxPendingRequests: 16
        http2MaxRequests: 40
        maxRetries: 4
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 5s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

这些数字用来读懂配置怎样落入 Envoy。Istio 1.32.0 的 `applyConnectionPool` 分别把 `http2MaxRequests`、`http1MaxPendingRequests` 写入 `MaxRequests`、`MaxPendingRequests` 阈值。它们对应代理的上游集群容量，不能按字段名推导它们只针对字面上的某个 HTTP 版本。`maxRetries` 控制同一上游集群的并发重试，上一篇的 `attempts` 控制每条请求最多重试几次。多个 source Sidecar 各有自己的熔断状态，部署 10 个 Sidecar 不会共享一个 20 连接额度。Envoy 对选中端点建立最低连接等细节还可能使观测连接数略高于设定阈值。 [DestinationRule API](https://istio.io/latest/docs/reference/config/networking/destination-rule/)，[固定版本的上游策略转换](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/networking/core/cluster_traffic_policy.go)

## 容量该按什么量纲估算

在流量和平均服务时间相对稳定时，可以用“到达率 × 平均占用时间”估算平均在途请求数。100 个请求每秒，每个在途 200 ms，平均约有 20 个活跃请求；若是 5 条长流每秒、平均持续 12 秒，平均约有 60 个活跃请求。此时连接数即使因为 HTTP/2 多路复用维持很低，40 个活动请求的阈值仍可能很紧。实际设置还要给突发留容量，并区分这个数字来自单个 Sidecar、一个 waypoint，还是多个入口网关实例。随着观测点改变，看到的并发与失败分布会改变。

## 摘除的对象是具体的端点

代理根据自己看到的失败，判断某个 endpoint 是否暂时退出它的负载均衡候选集。`consecutive5xxErrors` 描述连续错误门槛，`baseEjectionTime` 给出基本摘除时长，反复被摘除会延长隔离时间。`interval` 用于周期性检查，不能解读成“连续 500 必须等 5 秒才触发”。`maxEjectionPercent` 限制能同时摘除的比例，低副本服务必须看实际选取行为与有效 Envoy 配置，尤其是只有一个端点或所有端点同时异常的情形。摘除也会把本来去该实例的请求压力转给剩余实例。 [Envoy outlier detection](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier)

例如把三个实例 A、B、C 共用一个 Service。B 的业务进程能响应健康探针，但因为本地缓存失效持续返回 500。当某个 Sidecar 观察到 B 的连续错误时，可以暂时把 B 从自己的候选里移走；另一个低流量 Sidecar 可能还没得到足够样本。在 Kubernetes 层，B 仍然可以显示 Ready。反过来，若 Pod 的 readiness 失败并改变了 EndpointSlice，Istiod 会从发现链路反映端点变化，代理无需先积累 B 的失败次数。这两类来源在故障恢复时都能改变可用端点，但传播和恢复时钟各自不同。

`splitExternalLocalOriginErrors` 可把连接失败等本地错误，与上游返回的错误分开观察和计数。使用它时，应随同 `consecutiveLocalOriginFailures` 设计相应门槛。访问日志的 `UO`、`UH` 等 Response Flags 分别提示上游溢出、没有健康上游等现象，具体原因还应结合 `upstream_rq_pending_overflow`、`upstream_rq_active_overflow`、摘除计数及当前端点状态。在流式推理中，活跃请求占用可能持续数十秒，同样的 RPS 可以对应很不一样的并发。评估阈值时按连接、活跃请求和流持续时间分别看，比用单个 RPS 指标更接近代理面对的压力。 [Envoy 日志字段](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage)

[Istio 流量管理专题]({{< relref "istio-traffic-learning/index.md" >}})

前置阅读 · [Istio 超时与重试怎么一起算？从一次尝试到流式交付]({{< relref "posts/istio-timeouts-retries-streaming/index.md" >}})

继续阅读 · [Istio 怎样认识集群外的服务？ServiceEntry 与出口网关]({{< relref "posts/istio-serviceentry-egress/index.md" >}})
