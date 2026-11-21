---
title: "Istio 怎样看见一次请求？指标、日志和 Trace 的边界"
date: "2026-11-21"
description: "从源与目标代理的视角理解 Istio 指标、Envoy 日志、采样和链路上下文。"
categories: ["Istio 流量管理"]
tags: ["Istio", "Telemetry", "OpenTelemetry", "可观测性"]
---

同一个 HTTP 请求，源端代理和目标端代理都可能记录一次；Ingress 和 waypoint 加进路径后，又多了观测点。Istio 的指标、日志、Trace 分别回答不同问题。把这些信号按“在哪儿观察、观察到了哪一段”组织起来，才能解释错误率、长尾延迟和追踪缺口。

## 指标先确定分母和观察边界

Istio 标准 HTTP 指标中的 `reporter` 标记报告视角。Sidecar 模式里，源端代理通常上报 `source`，目标端代理通常上报 `destination`。同一调用被两端各记一次时，把两个 reporter 求和会重复计数。看 catalog 的目标 HTTP 错误率，可以分别用同样的命名空间与 reporter 求 5xx 和总请求。若请求在客户端代理被熔断或没有找到健康上游，目标侧根本没收到，对定位这类问题应补看源侧和入口的统计。对于 Ambient，纯 ztunnel 只能提供四层可见性，需要 waypoint 或其他七层观测点才能补齐对应 HTTP 指标。迁移指南记录的 waypoint 指标使用 `reporter="waypoint"`，复用 Sidecar 面板前应检查新数据实际携带的 reporter。 [Ambient 遥测迁移](https://istio.io/latest/docs/ambient/migrate/enable-ambient-mode/#post-migration-observability-changes) [Istio 标准指标及标签](https://istio.io/latest/docs/reference/config/metrics/)，[Ambient L7 能力](https://istio.io/latest/docs/ambient/usage/l7-features/)

```promql
sum(rate(istio_requests_total{
  reporter="destination",
  destination_service_name="catalog",
  destination_service_namespace="mesh-study",
  response_code=~"5.."
}[5m]))
/
sum(rate(istio_requests_total{
  reporter="destination",
  destination_service_name="catalog",
  destination_service_namespace="mesh-study"
}[5m]))
```

查询假设抓取了符合这些标签的 Istio 指标。空样本与零流量要单独处理；gRPC 应按 `grpc_response_status` 理解应用结果，很多 gRPC 错误的 HTTP 状态仍是 200。请求历时的分位数也应先按相同观察边界合计直方图 bucket，再计算分位数。不同 Pod 的 P99 不能求平均来得到整个服务的 P99。Envoy HTTP 请求历时表示代理看来整个流的时间；模型应用的 TTFT、输出间隔和用户收到数据的间隔需要应用或客户端相应埋点。将 request ID、用户 ID 等高基数字段写进指标标签会大幅增加时间序列，单请求明细适合留在受控日志或 Trace 中。 [Prometheus 直方图说明](https://prometheus.io/docs/practices/histograms/)

请求的分位数发生变化时，先固定服务、报告视角和观测窗口，再确定是否要把版本标签保留下来。例如 v1 流量占 95%，v2 占 5%，总 p95 可能把低流量版本的退化吞在总体分布里。先分别得到每个版本的 bucket，再并行计算总量与分版本分位数。如果 v2 在入口被拒绝，目标端的样本可能根本没有 v2，进一步需要看路由日志和源视角。延迟图和错误率图的分母应该指的是清楚定义的一组请求。

## 日志解释单次结果，Trace 串起分段时间

访问日志可记录路由、状态码、Response Flags 和上游地址，用来回答某个请求在哪个代理结束。例如“入口返回 503”只告诉我们状态，结合 `%RESPONSE_CODE_DETAILS%`、`%RESPONSE_FLAGS%` 与上游访问记录，才能区分入口本地回复或上游服务回了 503。应用日志另有职责，它知道业务处理到哪步。日志中一般不记录授权头和敏感正文。下面在隔离的 Sidecar 样例中显式使用 `envoy` provider 打开 catalog 的访问日志，前提是安装具有该日志提供方；同样诉求迁到 waypoint 时，应按 Telemetry 支持的 `targetRefs` 绑定。 [Envoy 日志字段](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage)，[Istio Telemetry API](https://istio.io/latest/docs/reference/config/telemetry/)

```yaml
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
  name: catalog-access-log
  namespace: mesh-study
spec:
  selector:
    matchLabels:
      app: catalog
  accessLogging:
    - providers:
        - name: envoy
      disabled: false
```

对于 Trace，Envoy 能为自己观察到的请求创建 span，但应用从一个入站 HTTP 请求发起另一个出站 HTTP 请求时，两个调用是应用进程内的因果关系。应用需要传播与 provider 匹配的上下文，例如 W3C 的 `traceparent` 和 `tracestate`，或相应的 B3 字段。`randomSamplingPercentage` 针对尚无采样决定的流量，已有决定时通常沿用；采集端过滤、丢弃与导出失败还会继续影响最终所见。 [Istio Trace 概览和上下文传播](https://istio.io/latest/docs/tasks/observability/distributed-tracing/overview/)

用服务间两跳的合成请求做对照很容易理解这种边界。先发起能覆盖链路的请求，从源代理看请求何时发送，目标代理看它何时到达，应用日志记进入和退出，Trace 查两个调用是否处于同一个 trace ID。如果访问日志证明两跳都处理了请求而 Trace 拆成两条，先检查应用传播和采样配置。一个日志或一个 span 的缺口，应从观测链路定位，不能当成请求本身消失。

[Istio 流量管理专题]({{< relref "istio-traffic-learning/index.md" >}})

前置阅读 · [Istio mTLS 建立谁的身份？从握手到请求授权]({{< relref "posts/istio-mtls-authorization/index.md" >}})

继续阅读 · [Istio 从 Sidecar 迁到 Ambient，先核对哪些语义？]({{< relref "posts/istio-sidecar-to-ambient-migration/index.md" >}})
