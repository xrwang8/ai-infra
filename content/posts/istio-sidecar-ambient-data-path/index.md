---
title: "Istio 请求经过谁？从 Sidecar 到 Ambient 数据平面"
date: "2026-11-21"
lastmod: "2026-11-22"
description: "从同一条服务调用理解 Istiod、Envoy、ztunnel、HBONE 和 waypoint 的职责。"
categories: ["Istio 流量管理"]
tags: ["Istio", "Sidecar", "Ambient", "服务网格"]
---

在 Kubernetes 里，`reader` 访问 `catalog`，首先涉及 DNS、Service 和接收连接的 Pod。接入 Istio 后，应用还发同一条请求，流量却可能经过完全不同的代理。理解这件事可以先问三个问题：**谁截获连接，谁能看到足够的信息来执行规则，谁把规则和端点状态送给代理。**

本文使用 Istio 1.32.0，围绕一个监听 HTTP 8080 的 Kubernetes Service `catalog` 展开。阅读只需知道 Service 会把请求交给 Pod；Sidecar、ztunnel、HBONE 和 waypoint 的职责在本文内定义。

## 控制面维护网格状态，数据面处理业务请求

Istiod 从 Kubernetes 和 Istio 获取工作负载、服务、端点、路由和策略等信息。它计算代理需要的配置，并参与建立工作负载身份的相关流程。数据面负责业务流量的转发。应用流量无需逐条经过 Istiod。Sidecar 模式通常让每个加入网格的 Pod 带一个 Envoy 代理。Ambient 把四层安全覆盖网络移到每节点的 ztunnel，按需再部署 Envoy waypoint 处理七层。 [Istio 架构](https://istio.io/latest/docs/ops/deployment/architecture/) · [Sidecar 与 Ambient](https://istio.io/latest/docs/overview/dataplane-modes/)

学习时把三个状态分开。Pod 状态描述应用进程是否起来，服务发现描述可用的后端，代理状态说明这条连接怎样处理。一个应用已经 Ready，并不证明它已进入期望的流量路径。API 能保存对象，也不能证明请求经过了使用这个对象的代理。

## Sidecar 的代理在 Pod 内

在常见 Sidecar 路径上，应用访问 Service，符合条件的出站连接会被重定向到同 Pod 的 Envoy。它按协议、目标和匹配规则选择上游集群，然后选端点。目标 Pod 也加入 Sidecar 网格时，其入站代理可先处理连接，再交给应用。应用仍用已有的服务地址请求。哪些连接被捕获、哪些端口被排除，由网格部署和工作负载配置共同决定。 [Istio CNI](https://istio.io/latest/docs/setup/additional-setup/cni/)

注入把代理及运行配置放进 Pod。捕获把符合条件的连接送到代理。两者都需要成立。旧版本的 iptables 链和端口列表只能说明那个版本的部署，不能拿作所有环境的固定拓扑。排查时先找实际处理该方向的代理。做源端灰度时，阅读目标端的入站规则不能代替核对源端的路由。

## Ambient 怎样把四层和七层拆开

ztunnel 为加入 Ambient 的工作负载提供安全覆盖网络、工作负载身份和连接转发。HBONE 使用 HTTP CONNECT 建立隧道，但隧道内装的是应用流量。外层使用 HTTP，并不意味着 ztunnel 会解析内层 HTTP 路径。要按 Header、URL 或其他七层属性做控制，需要请求经过 waypoint。 [HBONE](https://istio.io/latest/docs/ambient/architecture/hbone/) · [Ambient L7](https://istio.io/latest/docs/ambient/usage/l7-features/)

waypoint 运行自己的 Envoy。部署它、把目标 Service 绑定给它、让某条来源的请求实际经过它，分别是不同的操作和状态。以 `catalog` 为例，访问它的 Service 和直接访问后端 Pod IP，原始目标类型不同。默认用于服务流量的 waypoint 不能由此自动覆盖访问 Pod IP 的路径。入口网关发起的连接与网格内连接也要分开核对。 [配置 waypoint](https://istio.io/latest/docs/ambient/usage/waypoint/)

| 路径 | 处理连接的组件 | 能执行 HTTP 规则的位置 |
|---|---|---|
| 典型双边 Sidecar | 应用 Pod 内的代理 | 参与该请求的 Envoy |
| Ambient 基础路径 | 节点 ztunnel | 此路径只提供四层网格处理 |
| Ambient 绑定 waypoint 的服务路径 | ztunnel 与 waypoint | 实际经过的 waypoint |
| 通过入口网关 | 入口 Envoy 和后续代理 | 入口规则在网关处理，后续路径另外确认 |

## 同一条请求有哪些地址

应用看到的目的地址通常来自 DNS，在这个例子里是 catalog 的 Service 地址和 8080 端口。代理作负载均衡后才选定 Pod 地址。HTTP 层还能有一个独立标识，即 Host 或 HTTP/2 的 `:authority`。它可能由应用决定，也可能在网关上被改写。用 Service IP、Pod IP 和修改 Host 三种方式发请求，改变的是不同层次的信息，所以不能把其中一个成功当作另外两个也会成功的证据。

若 HTTP 被 TLS 包住，能看到 HTTP 的节点取决于 TLS 在哪里终止。例如，网关做普通 HTTPS 终止，便能基于解密后的路径路由；网关透传 TLS，只能依据能看到的连接信息和 TLS ClientHello 中的 SNI 等条件。网格代理间的 mTLS 与应用自己的 HTTPS 可以同时存在，分别有各自的终止点。 [Istio TLS 连接与终止](https://istio.io/latest/docs/ops/configuration/traffic-management/tls-configuration/)

## 从 Pod 网络命名空间跟到下一跳

Sidecar 模式先要有注入。注入 webhook 改的是 Pod 创建请求，已经存在的 Pod 不会因为 Namespace 多了一个标签就长出 Envoy。Pod 内没有启用 Istio CNI 时，常见安装用 `istio-init` 配置重定向；启用后则由 CNI 接管这部分网络初始化。iptables 的实现细节和端口可以随安装变化，但因果顺序很稳定：业务先向原地址发起连接，重定向让 Envoy 接受连接，Envoy 根据被识别的地址和协议建上游连接。

Ambient 的 CNI 也在工作负载的网络命名空间配置捕获。它另外把 Pod 的网络命名空间信息交给节点上的 ztunnel，使 ztunnel 能在相应命名空间建立代理监听，而不向 Pod 注入代理容器。源码上，Istio 1.32.0 的 `NetServer.reconcileExistingPod` 先取出 Pod netns，再在该 netns 运行 `CreateInpodRules`；这条函数处理已有 Pod 的规则协调，可见“节点上有一个 ztunnel Pod”和“应用所在 netns 已完成捕获”是不同状态。 [节点代理规则协调](https://github.com/istio/istio/blob/1.32.0/cni/pkg/nodeagent/net_linux.go#L207-L223) · [Ambient 流量重定向](https://istio.io/latest/docs/ambient/architecture/traffic-redirection/)

以请求 `http://catalog.mesh-study.svc.cluster.local:8080/items/42` 为例，可把下一跳决定逐层拆开。

```text
Sidecar
reader → reader Envoy → 目标 Pod 的 Envoy → catalog 进程
         HTTP 匹配、选集群和端点             入站请求处理

Ambient + Service waypoint
reader → 源 ztunnel → catalog waypoint → 目标 ztunnel → catalog 进程
         查服务和下一跳  读取 HTTP，随后选后端   目标侧四层约束和交付
```

图中的箭头表示逻辑路径，组件可能同节点。对 HBONE，ztunnel 源码的 `create_hbone_request` 将目标 URI 填进外层 CONNECT；隧道建立后转发的是连接内的字节流。外层对端、CONNECT 要承载的目的地址、内层 HTTP 的 `:authority` 是三件事。waypoint 被选中时，外层 TLS 会连接 waypoint，waypoint 仍可用内层 HTTP Host 做应用层匹配；未走 waypoint 的基础路径，则没有对应的七层路由点。 [Istio 1.32.0 对应的 ztunnel 出站源码](https://github.com/istio/ztunnel/blob/82d679bb93131143902a9e7c2650821edc634e14/src/proxy/outbound.rs#L535-L585)

## Waypoint Ready 为什么还没覆盖这条请求

已有 Ambient 安装时，可以给 `mesh-study` 中的 Kubernetes Service `catalog` 创建只处理服务流量的 waypoint。工作负载在该命名空间加入 Ambient；下面的标签与 Gateway 是两个独立对象。已有 Service 应保留自己的 port、selector 和其他字段。

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: catalog-waypoint
  namespace: mesh-study
  labels:
    istio.io/waypoint-for: service
spec:
  gatewayClassName: istio-waypoint
  listeners:
    - name: mesh
      port: 15008
      protocol: HBONE
```

```sh
kubectl -n mesh-study label service catalog istio.io/use-waypoint=catalog-waypoint
kubectl -n mesh-study get gateway catalog-waypoint -o yaml
istioctl ztunnel-config services -n istio-system
```

第一条命令只在为练习准备的 Service 上使用。后两条分别看 Gateway 状态和 ztunnel 的 Service 视角。一个已经 Programmed 的 waypoint 可以同时有若干绑定方，也可以暂时没有目标流量。读 `fetchWaypointForTarget` 会看到，资源显式指定的 waypoint 先于 Namespace 继承。如果资源明确点名的 Gateway 尚未进入可用集合，该分支报告未就绪，也不会去找一个现成的 Namespace waypoint 代替。`fetchWaypointForService` 接着检查这个 waypoint 是否支持 `service` 或 `all`。由此能推导出四种不同配置状态。

| 配置 | 应从哪个路径判断 |
| --- | --- |
| Service 未显式绑定，Namespace 有可用 Service waypoint | 网格客户端访问 Service，检查 Namespace 继承 |
| Service 显式指向另一个已就绪的 waypoint | 网格客户端访问该 Service，检查资源级优先 |
| Service 显式绑定的 waypoint 尚未就绪 | 查绑定状态及这类路径实际如何处理，不能假定继承还在 |
| waypoint 只服务 `service`，请求目标为 Pod IP | 从 workload 原始目标走，单独设计 waypoint 或四层约束 |

Waypoint 负责选路或执行授权时，必须保证相应入口确实走到它。流量绑定描述要走的路径，目标侧四层授权可以约束允许到达目标的身份。Ingress、带 Sidecar 的客户端和直接到 Pod 的请求都值得单独建模。 [Waypoint 绑定代码](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/serviceregistry/ambient/waypoints.go#L99-L179) · [Waypoint 流量类型与授权](https://istio.io/latest/docs/ambient/usage/waypoint/)

## 换了代理部署单位，资源与故障边界也变了

Sidecar 以 Pod 为部署单位。每个应用 Pod 的入站和出站连接、路由配置与证书都落在相邻代理，资源可以按应用副本预算。但更新代理镜像或某些注入配置需要重建 Pod，同一节点上的大量副本也会相应增加代理数量。Ambient 的 ztunnel 以节点为单位处理基础四层能力，同一个进程承载多个工作负载，但 mTLS 身份仍按工作负载使用，不会统一变成节点身份。共享 waypoint 负责附着到它的多项服务时，连接池、CPU 使用和故障范围集中到了 waypoint 副本。估算开销时要同时看 Pod 数、节点数、需要七层的流量比例与共享代理的容量。

这也改变了升级验收。Sidecar 升级先查看业务 Pod 模板注入了哪个 revision，再看重建后的代理版本。Ambient 升级要检查节点上 ztunnel 与 CNI 纳管、以及 waypoint Deployment 的副本状态。流量仍能到达只说明基础转发可用；如果 HTTP 策略要求所有受控入口都读取路径，还要从 Service、入口和独立运维路径验证。 [数据面模式与操作边界](https://istio.io/latest/docs/overview/dataplane-modes/)

## 用能力和边界决定选型

只需要网格身份、传输加密和四层授权的服务，可以先理解 Ambient 的基础路径。依赖已有 Sidecar 扩展、逐 Pod 代理定制或特殊协议处理的服务，要查对应功能状态。Istio 1.32 把 waypoint 上的 Stable Channel HTTPRoute、GRPCRoute 列为 Stable，VirtualService 在 waypoint 上仍列为 Alpha。 [Istio 功能阶段](https://istio.io/latest/docs/releases/feature-stages/)

真正切换模式前，可以给同一请求列三张路径卡。分别写网格内 Service 请求、直接 Pod 地址、从入口进入的请求，然后标出哪个组件做路由、哪个做身份验证、哪里采日志。共享代理的资源与故障影响范围也要结合部署理解。这样后面讨论路由和授权时，读者能把配置放回复用的请求路径，而不是只记住标签。

---

收录于 [Istio 流量管理]({{< relref "istio-traffic-learning/index.md" >}})。
