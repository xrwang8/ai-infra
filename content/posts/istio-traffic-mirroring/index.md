---
title: "Istio 流量镜像复制了什么？主请求、影子请求和副作用"
date: "2026-11-21"
description: "理解 Istio 镜像的复制比例、返回路径和影子服务的数据边界，再与灰度对照。"
categories: ["Istio 流量管理"]
tags: ["Istio", "流量镜像", "VirtualService"]
---

给 `catalog-v2` 5% 的镜像流量，用户仍然应从 `catalog-v1` 得到结果。代理在处理这条路由时额外发出一份请求。主请求和镜像请求分别选择各自的上游，镜像响应由代理忽略。这个机制适合让候选版本接触有代表性的请求形状，但要捕捉 v1 和 v2 回答的差异，仍需从服务侧收集和比较。

## 镜像一次请求时经过哪些地方

假设客户端用 HTTP 请求 `GET /items/42`。处理请求的 Envoy 先找到匹配规则，确定主目标；附在该规则上的镜像策略再决定是否复制。命中镜像比例时，代理为 shadow 目标安排额外请求。Istio 镜像任务记录了镜像请求的 Host 或 `:authority` 通常追加 `-shadow` 的行为。候选服务若按 Host 分配虚拟主机，必须核对它实际接收和匹配的值。配置应部署在会观察 HTTP 的代理上。处于 Ambient 的纯 ztunnel 路径没有这条 HTTP 路由可执行。 [Istio 流量镜像任务](https://istio.io/latest/docs/tasks/traffic-management/mirroring/)

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: catalog-mirror
  namespace: mesh-study
spec:
  hosts: [catalog.mesh-study.svc.cluster.local]
  http:
    - name: shadow-read
      match:
        - method:
            exact: GET
          uri:
            prefix: /items/
      route:
        - destination:
            host: catalog-v1.mesh-study.svc.cluster.local
            port:
              number: 8080
      mirror:
        host: catalog-v2.mesh-study.svc.cluster.local
        port:
          number: 8080
      mirrorPercentage:
        value: 5.0
    - name: primary-only
      route:
        - destination:
            host: catalog-v1.mesh-study.svc.cluster.local
            port:
              number: 8080
```

这是与灰度篇分开的配置样例。`catalog`、`catalog-v1` 和 `catalog-v2` 的定义见专题。显式写出 `mirrorPercentage` 可避免把默认的全量镜像误带入目标环境。规则中的普通 `route` 对所有匹配的请求照常执行，镜像百分比直接描述额外副本的份额。若有 200 个符合条件的请求每秒流入且每次以 5% 概率复制，候选服务平均额外收到约 10 个请求每秒。这还要乘上请求大小、执行成本和连接占用，才是容量估算。 [VirtualService 镜像字段](https://istio.io/latest/docs/reference/config/networking/virtual-service/)

Istio 1.32.0 的 `applyHTTPRouteDestination` 把 `mirror` 或 `mirrors` 转成 Envoy RouteAction 的 `RequestMirrorPolicies`。该策略附在主路由动作上，和 `WeightedClusters` 是不同字段。看配置时，应同时找到主 cluster 与 shadow cluster，再检查镜像比例和两端是否有端点。 [固定版本的镜像转换源码](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/networking/core/route/route.go)

## 只完成主请求，就算镜像成功了吗

镜像比例是代理对符合规则的请求做的选择。影子 Pod 上真正看到的次数还受连接、上游与影子服务容量影响。可以先用每条请求都能关联的合成 ID，分别记录主侧收到的请求数、影子侧收到的请求数和影子侧成功完成的请求数。若主侧有 1000 个样本，估计应复制约 50 个，影子侧却只完成 20 个，需要分清候选端是否根本没收到其余请求，或者已经收到但超时或失败。只用客户端 1000 条成功记录解释影子覆盖率，会把两个阶段混在一起。

## 影子系统可以做什么

HTTP 方法是筛选条件，副作用取决于服务实现。一个 GET 可能写审计记录、刷新缓存、回源计费或触发模型计算。POST 甚至可能带着真实订单到候选服务。镜像会涉及必要的请求头和请求内容，故需要清楚知道令牌、用户数据和访问记录能否进入影子环境。工程上可以限定合成读请求，在影子环境使用独立数据源以及受控身份，再观察响应和依赖调用。

镜像是单独的请求，因此 v2 变慢时用户不会收到它的慢响应，但共享端口、代理 CPU、网络和下游依赖仍可能相互影响。对 SSE 或大请求体，先验证 Envoy 版本、路由行为和流持续时间，并从影子侧测实际负载，再决定是否放大比例。这个区别在推理服务里很明显，一份影子请求如果完整跑 Prefill 和 Decode，影子 GPU 会承担真实成本。

| 观测点 | 能回答的问题 |
| --- | --- |
| 客户端 | 主服务是否在可接受的时间内响应 |
| 主服务 | 该请求实际走了哪个版本，以及错误出现在哪里 |
| 影子服务 | 候选版本收到了多少、如何响应、消耗多少资源 |
| 两侧按请求关联 | 对同样的输入，结果和阶段耗时是否发生变化 |

关联最好使用由实验客户端生成、允许记录的标识，并先检查它在两条路径的实际传递。候选服务能接住真实请求只是镜像验收的一部分；切到灰度后，用户才会直接体验该版本的响应质量。

[Istio 流量管理专题]({{< relref "istio-traffic-learning/index.md" >}})

前置阅读 · [Istio 灰度为什么先选版本，再选实例？]({{< relref "posts/istio-weighted-canary-routing/index.md" >}})

继续阅读 · [Istio 故障注入验证了哪一层？延迟、终止与真实后端故障]({{< relref "posts/istio-fault-injection/index.md" >}})
