---
title: "AgentENV 为什么能高密度运行 Agent 沙箱：Firecracker、OverlayBD 与 ublk"
date: 2026-07-30 20:00:00 +0800
categories: [AI Agent, 基础设施]
tags: [AgentENV, Firecracker, OverlayBD, ublk, Sandbox]
description: "从一次 Codex 源码阅读出发，拆解 AgentENV 如何兼顾微虚机隔离、增量快照、快速恢复与存储复用。"
---

Coding Agent 的执行环境有一个天然矛盾：既要像虚拟机一样隔离，又要像容器一样快速创建；既要允许安装依赖和修改文件，又不能为每个任务复制一整块根磁盘。一次历史 Codex 会话里，我沿着 `AgentENV` 的 Firecracker、OverlayBD 和 ublk 实现追了一遍，才发现它的核心并不是某一个“快”组件，而是把隔离、存储与生命周期放在了正确的边界上。

## Firecracker 负责隔离，不理解镜像分层

每个 Agent 环境运行在 Firecracker microVM 中。VM 里看到的仍是普通 ext4 文件系统和 virtio-blk 磁盘；它不需要理解 OCI layer、快照仓库或远程对象存储。

宿主机上的数据路径大致是：

```text
VM 内 ext4
  → Firecracker virtio-blk
  → /dev/ublkbN
  → Linux ublk
  → 用户态 OverlayBD target
  → writable upper + snapshots + OCI layers
```

这种分工很干净：Firecracker 只处理 CPU、内存、设备与 VM 生命周期，OverlayBD 负责块级分层，ublk 则把用户态存储实现桥接成内核可见的块设备。

## 创建沙箱时不复制完整根文件系统

传统 VM 模板常以完整磁盘镜像为单位。创建 100 个实例，即使大部分内容相同，也容易产生大量复制。AgentENV 为每个沙箱保留一个很小的可写 upper，基础 OCI 层和只读快照由多个实例共享。

当 VM 修改逻辑块时，新内容追加到 upper，并更新“逻辑块 → 物理位置”的索引；没有修改过的块继续从 lower layer 读取。这是块设备级 Copy-on-Write，而不是在文件路径层拷贝整个目录。

读取也不是每次从最上层线性扫描到底。层栈打开后会合并索引，并可复用持久化的 premerged index。大量沙箱来自同一个模板时，基础镜像、快照层和页缓存都能复用，私有数据只随实际写入增长。

## ublk 是连接性能与可组合性的关键

OverlayBD 是用户态存储逻辑，Firecracker 需要的却是一个块设备。ublk 让 Linux 内核创建 `/dev/ublkbN`，再把 I/O 请求交给用户态队列处理。

这样做有两个好处。第一，VM 侧保持标准 virtio-blk，无需注入定制文件系统；第二，宿主侧仍能使用 Rust、io_uring、分层索引、压缩和远程缓存等实现。存储系统可以演进，而 VM 的设备契约保持稳定。

写入路径也针对沙箱负载做了取舍：默认日志结构模式顺序追加，快照天然就是增量；小块本地写可以直接 `pwrite`，较大的 I/O 继续走 io_uring；丢弃和写零可记录映射，不必真的写满零数据。

## 快照不只是“保存一下磁盘”

Agent 任务经常需要暂停、恢复和分叉。AgentENV 的快照同时考虑内存与文件系统增量。README 给出的项目指标是：快照支持重负载下百毫秒级完成，基于快照的环境可在 50 ms 内启动或恢复，暂停低于 100 ms。这里应把它理解为项目报告的目标环境结果，而不是所有硬件上的无条件 SLA。

fork 的价值尤其适合 Agent 搜索：从同一个中间状态派生多个独立沙箱，让不同候选方案并行试验，而不重复准备依赖和工作树。快照可以落到 S3-compatible 对象存储或共享文件系统，节点本地盘只作为有界热缓存；热数据保留，冷数据淘汰，因此镜像集合可以大于单机磁盘。

内存侧还有 ballooning，把 guest 内可回收页面还给宿主。否则沙箱运行时间越长、内部页缓存越多，理论上的高超卖密度会逐渐消失。

## 单机运行时与集群控制面分层

集群模式下，Gateway 接收请求，Scheduler 维护 sandbox 到 node 的绑定，运行节点通过心跳上报沙箱数量与 CPU、内存、磁盘快照。资源阈值先过滤候选节点，调度策略只在合格节点中选择。

数据面路由可以根据 sandbox header 或专用 host 名定位已有实例；列表请求则由 Gateway 向各节点 fan-out 后合并。绑定既可驻留内存，也可用 Redis 支持数据面查询的高可用。但控制面创建和调度仍依赖主 Scheduler，这个边界需要在部署设计里明确。

## 最容易忽略的是安全默认值

AgentENV 当前 README 明确提示服务本身不提供授权，API 不应直接暴露公网。Firecracker 提供的是计算隔离，不等于控制面自动安全。实际部署至少要放在可信网络或认证代理之后，并限制沙箱出网、宿主设备访问和管理 API。

这次源码阅读给我的最大启发是：高密度沙箱不是靠“把 VM 做得更像容器”，而是让 VM 保留隔离职责，再把可共享的数据、快照和缓存下沉到块设备层。边界清楚之后，快照、fork 和集群调度才能自然组合。

项目地址：[kvcache-ai/AgentENV](https://github.com/kvcache-ai/AgentENV)
