---
title: "多阶段构建打包成功了，为什么最终镜像起不来？"
date: "2026-11-21"
description: "用可交叉编译的 Go 健康检查服务解释多阶段 Dockerfile 的产物交接与最小运行时验收。"
categories: ["Docker 工程笔记"]
tags: ["Docker", "Dockerfile", "Go", "镜像构建"]
---

构建机里有编译器、头文件、测试工具和依赖缓存。线上运行时只需要能够启动和提供服务的内容。Docker 多阶段构建让这两套环境可以独立描述，并通过一个明确的路径交接产物。学会这个机制后，也就能解释构建成功而运行失败时，中间少了哪项约定。

## 每个 FROM 开一个阶段 {#做一个依赖够少的例子}

阶段可以继承一个已有镜像或依赖别的阶段。通过 `COPY --from` ，最终阶段只取需要的产物。工具链留在哪个阶段，取决于 Dockerfile 的阶段关系。若编译依赖动态链接库，交接也要包括与二进制相容的运行库和加载器。 [Docker 多阶段构建](https://docs.docker.com/build/building/multi-stage/)

下面使用一个无需第三方模块的 Go 服务说明交接。两个文件位于同一个构建上下文。

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

第一个平台是执行编译器的平台。BuildKit 的 `TARGETOS` 和 `TARGETARCH` 则是产物要运行的平台，需要在该阶段声明后才能给 `RUN` 使用。编译生成 `/out/healthd`，最终阶段在自己的根目录生成 `/healthd`。数值 UID 和 GID 可以在没有用户数据库文件的镜像中提供身份。这里使用纯 Go，并选了非特权监听端口。 [Docker 多平台构建](https://docs.docker.com/build/building/multi-platform/)

## 交接除了二进制，还要考虑哪些运行条件 {#发布前对一次最终阶段}

| 需求 | 最终阶段要留下什么 |
|---|---|
| 启动动态链接程序 | 对应架构的解释器和依赖库 |
| 向外部 HTTPS 服务发请求 | 可用的证书信任链与对应配置 |
| 按用户名查询身份 | 所需用户数据库或相应应用处理 |
| 写缓存或日志文件 | 明确的目录、权限与存储来源 |
| 使用 GPU/NPU | 应用依赖、兼容的库以及节点运行时实际提供的设备环境 |

可以尝试用 `docker buildx build --platform linux/arm64 --load -t healthd:local .` 只构建一个目标，再到支持目标平台的 Engine 上验证 `/health`。如果最终镜像无法启动，先区分架构错误、入口文件缺失、动态加载器缺失和权限错误。这个简单示例的最终阶段没有 shell，调试方式也需要提前考虑。

## 两种找不到文件为什么表现相似

第一种是入口路径真的没有文件。可能是编译输出写到一个位置，最终阶段复制的是另一个。第二种是 ELF 文件已经在，但它引用的动态加载器不在。Linux 在加载时也可能给出类似找不到文件的错误。这时先从构建侧提取最终二进制，读它的架构、是否动态链接和解释器路径。按目标平台检查最终阶段是否具备该路径。类型和路径核对完，再到对应设备启动。一个 amd64 二进制也不能直接交给不具备相应支持的 arm64 环境运行。

对比两个常用取舍。全静态、依赖很少的服务，可以从很小的根文件系统开始。需要 Python、CUDA、动态库和 TLS 的服务，则适合用受维护的兼容基础镜像，清楚知道哪些包按约定提供。两类最终镜像都可以通过多阶段构建，只是交接物料不同。对该 Go 例，`CGO_ENABLED=0` 足以让这里的标准库实现生成静态程序，其他应用要核对源码依赖与构建实际结果。

## 镜像小了以后验收什么

目标是把全部运行依赖说清楚。开发环境可以保留一个调试阶段，生产阶段按交付约定构建。对推理引擎一类依赖众多的服务，宁可先核对本地与目标设备的库版本和驱动兼容，再砍工具或系统文件。测收益时分别记录制品尺寸、节点传输及解包、应用实际启动时间。尺寸变化不能替代业务就绪验证。 [Docker 构建最佳实践](https://docs.docker.com/build/building/best-practices/)

[Docker 工程笔记专题]({{< relref "docker-field-notes/index.md" >}})
