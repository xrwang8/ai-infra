---
title: "HTTPRoute 挂到哪一层？Istio 入口、网格内路由与 Gateway API"
date: "2026-11-21"
description: "区分传统 Gateway、Gateway API Listener、GAMMA Service 绑定和跨命名空间引用。"
categories: ["Istio 流量管理"]
tags: ["Istio", "Gateway API", "HTTPRoute", "GAMMA"]
---

Istio 同时支持两种常被都叫作 Gateway 的资源。一种在 `networking.istio.io` 下，常与 VirtualService 配合。另一种属于 Kubernetes Gateway API，与 HTTPRoute 等路由类型合作。同一个 HTTPRoute，也可以在网格内直接挂到 Service 上。理解这几种用法，要分清负责入口的资源、规则绑定的对象和后端引用的权限。

## 两套 API 划分角色的方式

| 职责 | Istio 原生 API | Kubernetes Gateway API |
|---|---|---|
| 选择实现 | Istio Gateway 选择相应的数据平面工作负载 | Gateway 通过 GatewayClass 选择控制器实现 |
| 声明入口 | Gateway server 定义端口、主机和 TLS | Gateway Listener 描述协议、端口、主机及 route 接纳 |
| 声明转发 | VirtualService 关联入口并定义匹配与目的地 | HTTPRoute 用 parentRefs 绑定，匹配后转发给 backendRefs |
| 表达后端 | Service 与可选的 DestinationRule subset | backendRefs 通常是 Service，其他类型应核对控制器 |

两套 API 最终都要生成代理配置。规则表达并非逐项对应。Istio 原生灰度常在一个 Service 内划分 subset。Gateway API 原生 Service 引用可以让两个版本分别有一个 Service。学习时先确定要使用哪种模型，再判断匹配范围和对象所有权。 [Istio 流量管理概念](https://istio.io/latest/docs/concepts/traffic-management/) · [Istio Gateway API](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/)

## Listener 接受 Route 与后端授权

假设已有名为 `catalog-edge` 的 Gateway，listener 名 `web`。以下 Route、Gateway 和后端 Service 都在 `mesh-study`，Service port 为 8080。

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: catalog-public
  namespace: mesh-study
spec:
  parentRefs:
  - name: catalog-edge
    sectionName: web
  hostnames:
  - catalog.example.com
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /items
    backendRefs:
    - name: catalog
      port: 8080
```

`parentRefs` 说明规则想挂在哪。Listener 的协议、主机匹配和 allowedRoutes 一起影响接纳结果。`backendRefs` 说明匹配后往哪里转发，这里的 port 是 Service port。跨命名空间引用后端通常需要被引用命名空间的 ReferenceGrant。它解决的是后端所有方是否允许被引用。Listener 是否接受某命名空间的 Route，是另一处权限。 [HTTPRoute](https://gateway-api.sigs.k8s.io/reference/api-types/httproute/) · [ReferenceGrant](https://gateway-api.sigs.k8s.io/reference/api-types/referencegrant/)

在 `status.parents` 中找对应的 parent 和 controller，核对 Accepted、ResolvedRefs 以及相关原因。API Server 保存 YAML 后，控制器还需要处理本次对象的 generation。看到旧状态时，应和 observedGeneration 对齐。最终要检查实际入口请求，而不仅是控制面的状态。

## Service 作为网格内 parent

GAMMA 的变化是可以把 route 关联到服务本身。对网格内访问 `catalog` 的请求，HTTPRoute 可以把同一命名空间的 Service 作为 parent。这样应用仍按该服务地址访问，路由规则的绑定范围随实际网格路径决定。对 Ambient 的七层行为，还需要让流量经过实际执行规则的 waypoint。HTTPRoute 在入口使用 Gateway 作为 parent，在网格使用 Service 作为 parent，不能假定两个配置覆盖的是同一条请求。 [Gateway API Mesh](https://gateway-api.sigs.k8s.io/docs/mesh/mesh-overview/) · [Ambient L7](https://istio.io/latest/docs/ambient/usage/l7-features/)

跨命名空间进入网格语义时，要分别看 producer route 和 consumer route。这也改变规则应该由服务方还是调用方配置，以及应以哪一侧流量做验收。不要把 ingress Listener 的接纳规则直接套到所有 mesh parentRefs。

## 看对象接纳，也要看每一个后端

如果路由 90% 指向有效的 catalog-v1，10% 指向了没有被授权的跨命名空间 Service，Gateway API 的 HTTPRoute 规范要求原本要送到无效后端的份额返回 HTTP 500。`ResolvedRefs=False` 描述的是引用问题，`Accepted=True` 仍可能表示路由已被所在 Listener 接纳。这样做保留了声明中的流量份额，用户能够观察到配置错误。

另一个例子是路径匹配。上面使用 `PathPrefix: /items`，Gateway API 按由斜杠分隔的路径元素匹配，它包含 `/items/42`，却不包含 `/items-backup`。迁移一条原来按字符串前缀写的 VirtualService 时，应该建立一组边界样本，而非只拿 `/items/42` 比一次。 [Gateway API v1.6.1 HTTPRoute 类型和规则](https://github.com/kubernetes-sigs/gateway-api/blob/v1.6.1/apis/v1/httproute_types.go)

## 源码转换与版本边界

Istio 1.32.0 的 `convertHTTPRoute` 读取 Gateway API 的规则，再转成可用于生成路由的内部表示。出现同样的 Envoy 目的集群，不等于两套 API 的接纳范围和字段支持相同。对某个字段，先核对 CRD 所在版本，再核对具体控制器、profile 和当前代理配置。 [Istio HTTPRoute 转换](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/config/kube/gateway/conversion.go#L102)

本系列以 Istio 1.32.0 为基线，其支持矩阵列出的 Gateway API 构建、测试版本为 v1.6.1。较新的 CRD 发布有助于跟踪机制，但不能仅从字段存在推导这里的实现支持。 [Istio 支持版本](https://istio.io/latest/docs/releases/supported-releases/)

[Istio 流量管理专题]({{< relref "istio-traffic-learning/index.md" >}})

前置阅读 · [从 Istio YAML 到 Envoy 转发：走一遍 xDS 配置链]({{< relref "posts/istio-envoy-xds-routing/index.md" >}})

继续阅读 · [Istio 灰度为什么先选版本，再选实例？]({{< relref "posts/istio-weighted-canary-routing/index.md" >}})
