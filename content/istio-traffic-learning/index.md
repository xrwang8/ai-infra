---
title: "Istio 流量管理"
description: "从 Sidecar、Ambient 与 xDS 出发，学习 Istio 路由、灰度、超时重试、出口、mTLS、可观测性和迁移。"
---

理解 Istio 流量管理，可以沿着一次服务调用往下走：请求先经过哪个代理，规则经过控制面怎样到达它，上游又怎样被选中。沿着这条路径，可以把灰度权重、重试、限流保护、身份和遥测放回它们实际发生的位置。

本系列以 Istio 1.32.0 为配置和源码基线。该版本的支持矩阵列出 Kubernetes 1.33–1.37，Gateway API 构建、测试版本为 v1.6.1。具体能力还随数据平面模式、安装配置和功能阶段变化。 [Istio 版本支持](https://istio.io/latest/docs/releases/supported-releases/) · [功能成熟度](https://istio.io/latest/docs/releases/feature-stages/)

## 请求路径与配置

先找执行点，再理解 Istiod 如何把声明转成代理行为，最后学会给路由选 parent。

1. [Istio 请求经过谁？从 Sidecar 到 Ambient 数据平面]({{< relref "posts/istio-sidecar-ambient-data-path/index.md" >}})
2. [从 Istio YAML 到 Envoy 转发：走一遍 xDS 配置链]({{< relref "posts/istio-envoy-xds-routing/index.md" >}})
3. [HTTPRoute 挂到哪一层？Istio 入口、网格内路由与 Gateway API]({{< relref "posts/istio-gateway-api-route-attachment/index.md" >}})

## 流量分配与实验

用灰度改变主目标，用镜像发送额外副本，用故障注入改变代理行为。

4. [Istio 灰度为什么先选版本，再选实例？]({{< relref "posts/istio-weighted-canary-routing/index.md" >}})
5. [Istio 流量镜像复制了什么？主请求、影子请求和副作用]({{< relref "posts/istio-traffic-mirroring/index.md" >}})
6. [Istio 故障注入验证了哪一层？延迟、终止与真实后端故障]({{< relref "posts/istio-fault-injection/index.md" >}})

## 韧性与出口

沿依赖调用给期限和并发定预算，最后把服务注册和真实出网路径接起来。

7. [Istio 超时与重试怎么一起算？从一次尝试到流式交付]({{< relref "posts/istio-timeouts-retries-streaming/index.md" >}})
8. [Istio 连接池、熔断和异常实例摘除分别做什么？]({{< relref "posts/istio-connection-pool-outlier-detection/index.md" >}})
9. [Istio 怎样认识集群外的服务？ServiceEntry 与出口网关]({{< relref "posts/istio-serviceentry-egress/index.md" >}})

## 安全、遥测与迁移

在路径上标出身份校验、观测和策略执行的位置，再考虑替换数据平面。

10. [Istio mTLS 建立谁的身份？从握手到请求授权]({{< relref "posts/istio-mtls-authorization/index.md" >}})
11. [Istio 怎样看见一次请求？指标、日志和 Trace 的边界]({{< relref "posts/istio-telemetry-trace-context/index.md" >}})
12. [Istio 从 Sidecar 迁到 Ambient，先核对哪些语义？]({{< relref "posts/istio-sidecar-to-ambient-migration/index.md" >}})

## 示例约定

多数例子使用隔离的 `mesh-study` 命名空间，服务程序监听 HTTP `8080`。假设两个版本的 Pod 都有 `app: catalog`，另外分别有 `version: v1` 和 `version: v2`。读接口 `GET /items/42` 以响应头 `X-Backend-Version` 标注版本。做延迟和流式对照时，用能够独立控制响应时间和事件间隔的测试后端。

| 资源 | 选择器或用途 |
| --- | --- |
| `catalog` Service | `app: catalog`，用于常规访问或作为网格路由的 parent |
| `catalog-v1` Service | `app: catalog, version: v1`，作为 Gateway API 或镜像的具体目标 |
| `catalog-v2` Service | `app: catalog, version: v2`，作为 Gateway API 或镜像的具体目标 |
| `reader` ServiceAccount | 用作授权示例客户端的工作负载身份 |

三个 Service 使用相同端口定义，下面给出公共 Service 的例子。其余两个分别增加版本 selector，并改用各自的对象名。示例中的多个路由对象可能都匹配 `catalog`，每次只应用正在阅读的配置。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: catalog
  namespace: mesh-study
spec:
  selector:
    app: catalog
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

文中的定量例子用于推导机制；示例 YAML 经过静态校验。要核对集群中的转发、时延或策略执行，请在匹配的安装和真实请求下观察。
