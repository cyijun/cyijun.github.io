---
title: "NCCL 明明能连却走错网：DGX Spark 双机 RoCEv2 排障笔记"
date: 2026-08-20 00:50:00 +0800
categories: [大模型推理, 网络]
tags: [RoCEv2, NCCL, RDMA, DGX-Spark, vLLM]
description: "区分 NIC 与 HCA、动态探测 GID index，并验证 NCCL collective 确实走 RoCE NET/IB。"
---

多节点推理最迷惑的故障，不一定是“连不上”。有时进程组能够建立，模型也能加载，但 collective 悄悄走了管理网、Docker bridge 或 socket transport。系统表面可用，吞吐和稳定性却完全不是预期。

在两台 DGX Spark 的 TP=2 部署中，我最终使用单条 200 Gb/s RoCEv2 fabric，并要求 NCCL 日志确认 `NET/IB`。这篇只记录网络层最容易混淆的几个点。

## Linux 网卡与 RDMA HCA 是两个名字

同一条物理链路会暴露两套名称。例如 Linux 网络接口可能叫 `enp1s0f1np1`，负责 IP、Gloo 和 NCCL socket bootstrap；RDMA HCA 可能叫 `rocep1s0f1`，供 NCCL IB transport 使用。

它们看起来相似，但不能互换，而且区分大小写。不要根据名字猜映射，应同时检查：

```bash
ip -br addr
rdma link show
ls /sys/class/infiniband
```

配置中也要分别保留 `FABRIC_NIC` 与 `FABRIC_HCA`。把 NIC 名填进 `NCCL_IB_HCA`，或者把 HCA 名当成 `GLOO_SOCKET_IFNAME`，都可能让 NCCL 回退到意外路径。

## GID index 不能作为永久常量

RoCEv2 的 GID table 会受到启动顺序、链路事件和地址配置影响。即使这次两台机器都使用 index 3，重启或重新配置网络后也不保证不变。

部署脚本在每个节点独立扫描 GID table，并同时匹配三个条件：类型为 `RoCE v2`、关联 ndev 等于目标 fabric NIC、地址为 IPv4-mapped 形式。探测结果再分别注入本节点的 `NCCL_IB_GID_INDEX`。

“分别”很关键。Head 与 Worker 的表结构可能不同，不能在 Head 探测一次后把同一数字复制给两边。动态发现也不是放弃可复现；恰恰相反，可复现的是选择规则，而不是某次机器状态产生的偶然编号。

## 所有数据路径变量要指向同一条 fabric

单 rail 起步时，我把以下变量同时固定到专用网络：

```text
VLLM_HOST_IP=<本节点 RoCE IP>
GLOO_SOCKET_IFNAME=<fabric NIC>
NCCL_SOCKET_IFNAME=<fabric NIC>
NCCL_IB_HCA=<fabric HCA>
NCCL_IB_GID_INDEX=<本节点探测值>
NCCL_IB_DISABLE=0
NCCL_NET=IB
```

Docker 使用 host network，避免容器网桥进入自动选择。先把一条 rail 验证正确，再讨论 multi-rail；未经基准就开放多个 HCA，会扩大路径组合，让“到底用了哪张卡”更难回答。

控制面仍然可以走管理网。SSH 能从哪个地址登录，与 NCCL collective 应走哪条 fabric，是两件不同的事。把二者强行统一反而会降低运维便利性。

## 物理链路可用，不等于 NCCL 在使用

启动前先读取 HCA 端口状态：

```bash
cat /sys/class/infiniband/<HCA>/ports/1/state
cat /sys/class/infiniband/<HCA>/ports/1/phys_state
cat /sys/class/infiniband/<HCA>/ports/1/rate
```

预期看到 `ACTIVE`、`LINK_UP` 与对应速率。但这些只能证明 RDMA 链路就绪，不能证明当前 vLLM 进程选中了它。

启动阶段应保留 `NCCL_DEBUG=INFO`，并在两端日志中查找 `NET/IB`、目标 HCA 和 RoCE 相关记录。如果看到 socket transport、管理网 IP 或 Docker bridge 地址，应停止服务并修正绑定，不要在错误路径上继续调 batch size 和 CUDA Graph。

最终还需要一次真实生成请求。只有 rank 0 和 rank 1 完成实际 collective，才能同时验证 rendezvous、设备映射、数据路径与模型执行。仅做端口 ping 或 NCCL 初始化日志，都覆盖不了完整请求链路。

## 排障时一次只改变一层

网络失败常与镜像、模型和运行时错误同时出现。我的顺序是：先验证链路与名称，再验证 GID，再看 NCCL transport，最后才看上层推理。如果日志已经显示走 socket，就不要先怀疑量化 kernel；如果 RDMA 设备没有正确映射进容器，也不要用调整超时掩盖问题。

RoCE 部署没有一个适用于所有机器的固定 GID 或接口名。真正可以沉淀的是一套探测、绑定和证明确实生效的方法。把这些步骤变成脚本后，网络从“经验配置”变成了启动契约。

项目地址：[cyijun/dgx-spark-dual-node-inference](https://github.com/cyijun/dgx-spark-dual-node-inference)
