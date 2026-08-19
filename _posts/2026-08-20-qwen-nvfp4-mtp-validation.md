---
title: "Qwen3.8-27B NVFP4 + MTP：如何证明投机解码真的启用了"
date: 2026-08-20 00:55:00 +0800
categories: [大模型推理, 性能工程]
tags: [Qwen, NVFP4, MTP, vLLM, FlashInfer]
description: "记录双 DGX Spark 上 Qwen3.8-27B-NVFP4 的加载选择、MTP 验收证据、内存代价与三个关键故障。"
---

模型目录里存在 MTP 权重，不代表服务已经在做投机解码；启动参数里写了 `mtp`，也不代表 draft token 真正被接受。要证明功能生效，需要把模型结构、运行时解析、加载日志、请求后指标和内存变化串成证据链。

下面是我在两台 DGX Spark 上验收 `unsloth/Qwen3.8-27B-NVFP4` 的记录。环境快照日期为 2026 年 8 月 19 日：vLLM 是特定开发版本，CUDA 13，镜像以完整 image ID 锁定，拓扑为两节点 TP=2、PP=1，最大上下文 262,144 tokens。nightly tag 会移动，因此这些结论不能脱离版本快照使用。

## 让模型配置决定量化 loader

这份模型在 config 中声明 `compressed-tensors`。最初如果强制传入 `--quantization modelopt`，vLLM 会在启动阶段报告模型配置与命令行量化方法不一致。

最终选择是不显式传 `--quantization`，让 vLLM 根据模型 config 自动解析 NVFP4 权重。运行时实际选择包括 FlashInfer 全注意力、FlashInfer XQA decode、FP8 E4M3 KV cache，以及 `FlashInferCutlassNvFp4LinearKernel` 执行 NVFP4 GEMM。

这里要区分格式与执行：`compressed-tensors` 是权重格式和 loader，不等于具体 GEMM 后端。观察“模型被识别为什么”与“kernel 实际由谁执行”需要看不同日志。

## 使用明确的 MTP CLI 参数

最终关键参数包括：

```text
--distributed-executor-backend mp
--tensor-parallel-size 2
--nnodes 2
--spec-method mtp
--spec-tokens 1
--gpu-memory-utilization 0.75
--max-model-len 262144
--max-num-seqs 8
--max-num-batched-tokens 8192
```

我没有通过一段 JSON 传 speculative config。原因不是 JSON 本身有问题，而是参数经过 SSH 远程 shell 时引号容易被再次解释，Head 能解析的字符串到 Worker 可能已经失真。等价的独立 CLI 参数跨 shell 边界更稳定，也更容易在日志里审计。

模型包含一层 `Qwen3_5MTP`，因此设置 1 个 draft token。盲目增加 token 数可能复用同一层，接受率和整体速度不一定提高，必须重新基准验证。

## 四层证据证明 MTP 在工作

第一层是静态模型证据：固定 revision 下存在约 811 MiB 的 `model_mtp.safetensors`，并包含对应 `mtp.*` tensors。它只证明权重存在。

第二层是架构与配置解析。日志需要出现 `Qwen3_5MTP`、解析后的 `SpeculativeConfig`，并明确 `num_spec_tokens=1`。

第三层是加载路径。运行时应报告 drafter model 正在加载，并识别 MTP 模型与目标模型共享 embedding 和 lm_head 权重。

第四层是请求后的动态指标。一次真实生成后，Prometheus 指标中应看到 drafted、draft tokens 与 accepted tokens 的累计值发生变化。我的 96-token 功能验收样本记录了 54 个 draft token，其中 41 个被接受，约 75.9%。

这只能证明 draft/verify 路径确实执行，不能当作吞吐结论。正式性能比较还需要固定 prompt、输出长度、并发、随机参数，并区分冷启动与热运行。

## MTP 用内存换减少解码步骤的机会

启用 MTP 后，每节点模型相关内存从约 10.67 GiB 增加到 11.07 GiB。75% 内存预算下，Head 的 FP8 KV cache 从约 74.64 GiB 降到 73.97 GiB，Worker 从约 74.27 GiB 降到 73.81 GiB。

全局 KV token 容量相应从约 4,676,407 降至 4,305,153；按 262,144-token 上下文估算的理论 KV 并发从约 17.84× 降至 16.42×。这提醒我，投机解码不是免费开关：draft head、CUDA Graph 和 cache padding 都会占用空间。

而且两节点内存不是一个共享大池。每个 TP rank 的权重、cache 分片和峰值都要分别满足约束，不能用两台机器的可见内存简单相加后做容量判断。

## 错误文本不等于致命失败

排障中还有一个有趣现象：Transformers 对视频 processor 某些 docstring 的提示带有 `[ERROR]` 字样，但在该版本里不是导致服务退出的异常。真正的失败判据应该组合容器 exit code、Python traceback、`/health` 和实际请求，而不是看到日志里一个醒目的词就下结论。

反过来也一样：容器还在运行，不代表模型已经正确提供服务。必须看到解析、加载、健康检查、模型列表、真实生成和指标变化的完整链路。

当前运行时还提示 speculative decoding 下 `min_p` 与 `logit_bias` 不生效。更换 vLLM commit 后，MTP 参数、模型 registry、指标名称和兼容限制都应该重新确认。实验可复现的核心不是永久固定某条命令，而是把版本、输入、判据和观察结果一起保存。

项目地址：[cyijun/dgx-spark-dual-node-inference](https://github.com/cyijun/dgx-spark-dual-node-inference)
