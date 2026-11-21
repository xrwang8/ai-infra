---
title: "Istio 灰度为什么先选版本，再选实例？"
date: "2026-11-21"
description: "对照 VirtualService 与 HTTPRoute 的路由匹配、权重和灰度验证口径。"
categories: ["Istio 流量管理"]
tags: ["Istio", "灰度发布", "VirtualService", "HTTPRoute"]
---

灰度的基本单位是一条匹配到路由的请求。代理先决定使用哪条路由，再从路由列出的目标中按权重选一个，最后从该目标的可用实例里挑一个连接。把这三步分开，就能理解两件事：给 v2 配 10% 流量和给 v2 部署 10% 的 Pod 是两种操作；一条长时间存在的 SSE 请求也只在建立时选择一次，后续事件不会各自重新参加灰度。

## 先匹配，再算权重

设 `catalog` 是一个 HTTP 服务。我们希望携带 `x-release-track: preview` 的读请求全部进 v2，剩下的请求按 90/10 分流。Istio `VirtualService.spec.http` 自上而下匹配并采用第一条命中的规则。下面这两条规则交换位置，第一条无条件匹配就会吃掉全部流量。单条规则内，一个 `match` 项中的条件需要同时满足，多个 `match` 项之间是“或”的关系。该请求头只用于受控预览流量，代码或网关应明确它的设置方。 [VirtualService API](https://istio.io/latest/docs/reference/config/networking/virtual-service/)

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: catalog
  namespace: mesh-study
spec:
  host: catalog.mesh-study.svc.cluster.local
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: catalog
  namespace: mesh-study
spec:
  hosts: [catalog.mesh-study.svc.cluster.local]
  http:
    - name: preview
      match:
        - method:
            exact: GET
          headers:
            x-release-track:
              exact: preview
      route:
        - destination:
            host: catalog.mesh-study.svc.cluster.local
            subset: v2
    - name: weighted
      route:
        - destination:
            host: catalog.mesh-study.svc.cluster.local
            subset: v1
          weight: 90
        - destination:
            host: catalog.mesh-study.svc.cluster.local
            subset: v2
          weight: 10
```

两个 subset 使用同一个 Service 的端点，`DestinationRule` 再根据工作负载标签筛选。路由权重分配给由 Service 和 subset 定义的逻辑目标。把某个 subset 扩容一倍，会改变该 subset 内部各实例处理请求的机会，路由声明的目标权重仍然是 90/10。缩容到没有可用 v2 端点时，被选中去 v2 的请求可能失败，所以配置正常下发也不能证明灰度可用。

在 Istio 1.32.0 源码里，`BuildHTTPRoutesForVirtualService` 遍历 HTTP 规则并在遇到 catch-all 时结束。翻译动作把多目标放到 Envoy 的 `WeightedClusters`，每个目标用 Service、端口、subset 形成上游集群名。排查时先看路由上的 cluster，再看 cluster 的端点，不然容易把“路由没选 v2”和“已经选到 v2，但没有健康实例”混在一起。 [固定版本的路由转换源码](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/networking/core/route/route.go)

## 用 HTTPRoute 表达同一件事

Gateway API 这套写法让 `catalog-v1` 和 `catalog-v2` 分别成为两个 Service，它们的 selector 包含各自的版本。示例挂在上游 `catalog` Service 上，适用于实现了 GAMMA 网格路由的路径，例如目标 Service 已绑定可处理 HTTP 的 waypoint。入口场景则把 parent 换成相应的 Gateway Listener。两份示例用于比较，用单一 API 验证同一条路径，避免把两类路由的重叠当成可移植的优先级。

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: catalog
  namespace: mesh-study
spec:
  parentRefs:
    - group: ""
      kind: Service
      name: catalog
      port: 8080
  rules:
    - matches:
        - method: GET
          headers:
            - name: x-release-track
              value: preview
      backendRefs:
        - name: catalog-v2
          port: 8080
    - backendRefs:
        - name: catalog-v1
          port: 8080
          weight: 9
        - name: catalog-v2
          port: 8080
          weight: 1
```

这里的 `9:1` 与 `90:10` 等价。HTTPRoute 权重按总和归一化，省略权重时单个 backend 默认为 1，权重为 0 的目标不应收到该规则的流量。匹配优先级由规范定义，包含路径类型和长度，以及方法、Header、Query 等条件。它和 VirtualService 的列表顺序规则不同。复杂规则应该先推导优先级，再用实际请求命中哪条规则来检验。 [HTTPRoute 规范](https://gateway-api.sigs.k8s.io/reference/api-types/httproute/)

## 版本路由与会话粘性解决不同的问题

随机分给 v2 的 10% 以请求为粒度。同一个客户端发第二次请求时，仍可能选中 v1；即使 DestinationRule 使用一致性哈希，哈希也作用在已选上游集群的实例选择上。如果功能依赖连续操作都走相同版本，例如多次续传共享协议状态，应当在选择版本的那一步用显式分组条件。可以由应用或入口基于可信会话分组，再用已知 Header 映射到 v1 或 v2；随后才在组内判断要不要做实例级粘性。这样版本人群分割与实例负载均衡可以分别验证。 [DestinationRule 负载均衡](https://istio.io/latest/docs/reference/config/networking/destination-rule/)

## 10% 到底怎么验收

让后端在响应头标记自己的版本，在没有预览头的请求中记录每个版本的计数，并单独记录错误。假设没有重试或失败，1000 次独立抽样以 10% 概率去 v2，期望命中 100 次，标准差约 9.5 次。这是便于理解样本波动的模型。若客户端应该发 1000 次，却只统计 900 条 2xx，并且 v2 错误更多，“v2 成功响应占比”就会低于“被选中去 v2 的请求占比”。这时还要按路由和上游 cluster 看代理证据。HTTP/2 连接可承载许多独立请求，单个长流则需要按请求计数。

一次业务灰度也应分开比较错误率、延迟、取消率和 v2 的资源负载。对长流，额外记录连接持续时间以及新进入和已存续的流数。规则变更主要影响新建请求，已经开始的流仍按原来选定的上游完成或退出。 [Istio 权重切换任务](https://istio.io/latest/docs/tasks/traffic-management/traffic-shifting/)

[Istio 流量管理专题]({{< relref "istio-traffic-learning/index.md" >}})

前置阅读 · [HTTPRoute 挂到哪一层？Istio 入口、网格内路由与 Gateway API]({{< relref "posts/istio-gateway-api-route-attachment/index.md" >}})

继续阅读 · [Istio 流量镜像复制了什么？主请求、影子请求和副作用]({{< relref "posts/istio-traffic-mirroring/index.md" >}})
