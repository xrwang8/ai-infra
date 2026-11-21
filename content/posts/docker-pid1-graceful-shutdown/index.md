---
title: "docker stop 了，服务为什么没走完自己的退出逻辑？"
date: "2026-11-21"
description: "顺着 Docker PID 1、shell 包装、停止信号与请求排空设计一次可复核的关停演练。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "PID 1", "优雅退出"]
---

容器通常用主进程代表自己的生命周期。主进程退出，容器就会随之结束。它还处在容器 PID namespace 的特殊位置，涉及信号处理和子进程回收。理解这一点后，Dockerfile 用哪种入口形式，以及如何安排上线时的优雅终止，就可以放在同一条链路上看。

## 先确定谁是 PID 1 {#找到第一位接收信号的进程}

主进程由镜像入口和运行参数共同决定。exec 形式可以直接让应用作为入口。shell 形式会经 shell 执行命令，应用可能成为它的子进程。真正收到信号的是哪个进程，要看使用的入口形式、包装脚本和进程树。参数也有边界。exec 形式的 `ENTRYPOINT` 定义可执行程序，`CMD` 提供默认参数，启动时传入的参数可以覆盖该默认值。 [Dockerfile ENTRYPOINT](https://docs.docker.com/reference/dockerfile/#entrypoint) · [Linux PID namespace](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html)

```dockerfile
ENTRYPOINT ["/opt/service/worker"]
CMD ["--grace-period=15s"]
```

这里的宽限参数只是应用接口示例。若必须先运行初始化脚本，常用 `exec /opt/service/worker "$@"` 替换脚本进程。脚本要留下来管理子进程时，就要处理转发、等待和退出状态。也可使用 `--init` 为容器加一个轻量 init，承担信号转发和回收部分子进程的职责。 [Docker 多进程容器](https://docs.docker.com/engine/containers/multi-service_container/)

## 回收子进程和处理请求是两项职责

子进程退出后，内核保留它的一部分退出状态，直到负责的进程执行等待调用收取它。长期不回收，僵尸进程可能积累。父进程先结束时，还涉及孤儿进程由谁接管。容器内的 PID 1 往往承担这一组进程中的重要管理责任。应用只关心网络事件时，不一定适合顺便做这些事。需要多进程时，可将 init 职责独立出来，再由应用自己实现业务终止顺序。 [Linux PID namespaces](https://man7.org/linux/man-pages/man7/pid_namespaces.7.html) · [Docker 的小型 init](https://docs.docker.com/engine/containers/multi-service_container/)

转发一个 SIGTERM 并不说明应用已经开始排空。应用可能没有注册处理函数，也可能把停止请求交给其他工作线程。检查时要把应用承诺的结束条件写清楚。比如流式服务在自己负责的这一层，需要等最后一个有效事件送出并结束流，然后才释放请求状态。只让 init 延长等待，并不能覆盖业务语义。

## 停止有两个不同的时限

Docker 通常先发 SIGTERM。具体信号可由 `STOPSIGNAL` 或运行设置影响。等配置的时间后仍未退出，Docker 可以用 SIGKILL 终止容器。应用可能也有自己的排空时长，两者要对齐。退出时间由容器层控制，是否完整处理应用工作由服务负责。 [Docker stop](https://docs.docker.com/reference/cli/docker/container/stop/)

例如允许容器等待 20 秒，而应用最慢的一批活动请求需要 35 秒。如果应用继续接活或一直等这些请求，又没有独立的退出策略，20 秒时就可能被强制结束。设计上可以把“停止接新流量、等待已接纳工作、到期取消、退出”写成明确顺序。宽限期至少覆盖计划允许完成的过程，剩余边界由业务的超时和重试策略定义。

## 一次发布怎么验收这条链路 {#有子进程时还有两层责任}

让一条长流开始输出，再触发停止。记录路由摘除、应用收到信号、旧请求结束、新请求是否还被接受、最后的退出状态。可以再对缩短过的预算测一次，以确认到期策略真实发生。进程退出后，客户端读到一段文本不等于收到了完整响应。验收应包括协议的结束信号。部署到 Kubernetes 时还应核对 Pod 终止期、退出钩子和入口流量的收敛。 [GPU 滚动发布与排空]({{< relref "posts/gpu-serving-rollout/index.md" >}})

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
