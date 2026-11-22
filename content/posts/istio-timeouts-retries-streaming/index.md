---
title: "Istio 超时与重试怎么一起算？从一次尝试到流式交付"
date: "2026-11-21"
lastmod: "2026-11-22"
description: "区分总期限、单次尝试与空闲计时，再看 SSE、gRPC 和重试成本。"
categories: ["Istio 流量管理"]
tags: ["Istio", "超时", "重试", "流式推理"]
---

同样是“2 秒超时”，连接没建立、上游迟迟没回响应头、SSE 中途停止输出，发生在不同阶段。把一个值放进 VirtualService 只解决其中一层预算。要读懂一次 504，得先知道哪个时钟先到期，以及代理在到期时是否已经向客户端开始交付响应。

本文使用 Istio 1.32.0 对应的 Envoy。集群配置示例假定 Sidecar 客户端访问 `mesh-study` 的 catalog HTTP Service，端口 8080；本文还提供一个独立的本地 SSE 小服务，用于区分响应头时钟和事件时钟。

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

## 一次失败是怎样进入重试队列的

Istio 1.32.0 的 `ConvertPolicy` 处理三类输入：没有显式路由策略时使用配置的默认值，`attempts: 0` 不生成该路由的重试策略，显式次数与条件则转换成 Envoy `RetryPolicy`。例如 `retryOn: "503,connect-failure"`，数字 503 会进入 `retriableStatusCodes` 并补上 `retriable-status-codes` 模式，`connect-failure` 仍是 RetryOn 条件。两类触发的证据不同，一个来自上游 HTTP 响应，一个来自建连错误。与 `5xx` 相比，用具体状态码可以明确哪类服务返回会重试。 [Istio RetryPolicy 转换](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/networking/core/route/retry/retry.go#L107-L164)

Envoy 会在收到上游响应头或上游连接重置时判断是否可重试。`RetryStateImpl::shouldRetry` 还检查这条请求剩余次数、cluster 有无可用的并发重试容量或容量预算，再安排下一次尝试。Router 自己持有总时钟，容量足够也可能因总期限先到而无法重试。若请求体没法重放，例如持续上传的请求无法完整缓冲，最大重试次数只是配置上限。已向下游交付响应头之后，代理也无法把用户看到的这一段当作还没有发生。 [Envoy RetryState](https://github.com/envoyproxy/envoy/blob/56e0748e3deaeb8af7efcbf2f97329bb15539767/source/common/router/retry_state_impl.cc#L271-L352) · [Envoy Router 超时处理](https://github.com/envoyproxy/envoy/blob/56e0748e3deaeb8af7efcbf2f97329bb15539767/source/common/router/router.cc#L1348-L1541)

| 事件 | 到期时代理可能已做什么 | 首先该观察什么 |
| --- | --- | --- |
| 上游 TCP 连接建立失败 | 还没有发 HTTP 请求 | cluster 的连接失败或 connect timeout |
| 单次等待响应超过 per-try timeout | 发送了请求，尚未向下游开始响应 | 上游尝试数、单次超时统计、失败条件 |
| 请求总期限耗尽 | 可能已经有若干尝试或刚开始流式交付 | 源侧总期限、代理 Response Flags、成功交付了多少 |
| HTTP stream 一直无活动 | 已有流，但长时间没有有效的流事件 | 哪一跳无进展、代理流空闲配置 |
| 客户端 deadline 更短 | 下游先关闭，代理和应用随后取消 | 客户端计时与上下游取消是否一致 |

上游的连接池空闲计时和下游流空闲计时应分别核对。TCP keepalive 只证明维持连接，不能当生成模型输出。gRPC 还需要考虑应用传递的 `grpc-timeout` 及代理侧 max stream duration。用一个闹钟解释所有期限会把真正结束请求的组件弄错。

## 谁重试，谁承担成本

以三个代理或客户端调用层都各允许两次重试为例，若错误在每一层都满足各自重试条件，理论上最里面的服务可能收到 `3 × 3 × 3 = 27` 次尝试，其中多数没有改善原依赖故障。配置时应先确定哪个层对这次依赖拥有最佳上下文，再对该层给预算，并考虑退避、并发重试容量与熔断。应用返回了超时也可能已经提交写操作。只有带幂等语义、幂等键或有明确定义的清理机制时，重试写请求才有依据。

对生成模型，HTTP 200 送到用户以后，后续响应流不能靠代理悄悄重新开始来修补。SSE 要单独测收到响应头、首个事件和完整交付的时间，并观察取消是否传到模型后端。gRPC 还需理解请求携带的期限和流类型。一次流式测试最好分成三个对照：首包前故障、事件间隔过长和总时长超限，并记录客户端收到的最后一段内容。只有在同一条路径知道客户端、入口、服务代理和推理服务各自的预算，才能解释是哪一层做出的结束决定。

在一个入口同时承接普通 JSON 和 SSE 时，可以拆成两条路由来配预算。普通 GET 适用有限总时长、明确失败条件和有限重试；SSE 可用显式无 route 总超时并关闭代理重试，再由应用、客户端和受支持的 idle/max stream duration 约束流的生命周期。另一种设计是保留明确的总期限，适合服务有产品层的最长生成时间。无总超时仍要处理空闲流，稳定发送心跳也仍会占用活跃请求，保持连接的开销需要和并发预算一起看。

若调用方能消费部分结果，重连机制也需要接口级定义。SSE 事件 ID 可作为具有设计的恢复协议的一部分；普通 proxy retry 只理解请求和响应，并不知道已生成 token、是否应去重，以及如何由新的推理实例恢复。对于一次事件级延迟验收，记录头部首字节、每个完整事件、最终状态和服务端取消后的资源释放时间，再判断调的是哪层预算。 [Envoy 针对流式请求的时钟选择](https://www.envoyproxy.io/docs/envoy/latest/faq/configuration/timeouts)

## 用本地服务区分慢响应头与慢事件

下面的小服务只依赖 Python 标准库。它在本机一条路径延后响应头，另一条路径马上交付 SSE 响应头和首个事件，再把第二个事件延后。运行 `python3 timing_demo.py` 后，从另一个终端执行后面的 curl。这个对照用于观察两个时间点，HTTP/1.0 demo 通过关闭连接结束响应。将同样行为部署到隔离集群的 catalog 后，才可以用 Istio 配置观察代理期限与重试。

```python
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from time import sleep

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/slow-headers":
            sleep(0.7)
            self.send_response(200)
            self.send_header("Content-Type", "text/plain")
            self.end_headers()
            self.wfile.write(b"ready\n")
        elif self.path == "/stream":
            self.send_response(200)
            self.send_header("Content-Type", "text/event-stream")
            self.end_headers()
            for event, delay in ((b"first", 0.0), (b"second", 0.7)):
                sleep(delay)
                try:
                    self.wfile.write(b"data: " + event + b"\n\n")
                    self.wfile.flush()
                except (BrokenPipeError, ConnectionResetError):
                    break
        else:
            self.send_error(404)

if __name__ == "__main__":
    ThreadingHTTPServer(("127.0.0.1", 18080), Handler).serve_forever()
```

```sh
for path in slow-headers stream; do
  curl -NsS --max-time 3 \
  -w "path=$path start=%{time_starttransfer} total=%{time_total}\n" \
  "http://127.0.0.1:18080/$path"
done
```

curl 的 `time_starttransfer` 记录收到首字节的时间，应用层首个 SSE 事件的时间还需要按完整事件边界记。两条路径可能有近似的总耗时，用户观感和代理能否在响应前安排 retry 却不同。若要用上文 `/items/` 路由验证，保持 path match 与测试后端一致，并分别记录应用实际接收次数、客户端响应是否已开始。

---

收录于 [Istio 流量管理]({{< relref "istio-traffic-learning/index.md" >}})。
