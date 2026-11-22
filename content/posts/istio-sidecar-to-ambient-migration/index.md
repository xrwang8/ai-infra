---
title: "Istio 从 Sidecar 迁到 Ambient，先核对哪些语义？"
date: "2026-11-21"
lastmod: "2026-11-22"
description: "用路径矩阵理解迁移中的标签、路由、身份、遥测与回滚。"
categories: ["Istio 流量管理"]
tags: ["Istio", "Ambient", "Waypoint", "迁移"]
---

把 Pod 的 Sidecar 去掉，工作负载仍可能正常接收流量。但如果之前的灰度、JWT 或审计依赖该代理读取 HTTP，那么“HTTP 200 还能返回”和“路由与策略保持一致”是不同的验收目标。迁移 Ambient 先按请求路径画出四层和七层分别由谁来接管，再按命名空间推进。

本文以 Istio 1.32.0 为基线，假设已有 Sidecar 网格，目标是把一个 Kubernetes Namespace 的 HTTP catalog 服务迁到 Ambient。路由决策与安全验证围绕这个例子展开，读者可以直接从本文理解 CNI、ztunnel、waypoint 和原有 Sidecar 的交接。

## 先为每一条入口路径定预期

用 `catalog` 举例。预期从服务域名访问时按 90/10 灰度，对目录请求要求 `reader` 身份和 GET，允许的入口还需留下 HTTP 指标。列出可能访问它的来源，并记下请求原来的目的地址。

| 来源和原始目的地 | 必须验证的路径 |
| --- | --- |
| Ambient 客户端 → catalog Service | 目标 Service 是否绑定处理 service 流量的 waypoint，HTTPRoute 是否在该 waypoint 生效 |
| Ambient 客户端 → 直连 catalog Pod | 是否存在处理 workload 流量的 waypoint，或该直连路径能否用四层策略拒绝 |
| 仍使用 Sidecar 的客户端 → catalog | 混合迁移期间，该来源可能绕过 waypoint，依赖它的七层策略不能作为连续保护 |
| Ingress gateway → catalog | 默认要按独立路径核验，启用 ingress 使用 waypoint 后再检查实际转发与授权 |
| 无网格身份的内部客户端 → catalog | 是否只能走允许的入口，直接连接是否由 STRICT 或对应 L4 策略拒绝 |

`istio.io/use-waypoint` 用于让目标选择 waypoint，waypoint 的 `istio.io/waypoint-for` 决定它接收 service、workload 或两类流量。Service 请求最后也会转给 Pod，但 waypoint 选择基于请求原来的目的地。对 ingress，需要目标 Service 或 Namespace 显式设置 `istio.io/ingress-use-waypoint=true`，Waypoints 和实际控制平面也要支持该路径。Istio 1.32.0 源码把 `ENABLE_INGRESS_WAYPOINT_ROUTING` 经 `registerAmbient` 注册，缺省取值区分 ambient 是否启用。因此应结合 Istiod 的实际配置、安装方式、版本和代理路由检查，不根据单一缺省值判断流量必经 waypoint。 [Waypoint 生效范围](https://istio.io/latest/docs/ambient/usage/waypoint/)，[1.32.0 功能开关源码](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/features/ambient.go#L52-L67)

## 迁移时优先保住安全与协议语义

对已经有 L7 策略的服务，官方迁移指南明确列出了 Sidecar 客户端可能绕过目标 waypoint，以及替换 selector 策略期间的执行缺口。需要连续强制 L7 策略时，先在迁移窗口隔离相关路径，或在已经受控的入口设等效限制并逐条证明，再切换。对要求所有允许访问都经过 waypoint 的目标，应当给目标工作负载配合适的四层授权。例如只允许正确的 waypoint 身份进入受管工作负载，避免仅靠一个 waypoint 标签做约束。启用前还要识别探针和内部运维流量的实际路径。 [Istio 启用 Ambient 的迁移指南](https://istio.io/latest/docs/ambient/migrate/enable-ambient-mode/)，[Waypoint 安全策略](https://istio.io/latest/docs/ambient/usage/waypoint/)

逐个命名空间迁移时，先部署并验证需要的 waypoint，把依赖成熟 API 的路由和 L7 策略按 Service 等资源重新绑定；接着让命名空间启用 ambient，确认纳管，再停用 Sidecar 注入，最后重建工作负载 Pod。不要认为修改注入标签会移除已经存在的代理。核对命名空间注入标签、revision、Pod 模板的显式注入设置以及实际 workload 状态。保留可恢复的原 Sidecar 策略与工作负载模板，回滚时使 Sidecar 再次接管实际路径，核对请求恢复后才收尾。迁移期间要安排旧 selector 型 L7 策略的撤换。Istio 1.32.0 的 `convertAuthorizationPolicy` 会从发往 ztunnel 的 ALLOW 中省略带 HTTP 条件的规则；包含 L4 与 L7 的 DENY 可能删去 L7 条件后变得更严格，只有 HTTP 条件且没有可执行匹配的 DENY 则可能不起作用。先核对策略状态和 1.32.0 源码，再用一组允许与拒绝的请求确认新策略。纯 L4 selector 策略可以按各自用途保留。 [1.32.0 ztunnel 授权转换](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/serviceregistry/ambient/authorization.go#L398-L489) [迁移准备与能力对照](https://istio.io/latest/docs/ambient/migrate/)

## 迁移一条规则时保留两类绑定

假设目标命名空间为 `mesh-study`，其中的 HTTP Service `catalog` 用 8080，Pod 都有 `app: catalog`。现有 Sidecar 规则限制 reader 只能 GET `/items/*`。完成切换后，把两种策略分开管理：waypoint 上按原始调用方身份及 HTTP 条件判定，ztunnel 上只允许正确的 waypoint 身份进入工作负载。下列为切换完成后的状态，禁止直接到达的约束应放在经过验证的切换节点应用。示例假设所有合法流量已经可经该 waypoint，真实环境需先核对健康检查、入口、残留 Sidecar 和运维流量。

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
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: catalog-waypoint-l7
  namespace: mesh-study
spec:
  targetRefs:
    - kind: Service
      group: ""
      name: catalog
  action: ALLOW
  rules:
    - from:
        - source:
            principals: [cluster.local/ns/mesh-study/sa/reader]
      to:
        - operation:
            methods: [GET]
            paths: ["/items/*"]
---
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: catalog-workload-l4
  namespace: mesh-study
spec:
  selector:
    matchLabels:
      app: catalog
  action: DENY
  rules:
    - from:
        - source:
            notPrincipals: [cluster.local/ns/mesh-study/sa/catalog-waypoint]
```

Service 通过 `istio.io/use-waypoint=catalog-waypoint` 绑定，先从 Gateway 与 Deployment 核对 waypoint Pod 实际 ServiceAccount。上面身份假定这个账户名为 `catalog-waypoint`，请以实际 `spec.serviceAccountName` 和网格 trust domain 为准。L7 配置在 waypoint 可见的原调用方身份上执行，目标 ztunnel 在最后一跳看到的是 waypoint 的身份。把两个 principal 对调，正常流也可能被挡住。如果业务还需要让指定的非 HTTP 入口一直连到同一 Pod，必须明确其安全模型，按支持的路径和端口逐项设计，不能用同一个期望覆盖所有流量。 [Istio 迁移授权策略](https://istio.io/latest/docs/ambient/migrate/migrate-policies/)

```sh
kubectl -n mesh-study get gateway catalog-waypoint -o yaml
kubectl -n mesh-study get pod \
  -l gateway.networking.k8s.io/gateway-name=catalog-waypoint \
  -o custom-columns=POD:.metadata.name,SA:.spec.serviceAccountName
istioctl ztunnel-config services -n istio-system
istioctl ztunnel-config workloads -n istio-system
istioctl proxy-config routes WAYPOINT_POD -n mesh-study
```

这些输出分别回答 Gateway 是否就绪、Waypoint 身份是什么、Service 和工作负载是否被 ztunnel 看见、HTTP 路由是否进入 Envoy。若从 ingress 来的请求仍没有 waypoint 观测，Istio 的 `ServicesWithWaypoint` 还要解析可用的 waypoint hostname，并由 `ingressUseWaypoint` 先看 Service 层显式值，未指定才看 Namespace。这解释了为什么 Namespace 采用该路径后，单个 Service 仍可能有不同结果。转发图要细到资源与来源。 [Ingress 使用 waypoint 的注册表源码](https://github.com/istio/istio/blob/1.32.0/pilot/pkg/serviceregistry/ambient/sidecar_interop.go#L172-L238)

| 对照请求 | 预期决策点 | 需要记录 |
| --- | --- | --- |
| reader 从网格内 GET 合法 Service 路径 | waypoint L7 允许、目标 L4 允许代理身份 | 收到 2xx，waypoint 与应用侧互相印证 |
| reader 用同一个 Service 调用其他方法 | waypoint L7 拒绝 | 看 waypoint 的拒绝，无应用请求 |
| 其他被纳管的账号请求同一 Service | waypoint L7 拒绝身份 | 同一入口，安全决策改变 |
| 受管客户端尝试直接连接目标 Pod | 目标侧 L4 拒绝非 waypoint 身份 | 没有抵达应用，区分连通性与显式拒绝 |
| 合法外部流量从 ingress 进入 | 按预期受控路径处理 | 同时核对入口和 waypoint，校验实际账号假设 |

## 回退要恢复数据路径与策略

如果故障只出现在仍用 Sidecar 的调用方，单独重启 Ambient 目标可能无法修复请求走向。按路径矩阵判断故障到底在入口转发、L7 规则绑定还是目标 L4 授权。若决定回退一个命名空间，按当前迁移进度恢复对应的 Sidecar 策略、注入设置和 Pod；已重建工作负载的场景要恢复 Sidecar 注入并再次重建。复用备份前应核对同名的新旧策略，避免整个备份一次覆盖另一条仍在使用的路径。回退验收仍使用之前的允许与拒绝样本，并查看目标进程是否接到预期请求。迁移后已经启动的长会话，以及回退中途处于不同模式的 Pod，需要单独统计。 [官方回滚步骤](https://istio.io/latest/docs/ambient/migrate/enable-ambient-mode/#rollback)

如果 Catalog 有 6 个副本，从刚纳管、仍存在注入的 Pod，到新的 Ambient Pod 都可能出现在同一 Service 的端点里。需要记下选择器、列出两类 Pod 的可接收条件，并逐步滚动；不要只数 Deployment 的 Ready 副本。调用方还有 Sidecar 时，从它到已迁移目标的那一路仍可能与 Ambient 调用方不同，因此迁移单元也可以从目标命名空间扩展到已枚举的调用方集合。生产操作时保留一条明确的停线条件，例如任何预期拒绝的请求变成成功，或已有流的关闭率超过容量允许值。

可观测面的交接同样需要基线。Sidecar 的日志和源/目标 reporter 换到共享 waypoint 与 ztunnel 后，原来的按 Pod 聚合图可能需要重写。保留旧表口径，同时用新的采集点提供同一条基准请求的完整证据，然后再改变告警表达式。记录当时控制平面版本、waypoint Gateway 和目标 Namespace 的生效状态，可以把迁移后出现的问题重新定位到配置、版本或数据面。 [迁移后的遥测变化](https://istio.io/latest/docs/ambient/migrate/enable-ambient-mode/#post-migration-observability-changes)

## 验收把正例、反例和版本放在一起

除了一条成功路径，至少要能说明未经授权的身份是否在 **每条预期入口** 被拒绝，以及灰度指标到底由谁采集。长流除了检查新请求，还要观察迁移时已有连接、滚动中的取消和重连。为验证保留一份 Sidecar 基线，包括客户端状态码、延迟和代理证据。切到 Ambient 后再用 `istioctl ztunnel-config workloads` 看工作负载注册，`istioctl proxy-config routes` 看 waypoint 路由，并用真实请求核对目标选择。路由有配置只是第一步，状态、数据路径和策略证据要能互相印证。

Istio 1.32 的成熟度表把核心 waypoint 与标准 HTTPRoute 等能力列为 Stable，waypoint 上的 VirtualService 仍在 Alpha。不要因为集群接受了原有 YAML，就认为所有功能已经自动平移。社区提交的 waypoint 在扩缩容期间旧连接返回 503 的报告可用来构造验证问题，例如“既有 keepalive 与新建连接表现一样吗”，它只描述报告者的条件。最有价值的迁移产物是一份能随版本复用的路径矩阵，记下入口、身份、策略执行点和验证结果。 [Istio 功能阶段](https://istio.io/latest/docs/releases/feature-stages/)，[社区问题 #62574](https://github.com/istio/istio/issues/62574)

---

收录于 [Istio 流量管理]({{< relref "istio-traffic-learning/index.md" >}})。
