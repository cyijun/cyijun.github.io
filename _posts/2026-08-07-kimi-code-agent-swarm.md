---
title: "Kimi Code 的群体模式：从命令入口到 AgentSwarm 调度"
date: 2026-08-07 20:00:00 +0800
categories: [AI Agent, 源码阅读]
tags: [Kimi-Code, Multi-Agent, Swarm, TypeScript, TUI]
description: "沿 Kimi Code 源码拆解 swarm 模式如何进入、申请权限、批量创建或恢复子 Agent，并约束并发工具调用。"
---

一次历史 Codex 会话里，我研究的是 Kimi Code 当时被称作 Tower 的群体模式。回看当前代码，这套能力已经收敛为 `/swarm` 命令和 `AgentSwarm` 工具。名称在演进，核心问题没变：怎样让模型把一组相似任务交给多个子 Agent，同时保留权限、生命周期和失败恢复的控制。

## `/swarm` 既是模式开关，也是任务入口

当前 TUI 命令支持三种用法：

```text
/swarm
/swarm on | off
/swarm <task prompt>
```

无参数时切换模式，`on/off` 显式设置状态，带普通文本则确保 swarm mode 已开启，然后把文本作为用户任务送入当前 session。

命令入口先检查 active session 和模型配置。如果当前权限模式是 `manual`，开启群体模式前会弹出选择，让用户决定保持手动，还是切换到 `auto` / `yolo`。权限变更先写入 Session，成功后才更新本地 TUI state，避免界面显示已切换、服务端实际没有生效。

状态里还记录模式是用户手动进入，还是为了某次任务自动进入。这个细节让退出、标记和后续 UI 行为能够解释“为什么现在处于 swarm mode”。

## AgentSwarm 把“批量任务”变成明确 schema

模型真正分派时调用 `AgentSwarm`。输入不是一串随意 prompt，而是一组结构化字段：

- 整体 `description`；
- 可选的 `subagent_type`，新建时默认 coder；
- 带 `{{item}}` 占位符的 `prompt_template`；
- 一组 `items`；
- 或需要继续工作的 `resume_agent_ids`。

每个 item 替换模板占位符后形成一个独立子任务。若使用 items，至少要有两个；恢复已有 Agent 则允许单个。总数上限为 128，但这是输入保护上限，不代表运行时会无条件同时启动 128 个进程。

工具还会拒绝模板没有占位符、生成重复 prompt、空 item 或总量超限。重复任务看似无害，却会制造两个无法区分的子 Agent，浪费成本并让结果映射含糊，因此在调度前就 fail fast。

## 新建与恢复走同一队列

每个任务被转换为 `QueuedSubagentTask`，包含 profile、prompt、父 tool call ID、swarm index、超时与取消信号。新建任务使用指定 profile；恢复任务保留原 Agent 身份，并可从 host 取回它之前对应的 item。

所有任务交给 `SessionSubagentHost.runQueued()`，而不是在工具里直接 `Promise.all`。这样并发上限、排队、取消、超时和 session 记录可以集中管理。父 tool call ID 又把每个子 Agent 的执行归因到同一次 swarm 调用，便于 TUI 和 trace 展示。

失败结果会保留可用的 `agent_id`，并在聚合输出中给出 resume hint。下一次调用可以通过 `resume_agent_ids` 继续未完成工作，不必重新创建上下文。

## 为什么一次响应只能调用一个 AgentSwarm

权限策略明确禁止在同一个模型响应中：

- 同时调用多个 AgentSwarm；
- 把 AgentSwarm 与其他工具混用。

这并不意味着永远只能运行一个 swarm，而是要求时序清楚：先发出一个 AgentSwarm，等待聚合结果，再做下一次工具调用。否则多个 swarm 与普通工具并列时，权限审批、模式生命周期、结果顺序和取消传播都会变得含糊。

swarm mode 激活后，专用策略可以自动批准 AgentSwarm 工具；但进入模式本身仍受用户选择控制。这体现了一个好原则：用户授权的是一段明确的协作阶段，不是永久取消所有边界。

## 结果必须保留逐 Agent 状态

聚合结果使用结构化块，先给 completed/failed/aborted 汇总，再为每个 subagent 保留 item、agent ID、是否 resume、是否真正启动、结果或错误。

主 Agent 因而可以判断：

- 哪些结果已经可用于合并；
- 哪些因队列或取消从未启动；
- 哪些需要原 Agent 继续，而不是换一个重做；
- 失败是否集中在某类 item。

如果工具只返回十段文本拼接，群体模式就只是并行 prompt；有了稳定身份、父子边、状态和恢复语义，它才成为可管理的执行模型。

## 群体模式适合什么任务

最适合的是独立、同构、结果易合并的工作，例如逐文件审计、跨模块测试、多个数据分片分析或候选方案比较。强顺序依赖、多人同时修改同一文件、需要频繁共享中间状态的任务，反而可能因为冲突增加成本。

群体能力的难点从来不是 spawn，而是控制。Kimi Code 这条链路把命令状态、权限选择、参数验证、队列调度、结果归因与恢复放进了不同层，让“多 Agent”不只是界面上的热闹动画。

项目地址：[MoonshotAI/kimi-code](https://github.com/MoonshotAI/kimi-code)
