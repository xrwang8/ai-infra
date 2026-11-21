---
title: "Kubernetes 1.37：GPU Pod 不改 YAML，也能走 DRA？"
date: "2026-10-15"
categories: ["AI Infra"]
tags: ["Kubernetes", "DRA", "GPU 调度"]
---

![Kubernetes GPU 分配封面](assets/k8s-gpu-cover-5x2.png)

*封面图：原创概念视觉，表达从整数资源请求到设备属性分配的变化；不是实际集群拓扑。*

一个值得注意的 Kubernetes 1.37 变化是：Pod 仍然可以写 `example.com/gpu: 1`，但满足这个请求的，不一定是传统 Device Plugin，也可以是 DRA 驱动。工作负载不必显式写 ResourceClaim；管理员把扩展资源名映射到 DeviceClass，调度器便能从 DRA 发布的设备中选择匹配项。这个能力在 1.37 进入稳定阶段，最大的意义不是“GPU 调度已经完美”，而是给现有扩展资源工作负载提供了一条渐进迁移路径。[1][2]

## 一、先看旧模型：Kubernetes 为什么只认“几张卡”

传统 Device Plugin 的职责，是让 kubelet 知道节点上有哪些特殊设备。插件向 kubelet 注册资源名，例如 `nvidia.com/gpu`，持续上报设备列表和健康状态；kubelet 再把可用数量发布到 Node 状态。Pod 通过容器资源限制请求一个整数，调度器主要据此判断哪个节点容得下它。[3]

真正的设备 ID 分配与容器注入，发生在节点侧。容器创建时，kubelet 调用 Device Plugin 的 `Allocate`；插件可以返回设备节点、环境变量、挂载、注解或 CDI 设备名，让容器运行时完成访问配置。也就是说，Pod 里写的 `gpu: 1` 并不是某一块 GPU 的身份，更不是显存、互连或应用性能的承诺。[3]

这种模型很实用：工作负载只需声明数量，平台也能比较容易地部署。它的局限同样清楚。扩展资源本质上是整数，无法在原生请求里直接表达“显存至少多少”“与另一块 GPU 处于什么拓扑关系”“需要哪种分区能力”。Kubernetes 的 GPU 调度文档也说明，混合型号集群往往需要额外依赖节点标签与亲和性。标签能筛节点，但它不是结构化的单设备属性选择。[3][4]

这不是说 Device Plugin 一无是处。它的 API 长期稳定承载了大量 GPU 工作负载，节点侧也可以通过 `GetPreferredAllocation` 提供设备选择建议。问题在于，调度器首先面对的是按资源名汇总的数量；更细的设备选择往往被推迟到节点上，或需要平台自己建立额外协议。对于同型号整卡分配，这种简单性可能正合适；对于混合卡型、分区、共享与复杂拓扑，信息损失就会逐渐显现。[3]

因此，很多平台在 Device Plugin 之外又叠加调度器插件、注解、Webhook 或自定义 CRD，补上型号、显存、共享和拓扑信息。这些扩展各有价值，却会把设备语义分散到多处：Pod 看见的是整数，调度器看的是标签和注解，节点侧插件还要再解释一次。出现分配不一致时，很难只凭 `kubectl describe pod` 还原完整决策。

## 二、DRA 改变的是设备描述与分配边界

DRA（Dynamic Resource Allocation）试图把设备从“不透明的整数”变成可描述、可选择的对象。驱动创建 ResourceSlice，公布设备池、设备属性、容量以及哪些节点可以访问；管理员或驱动创建 DeviceClass，把一类设备的选择条件和配置抽象出来；工作负载可通过 ResourceClaim 或 ResourceClaimTemplate 声明需要什么。调度器在分配资源时，不再只能对照 Node 上的一个可用数量。[5][6]

这套对象不是为了让每个业务 Pod 都写复杂 YAML。DeviceClass 的价值恰恰在于把硬件细节收进平台边界：业务侧可以说“我要这一类设备”，平台侧定义这一类设备具体满足哪些属性。Kubernetes 官方文档列出的收益包括基于 CEL 的细粒度筛选、设备共享、按工作负载配置和集中分类；这些能力需要相应 DRA 驱动与配置支持，不是升级控制平面后自动获得。[5]

分配和实际可用之间仍有一个节点侧动作。DRA 驱动需要发布准确的 ResourceSlice，并与 kubelet 协作准备设备，把设备访问交给容器运行时；官方文档描述了通过 CDI 注入容器的路径。[5][6] 所以 DRA 不是“调度器自己会操作 GPU”，而是把设备信息、选择条件和准备动作放进更明确的协议里。

![同一 Pod 的两种设备分配路径](assets/gpu-allocation-flow.png)

*图 1：根据 Kubernetes Device Plugin 与 DRA 文档重绘的通用流程。两侧表示同一扩展资源请求可能对应不同分配机制；图中不表示某个现成 GPU 驱动已完成迁移。*

## 三、1.37 的桥梁：旧请求如何接入新分配

Kubernetes 1.34 的发布说明记录了 DRA 核心 API 进入 GA。由 DRA 满足传统扩展资源请求的能力在 1.37 进入 GA。管理员可以在 DeviceClass 上设置 `spec.extendedResourceName`，例如 `example.com/gpu`。旧 Pod 仍请求这个名字，调度器从符合该类条件的 DRA 设备中分配，无需工作负载显式创建 ResourceClaim。[1][2][8]

这不是说 ResourceClaim 在实现里彻底消失了。Kubernetes 文档说明，DRA 扩展资源路径会生成用于分配的 claim；“不需要 ResourceClaim”准确地说是“Pod 作者不需要自己在工作负载里声明它”。这一点很重要：面向用户的 API 可以保持简单，平台内部仍需要可追踪的设备分配状态。[2]

举个不带厂商假设的例子：原本某个 Job 模板只有 `limits: {example.com/gpu: 1}`。迁移后模板仍可保留这行；变化发生在集群侧，驱动发布 ResourceSlice，DeviceClass 用选择条件圈出符合该资源名的设备。调度器不只是看到节点上的总数，还能在 DRA 路径里找出具体匹配设备。这个例子只说明接口兼容性，不说明任何已有 GPU 插件会自动转换成 DRA 驱动。[1][2]

过渡期可以让一部分节点通过 Device Plugin 提供某个扩展资源名，另一部分节点通过 DRA 提供同名资源。[2] 逐节点迁移时，要核对资源名、设备身份和配额，并明确每块物理设备由哪条路径提供，避免重复计入容量。

API 参考还提醒，`extendedResourceName` 应在 DeviceClass 之间保持唯一。若多个 DeviceClass 意外使用同名，系统有选择规则，但这不等于应该依赖这种规则做业务路由。[7] 对平台维护者来说，资源名是兼容契约，不只是一个字符串；更换背后的提供者之前，应确认用户原来以为拿到的设备能力是否仍然成立。

## 四、不改 Pod YAML，究竟保留了什么、丢了什么

保留的是旧工作负载入口：应用仍可用整数请求某类资源，不必让所有 Deployment、Job 或训练 Operator 同一天改成 ResourceClaim。对已有大量 Helm Chart、模板和自动生成 Pod 的平台，这比一次性改 API 更现实。DRA 驱动和 DeviceClass 可以先在少量节点试点，验证分配正确性，再逐步扩大范围。[1][2]

保留的还包括旧接口的一部分限制：扩展资源请求仍按数量表达，不会因为背后改由 DRA 分配，就自动获得“按显存容量请求 0.5 张 GPU”这样的语义。DRA 其他功能可能支持设备共享或可消费容量，但能否用于某个硬件、以什么粒度提供，取决于驱动、DeviceClass 和具体功能状态。不要把 DRA 的能力清单误读为一个旧 `gpu: 1` Pod 已经拥有的能力。[5]

但整数请求仍然只是整数请求。如果业务真正需要“至少 80 GB 显存”“两张卡彼此高速互连”“可切分但不能与别的租户共享”，旧 Pod 语法无法把这些约束逐项说清。管理员可以用 DeviceClass 定义一个较固定的类别，可当同一类里还存在多种选择、不同任务要不同配置时，显式 ResourceClaim 才能表达更丰富的选择与配置。把 DRA 接在旧资源名后面，并不会自动让旧 YAML 变成完整的硬件意图。[5][6]

也不要把“设备分配成功”当作应用性能保证。DRA 可以让调度器看到更细的属性，驱动可以准备设备，但 GPU 训练或推理是否达到目标，仍受驱动、容器运行时、NUMA、PCIe/NVLink、批量与模型配置影响。对 AI Infra 来说，DRA 改善的是设备请求和分配的表达力，性能仍要通过目标负载压测证明。

最后还有一个容易被跳过的调度边界：Kubernetes 官方 DRA 概览指出，DRA 资源当前不支持通过优先级抢占正在使用它的 Pod；高优先级 Pod 可能继续 Pending，直到设备释放。[5] 因此，训练队列、交互式推理和在线服务共用设备池时，不能只把旧资源名迁过来，还要检查原先依赖的优先级、排队和抢占预期。

## 五、平台迁移应该按什么顺序验收

第一步是盘点现状：所有扩展资源名、Device Plugin、运行时注入方式、节点标签、NUMA/拓扑约束、配额和监控。第二步定义设备语义：一个资源名到底代表整卡、分区、共享份额还是某种能力类。第三步才决定入口：只需要兼容旧 Pod，就用 DeviceClass 的 `extendedResourceName`；需要按工作负载表达复杂约束，就引入显式 ResourceClaim。[2][5]

第四步检查驱动闭环。ResourceSlice 是否准确发布每个设备和可访问节点？设备健康变化是否能及时反映？调度器选中的设备，kubelet 和 DRA 驱动是否真正准备了同一设备？容器内看到的设备、驱动版本、权限和 CDI 注入是否正确？这些问题比“Pod 已经 Running”更接近迁移成败。[5][6]

第五步做同负载 A/B：相同模型、相同请求或训练任务，在旧 Device Plugin 与 DRA 路径下分别记录 Pending 时间、分配失败、设备 ID/拓扑、容器可见性和应用性能。若某个环节变化，先区分是资源调度语义变了，还是节点运行时配置变了。不要拿只跑通一次样例 Pod 当作生产验收。

迁移测试还应覆盖不那么顺利的路径：设备在分配前变为不健康、驱动进程重启、Pod 删除后设备清理、同资源名在新旧节点间调度，以及配额接近上限时的行为。正向样例证明功能存在，异常路径才证明平台知道如何恢复。监控至少要能把 Pod、ResourceClaim、ResourceSlice、DeviceClass、节点和设备标识串起来，否则出错时仍会回到“只知道某个 Pod Pending”的老问题。

![DRA 迁移检查流程](assets/dra-migration-check.png)

*图 2：原创迁移检查清单，不含实测数据。每一步是平台验收建议，并非 Kubernetes 自动执行的迁移流程。*

第六步保留回滚与配额观测。旧资源名可以保留，底层提供方式却可能按节点逐步变化；平台需要知道某个 Pod 最终走的是哪条路径、用了哪块设备、失败发生在调度还是准备阶段。Kubernetes 1.37 给的是渐进迁移能力，真正的安全感来自清晰的资源归属、故障可见性和可恢复的发布节奏。

## 结论

Kubernetes 1.37 的 DRA 扩展资源 GA，让“Pod 不改 YAML、底层逐步迁移”成为现实选项。它不会自动安装 GPU 驱动，也不会自动理解显存、互连、共享或性能目标。Device Plugin 解决了设备如何作为整数资源被请求；DRA 让设备属性、类别与分配状态有了更结构化的表达；扩展资源桥梁则把两代工作负载接口接在一起。对平台来说，下一步不是立刻重写所有 Pod，而是先把资源名背后的设备语义和验收证据做实。

## 资料来源

[1] [Kubernetes v1.37 DRA 更新](https://kubernetes.io/blog/2026/09/03/kubernetes-v1-37-dra-updates/)

[2] [Kubernetes 文档 · 由 DRA 分配扩展资源](https://kubernetes.io/docs/concepts/resource-management/dynamic-resource-allocation/dra-api/#extended-resource-allocation-by-dra)

[3] [Kubernetes 文档 · Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)

[4] [Kubernetes 文档 · GPU 调度](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)

[5] [Kubernetes 文档 · Dynamic Resource Allocation](https://kubernetes.io/docs/concepts/resource-management/dynamic-resource-allocation/)

[6] [Kubernetes 文档 · DRA 如何工作](https://kubernetes.io/docs/concepts/resource-management/dynamic-resource-allocation/how-dra-works/)

[7] [Kubernetes API 参考 · DeviceClass](https://kubernetes.io/docs/reference/kubernetes-api/resource/device-class-v1/)

[8] [Kubernetes v1.34 DRA 更新](https://kubernetes.io/blog/2025/09/01/kubernetes-v1-34-dra-updates/)
