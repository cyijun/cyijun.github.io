---
title: "两台 DGX Spark 跑一个模型：TP=2 双机推理架构与启动顺序"
date: 2026-08-20 00:45:00 +0800
categories: [大模型推理, 系统工程]
tags: [DGX-Spark, vLLM, Tensor-Parallel, NCCL, ARM64]
description: "在两台 ARM64 DGX Spark 上用 vLLM 原生多节点执行器部署单个 TP=2 模型实例。"
---

两台机器各启动一个推理服务很容易，但那只是两个模型副本。我要验证的是另一种拓扑：同一个模型实例拆成两个 Tensor Parallel rank，每一步推理都由两台 DGX Spark 共同完成。

我最近把这次实验整理成 `dgx-spark-dual-node-inference`。仓库不是一条“神奇启动命令”，而是一组会在启动前检查镜像、模型、网络、设备、端口和安全边界的脚本。以下记录基于 2026 年 8 月 19 日的真实双机验收，不代表厂商基准，也不保证其他 nightly 版本得到相同结果。

## 拓扑：一个 API，两个 rank

Head 节点同时运行 OpenAI-compatible API、EngineCore 和 TP rank 0；Worker 节点以 headless 模式运行 TP rank 1，不提供 HTTP API。两端通过 `master-addr` 与 `master-port` 建立进程组，collective 通信走专用 RoCEv2 fabric。

```text
Head                                 Worker
API + EngineCore                     headless worker
TP rank 0 / cuda:0  <--- NCCL --->   TP rank 1 / cuda:0
127.0.0.1:8888                       no HTTP API
```

管理流量与数据流量需要分开理解。SSH、Docker 控制和日志读取可以走普通管理网络；`VLLM_HOST_IP`、Gloo、NCCL socket 与 IB HCA 则显式绑定 RoCE 网络。容器使用 host network，避免 bridge 地址被误选为跨节点路径。

这不是把两台机器的内存拼成一个透明池。每个 rank 都有自己的权重分片、KV cache 和运行时开销，容量规划必须逐节点检查。

## 两节点为什么不引入 Ray

本次拓扑固定为两节点、每节点一张 GPU、TP=2、PP=1。vLLM 原生多节点 multiprocessing 执行器已经能直接表达，因此没有额外启动 Ray head、worker 或 dashboard。

减少控制面组件意味着更少的端口、版本组合和故障面。它不表示 Ray 永远不合适；需要多副本调度、异构角色或上层框架只提供 Ray backend 时，仍然应该采用相应设计。选择执行器要从拓扑出发，而不是从流行度出发。

## 启动顺序本身就是协议

多节点程序不能随意并行 `docker run`。仓库采用明确顺序：

1. Head 本地执行 preflight，并通过 SSH 检查 Worker；
2. 两节点分别探测当前 RoCEv2 GID index；
3. 先启动 Worker rank 1，让它等待 rendezvous；
4. 再启动 Head rank 0 与 API；
5. 同时观察两个容器状态和 `/health`；
6. 健康后检查 `/v1/models`，最后发送真实生成请求。

`wait-ready.sh` 不只是循环 curl。如果任一侧容器提前退出，它会收集两端日志，让第一次失败保留足够证据。仅有 `/health` 也不算完整验收：服务可能活着，但模型名、生成路径或跨节点 collective 仍有问题，所以必须执行实际 Chat Completions smoke test。

## Fail closed 的启动前检查

双机部署最浪费时间的情况，是加载几十 GB 内容后才发现两边不一致。preflight 因此提前检查：

- 两端 ARM64 架构、GPU 与 RDMA 设备；
- Docker/NVIDIA runtime 和端口占用；
- 同名镜像是否对应相同完整 image ID；
- 模型目录、revision、断链和关键文件；
- RoCE link 是否 `ACTIVE / LINK_UP`；
- 网卡、HCA 与动态 GID 是否匹配；
- API 绑定与鉴权是否符合安全策略；
- 是否已有同名容器，避免误覆盖其他任务。

任何关键条件不满足都停止，而不是带着警告继续加载。可复现环境里，镜像 tag 和模型路径都只是人类可读标签，完整 image ID、revision 与逐文件 checksum 才是机器能验证的身份。

## 缓存、容器和安全边界

模型分别保存在两节点本地 NVMe，不依赖共享文件系统。如果只有 Head 已下载，脚本使用可续传同步，再在两端逐文件校验。容器以只读方式挂载模型目录，并启用 Hugging Face/Transformers 离线模式，防止启动期间拉到变化的远端 revision。

API 默认只绑定 Head 的 `127.0.0.1:8888`。远端访问优先使用 SSH tunnel；若主动绑定局域网地址，必须同时显式允许远程 API 并设置足够长度的 key。部署脚本还只停止自己创建的两个容器，不把“清理环境”理解成删除所有相似工作负载。

这套模板的范围很清楚：两节点、单 GPU/节点、TP=2、vLLM。它不是通用集群编排器，也不是生产 SLA。它的价值是把一次真实部署中最容易漂移的条件变成可重复执行的检查，让下一次失败发生得更早、更清楚。

项目地址：[cyijun/dgx-spark-dual-node-inference](https://github.com/cyijun/dgx-spark-dual-node-inference)
