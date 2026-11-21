---
title: "Istio mTLS 建立谁的身份？从握手到请求授权"
date: "2026-11-21"
description: "区分 PeerAuthentication、DestinationRule、请求认证与授权的执行范围。"
categories: ["Istio 流量管理"]
tags: ["Istio", "mTLS", "AuthorizationPolicy", "Ambient"]
---

客户端与服务端之间的 mTLS 能回答“和我建连的是谁”。一次 HTTP 请求该不该被允许，还要考虑工作负载身份、最终用户身份、操作和路径。Istio 用几类对象分别处理这些问题。它们执行在不同位置，所以同样的 YAML 放进 Sidecar 与 Ambient，实际要检查的执行点也不同。

## 入站要求和出站握手是两件事

| 问题 | 配置 | 谁执行 |
| --- | --- | --- |
| 入站能否接收明文或必须用 mTLS | PeerAuthentication | 目标 Sidecar 或 Ambient ztunnel |
| 出站是否对目标建立 TLS 或使用网格 mTLS | DestinationRule 中的 TLS 模式，或默认的 Auto mTLS | 能执行该配置的出站 Envoy |
| HTTP 请求的 JWT 是否有效 | RequestAuthentication | 可读取 HTTP 的服务端代理 |
| 此身份是否能调用该目标 | AuthorizationPolicy | 处理数据路径的目标代理 |

以 Sidecar 为例，目标用 `STRICT` 时，来自网格内的客户端通常由 Auto mTLS 根据目标状态决定使用网格证书；若给该出站显式设置不兼容的 TLS 类型，入站策略不能代替客户端完成握手。至于应用收到的连接，Sidecar 通常在本地代理终止网格 mTLS，再把请求送给应用。`portLevelMtls` 写的是工作负载端口，Service 将 `80` 映射到 `8080` 的时候，入站端口选择应理解为 `8080`。Ambient 的代理间 HBONE 自带 mTLS，`DISABLE` 不受支持，入站的 `STRICT` 仍能用于拒绝到目标的非网格流量。 [TLS 配置方式](https://istio.io/latest/docs/ops/configuration/traffic-management/tls-configuration/)，[PeerAuthentication API](https://istio.io/latest/docs/reference/config/security/peer_authentication/)

## 用工作负载身份做授权

假设安装采用默认的 `cluster.local` 信任域，客户端 Pod 使用 `mesh-study` 中的 `reader` ServiceAccount。下面在隔离的 Sidecar 练习环境要求入站 mTLS，并把 `catalog` 的 GET 读取限定给 reader。一个 ServiceAccount 可被多个 Pod 使用，匹配身份等于信任这些 Pod 共同持有的服务账号，不等于认出了某个 Pod 名或某位具体用户。

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: mesh-study
spec:
  mtls:
    mode: STRICT
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: catalog-reader
  namespace: mesh-study
spec:
  selector:
    matchLabels:
      app: catalog
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/mesh-study/sa/reader"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/items/*"]
```

目标收到请求后，认证连接给出 peer 的网格身份，授权才对这个身份和 HTTP 条件做决策。若目标存在适用的 ALLOW 策略，未命中任何 ALLOW 规则就拒绝；适用的 DENY 命中时，ALLOW 不会覆盖它。可用四种互相独立的请求查验：reader 读目录、另一个 ServiceAccount 读目录、reader 发不同方法、非网格明文客户端连接。比较握手失败与 HTTP 授权拒绝两类结果，才能定位是哪层决策。如果需要按最终用户授权，用 RequestAuthentication 校验 JWT 的来源、签名和受众，再用 AuthorizationPolicy 要求有效的 `requestPrincipals` 或特定 claims。RequestAuthentication 默认按 `jwtRules[].presence: OPTIONAL` 处理，拒绝无效令牌；缺失令牌的请求以无请求身份继续，由授权策略决定能否访问。Istio 1.32 新增的 `presence: REQUIRED` 可在认证层要求该条规则的令牌，缺失时返回 401；多条规则都设为 REQUIRED 时须全部满足。 [1.32 JWT 变更](https://istio.io/latest/news/releases/1.32.x/announcing-1.32/change-notes/) [AuthorizationPolicy API](https://istio.io/latest/docs/reference/config/security/authorization-policy/)，[RequestAuthentication API](https://istio.io/latest/docs/reference/config/security/request_authentication/)

## 区分同一用户和同一服务

假设两位不同用户都通过前端 `reader` 调用 catalog。catalog 的 mTLS 会把这两个调用都识别为 reader。如果两人的数据访问权限不同，就需要独立携带、校验并授权最终用户凭证，或由可信应用层做相应决策。反过来，收到合法 JWT 也只说明请求凭证通过了校验，仍应检查发起调用的工作负载是不是预期来源。把用户身份和工作负载身份同时放进模型，支持“只有前端服务能调用，而且 JWT 用户只能读自己的目录”这样的要求。Istio 的策略能做来源、声明及路径匹配；资源归属、数据库记录和业务动作的一致性由应用判断。 [Istio 安全架构与身份](https://istio.io/latest/docs/concepts/security/)

## Ambient 要把策略交给读得懂的代理

ztunnel 在四层可检查工作负载身份与端口。HTTP method、path、JWT 等条件属于七层，由 waypoint 或实际观察 HTTP 的网关代理做判断。同一个对象从 selector 换到 waypoint 时，需要按支持范围使用 `targetRefs`，例如关联 waypoint 所服务的同命名空间 Service，并确认这条流量确实经过该 waypoint。只迁移策略绑定不够，所有要求经过 waypoint 的入口或直连路径都需要相应网络和授权约束。旧版本与多控制平面 revision 环境中，还要留意旧控制平面对 `targetRefs` 的解释。以 1.32 为例，授权代码区分工作负载策略与 HTTPRoute 附着的策略索引，1.32 的 AuthorizationPolicy 附着 HTTPRoute 为实验特性，默认关闭，适用范围是 ingress gateway。对 waypoint 使用其明确支持的 Service 或 Gateway 等资源进行绑定。 [Ambient L7 能力](https://istio.io/latest/docs/ambient/usage/l7-features/)，[Istio 1.32 变更说明](https://istio.io/latest/news/releases/1.32.x/announcing-1.32/change-notes/)，[1.32.0 授权索引源码](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/model/authorization.go)

[Istio 流量管理专题]({{< relref "istio-traffic-learning/index.md" >}})

前置阅读 · [Istio 怎样认识集群外的服务？ServiceEntry 与出口网关]({{< relref "posts/istio-serviceentry-egress/index.md" >}})

继续阅读 · [Istio 怎样看见一次请求？指标、日志和 Trace 的边界]({{< relref "posts/istio-telemetry-trace-context/index.md" >}})
