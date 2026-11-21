---
title: "容器 80 端口有返回，为什么宿主访问还是失败？"
date: "2026-11-21"
description: "按进程监听、Docker bridge、端口发布和宿主机入口四处定位容器网络问题。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "Linux 网络", "端口映射"]
images: ["images/cover.png"]
---

应用日志显示已经启动，容器里请求 80 端口也能返回，笔记本上却访问失败。排查时把三件事分别记下：进程监听在哪个地址、Docker 发布到哪个宿主地址、客户端打的是哪台机器。

![宿主入口经网桥通往独立服务舱的原创概念插画](images/cover.png)

## 先从离进程最近的地方查

1. 进容器检查有效监听地址。只监听容器内的 `127.0.0.1` 时，外部转发进来的流量通常会打到容器接口的地址。应用需要在对应接口或通配地址监听。
2. 用 `docker port <容器> 80/tcp` 核对发布结果。没有发布到宿主机时，同一网络里的容器仍可能访问它。
3. 从宿主侧先访问实际发布地址，再去查远程路由、防火墙或反代。Linux bridge、Docker Desktop 虚拟机和 host 网络的观察点各有区别。

Dockerfile 的 `EXPOSE` 描述预期端口。`-p` 配置显式映射，`-P` 会把已声明端口映射到动态宿主端口。在用户自定义的 Docker bridge 上，相同网络的服务还可以用容器名等 DNS 别名沟通。 [Docker 端口发布](https://docs.docker.com/engine/network/port-publishing/) · [Bridge 驱动](https://docs.docker.com/engine/network/drivers/bridge/)

## 从随机宿主端口读一次真实映射

在本地 Docker Engine 做这个例子，可以直接读取分配到的端口。容器使用默认 NGINX 配置，测试入口指定为宿主回环地址。

```sh
demo=$(docker run -d --rm -p 127.0.0.1::80 nginx:alpine)
trap 'docker stop -t 1 "$demo" >/dev/null 2>&1 || true' EXIT
endpoint=$(docker port "$demo" 80/tcp)
printf 'host endpoint = %s\n' "$endpoint"
curl --fail --silent --show-error --retry 5 --retry-connrefused "http://$endpoint/"
```

这份配方用来验证本机路由。排查两台 Linux 节点时，先写明是从宿主端口发布进容器，还是跨节点直连容器 IP。前者重点查发布规则，后者要继续查底层路由或覆盖网络。 [跨节点 Overlay 网络笔记]({{< relref "posts/docker-vxlan-cross-node-troubleshooting/index.md" >}})

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
