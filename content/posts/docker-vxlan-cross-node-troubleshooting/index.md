---
title: "同节点容器能通，跨节点 Pod 为什么超时？"
date: "2026-11-21"
description: "从 Pod IP、隧道外层和底层节点路由排查 VXLAN 跨节点连通性，并区分 Docker overlay 与 Kubernetes CNI。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "VXLAN", "Flannel", "Kubernetes 网络"]
---

容器的 veth 与 Linux 网桥解决了同节点转发。跨节点还需要一个方法，把目标工作负载的地址映射到下一跳节点。可以依赖实际网络直接路由 Pod 网段，也可以在节点间已有网络上再建立覆盖网络。VXLAN 是后一种做法中的一种。理解它，可以用**内层工作负载地址、外层节点地址和隧道处理方式**三项把路径串起来。

## Underlay 与 Overlay 分别转发什么 {#沿两个地址查一条通信}

Underlay 是节点之间实际能通信的网络。它负责寻找外层目标节点，可能跨交换机和路由器。Overlay 则让工作负载看到另一套逻辑网络。采用 VXLAN 时，源节点把原来的帧装到 VXLAN 封装内，通过 UDP 送往目标节点。对端解封装后，再按本地网络规则交给目标 Pod。沿途设备通常只需要能路由外层节点地址，工作负载地址由端点所在的软件处理。 [RFC 7348 VXLAN](https://www.rfc-editor.org/rfc/rfc7348) · [Flannel VXLAN 后端](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md)

在 Flannel 的典型 Linux VXLAN 配置中，节点根据远端 Pod 网段认出目标隧道。具体下一跳、VXLAN 接口、目的 UDP 端口以及是否使用直接路由，应按插件当前配置确认。如果实现开启可用时直接路由的路径，部分节点之间的流量也可能根本不走 VXLAN。Docker 的 overlay driver 则以 Swarm 为管理前提。理解两套部署的相同原理后，仍要分别核对实际配置。 [Docker Overlay 驱动](https://docs.docker.com/engine/network/drivers/overlay/) · [Flannel 后端配置](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md)

## 远端节点怎样知道该往哪里送

覆盖网络除了封装格式，还要分发端点和网段信息。源节点首先要知道某个远端 Pod IP 属于哪段网络，以及对应哪个远端节点。接收端也要有将解封装后的流量送往目标接口的路径。VXLAN 的 VNI 用于标识逻辑网络，数据包的外层 UDP 端口和路由属于底层通信。看到一个 UDP 包到达，仍需确认走的是正确的逻辑网络、并且解封装后的本地路径有效。 [RFC 7348](https://www.rfc-editor.org/rfc/rfc7348)

比较两种路线，可将路径拆为学习问题。如果节点之间已经明确路由每个 Pod 网段，可以在支持的部署中直接转发，省掉封装开销，但底层路由和变更需要能承载这些网段。使用覆盖时，底层主要处理节点地址，网段信息和隧道端点由覆盖网络维护。这样把一部分复杂度挪到了端点，在决定方案时应同时看网络可控性、流量特征和可观测性。 [Flannel 后端取舍](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md)

## 封装如何影响 MTU

以不含额外 VLAN 或其他隧道的常见 IPv4 VXLAN 为算例，外层 IPv4、UDP、VXLAN 和内层以太网头部合计约 50 字节。若 underlay 的三层 MTU 为 1500，则这组假设下，内层 IP 包的预算约为 1450。实际还会受到外层 IPv6、实际路径 MTU、额外封装等条件影响，应测路径并看接口配置，不能把算例直接当集群参数。

短 HTTP 请求正常而大数据传输失败时，可以优先比较不同包长、不同方向的行为。成功发一条小请求只证明这一种数据能到达。连接已经建立后，如果较大的数据在路径中被丢弃，应用也可能表现为卡在读取或上传。此时还需结合同步采样判断原因，而不是立刻修改整个集群的 MTU。

## 一条请求的证据可以放在哪些位置

| 观察点 | 关注什么 |
|---|---|
| 发端 Pod | 实际目的 IP、应用请求与返回 |
| 发端节点 | 到远端 Pod IP 的路由，是否发出了预期外层流量 |
| 接收节点 | 外层包是否到达，是否进入了接收路径 |
| 目标 Pod | 本地转发能否送达，应用有没有接收 |
| 回程 | 是否存在不对称路由、防火墙或另一方向的 MTU 问题 |

学这条路径时，可以分别比较同节点和跨节点直连 Pod IP。再把 Service 名称、反代和访问策略一层一层加回去。节点互相可达说明了 underlay 的部分状态，能不能承载目标 Pod 的双向流量还要继续验证。

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
