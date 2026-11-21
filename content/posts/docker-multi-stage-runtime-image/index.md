---
title: "多阶段构建打包成功了，为什么最终镜像起不来？"
date: "2026-11-21"
description: "用可交叉编译的 Go 健康检查服务解释多阶段 Dockerfile 的产物交接与最小运行时验收。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "Dockerfile", "Go", "镜像构建"]
---

构建镜像时每一步都绿了，容器一启动却报找不到程序。先确认报错来自最终镜像，并在最终环境里核对二进制、架构、动态库和运行用户。构建阶段有的文件，只有通过显式复制或继承才会进入最终阶段。

## 做一个依赖够少的例子

下面的 Go 程序只监听容器的非特权端口 8080。两个文件放在同一个构建上下文内。

```go
package main

import (
    "fmt"
    "log"
    "net/http"
)

func main() {
    mux := http.NewServeMux()
    mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "ready")
    })
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

```dockerfile
FROM --platform=$BUILDPLATFORM golang:1.27 AS compile
ARG TARGETOS TARGETARCH
WORKDIR /src
COPY main.go ./
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -trimpath -o /out/healthd main.go

FROM scratch
COPY --from=compile /out/healthd /healthd
USER 65532:65532
EXPOSE 8080
ENTRYPOINT ["/healthd"]
```

构建阶段用构建机的平台运行编译器，通过 BuildKit 的目标参数生成对应的二进制。交接到最终阶段的只有 `healthd`。这份仅提供 HTTP 接口的程序适合用 `scratch` 展示最小运行时。可用 `docker buildx build --platform linux/arm64 --load -t healthd:local .` 构建单平台镜像，再用支持该平台的 Engine 启动和请求 `/health`。 [Docker 多阶段构建](https://docs.docker.com/build/building/multi-stage/) · [Docker 多平台构建](https://docs.docker.com/build/building/multi-platform/)

## 发布前对一次最终阶段

有动态库或 CGO 时先检查 ELF 解释器与依赖。要出站访问 HTTPS 时确认 CA 证书和 TLS 行为。需要写模型缓存时核对路径和运行用户的权限。推理服务还要按 CUDA 或 CANN 的运行时依赖验收，不能仅根据镜像变小判断能否运行。

比较镜像大小时，用相同平台和量纲，分别记录镜像层、传输体积和冷启动耗时。这里的小例子只说明构建产物怎么进入目标镜像。生产的基础镜像可以固定 digest，并按安全更新节奏主动刷新。

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
