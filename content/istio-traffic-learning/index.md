---
title: "Istio 流量管理"
description: "从 Sidecar、Ambient 与 xDS 出发，学习 Istio 路由、灰度、超时重试、出口、mTLS、可观测性和迁移。"
---

选一个你手上的 Istio 问题直接读。这里收录 12 篇独立技术笔记，每篇说明版本、示例环境与分析所需的背景。

本系列以 Istio 1.32.0 为配置和源码基线。该版本的支持矩阵列出 Kubernetes 1.33–1.37，Gateway API 构建、测试版本为 v1.6.1。具体能力还随数据平面模式、安装配置和功能阶段变化。 [Istio 版本支持](https://istio.io/latest/docs/releases/supported-releases/) · [功能成熟度](https://istio.io/latest/docs/releases/feature-stages/)

## 请求路径与配置

数据平面对比、xDS 配置下发与路由绑定。

- [Istio 请求经过谁？从 Sidecar 到 Ambient 数据平面]({{< relref "posts/istio-sidecar-ambient-data-path/index.md" >}})
- [从 Istio YAML 到 Envoy 转发：走一遍 xDS 配置链]({{< relref "posts/istio-envoy-xds-routing/index.md" >}})
- [HTTPRoute 挂到哪一层？Istio 入口、网格内路由与 Gateway API]({{< relref "posts/istio-gateway-api-route-attachment/index.md" >}})

## 流量分配与实验

用灰度改变主目标，用镜像发送额外副本，用故障注入改变代理行为。

- [Istio 灰度为什么先选版本，再选实例？]({{< relref "posts/istio-weighted-canary-routing/index.md" >}})
- [Istio 流量镜像复制了什么？主请求、影子请求和副作用]({{< relref "posts/istio-traffic-mirroring/index.md" >}})
- [Istio 故障注入验证了哪一层？延迟、终止与真实后端故障]({{< relref "posts/istio-fault-injection/index.md" >}})

## 韧性与出口

依赖调用的期限和并发预算，以及外部服务如何进入网格。

- [Istio 超时与重试怎么一起算？从一次尝试到流式交付]({{< relref "posts/istio-timeouts-retries-streaming/index.md" >}})
- [Istio 连接池、熔断和异常实例摘除分别做什么？]({{< relref "posts/istio-connection-pool-outlier-detection/index.md" >}})
- [Istio 怎样认识集群外的服务？ServiceEntry 与出口网关]({{< relref "posts/istio-serviceentry-egress/index.md" >}})

## 安全、遥测与迁移

工作负载身份、代理观测、迁移期间的策略交接。

- [Istio mTLS 建立谁的身份？从握手到请求授权]({{< relref "posts/istio-mtls-authorization/index.md" >}})
- [Istio 怎样看见一次请求？指标、日志和 Trace 的边界]({{< relref "posts/istio-telemetry-trace-context/index.md" >}})
- [Istio 从 Sidecar 迁到 Ambient，先核对哪些语义？]({{< relref "posts/istio-sidecar-to-ambient-migration/index.md" >}})
