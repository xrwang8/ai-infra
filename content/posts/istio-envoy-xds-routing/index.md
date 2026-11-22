---
title: "从 Istio YAML 到 Envoy 转发：走一遍 xDS 配置链"
date: "2026-11-21"
lastmod: "2026-11-22"
description: "看 Listener、Route、Cluster、Endpoint 的依赖，以及如何分辨 ADS 确认和路由生效。"
categories: ["Istio 流量管理"]
tags: ["Istio", "Envoy", "xDS"]
---

VirtualService 给出用户期望的流量行为。真正执行转发的 Envoy 使用的是监听器、路由配置、上游集群和端点。学 xDS 可以沿一条 `reader → catalog` 的 HTTP 请求走一遍，然后反过来看，修改 Kubernetes 中的不同对象为什么会改变代理里的不同资源。

这里的 xDS 指 Envoy 动态发现协议。示例以 Istio 1.32.0 的 Sidecar 出站为主，假设 `mesh-study` 命名空间有 HTTP catalog Service，端口 8080，reader Pod 已注入代理。文中 JSON 字段是代理有效配置的读法，格式可能随代理版本调整。

## 四种资源回答四个问题

| 对象 | 负责的决定 | 与变更有关的例子 |
|---|---|---|
| Listener / LDS | 接到连接时用哪条过滤链 | 监听端口或协议变化 |
| Route configuration / RDS | HTTP 主机、路径、Header 怎样选择上游 | 添加条件或修改权重 |
| Cluster / CDS | 怎样建立上游连接、使用何种 TLS 和池策略 | 改 DestinationRule 或新增 subset |
| Endpoint / EDS | 可供上游集群选择哪些具体地址 | Pod 更新或就绪变化 |

这是用了 HTTP 过滤链和动态端点发现的常见路径。TCP 连接不一定走 HTTP route configuration，DNS 上游也可能采用不同的地址解析。SDS 处理 TLS 相关的秘密发现。ADS 让多种资源通过同一条聚合发现流交换，各资源之间仍然有独立关系。 [Envoy xDS 协议](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol)

例如，只增加一个 v2 Pod，最直接影响的是服务端点信息。只把 v2 权重从 10 改成 20，主要影响路由配置。新建 v2 subset，还需要让 cluster 和对应端点一起出现。掌握每一层的作用，就能判断应该去查哪个对象。

## 用具体主机和 cluster 名对账

假设客户端访问 `catalog.mesh-study.svc.cluster.local:8080`，匹配实验 Header 后应选 v2。先找到该次连接被交给哪条 HTTP 过滤链，再看 route 的虚拟主机和目标 cluster。典型 Sidecar 子集场景里，可能看到 `outbound|8080|v2|catalog.mesh-study.svc.cluster.local`。从实际 route 读取名字比猜名字可靠。最后检查该 cluster 的端点、版本和实际连通性。

```sh
istioctl proxy-status
istioctl proxy-config listeners -n mesh-study CLIENT_POD
istioctl proxy-config routes -n mesh-study CLIENT_POD
istioctl proxy-config clusters -n mesh-study CLIENT_POD \
  --fqdn catalog.mesh-study.svc.cluster.local
istioctl proxy-config endpoints -n mesh-study CLIENT_POD \
  --cluster 'outbound|8080|v2|catalog.mesh-study.svc.cluster.local'
```

这是带 Sidecar 的客户端观察配方。`CLIENT_POD` 换成实际 Pod。如果这次访问经过入口网关，应查网关代理。如果要验证 Ambient 七层策略，则查实际执行的 waypoint。反复看同一份控制面 YAML 不会提供这些不同视角。 [Istio proxy-config](https://istio.io/latest/docs/ops/diagnostic-tools/proxy-cmd/)

## ACK、NACK 和请求命中分别证明什么

状态型 xDS 交互的响应有 nonce。客户端可以针对特定响应确认，或通过错误信息表示拒绝。NACK 因而是相应更新未被接受的证据，还要检查代理正在使用的配置。ACK 证明的范围也是一份资源更新，不能直接代替请求命中。Delta xDS 又以变更过的资源为单位传送，与整份状态发布的方式不同。 [Envoy ACK 与 NACK](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol)

比较同一次上线的三份状态。第一份是 API 对象及控制器给出的反馈，第二份是目标代理的有效配置，第三份是请求实际到了哪个后端。前两份即使一致，也可能因为选错观测代理、不同流量绑定或长连接的生命周期，而无法解释后端交付。

## 新路由要等到哪些资源到位

例如上线前只有 v1 集群，现在把 v2 权重加到 10。控制面下发新 cluster 时，代理还要获得它依赖的端点更新，才能按新路由完整工作。Envoy 对不同类型资源有依赖和 warming 行为，ADS 可用来编排下发顺序；这不等于所有代理在同一时刻完成更新。安全发布可以先让 v2 端点就绪，确认目标代理能看到 cluster 与端点，再改灰度权重。

有个相反的信号也值得看：负载均衡仍使用原有 v1，而 `proxy-status` 显示 `SYNCED`。这可能是代理已经正确接收了自己的配置，却从来没有收到你要找的规则。例如 HTTPRoute 只绑定入口 Gateway，发请求的是网格内 reader。此时问题在配置绑定的范围，应从请求入口回到对应代理，而非反复等待一次新的 ACK。 [Envoy xDS 的资源更新顺序](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol)

## 1.32.0 源码里有两个可读入口

在 `DiscoveryServer.processRequest` 中，可以看到状态型 ADS 请求进入后，Istio 调用 `ShouldRespond` 判断是否需要生成响应。后续的推送使用该连接关联的代理与 PushContext。这段代码说明订阅、确认和配置生成如何衔接。HTTP route 的入口 `BuildHTTPRoutes` 又会根据代理视角走 Sidecar 或 Gateway 的生成流程。两个代理可能看到不同的服务可见范围和配置。 [ADS 处理](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/xds/ads.go#L142) · [ShouldRespond](https://github.com/istio/istio/blob/1.32.0/pkg/xds/server.go#L353) · [HTTP 路由构造](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/networking/core/httproute.go#L56)

## 一次推送里的三个不同编号

在 SotW ADS 流上，`type_url` 选择资源种类，`version_info` 表明客户端对该资源种类掌握的版本，`response_nonce` 用于把请求与此前收到的某次响应对应起来。一个流上可以交替发送 CDS 和 RDS，因此 nonce 应在相应的类型和连接上解释。API Server 上的对象 `resourceVersion`、Istiod 发布的 xDS 版本、Envoy 回报的 nonce 属于不同层，不宜直接比较大小。

下面是为了教学压缩过的时序。`r8` 和 `n101` 是示意值，省略 Node、实际资源与传输细节。

```text
代理 → Istiod  RDS 订阅 [8080]，response_nonce 为空
Istiod → 代理  RDS 响应 version=r8，nonce=n101
代理 → Istiod  RDS 请求 version=r8，response_nonce=n101
                如果只确认旧订阅，控制面不需要再发一遍
代理 → Istiod  RDS 请求 version=r8，response_nonce=n101，
              资源订阅改成 [8080, 8081]
                新订阅本身需要响应，尽管 version 和 nonce 都没变
```

`ShouldRespond` 所做的正是这层区分。进入函数后先看 `ErrorDetail`，有错误便记录拒绝及原因；初次订阅或控制端没有 watcher 记录时建立 watcher；nonce 落后于已发送的响应时走 stale 分支；nonce 对上后保存确认状态，再比较订阅集合，必要时为新增资源或 warming 发响应。可用自己的请求时序理解下面的伪代码，序列是教学摘要。

```text
有 error_detail                     → 记录拒绝
首次订阅或重连后没有 watcher        → 建立订阅并响应
nonce 已经过期                      → 计数并结束这次处理
nonce 匹配 + 新的订阅内容            → 更新确认与订阅，按类型决定响应
nonce 匹配 + 订阅未变                → 记录 ACK，通常到这里结束
```

Delta ADS 更适合理解“这次变动了哪些资源”，它有显式订阅或退订字段；Ambient 的 ztunnel 另用地址与工作负载策略资源，不能把 waypoint 的 Envoy RDS 表直接当成 ztunnel 的状态。Istio 1.32.0 的 `WorkloadGenerator.GenerateDeltas` 针对地址订阅及推送变更产出资源，`WorkloadRBACGenerator` 则处理其策略。排查 Ambient 时，找 L4 发现看 ztunnel，找 HTTP 匹配看执行规则的 Envoy。 [SotW 响应条件](https://github.com/istio/istio/blob/1.32.0/pkg/xds/server.go#L353-L447) · [Ambient 工作负载资源](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/xds/workload.go#L43-L137)

## 把 routes、clusters 和 endpoints 的输出连起来

在已经运行的 Sidecar 集群中，从捕获 reader 请求的 Pod 读取下面三份结果。过滤时先观察有效内容再挑字段，检查途中没有把入口代理当成源代理。`--name` 限定的是 RouteConfiguration 名称，和 VirtualService 中的 HTTP 路由名不同。

```sh
ns=mesh-study
proxy=READER_POD
service=catalog.mesh-study.svc.cluster.local

istioctl proxy-config routes "$proxy" -n "$ns" -o json |
  jq -r --arg host "$service" '
    .[] | .virtualHosts[]? |
    select(any(.domains[]?; . == $host or startswith($host + ":"))) |
    .routes[] |
    [.name, (.route.cluster // ([.route.weightedClusters.clusters[]?.name] | join(",")))] | @tsv'

istioctl proxy-config clusters "$proxy" -n "$ns" --fqdn "$service"
istioctl proxy-config endpoints "$proxy" -n "$ns" \
  --cluster "outbound|8080|v2|$service"
```

例如 v2 权重可见而对应 cluster 没有健康端点，去查 Service 选择器、Pod 标签与 EndpointSlice；如果 v2 cluster 有健康 Pod，却是另一条无条件匹配先消费了请求，去查 route 的顺序和 match；如果配置里根本没这个主机，先核对发出的 authority、Sidecar 的服务可见性和查的是哪一个代理。与查询相反的路径同样重要：看到后端回了 v2，可以沿访问日志往上确认使用的是预期 route，而非其他直接转发入口。

## 哪一类变更会牵动哪几类资源

| 案例 | 通常先变化的代理资源 | 要留意的依赖 |
| --- | --- | --- |
| Ready Pod 替换另一个 Pod，Service、端口、策略都没变 | EDS 中该集群的成员 | 旧连接仍有生命周期，新的端点可见不等于旧连接立刻消失 |
| 给已经存在的 v2 权重从 10 调到 20 | RDS 中规则的动作 | 查的是执行该规则的代理，当前 cluster 及端点仍有效 |
| 首次为 v2 新建 subset 且启用流量 | CDS、EDS 与 RDS | 新 cluster 有可用的 endpoints，相关资源状态能一起解释请求 |
| 把新的端口加入监听，增加一套 HTTP 过滤链 | LDS，常伴随对应 RDS | 能接收连接、选中正确的 filter chain，再谈路由 |

这张表描述的是最直接受影响的配置表示。控制面可能把多个事件合并，或按自身增量逻辑计算不同类型，因此运行日志里的 push 次数与 API 修改次数无需一一对应。把一次待验收的变更记成“对象变更 → 配置资源 → 明确的客户端行为”，能比按单个时间点比较配置快照得到更多信息。 [xDS 顺序和最终一致性](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol)

掌握这条链之后，可以把“没生效”拆成具体问题。连接进了哪条过滤链？HTTP 规则选了哪个上游？这个上游是否拥有预期的实例？把答案连起来，后面学习灰度和重试时也能找到相应的执行位置。

---

收录于 [Istio 流量管理]({{< relref "istio-traffic-learning/index.md" >}})。
