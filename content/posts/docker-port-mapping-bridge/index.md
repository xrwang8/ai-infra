---
title: "容器 80 端口有返回，为什么宿主访问还是失败？"
date: "2026-11-21"
description: "按进程监听、Docker bridge、端口发布和宿主机入口四处定位容器网络问题。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "Linux 网络", "端口映射"]
---

Docker 容器内的 `80/tcp`、宿主机上的 `8080/tcp` 和其他容器用来访问它的服务名，可以同时指向同一个应用。把它们连起来之前，得分清**进程监听、网络转发和端口发布**。只有进程真的监听了目标容器地址，转发过去的流量才有地方接收。

## 容器网络里的中间一段 {#先从离进程最近的地方查}

在常见 Linux bridge 配置里，容器有自己的 network namespace。Docker 通常用一对 veth，把一端放进容器，另一端接到宿主机上的 Linux 网桥。veth 的两端相连，本机网桥在连接的端口之间转发。同一自定义 bridge 上的容器也可通过 Docker 提供的名字解析发现对方，应用连接的是目标容器地址和应用端口。 [Docker Bridge 驱动](https://docs.docker.com/engine/network/drivers/bridge/)

对发往外部网络的数据，常见默认 bridge 配置会使用 masquerading，改写源地址，让外部看到流量来自宿主。对外部发来并指向宿主已发布端口的数据，常见的 NAT 路径会转发到容器地址及端口。这两个方向处理的字段不同。Docker 的具体防火墙后端、网桥模式和桌面虚拟机，可能改变你该在哪里查看规则。 [Docker 防火墙与数据包过滤](https://docs.docker.com/engine/network/packet-filtering-firewalls/)

## 为什么网络模式会改变你能看见的东西

Docker 的自定义 bridge 给同一主机的一组容器准备一个独立网络。它们可以用服务别名直接交互，访问路径不必绕一圈宿主发布端口。host 模式在常见 Linux 配置中使用宿主网络栈，用了端口就和宿主其他进程共享这一组绑定。none 模式不给容器准备可用的外部连通路径。还可以让两个容器共享其中一个的网络视图。多宿主机方案则要再理解底层转发或隧道。前两篇讲过的 namespace，在这里直接变成通信边界。 [Docker 网络模式](https://docs.docker.com/engine/network/) · [Docker Host 网络](https://docs.docker.com/engine/network/drivers/host/)

可以读两个地址区分问题。容器访问同网络的同伴时，通常访问对方的容器地址和应用端口，例如在业务网络中访问 `api:80`。宿主侧测试发布地址则可能用 `127.0.0.1:8080`，发布规则会映射到后端的容器端口。若换成另一台客户端，那个回环地址就属于另一台客户端了。连通范围取决于实际绑定和路由。两个入口最终能访问同一应用，也可以因为监听或传播范围不同而只有一个入口成功。

## EXPOSE、监听与宿主发布

| 做的事 | 影响谁 | 用什么确认 |
|---|---|---|
| 应用监听 `0.0.0.0:80` 或具体容器接口 | 能接受哪个目标地址及端口的连接 | 容器内的监听信息和请求 |
| Dockerfile `EXPOSE 80` | 镜像对外说明应用预期使用的端口 | 镜像配置 |
| `docker run -p 127.0.0.1:8080:80` | 宿主绑定的地址及端口与容器端口的映射 | `docker port` |
| `docker run -P` | 将已声明端口发布到宿主动态分配的端口 | 每个端口的实际映射 |

应用只监听容器内的回环地址时，发往容器接口的请求通常接不到。容器内的 `127.0.0.1` 和宿主侧的 `127.0.0.1` 属于不同视图。`EXPOSE` 本身也不会改变应用监听或宿主映射。发布时如果省略绑定地址，需要按 Engine 的默认行为和实际网络配置确认暴露范围。 [Dockerfile EXPOSE](https://docs.docker.com/reference/dockerfile/#expose) · [端口发布](https://docs.docker.com/engine/network/port-publishing/)

## 一次只验证宿主到容器的这一段 {#从随机宿主端口读一次真实映射}

在本地 Engine 运行一个临时 NGINX，给宿主选择动态端口。脚本会读出分配结果并请求对应地址。

```sh
demo=$(docker run -d --rm -p 127.0.0.1::80 nginx:alpine) || exit
trap 'docker stop -t 1 "$demo" >/dev/null 2>&1 || true' EXIT
endpoint=$(docker port "$demo" 80/tcp)
printf 'host endpoint = %s\n' "$endpoint"
curl --fail --silent --show-error --retry 5 --retry-connrefused "http://$endpoint/"
```

能成功取到页面时，至少说明这次宿主地址、发布规则和进程监听连通。从另一台机器访问时，还要对客户端路由和宿主绑定地址。跨节点直连容器或 Pod IP 又涉及另一条路径，不能用本机映射的成功证明。 [跨节点 VXLAN]({{< relref "posts/docker-vxlan-cross-node-troubleshooting/index.md" >}})

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
