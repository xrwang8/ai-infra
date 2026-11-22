---
title: "Istio 灰度为什么先选版本，再选实例？"
date: "2026-11-21"
lastmod: "2026-11-22"
description: "对照 VirtualService 与 HTTPRoute 的路由匹配、权重和灰度验证口径。"
categories: ["Istio 流量管理"]
tags: ["Istio", "灰度发布", "VirtualService", "HTTPRoute"]
---

灰度的基本单位是一条匹配到路由的请求。代理先决定使用哪条路由，再从路由列出的目标中按权重选一个，最后从该目标的可用实例里挑一个连接。把这三步分开，就能理解两件事：给 v2 配 10% 流量和给 v2 部署 10% 的 Pod 是两种操作；一条长时间存在的 SSE 请求也只在建立时选择一次，后续事件不会各自重新参加灰度。

配置基线为 Istio 1.32.0。Sidecar 示例只需要一个名为 `catalog` 的 HTTP Service，在 `mesh-study` 暴露 8080，selector 为 `app: catalog`，其 Pod 分别标 `version: v1`、`version: v2`。HTTPRoute 示例还需要两个同端口的版本 Service，selector 各加自己的 version。两套配置是独立对照。

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

## 从 route 条目读到权重区间

Istio 在 `applyHTTPRouteDestination` 中先分单目标和多目标。单目标直接写入 cluster，多目标依次翻译每个有效目的地，不把其中权重为 0 的目的地加入 weighted clusters。Envoy 收到后，`WeightedClusterSpecifierPlugin` 把路由动作里的权重映射成连续区间。它以选取值对总权重取模，再找落入的区间。权重 90/10 划出 `[0, 90)` 和 `[90, 100)`，v1、v2 在各自区间各占 90 格和 10 格。下面是对应的代理字段摘录，省略了 route match 和其他默认字段。

```json
{
  "weightedClusters": {
    "clusters": [
      {"name": "outbound|8080|v1|catalog.mesh-study.svc.cluster.local", "weight": 90},
      {"name": "outbound|8080|v2|catalog.mesh-study.svc.cluster.local", "weight": 10}
    ]
  }
}
```

从这里开始，v2 的 cluster 有自己的连接池、端点和负载均衡。即便它目前没有后端，上面的权重表仍然合法；出错发生在选择 cluster 之后。若同时配置超时或重试，客户端最终成功版本、路由最初选择的 cluster 和上游尝试次数也会出现不同口径。做单一变量验证时可以先禁用这条路由的重试，或者额外记录重试的 proxy 证据。 [Istio 路由动作转换](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/networking/core/route/route.go#L625-L750) · [Istio 1.32.0 所用 Envoy 的权重选取](https://github.com/envoyproxy/envoy/blob/56e0748e3deaeb8af7efcbf2f97329bb15539767/source/common/router/weighted_cluster_specifier.cc#L25-L75)

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

## 配置写在哪儿，会影响谁做选择

Sidecar 内部路由的 `VirtualService` 默认绑定保留的 `mesh` gateway，出站 Sidecar 用它来给网格内服务路由。如果只把对象绑到名为 `edge` 的 Istio Gateway，选择逻辑转移到实际使用这条规则的入口。两类流量都需要时，可以显式列出入口和 `mesh`，同时核对针对不同 Host 和 Gateway 的 match。DestinationRule 所有 subset 标签也必须选在该 host 真实发现的 endpoint 集合内。在两个命名空间有同名 Service 时，优先用 FQDN 消除短名按配置命名空间解析的歧义。

HTTPRoute 中 Service 作为 parent、Service 作为两个 backend 的范式，有一个单独的架构代价：三个 Service 的 selector、端口名和端点状态都要能独立解释。在 v1 和 v2 可能搭配不同协议、会话或端口配置时，单独 Service 又让所有者边界更清楚。本文两种实现都以版本权重选择 cluster；实际控制器与代理的位置依然需要按相应路径确定。 [VirtualService 主机及绑定](https://istio.io/latest/docs/reference/config/networking/virtual-service/) · [Gateway API 网格模型](https://gateway-api.sigs.k8s.io/docs/mesh/mesh-overview/)

## 10% 到底怎么验收

让后端在响应头标记自己的版本，在没有预览头的请求中记录每个版本的计数，并单独记录错误。假设没有重试或失败，1000 次独立抽样以 10% 概率去 v2，期望命中 100 次，标准差约 9.5 次。这是便于理解样本波动的模型。若客户端应该发 1000 次，却只统计 900 条 2xx，并且 v2 错误更多，“v2 成功响应占比”就会低于“被选中去 v2 的请求占比”。这时还要按路由和上游 cluster 看代理证据。HTTP/2 连接可承载许多独立请求，单个长流则需要按请求计数。

一次业务灰度也应分开比较错误率、延迟、取消率和 v2 的资源负载。对长流，额外记录连接持续时间以及新进入和已存续的流数。规则变更主要影响新建请求，已经开始的流仍按原来选定的上游完成或退出。 [Istio 权重切换任务](https://istio.io/latest/docs/tasks/traffic-management/traffic-shifting/)

## 用一个能标记版本的后端读结果

下面的 Python 模型只复核权重区间的数学含义。它遍历全部 100 个可能的抽样槽位，两个输入向量归一化后应得到相同份额。它可以在本机直接运行，实际 Envoy 的随机来源、端点状态与连接生命周期不在这个模型里。

```python
from collections import Counter

def choose_cluster(weights, draw):
    slot = draw % sum(weights.values())
    for cluster, weight in weights.items():
        if slot < weight:
            return cluster
        slot -= weight
    raise ValueError("a positive total weight is required")

for weights in ({"v1": 90, "v2": 10}, {"v1": 9, "v2": 1}):
    results = Counter(choose_cluster(weights, draw) for draw in range(100))
    assert results == {"v1": 90, "v2": 10}
    print(weights, dict(sorted(results.items())))
```

在 Sidecar 隔离集群做对照时，可让 v1、v2 对 `GET /items/42` 分别返回正文 `v1`、`v2`。用一个已注入、装了 curl 的 reader，从同一来源跑三轮：无预览头、带预览头、再去掉预览头。下面统计正常样本与 HTTP 错误。设有限速和固定尝试次数，避免把灰度观察变成负载压测。

```sh
kubectl -n mesh-study exec deploy/reader -c reader -- sh -c '
  url=http://catalog.mesh-study.svc.cluster.local:8080/items/42
  i=0
  while [ "$i" -lt 100 ]; do
    if version=$(curl -fsS --max-time 2 "$url"); then
      printf "%s\n" "$version"
    else
      printf "%s\n" error
    fi
    i=$((i + 1))
  done
' | sort | uniq -c
```

将 curl 加上 `-H "x-release-track: preview"` 可以对照匹配分支。样本中还应保留哪些请求来自显式预览，防止把这些必然落到 v2 的结果混入随机样本。如果 v2 除了命中率变动还涉及错误分布，可固定 v2 的负载输入并单独查看 v2 cluster 的请求与端点。排查常用的三个数是目标权重、上游选取数和客户端成功交付数，各自对应不同阶段。

---

收录于 [Istio 流量管理]({{< relref "istio-traffic-learning/index.md" >}})。
