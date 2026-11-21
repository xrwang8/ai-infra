---
title: "Istio 从 Sidecar 迁到 Ambient，先核对哪些语义？"
date: "2026-11-21"
description: "用路径矩阵理解迁移中的标签、路由、身份、遥测与回滚。"
categories: ["Istio 流量管理"]
tags: ["Istio", "Ambient", "Waypoint", "迁移"]
---

把 Pod 的 Sidecar 去掉，工作负载仍可能正常接收流量。但如果之前的灰度、JWT 或审计依赖该代理读取 HTTP，那么“HTTP 200 还能返回”和“路由与策略保持一致”是不同的验收目标。迁移 Ambient 先按请求路径画出四层和七层分别由谁来接管，再按命名空间推进。

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

## 回退要恢复数据路径与策略

如果故障只出现在仍用 Sidecar 的调用方，单独重启 Ambient 目标可能无法修复请求走向。按路径矩阵判断故障到底在入口转发、L7 规则绑定还是目标 L4 授权。若决定回退一个命名空间，按当前迁移进度恢复对应的 Sidecar 策略、注入设置和 Pod；已重建工作负载的场景要恢复 Sidecar 注入并再次重建。复用备份前应核对同名的新旧策略，避免整个备份一次覆盖另一条仍在使用的路径。回退验收仍使用之前的允许与拒绝样本，并查看目标进程是否接到预期请求。迁移后已经启动的长会话，以及回退中途处于不同模式的 Pod，需要单独统计。 [官方回滚步骤](https://istio.io/latest/docs/ambient/migrate/enable-ambient-mode/#rollback)

## 验收把正例、反例和版本放在一起

除了一条成功路径，至少要能说明未经授权的身份是否在 **每条预期入口** 被拒绝，以及灰度指标到底由谁采集。长流除了检查新请求，还要观察迁移时已有连接、滚动中的取消和重连。为验证保留一份 Sidecar 基线，包括客户端状态码、延迟和代理证据。切到 Ambient 后再用 `istioctl ztunnel-config workloads` 看工作负载注册，`istioctl proxy-config routes` 看 waypoint 路由，并用真实请求核对目标选择。路由有配置只是第一步，状态、数据路径和策略证据要能互相印证。

Istio 1.32 的成熟度表把核心 waypoint 与标准 HTTPRoute 等能力列为 Stable，waypoint 上的 VirtualService 仍在 Alpha。不要因为集群接受了原有 YAML，就认为所有功能已经自动平移。社区提交的 waypoint 在扩缩容期间旧连接返回 503 的报告可用来构造验证问题，例如“既有 keepalive 与新建连接表现一样吗”，它只描述报告者的条件。最有价值的迁移产物是一份能随版本复用的路径矩阵，记下入口、身份、策略执行点和验证结果。 [Istio 功能阶段](https://istio.io/latest/docs/releases/feature-stages/)，[社区问题 #62574](https://github.com/istio/istio/issues/62574)

[Istio 流量管理专题]({{< relref "istio-traffic-learning/index.md" >}})

前置阅读 · [Istio 怎样看见一次请求？指标、日志和 Trace 的边界]({{< relref "posts/istio-telemetry-trace-context/index.md" >}})
