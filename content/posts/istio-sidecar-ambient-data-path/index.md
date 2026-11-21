---
title: "Istio 请求经过谁？从 Sidecar 到 Ambient 数据平面"
date: "2026-11-21"
description: "从同一条服务调用理解 Istiod、Envoy、ztunnel、HBONE 和 waypoint 的职责。"
categories: ["Istio 流量管理"]
tags: ["Istio", "Sidecar", "Ambient", "服务网格"]
---

在 Kubernetes 里，`reader` 访问 `catalog`，首先涉及 DNS、Service 和接收连接的 Pod。接入 Istio 后，应用还发同一条请求，流量却可能经过完全不同的代理。理解这件事可以先问三个问题：**谁截获连接，谁能看到足够的信息来执行规则，谁把规则和端点状态送给代理。**

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

## 用能力和边界决定选型

只需要网格身份、传输加密和四层授权的服务，可以先理解 Ambient 的基础路径。依赖已有 Sidecar 扩展、逐 Pod 代理定制或特殊协议处理的服务，要查对应功能状态。Istio 1.32 把 waypoint 上的 Stable Channel HTTPRoute、GRPCRoute 列为 Stable，VirtualService 在 waypoint 上仍列为 Alpha。 [Istio 功能阶段](https://istio.io/latest/docs/releases/feature-stages/)

真正切换模式前，可以给同一请求列三张路径卡。分别写网格内 Service 请求、直接 Pod 地址、从入口进入的请求，然后标出哪个组件做路由、哪个做身份验证、哪里采日志。共享代理的资源与故障影响范围也要结合部署理解。这样后面讨论路由和授权时，读者能把配置放回复用的请求路径，而不是只记住标签。

[Istio 流量管理专题]({{< relref "istio-traffic-learning/index.md" >}})

继续阅读 · [从 Istio YAML 到 Envoy 转发：走一遍 xDS 配置链]({{< relref "posts/istio-envoy-xds-routing/index.md" >}})
