---
title: "Istio 流量镜像复制了什么？主请求、影子请求和副作用"
date: "2026-11-21"
lastmod: "2026-11-22"
description: "理解 Istio 镜像的复制比例、返回路径和影子服务的数据边界，再与灰度对照。"
categories: ["Istio 流量管理"]
tags: ["Istio", "流量镜像", "VirtualService"]
---

给 `catalog-v2` 5% 的镜像流量，用户仍然应从 `catalog-v1` 得到结果。代理在处理这条路由时额外发出一份请求。主请求和镜像请求分别选择各自的上游，镜像响应由代理忽略。这个机制适合让候选版本接触有代表性的请求形状，但要捕捉 v1 和 v2 回答的差异，仍需从服务侧收集和比较。

本文使用 Istio 1.32.0 和已注入 Sidecar 的 HTTP 客户端。`catalog` 是固定的访问入口，`catalog-v1` 为主后端，`catalog-v2` 为隔离影子后端。后端都暴露 8080；示例只发合成读请求。

## 镜像一次请求时经过哪些地方

假设客户端用 HTTP 请求 `GET /items/42`。处理请求的 Envoy 先找到匹配规则，确定主目标；附在该规则上的镜像策略再决定是否复制。命中镜像比例时，代理为 shadow 目标安排额外请求。Istio 镜像任务记录了镜像请求的 Host 或 `:authority` 通常追加 `-shadow` 的行为。候选服务若按 Host 分配虚拟主机，必须核对它实际接收和匹配的值。配置应部署在会观察 HTTP 的代理上。处于 Ambient 的纯 ztunnel 路径没有这条 HTTP 路由可执行。 [Istio 流量镜像任务](https://istio.io/latest/docs/tasks/traffic-management/mirroring/)

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: catalog-mirror
  namespace: mesh-study
spec:
  hosts: [catalog.mesh-study.svc.cluster.local]
  http:
    - name: shadow-read
      match:
        - method:
            exact: GET
          uri:
            prefix: /items/
      route:
        - destination:
            host: catalog-v1.mesh-study.svc.cluster.local
            port:
              number: 8080
      mirror:
        host: catalog-v2.mesh-study.svc.cluster.local
        port:
          number: 8080
      mirrorPercentage:
        value: 5.0
    - name: primary-only
      route:
        - destination:
            host: catalog-v1.mesh-study.svc.cluster.local
            port:
              number: 8080
```

这是与灰度篇分开的配置样例。三个 Service 在同一命名空间且暴露 HTTP 8080。catalog 选择全部版本，后两个 Service 的选择器分别再约束 `version: v1` 和 `version: v2`。显式写出 `mirrorPercentage` 可避免把默认的全量镜像误带入目标环境。规则中的普通 `route` 对所有匹配的请求照常执行，镜像百分比直接描述额外副本的份额。若有 200 个符合条件的请求每秒流入且每次以 5% 概率复制，候选服务平均额外收到约 10 个请求每秒。这还要乘上请求大小、执行成本和连接占用，才是容量估算。 [VirtualService 镜像字段](https://istio.io/latest/docs/reference/config/networking/virtual-service/)

Istio 1.32.0 的 `applyHTTPRouteDestination` 把 `mirror` 或 `mirrors` 转成 Envoy RouteAction 的 `RequestMirrorPolicies`。该策略附在主路由动作上，和 `WeightedClusters` 是不同字段。看配置时，应同时找到主 cluster 与 shadow cluster，再检查镜像比例和两端是否有端点。 [固定版本的镜像转换源码](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/networking/core/route/route.go)

## 从 Envoy 的哪一行分出影子流

Istio 可接受单目标 `mirror` 与列表 `mirrors` 两种配置。单目标分支先看 `mirrorPercentage`，显式为零会跳过生成策略，百分比缺省则按全量处理；`mirrors` 为每个目标单独计算 `percentage`。理念上多个目标可以同时收到副本，所以两个各 10% 的镜像目标并不意味着请求只能二选一。容量估算应把各目标可能处理的副本分开计数后加总。 [Istio 镜像比例转换](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/networking/core/route/route.go#L982-L1028)

在 Istio 1.32.0 pin 的 Envoy 中，Router 先选本请求的 shadow policy，复制还未被主上游修改的请求头，然后开始主请求的 upstream 路径。没有请求体的请求可以直接交给 `ShadowWriterImpl.shadow`；带请求体的路径通过 `streamingShadow` 建立异步请求，并关联 buffer limit 和 watermark。代理还把镜像选项设置成丢弃响应体。这样就能解释为什么候选端响应不会被转回调用方，同时也能解释上传请求体仍然会对代理产生缓冲和带宽成本。影子 cluster 如果已从 CDS 撤销，ShadowWriter 检查不到 cluster 后会结束该副本。 [Envoy Router 的镜像分流](https://github.com/envoyproxy/envoy/blob/56e0748e3deaeb8af7efcbf2f97329bb15539767/source/common/router/router.cc#L865-L1001) · [Envoy ShadowWriter](https://github.com/envoyproxy/envoy/blob/56e0748e3deaeb8af7efcbf2f97329bb15539767/source/common/router/shadow_writer_impl.cc#L14-L75)

Host 后缀是在影子 Writer 再处理，所以 `catalog.mesh-study.svc.cluster.local:8080` 默认会变成 `catalog.mesh-study.svc.cluster.local-shadow:8080`，后缀插在端口之前。若候选后端有严格的 Host 支持范围，影子请求得到 404 仍可能说明镜像确实送达了。要比较候选业务行为，先确认它进入了与主请求同意义的处理分支。

## 只完成主请求，就算镜像成功了吗

镜像比例是代理对符合规则的请求做的选择。影子 Pod 上真正看到的次数还受连接、上游与影子服务容量影响。可以先用每条请求都能关联的合成 ID，分别记录主侧收到的请求数、影子侧收到的请求数和影子侧成功完成的请求数。若主侧有 1000 个样本，估计应复制约 50 个，影子侧却只完成 20 个，需要分清候选端是否根本没收到其余请求，或者已经收到但超时或失败。只用客户端 1000 条成功记录解释影子覆盖率，会把两个阶段混在一起。

## 影子系统可以做什么

HTTP 方法是筛选条件，副作用取决于服务实现。一个 GET 可能写审计记录、刷新缓存、回源计费或触发模型计算。POST 甚至可能带着真实订单到候选服务。镜像会涉及必要的请求头和请求内容，故需要清楚知道令牌、用户数据和访问记录能否进入影子环境。工程上可以限定合成读请求，在影子环境使用独立数据源以及受控身份，再观察响应和依赖调用。

镜像是单独的请求，因此 v2 变慢时用户不会收到它的慢响应，但共享端口、代理 CPU、网络和下游依赖仍可能相互影响。对 SSE 或大请求体，先验证 Envoy 版本、路由行为和流持续时间，并从影子侧测实际负载，再决定是否放大比例。这个区别在推理服务里很明显，一份影子请求如果完整跑 Prefill 和 Decode，影子 GPU 会承担真实成本。

| 观测点 | 能回答的问题 |
| --- | --- |
| 客户端 | 主服务是否在可接受的时间内响应 |
| 主服务 | 该请求实际走了哪个版本，以及错误出现在哪里 |
| 影子服务 | 候选版本收到了多少、如何响应、消耗多少资源 |
| 两侧按请求关联 | 对同样的输入，结果和阶段耗时是否发生变化 |

关联最好使用由实验客户端生成、允许记录的标识，并先检查它在两条路径的实际传递。候选服务能接住真实请求只是镜像验收的一部分；切到灰度后，用户才会直接体验该版本的响应质量。

## 镜像负载怎样估算

把匹配的请求速率写成 `r`，复制比例写成 `p`，平均请求体大小写成 `b`。对于一个影子目标，预期副本速率为 `r × p`，被送入影子方向的请求体带宽近似 `r × p × b`。若 200 req/s、5% 比例、平均请求体 64 KiB，请求体就约有 640 KiB/s 的额外吞吐，尚未算响应、协议和重传。对于应用耗时，10 req/s × 平均 3 秒约对应 30 个在途影子请求。即使用户的主请求没有增加任何上游尝试，这些资源仍会进入共享网络、连接池和候选计算设备。

低比例且低流量下也要估计等待时间。假设独立抽样，一个满足条件的请求按 5% 概率复制，连续 20 次都没收到副本的概率为 `0.95^20 ≈ 36%`。先确认高比例的小样本能到达，再用足够样本做低比例观察，可以把错误配置和小样本偶然性区分开。相同思路适用于候选 shadow 集群多副本下的单 Pod 无日志。

## 用同一请求 ID 识别三个阶段

在已有实验命名空间里，可以先把镜像比例临时设为 100%，只发一条带实验 ID 的 GET，然后回到低比例取样。假设两个后端 Deployment 名为 `catalog-v1` 和 `catalog-v2`，应用用非敏感实验 ID 记接收、完成和状态。只记录必要字段，响应体是否需要留存由测试输入与数据边界决定。

```sh
kubectl -n mesh-study exec deploy/reader -c reader -- \
  curl -fsS --max-time 3 \
  -H 'x-lab-id: shadow-one' \
  http://catalog.mesh-study.svc.cluster.local:8080/items/42

kubectl -n mesh-study logs deploy/catalog-v1 -c catalog --since=2m |
  grep -F shadow-one
kubectl -n mesh-study logs deploy/catalog-v2 -c catalog --since=2m |
  grep -F shadow-one
```

将候选服务设置为返回可区分的 404 可以帮助读出路径：客户端状态来自主请求，候选日志里的 404 对应独立副本。若候选服务无日志，要把“没有复制”“复制后未能连接”“请求到了另一个实例”和“日志未采到”分成四个假设。多副本部署需采集对应标签的 Pod 日志，不能把一个 Deployment 默认选中的 Pod 当所有副本。最后用一组预估比例低、流量受控的测试样本检查候选资源，调整比例后再测同一批输入。

---

收录于 [Istio 流量管理]({{< relref "istio-traffic-learning/index.md" >}})。
