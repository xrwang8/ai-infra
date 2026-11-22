---
title: "Istio 怎样认识集群外的服务？ServiceEntry 与出口网关"
date: "2026-11-21"
lastmod: "2026-11-22"
description: "拆解外部服务注册、域名解析、WorkloadEntry、出口网关与网络约束。"
categories: ["Istio 流量管理"]
tags: ["Istio", "ServiceEntry", "Egress Gateway"]
---

向外部域名发起 HTTPS 时，有三个相互独立的问题。应用能否把名字解析成地址，代理能否把这条连接识别为某个已知服务，网络是否强制它经过某条受控路径。ServiceEntry 主要处理第二个问题，部分场景也参与代理侧的解析。要理解出口网关，先把这些问题分开。

本文以 Istio 1.32.0 的 Sidecar 出站为示例。`mesh-study` 是练习命名空间，`supplier.example.com` 是文档用域名。应用 DNS、ServiceEntry 注册、代理解析、两跳出网以及 TLS 的角色都在正文定义。

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

## 解析模式会改变 Envoy 如何得到上游

同一个域名能通过不同的方式解析。Istio 把 ServiceEntry 的 `resolution` 转成内部服务模型里的地址发现类型，再按该类型生成 cluster。`convertServices` 对 `NONE`、`STATIC`、`DNS`、`DNS_ROUND_ROBIN` 和 `DYNAMIC_DNS` 分别赋予独立内部模式。把它们当成同名 DNS 的几种写法，容易在目标地址、连接池和多 IP 行为上得到错误预期。 [Istio ServiceEntry 到服务模型的转换](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/serviceregistry/serviceentry/conversion.go#L201-L242)

| 模式 | 代理的上游来源 | 适合核对的字段 |
| --- | --- | --- |
| NONE | 沿用连接的原始目标 | 捕获时识别的地址、路由匹配 |
| STATIC | 显式端点或由工作负载选择提供的端点 | endpoint 地址、端口、选择器 |
| DNS | 按服务或指定 endpoint 名称持续解析，使用得到的地址集合 | DNS cluster、代理维护的上游地址 |
| DNS_ROUND_ROBIN | 建立连接时使用 DNS 获得的地址，面向 DNS 端做轮换的大型服务 | cluster DNS 行为和已有连接的寿命 |
| DYNAMIC_DNS | 使用真实请求的 HTTP Host 或 TLS SNI 标识动态解析的域名 | 当前数据平面支持、通配域名与七层可见性 |

通配域名例如 `*.example.com` 本身不是可以解析的 DNS 名称。动态 DNS 需要代理知道真实被访问的主机，还要结合具体模式和版本阅读可用范围。对只有普通 TCP 且无法恢复 hostname 的路径，不能预期它获得 HTTP Host。DNS capture 也可以视为“捕获应用查名字的请求”的组件；只有应用拿到的地址而没有后续代理 cluster 选择，无法证明应用用了哪条出网路径。 [ServiceEntry 解析规则](https://istio.io/latest/docs/reference/config/networking/service-entry/#ServiceEntry-Resolution)

## WorkloadEntry 管的是服务里面的实例

当团队把部分进程从传统 VM 搬到 Kubernetes，逻辑服务仍叫 `catalog`，服务内的可选端点可能分布于两边。`WorkloadEntry` 描述 VM 等工作负载实例的地址、标签、账户或其他属性；带 `workloadSelector` 的 `ServiceEntry` 可按标签选择同一命名空间的已注册实例和 Pod。真实的服务迁移仍需校验地址可达、端口映射、VM bootstrap、服务账号凭证和网络边界。写了 WorkloadEntry 并不会替 VM 安装代理或签发身份证书。固定代码中 `convertWorkloadInstanceToInstances` 为具体工作负载与模型 Service 组合生成实例，正好能看到服务与工作负载这两个层次怎样汇合。 [WorkloadEntry API](https://istio.io/latest/docs/reference/config/networking/workload-entry/)，[Istio 1.32.0 注册表转换](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/serviceregistry/serviceentry/conversion.go)

下面的注册示意用 STATIC 服务和一个文档保留地址，展示 `workloadSelector` 如何把单独声明的实例挂到服务。实际 VM 场景应先按所在网格的虚拟机接入流程注册证书与 sidecar，再替换地址和账号。示例不依赖该地址提供真实服务。

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: catalog-hybrid
  namespace: mesh-study
spec:
  hosts: [catalog.hybrid.example.com]
  addresses: [198.51.100.40]
  ports:
    - number: 8080
      name: http
      protocol: HTTP
  resolution: STATIC
  location: MESH_INTERNAL
  workloadSelector:
    labels:
      app: catalog-hybrid
---
apiVersion: networking.istio.io/v1
kind: WorkloadEntry
metadata:
  name: catalog-vm
  namespace: mesh-study
spec:
  address: 192.0.2.24
  labels:
    app: catalog-hybrid
  ports:
    http: 18080
  serviceAccount: catalog
```

ServiceEntry 给出的 `addresses` 是识别这项逻辑服务的地址，WorkloadEntry 的 `address` 才是实例地址。`ports.http` 通过端口名把 `8080` 对应到该实例的 `18080`。Kubernetes 中同标签的工作负载是否加入端点集合，还需核对 Pod 所在命名空间与导入状态。ServiceAccount 名描述注册期望的服务身份，真正哪张证书可用于 mTLS 仍由工作负载的受信任接入和证书下发决定。 [Istio 虚拟机接入](https://istio.io/latest/docs/setup/install/virtual-machine/) · [WorkloadEntry 端口和标签](https://istio.io/latest/docs/reference/config/networking/workload-entry/)

## 外部域名发现后怎样更新端点

把地址写进 `STATIC` endpoints 的服务，依赖配置或工作负载注册更新地址。DNS 型服务依靠代理后续解析跟踪地址变化，应用端缓存、代理端 DNS 缓存和已有连接的寿命都可能不同。因此，外部系统切换地址时，一个新建请求与一条仍然存活的旧长连接可能指向不同的地址。核对时把 DNS 应答时间、代理实际选用的上游 IP 和连接建立时间放到一起。对需要周期变化的域名，先确认自己使用的 ServiceEntry 解析模式和协议支持。 [ServiceEntry 解析模式](https://istio.io/latest/docs/reference/config/networking/service-entry/)

## egress gateway 补上受控转发点

出口网关常用于集中连接外部系统，便于固定出口地址、统一访问记录或集中做 TLS origination。一个显式路径有两段，应用侧代理把请求转给网关，网关再匹配外部目的地。每段都有自己的路由与 TLS 设置。如果由应用建立 HTTPS 并一路透传，网关看到的是 TLS 连接。如果选择由网关对上游发起 TLS，必须确定上游地址、SNI、证书校验和下游到网关的安全设置，避免把自己发起的 HTTPS 再错误包一层 TLS。 [出口网关](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/)，[Istio TLS 链路说明](https://istio.io/latest/docs/ops/configuration/traffic-management/tls-configuration/)

`REGISTRY_ONLY` 让代理拒绝未注册服务，但 Istio 说明它不能当作出口安全策略。需要强制业务只从网关出网时，最终边界在 CNI 支持的网络策略、网络 ACL 或其他独立出口控制，并要切实校验网关还能分辨和限制允许的目的地。验证思路是分别确认应用 DNS 与已注册请求、网关的真实上游日志，以及租户在网络控制下有无直接外连。仅看到一个网关日志条目，可以证明那条请求走过网关，却无法证明整个数据路径都受约束。 [Istio 出口流量控制](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-control/)

## 把 TLS origination 配成一条可读链路

上一种 `HTTPS:443` 示意由应用发起 TLS，代理转发加密后的流。另一种可选方案由客户端 Sidecar 负责发起 TLS。应用访问明文 `http://supplier.example.com`，Envoy 选择服务逻辑端口 80 的 cluster，然后连接受控上游的 443。在隔离练习中，以下配置与前述只注册 HTTPS 的 ServiceEntry 二选一。域名和证书都要换成自己的测试系统，示例没有向外部发请求。

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: supplier-origination
  namespace: mesh-study
spec:
  hosts: [supplier.example.com]
  location: MESH_EXTERNAL
  resolution: DNS
  ports:
    - number: 80
      name: http
      protocol: HTTP
      targetPort: 443
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: supplier-origination
  namespace: mesh-study
spec:
  host: supplier.example.com
  trafficPolicy:
    portLevelSettings:
      - port:
          number: 80
        tls:
          mode: SIMPLE
          sni: supplier.example.com
          subjectAltNames: [supplier.example.com]
          caCertificates: /etc/ssl/certs/ca-certificates.crt
```

`number: 80` 是服务端口，用于匹配流和 DestinationRule。`targetPort: 443` 是实际连接的后端端口。SNI 是握手告诉服务端的名字，`subjectAltNames` 是客户端接受证书身份时使用的条件，CA 路径必须在代理容器内存在且能验证上游证书。把这些字段都叫作“HTTPS 配置”会漏掉名字验证和端口绑定。需要由 egress gateway 做 origination 时，把对外 TLS 决策移到实际发起第二跳的网关 cluster，同时保持并独立验证应用侧到网关的路由与身份。 [TLS origination 示例](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-tls-origination/) · [DestinationRule ClientTLSSettings](https://istio.io/latest/docs/reference/config/networking/destination-rule/#ClientTLSSettings)

```sh
istioctl proxy-config clusters CLIENT_POD -n mesh-study \
  --fqdn supplier.example.com -o json |
  jq '.[] | {name, type, loadAssignment, transportSocket}'
```

确认 `transportSocket` 有上游 TLS，端点选择落在外部测试域名和目标 443，再观察连接和证书校验结果。egress gateway 的第一跳 cluster 指向网关，第二跳才指向 supplier；同一个命令应针对两个对应代理分别执行。用第二跳的身份和流量证据标记它是真正的受控出口。

---

收录于 [Istio 流量管理]({{< relref "istio-traffic-learning/index.md" >}})。
