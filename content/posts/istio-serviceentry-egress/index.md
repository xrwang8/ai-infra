---
title: "Istio 怎样认识集群外的服务？ServiceEntry 与出口网关"
date: "2026-11-21"
description: "拆解外部服务注册、域名解析、WorkloadEntry、出口网关与网络约束。"
categories: ["Istio 流量管理"]
tags: ["Istio", "ServiceEntry", "Egress Gateway"]
---

向外部域名发起 HTTPS 时，有三个相互独立的问题。应用能否把名字解析成地址，代理能否把这条连接识别为某个已知服务，网络是否强制它经过某条受控路径。ServiceEntry 主要处理第二个问题，部分场景也参与代理侧的解析。要理解出口网关，先把这些问题分开。

## 服务注册与 DNS 在哪里接上

假设网格中的应用要访问一项外部 HTTPS API。下面把一个示意域名注册成可被 Sidecar 识别的服务。域名只用于说明配置，不要求它实际存在。`hosts` 给出服务名，`location: MESH_EXTERNAL` 表示网格外目的地，`resolution: DNS` 指示代理用 DNS 解析后端。省略显式 endpoints 时，此例的域名同时充当解析目标。HTTP 加密发生在应用到外部服务之间时，代理正向转发的是 TLS，会根据配置匹配连接和可见的 SNI，不能在未终止 TLS 的情况下依据通用 HTTP 路径分流。 [ServiceEntry API](https://istio.io/latest/docs/reference/config/networking/service-entry/)

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: supplier
  namespace: mesh-study
spec:
  hosts: [supplier.example.com]
  location: MESH_EXTERNAL
  resolution: DNS
  ports:
    - number: 443
      name: https
      protocol: HTTPS
```

应用的 resolver 与 Envoy 的后端解析是两个视角。应用先根据运行环境的 DNS 机制发现名字并试图连接。启用 DNS capture 时，Istio 可以参与应答，某些注册服务还能获得合成的地址来区分服务。Envoy 再按自己的发现和 DNS 策略选真实上游地址。因此，看到应用解析到某地址，只能证明应用得到了地址。核对代理应该看它认出的 cluster 和 endpoints；当应用使用以 IP 为目的地址的连接时，还要核对服务匹配取决于什么。把 `resolution: NONE` 改成 `DNS` 也改变了代理使用原目标地址或按服务解析的行为。 [Istio DNS proxying](https://istio.io/latest/docs/ops/configuration/traffic-management/dns-proxy/)

## WorkloadEntry 管的是服务里面的实例

当团队把部分进程从传统 VM 搬到 Kubernetes，逻辑服务仍叫 `catalog`，服务内的可选端点可能分布于两边。`WorkloadEntry` 描述 VM 等工作负载实例的地址、标签、账户或其他属性；带 `workloadSelector` 的 `ServiceEntry` 可按标签选择同一命名空间的已注册实例和 Pod。真实的服务迁移仍需校验地址可达、端口映射、VM bootstrap、服务账号凭证和网络边界。写了 WorkloadEntry 并不会替 VM 安装代理或签发身份证书。固定代码中 `convertWorkloadInstanceToInstances` 为具体工作负载与模型 Service 组合生成实例，正好能看到服务与工作负载这两个层次怎样汇合。 [WorkloadEntry API](https://istio.io/latest/docs/reference/config/networking/workload-entry/)，[Istio 1.32.0 注册表转换](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/serviceregistry/serviceentry/conversion.go)

## 外部域名发现后怎样更新端点

把地址写进 `STATIC` endpoints 的服务，依赖配置或工作负载注册更新地址。DNS 型服务依靠代理后续解析跟踪地址变化，应用端缓存、代理端 DNS 缓存和已有连接的寿命都可能不同。因此，外部系统切换地址时，一个新建请求与一条仍然存活的旧长连接可能指向不同的地址。核对时把 DNS 应答时间、代理实际选用的上游 IP 和连接建立时间放到一起。对需要周期变化的域名，先确认自己使用的 ServiceEntry 解析模式和协议支持。 [ServiceEntry 解析模式](https://istio.io/latest/docs/reference/config/networking/service-entry/)

## egress gateway 补上受控转发点

出口网关常用于集中连接外部系统，便于固定出口地址、统一访问记录或集中做 TLS origination。一个显式路径有两段，应用侧代理把请求转给网关，网关再匹配外部目的地。每段都有自己的路由与 TLS 设置。如果由应用建立 HTTPS 并一路透传，网关看到的是 TLS 连接。如果选择由网关对上游发起 TLS，必须确定上游地址、SNI、证书校验和下游到网关的安全设置，避免把自己发起的 HTTPS 再错误包一层 TLS。 [出口网关](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/)，[Istio TLS 链路说明](https://istio.io/latest/docs/ops/configuration/traffic-management/tls-configuration/)

`REGISTRY_ONLY` 让代理拒绝未注册服务，但 Istio 说明它不能当作出口安全策略。需要强制业务只从网关出网时，最终边界在 CNI 支持的网络策略、网络 ACL 或其他独立出口控制，并要切实校验网关还能分辨和限制允许的目的地。验证思路是分别确认应用 DNS 与已注册请求、网关的真实上游日志，以及租户在网络控制下有无直接外连。仅看到一个网关日志条目，可以证明那条请求走过网关，却无法证明整个数据路径都受约束。 [Istio 出口流量控制](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-control/)

[Istio 流量管理专题]({{< relref "istio-traffic-learning/index.md" >}})

前置阅读 · [Istio 连接池、熔断和异常实例摘除分别做什么？]({{< relref "posts/istio-connection-pool-outlier-detection/index.md" >}})

继续阅读 · [Istio mTLS 建立谁的身份？从握手到请求授权]({{< relref "posts/istio-mtls-authorization/index.md" >}})
