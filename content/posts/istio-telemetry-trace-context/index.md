---
title: "Istio 怎样看见一次请求？指标、日志和 Trace 的边界"
date: "2026-11-21"
lastmod: "2026-11-22"
description: "从源与目标代理的视角理解 Istio 指标、Envoy 日志、采样和链路上下文。"
categories: ["Istio 流量管理"]
tags: ["Istio", "Telemetry", "OpenTelemetry", "可观测性"]
---

同一个 HTTP 请求，源端代理和目标端代理都可能记录一次；Ingress 和 waypoint 加进路径后，又多了观测点。Istio 的指标、日志、Trace 分别回答不同问题。把这些信号按“在哪儿观察、观察到了哪一段”组织起来，才能解释错误率、长尾延迟和追踪缺口。

以下指标与配置以 Istio 1.32.0 为基线。`mesh-study` 中的 HTTP catalog Service 接受 reader 的调用。用的是经典 Prometheus 直方图；默认拥有 source、destination 的例子针对 Sidecar，Ambient 的 reporter 单独说明。

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

## 用直方图找候选版本的尾延迟

下面按版本统计 catalog 在目标端看到的 P99。保留 `le` 是因为每条时间序列是一个累计区间，先在相同区间和版本上求和，再把分布交给 `histogram_quantile`。结果单位是毫秒。将 `destination_version` 去掉能得到同一口径的聚合分布。对只有很少请求的候选版本，P99 也可能由一个桶内插值主导，应同时展示请求数和 bucket 范围。 [Prometheus 直方图和分位数](https://prometheus.io/docs/practices/histograms/)

```promql
histogram_quantile(
  0.99,
  sum by (le, destination_version) (
    rate(istio_request_duration_milliseconds_bucket{
      reporter="destination",
      destination_service_name="catalog",
      destination_service_namespace="mesh-study"
    }[5m])
  )
)
```

前面的错误率表达式若没有任何 5xx 标签，分子可能没有时间序列；真实服务有成功流量时，可用 `or on() vector(0)` 为单服务聚合分子补 0，再用流量分母大于零作为展示条件。不要用同样方式把整个服务抓取中断当成零错误率。对 SSE，一个未结束的流还可能没有完成时的请求历时记录，实时负载要另查在途请求和应用阶段指标。

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

## 访问日志字段应能区分本地回复和上游回复

对于推理入口，下面这些 Envoy 格式操作符适合用于结构化访问日志。同一个状态码要结合生成位置和上游选取来解释，有上游地址仍不能证明应用完成了用户请求。

| 字段 | 用来回答什么 |
| --- | --- |
| `%ROUTE_NAME%` | 请求匹配了哪一个代理 route |
| `%RESPONSE_CODE%`、`%RESPONSE_CODE_DETAILS%` | 代理最终返回了什么，附加原因是什么 |
| `%RESPONSE_FLAGS%` | 代理是否报告溢出、本地故障或没有健康上游 |
| `%UPSTREAM_CLUSTER%`、`%UPSTREAM_HOST%` | 被选的逻辑集群及可见的上游实例 |
| `%DURATION%` | 当前代理观测到的总历时，适合对照同一跳 |
| `%REQ(X-REQUEST-ID)%` | 查找对应日志的请求标识，注意日志留存与最小化 |

把日志里的网格视角和应用处理阶段对齐，可以解释为什么某个 trace 非常长。跨跳比较时应核对时钟同步和采样。Metrics 可告诉我们过去 5 分钟延迟升高，固定 route 的日志负责说明哪个分支有变化，从代表性样本的 Trace 再看等待落在哪一段。三者的样本范围各自独立，不能将未采到的 Trace 计为失败请求。 [Envoy 访问日志格式](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage)

## 读 Telemetry 怎样落进代理与应用

Istio 1.32.0 的 `applicableTelemetries` 先合并 root Namespace 配置和代理 Namespace 配置，再对匹配的 selector 或 targetRefs 做处理。waypoint 跨命名空间服务时，代码还会纳入服务 Namespace 的策略。`configureTracingFromTelemetry` 根据 listener 选 client 或 server 视角，查 provider 和采样配置，再写入 Envoy 的 HTTP Connection Manager。若 provider 配了自定义的 OpenTelemetry sampler，Istio 为那条采样路径把 Envoy 采样比例调到 100%，将选择交给 sampler。由此可以判断修改 Telemetry 的随机比例为何不一定改变实际导出数。 [Telemetry 策略绑定及合并](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/model/telemetry.go#L411-L507) · [HTTP tracing 配置](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/networking/core/tracing.go#L72-L151)

为了让应用处理两段连接时仍处于同一条 Trace，可以在 reader 的 HTTP handler 中显式提取和注入上下文。以下 Python 函数只依赖 OpenTelemetry API 与标准库，进程启动时还需配置 OpenTelemetry SDK 和 exporter。已经使用框架自动 instrumentation 时，让框架负责相同的步骤，避免重复生成 span。这里选用 W3C propagator，只传递 trace 上下文，调用的目标也固定为内部 catalog。

```python
from urllib.request import Request, urlopen

from opentelemetry import trace
from opentelemetry.trace import SpanKind
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator

tracer = trace.get_tracer("reader")
propagator = TraceContextTextMapPropagator()
CATALOG = "http://catalog.mesh-study.svc.cluster.local:8080/items/42"

def fetch_catalog(incoming_headers):
    parent = propagator.extract(
        {key.lower(): value for key, value in incoming_headers.items()}
    )
    with tracer.start_as_current_span("reader.catalog", context=parent, kind=SpanKind.CLIENT):
        outgoing_headers = {}
        propagator.inject(outgoing_headers)
        request = Request(CATALOG, headers=outgoing_headers)
        with urlopen(request, timeout=2) as result:
            return result.read()
```

若入口收到的 trace ID 为 T，reader 创建子 span 后发给 catalog 的 `traceparent` 仍然应带 T，但 parent span ID 变成 reader 这次客户端 span 的 ID。日志中的 `x-request-id` 可以用于查单次请求，不能代替分布式上下文的父子关系。若上下游已在同一 ID 而某一段不可见，再按采样、provider、导出端和采集端顺序检查。 [OpenTelemetry Python 上下文传播](https://opentelemetry.io/docs/languages/python/propagation/)

---

收录于 [Istio 流量管理]({{< relref "istio-traffic-learning/index.md" >}})。
