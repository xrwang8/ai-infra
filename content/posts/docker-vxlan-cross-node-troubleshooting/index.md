---
title: "同节点容器能通，跨节点 Pod 为什么超时？"
date: "2026-11-21"
description: "从 Pod IP、隧道外层和底层节点路由排查 VXLAN 跨节点连通性，并区分 Docker overlay 与 Kubernetes CNI。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "VXLAN", "Flannel", "Kubernetes 网络"]
---

两台节点上的服务单独看都正常。同节点 Pod 之间能连，跨节点访问却一直等。这样的现象值得先把 Service 解析和底层网络分开。找两组同节点、跨节点的直接 Pod IP 流量，再记录各自有没有到达目标。

## 沿两个地址查一条通信

以使用 Flannel VXLAN 后端的 Linux 集群为例，跨节点路径同时涉及内层 Pod 地址和外层节点地址。本机根据远端网段选择隧道，把内层包封装到节点之间可路由的 UDP 流量里。对端收到后解封装，再送往目标工作负载。节点能互相 ping，只检查了部分底层连通性。 [Flannel 后端说明](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md)

| 观察到的结果 | 下一处检查 |
|---|---|
| 同节点与跨节点直连都失败 | 先查应用监听、目标地址和策略 |
| 同节点正常，跨节点发端没有看到外层报文 | 查到远端 Pod IP 的有效路由及后端配置 |
| 发端已有外层报文，对端没有观察到 | 用同一时间窗口查节点路由、防火墙及底层链路 |
| 对端已收到封装，目标进程仍无流量 | 查隧道解封装和目标 Pod 的本地转发路径 |
| 短请求正常，大包或长连接出问题 | 检查链路 MTU、封装开销和方向，避免只做一种包长 |

查验时需要实际的双节点网络环境。用 `ip route get <远端PodIP>` 读出有效路径，再按插件配置确认隧道接口、远端地址与 UDP 端口。具体名称和端口随环境变化。变更 MTU 或防火墙前先保留双方的证据。

Docker overlay 同样解决多宿主机容器互联，但它需要 Docker Swarm 的管理状态。Kubernetes 的 Pod 网络要核对当前 CNI 插件及对应后端。不要直接把一种部署的名称和端口套到另一种。 [Docker overlay 驱动](https://docs.docker.com/engine/network/drivers/overlay/)

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
