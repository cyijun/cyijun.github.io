---
title: "把 Coding Agent 会话变成可观测 Trace：AgentsTrace 的 Langfuse 实践"
date: 2026-08-04 20:00:00 +0800
categories: [AI Agent, 可观测性]
tags: [Agent, Langfuse, OpenTelemetry, Codex, 隐私]
description: "从扁平 JSONL 到 agent、generation、tool、subagent 树，并在同步 Langfuse 前完成脱敏、校验和幂等控制。"
---

Coding Agent 的本地日志很丰富：用户输入、模型输出、思考片段、工具调用、终端结果和子 Agent 事件都在里面。但日志文件多，不等于系统可观测。把所有内容压成一列 `messages` 适合做数据集，却很难回答一次任务究竟慢在哪里、哪个工具失败、某个子任务从哪里被派生。

`AgentsTrace` 最近加入了 Langfuse 导出，并在随后的 v1.1.0 更新中加固运行时、脱敏和发布流程。我的目标不是“把 JSON 上传”，而是构造一棵保留因果关系、时间边界和证据质量的 trace。

## 先建立独立的规范模型

原有导出器继续生成兼容的 JSONL；Langfuse 路径则使用独立的 canonical trace model。一条 trace 内包含唯一根节点，以及 agent、subagent、generation、tool 和 event 等 observation。

这个中间层很重要。Claude Code、Codex CLI、Kimi Code CLI、OpenCode、OpenClaw 和 Gemini CLI 的本地格式并不相同。如果每个解析器直接拼 Langfuse payload，父子关系、时间语义和错误处理很快会散落到多处。先归一化，再统一脱敏、验证和发送，才能让跨来源的查询有一致含义。

规范模型还会记录 `timestamp_quality`、`parent_link_quality`、`missing_parent` 等元数据。源日志没有精确开始时间时，我宁愿明确标记“推断”，也不生成一个看似精确的假时间。

## 工具和模型调用为什么是兄弟节点

在 Langfuse 的 agent 视图里，generation 与它发起的 tool observation 被放在 agent 下作为兄弟节点，二者的因果关系通过稳定 metadata 和 OpenTelemetry span link 保留。这样既符合常见的 Agent 展示方式，又不会丢掉“哪个模型调用触发了哪个工具”。

如果工具进一步启动 subagent，subagent 则直接挂在该 tool 下。这条边对于理解多 Agent 执行尤其关键：只看时间先后，无法区分并行分派、嵌套调用和普通的下一轮推理。

## 优先消费官方运行时证据

Codex 新会话可以选择生成官方 `codex-rollout-trace` bundle。它用连续 `seq` 保存推理、code cell、终端、工具 dispatch、子 Agent 边和结果，并把结构化 payload 放在独立文件中。相比从会话文本里反推 JavaScript 调用了哪个工具，这种运行时边界更可靠。

AgentsTrace 因而优先读取 rollout trace；未启用录制的历史会话再回退到 session JSONL。回退适配器只对静态字面量做安全解析，不执行日志中的代码；动态表达式会降级为 `arguments_code`，纯编排脚本也不会被伪造成不存在的子 span。

这是可观测系统必须坚持的原则：能证明多少就表达多少。为了让图更漂亮而补造事件，最终只会误导排障。

## 同步之前先处理隐私

Agent 日志的敏感度通常高于普通应用日志。它可能含有 prompt、源代码、终端输出、本机路径、用户名、API key，甚至完整工具参数。AgentsTrace 默认对嵌套字符串递归执行 secrets、PII、用户名和路径脱敏，然后才进入网络发送阶段。

Langfuse 凭据只从环境变量读取，不写入配置和同步账本。私网地址、`localhost`、`.local` 主机默认绕过系统 HTTP 代理，避免内网认证头误发给代理。单个 observation 还有字符上限，超限时上传带原长度的预览，而不是无限扩张请求体。

项目保留了关闭部分 secrets 规则的危险选项，但用户名和路径仍然匿名化。我的建议很简单：除非 Langfuse 完全自控且有明确理由，否则不要关闭默认脱敏。

## 用 preview、smoke、sync 拆开风险

导出流程分成几个层级：

```bash
agentstracer langfuse doctor
agentstracer langfuse smoke
agentstracer langfuse preview --source all
agentstracer langfuse sync --source all --verify
```

`doctor` 检查版本和凭据；`smoke` 只写入一条不含本机会话的合成 trace；`preview` 在本机解析、脱敏和校验，不发网络请求；最后 `sync --verify` 才真正增量写入，并通过 Observations API 回读验证。

本地 SQLite 账本记录已被服务端接受的不可变快照。同一个活跃日志增长后会产生新的稳定 trace ID，已经成功接收的快照不会重复发送。网络临时错误和 429/5xx 使用退避重试，回读延迟也不会错误地把已接收数据重新排队。

Agent 可观测性的难点从来不是画瀑布图，而是忠实保留结构、承认不确定性，并在数据离开机器前建立明确的隐私边界。做到这些之后，Langfuse 才真正成为排障工具，而不是另一份难以信任的日志副本。

项目地址：[cyijun/agentstracer](https://github.com/cyijun/agentstracer)

