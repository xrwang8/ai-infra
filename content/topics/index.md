---
title: "专题导航"
description: "智算手记技术专题索引，涵盖 Kubernetes 与 GPU 部署、推理运行时、vLLM KV Cache、压测与可观测性。"
---

从基础设施到智能应用，这里逐步整理技术文章与真实案例。

## Kubernetes 与云原生

关注容器、集群调度、网络、存储和可观测性。

- [Docker 工程笔记（13 篇）]({{< relref "docker-field-notes/index.md" >}})
- [Istio 流量管理（12 篇）]({{< relref "istio-traffic-learning/index.md" >}})
- [Kubernetes 1.37：GPU Pod 不改 YAML，也能走 DRA？]({{< relref "posts/k8s-dra-extended-resource/index.md" >}})
- [GPU 满了，模型还怎么升级：滚动发布的容量与排空边界]({{< relref "posts/gpu-serving-rollout/index.md" >}})
- [双卡 Pod 已经调度，为什么 13 分 51 秒才 Ready？]({{< relref "posts/qwen-dual-gpu-cold-start/index.md" >}})

## AI Infra

关注 GPU/NPU 资源、模型服务、推理引擎与性能优化。

### 推理运行时与设备

- [Qwen3.5 的 24 层 Gated DeltaNet，为什么核心只占约 2.2% 的 Kernel 时间？]({{< relref "posts/qwen35-gated-deltanet-prefill/index.md" >}})
- [HAMi vNPU 性能实测：从原生独占到双 Pod 共卡，性能差在哪里？]({{< relref "posts/hami-vnpu-performance/index.md" >}})
- [明明没禁用，vLLM 为什么还是关闭了 Custom AllReduce？]({{< relref "posts/vllm-custom-allreduce-selection/index.md" >}})
- [Gate/Up 合并后，TP=2 为什么不能把权重直接切成两半？]({{< relref "posts/vllm-merged-mlp-tp-layout/index.md" >}})
- [5 行补成 8 行，CUDA Graph 为什么反而更快？]({{< relref "posts/cuda-graph-padding/index.md" >}})

### KV Cache

- [TP 从 4 加到 8，KV 显存为什么可能不再减半？]({{< relref "posts/vllm-tp-kv-head-replication/index.md" >}})
- [都是 10.8 GiB，vLLM 的 KV Cache 容量为什么多了近 20 万 Token？]({{< relref "posts/vllm-kv-capacity-denominator/index.md" >}})
- [同样共享 4K 前缀，为什么 vLLM 一次命中 3,696 Token，另一次命中 0？]({{< relref "posts/vllm-prefix-checkpoint-history/index.md" >}})
- [请求结束，显存却没降：vLLM 的 KV Block 释放到了哪一层？]({{< relref "posts/vllm-kv-cache-memory/index.md" >}})
- [KV Cache 命中率 64.4%，到底测到了什么？]({{< relref "posts/inference-cache-hit-coverage/index.md" >}})

### 压测与可观测性

- [SGLang 日志 50 Token/s，压测报告为什么有 464 Token/s？]({{< relref "posts/sglang-token-throughput-measurement/index.md" >}})
- [并发越高，为什么大模型总输出吞吐可能增加？]({{< relref "posts/inference-concurrency-output-throughput/index.md" >}})
- [8 卡 H100 上压测 SGLang，为什么加并发，首 token 反而慢了]({{< relref "posts/sglang-h100-prefill-profiling/index.md" >}})
- [EvalScope 多轮压测，请求成功率 99.1%，为什么 12% 的会话仍有失败？]({{< relref "posts/evalscope-multiturn-latency-population/index.md" >}})
- [vLLM 分块 Prefill 已经采样四次，为什么只输出一个 Token？]({{< relref "posts/vllm-chunked-prefill-sampling/index.md" >}})
- [请求全部成功，停发后却还要等 86 秒]({{< relref "posts/inference-open-loop-drain/index.md" >}})
- [P99 算出 2.49 秒，但样本上限不到 1.62 秒]({{< relref "posts/vllm-histogram-quantile/index.md" >}})
- [vLLM 跑出 3753 Token/s，为什么换个分子，成绩就变了？]({{< relref "posts/vllm-throughput-goodput/index.md" >}})
- [并发从 1 到 4，为什么 vLLM 已经开始输出，还会一顿一顿地到？]({{< relref "posts/vllm-streaming-concurrency/index.md" >}})

## AI Agent

关注智能体架构、工具调用、上下文、记忆及工程实践。相关内容正在整理。

## 实践与思考

记录故障排查、性能测试、开源贡献和架构设计。具体案例的测试条件和适用范围见各篇正文。
