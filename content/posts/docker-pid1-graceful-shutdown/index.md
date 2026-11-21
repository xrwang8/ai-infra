---
title: "docker stop 了，服务为什么没走完自己的退出逻辑？"
date: "2026-11-21"
description: "顺着 Docker PID 1、shell 包装、停止信号与请求排空设计一次可复核的关停演练。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "PID 1", "优雅退出"]
---

发布时容器确实退出了，最后一批流式请求却断在中途。与其先把宽限期加长，我会沿着应用日志对齐四个时刻：收到停止信号、停止接新流量、存量请求完成、主进程退出。只看到容器消失，四个环节里究竟走完了哪些还不清楚。

## 找到第一位接收信号的进程

Docker 通常先向容器主进程发送 SIGTERM，等待指定时间，再用 SIGKILL 结束仍在运行的容器。镜像的 `STOPSIGNAL` 和运行设置可以指定不同信号。用 shell 形式写入口时，主进程经常是 `/bin/sh -c`。应用在子进程里时，信号怎么转发要检查。 [Docker stop](https://docs.docker.com/reference/cli/docker/container/stop/) · [Dockerfile ENTRYPOINT](https://docs.docker.com/reference/dockerfile/#entrypoint)

直接启动应用时可以用 exec 形式。以下路径和参数是服务配置示例。

```dockerfile
ENTRYPOINT ["/opt/service/worker"]
CMD ["--grace-period=15s"]
```

必须先执行初始化脚本时，通常在最后使用 `exec /opt/service/worker "$@"`，让应用替代脚本进程。应用还要自己处理收到的信号。额外定义 `--grace-period` 前应核对服务真的支持这个参数。

## 有子进程时还有两层责任

用 `--init` 或一个小型 init 处理信号转发和僵尸进程回收，可以省去自己写一部分生命周期逻辑。已接纳请求如何排空、检查点是否写完、超时怎么结束，仍由应用决定。 [Docker 容器内多进程](https://docs.docker.com/engine/containers/multi-service_container/)

验收可以固定一个长流。先让它开始交付，再发停止动作，同时发一条新请求检查服务是否停止接纳。收集请求完整结束标记、终止信号、等待时间和退出码。若所有活动最终都是被强制杀掉，只有宽限期或信号配置还不能说明根因。把这一步接到滚动发布验收，再比较终止前后的可用容量。 [GPU 服务滚动发布与排空]({{< relref "posts/gpu-serving-rollout/index.md" >}})

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
