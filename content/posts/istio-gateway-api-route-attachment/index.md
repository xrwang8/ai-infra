---
title: "HTTPRoute 挂到哪一层？Istio 入口、网格内路由与 Gateway API"
date: "2026-11-21"
lastmod: "2026-11-22"
description: "区分传统 Gateway、Gateway API Listener、GAMMA Service 绑定和跨命名空间引用。"
categories: ["Istio 流量管理"]
tags: ["Istio", "Gateway API", "HTTPRoute", "GAMMA"]
---

Istio 同时支持两种常被都叫作 Gateway 的资源。一种在 `networking.istio.io` 下，常与 VirtualService 配合。另一种属于 Kubernetes Gateway API，与 HTTPRoute 等路由类型合作。同一个 HTTPRoute，也可以在网格内直接挂到 Service 上。理解这几种用法，要分清负责入口的资源、规则绑定的对象和后端引用的权限。

本文的实现基线是 Istio 1.32.0 和 Gateway API v1.6.1。`catalog` 代表一个暴露 HTTP 8080 的 Kubernetes Service。入口与网格内示例分别说明两种 attach 关系，读者无需先部署其他文章中的配置。

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

## 把入口、规则和后端分别交给三方

假设平台团队在 `edge` 提供入口，应用团队在 `apps` 管路由，后端团队在 `data` 维护 `catalog` Service。Route 的 parent 指向 edge，backend 指向 data。应用团队有权写自己的 Route，但两个跨命名空间关系还分别需要平台和后端两侧声明。下列对象假设三个 Namespace 已创建，`data` 内已有名为 catalog 的 HTTP Service，Service port 为 `8080`。ReferenceGrant 使用 Gateway API v1.6.1 已提供的 `v1`，部署到其他版本时查实际提供的 CRD 版本。

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-web
  namespace: edge
spec:
  gatewayClassName: istio
  listeners:
    - name: web
      hostname: catalog.example.com
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              catalog-edge-access: "true"
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: catalog
  namespace: apps
spec:
  parentRefs:
    - name: shared-web
      namespace: edge
      sectionName: web
  hostnames: [catalog.example.com]
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /items
      backendRefs:
        - name: catalog
          namespace: data
          port: 8080
---
apiVersion: gateway.networking.k8s.io/v1
kind: ReferenceGrant
metadata:
  name: catalog-from-apps
  namespace: data
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: apps
  to:
    - group: ""
      kind: Service
      name: catalog
```

只有经过平台认可的 Namespace 加上 `catalog-edge-access=true`，才通过这个 Listener 的命名空间选择。ReferenceGrant 授权的是 apps 中的 HTTPRoute 对特定后端的引用；它不能按来源 Route 名细分，因为 `from` 没有该字段。反过来，授予后端引用权限不会让路由自动挂上某个 Listener。权限生效之后，仍应分别检查 Gateway 对应 listener 与 Route 上这条 parent 的反馈。

```sh
kubectl -n edge get gateway shared-web -o json |
  jq '.status.listeners[] | select(.name == "web") | {attachedRoutes, conditions}'
kubectl -n apps get httproute catalog -o json |
  jq '.metadata.generation as $generation |
      .status.parents[] |
      select(.parentRef.name == "shared-web" and .parentRef.sectionName == "web") |
      {controllerName, generation: $generation, conditions}'
```

| 变化 | 预期哪个层的反馈先变化 |
| --- | --- |
| apps 尚未获得 Listener 的 Namespace 标签 | 接纳条件；Route 不能在此 Listener 生效 |
| apps 已被接纳，data 尚未给 ReferenceGrant | 引用条件；跨命名空间后端未被允许 |
| 两侧都授权，但 port 填的是 Pod 的 8081，而 Service port 是 8080 | 后端解析；控制器找不到所声明的 Service port |
| 对象与状态都就绪，客户端发错了 Host | 实际请求匹配；Gateway 状态不会因此变成无效 |

GAMMA 采用 Service parent 时，`sectionName` 可用于选择服务命名端口，`port` 则用端口数字来约束 parent。Route 和 Service 在同一命名空间时是 producer 从服务所有方的视角定义行为；跨命名空间的 consumer route 只涉及相应的消费者作用域和当前实现支持。把入口配置等价迁到 Service parent 之前，要先确认消费者命名空间、真实访问地址和执行代理。 [Gateway API 网格路由](https://gateway-api.sigs.k8s.io/docs/mesh/mesh-overview/) · [跨命名空间引用](https://gateway-api.sigs.k8s.io/reference/api-types/referencegrant/)

## 两个重叠的匹配由谁赢

假设一条规则匹配 PathPrefix `/items`，还要求指定 Header；另一条只匹配更长的 PathPrefix `/items/archive`。请求地址为 `/items/archive/42` 且携带该 Header 时，较长前缀优先。Header 匹配数用于后续比较；它不会超过更高优先级的路径规则。若类型、长度、方法、Header 和 Query 的匹配优先级均相同，规范还定义了跨 Route 的顺序，优先较早创建的对象，再按命名空间与名称比较。规则写入顺序只在规范所规定的同类平手情形下决定结果。

Istio 1.32.0 的 `httpRouteLess` 按 URI 类型和长度、方法是否存在、Header 个数、Query 个数比较。不同 Route 合并时，代码还保留 origin 信息参与排序与策略归属。对复杂配置，可以先按规范手推一个请求，再在 Envoy 的 virtual host 中定位两条 route，查看顺序是否同推导一致。 [HTTPRoute 优先级源码](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/config/kube/gateway/conversion.go#L462-L494) · [跨对象合并源码](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/config/kube/gateway/route_collections.go#L815-L978)

一个容易漏掉的生命周期是删除 ReferenceGrant 或修改目标 Service port。Route 自己未被编辑，控制器也需要重新计算后端引用；从 API 上看到 Route 的 generation 没动，不能推导其前向依赖没有变化。Istio 的 `buildDestination` 先把 `group/kind` 正规化为资源类型，再对跨 Namespace 后端检查授权，最后构造带集群域的 Service 主机名。于是 `backendRefs.name` 应写 Kubernetes Service 对象名 `catalog`，而非用来发请求的 FQDN。引用写错时，应先看 `ResolvedRefs` 的 reason，再检查目标方的实际对象。 [Istio 1.32.0 后端引用转换](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/config/kube/gateway/conversion.go#L1153-L1225)

## 源码转换与版本边界

Istio 1.32.0 的 `convertHTTPRoute` 读取 Gateway API 的规则，再转成可用于生成路由的内部表示。出现同样的 Envoy 目的集群，不等于两套 API 的接纳范围和字段支持相同。对某个字段，先核对 CRD 所在版本，再核对具体控制器、profile 和当前代理配置。 [Istio HTTPRoute 转换](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/config/kube/gateway/conversion.go#L102)

Istio 1.32.0 支持矩阵列出的 Gateway API 构建、测试版本为 v1.6.1。较新的 CRD 发布有助于跟踪机制，但不能仅从字段存在推导这里的实现支持。 [Istio 支持版本](https://istio.io/latest/docs/releases/supported-releases/)

---

收录于 [Istio 流量管理]({{< relref "istio-traffic-learning/index.md" >}})。
