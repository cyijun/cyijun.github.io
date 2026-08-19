---
title: "把实验框架从 PowerShell 迁到 Linux：为什么不只是改成 Bash"
date: 2026-08-02 20:00:00 +0800
categories: [工程实践, 实验平台]
tags: [Linux, tmux, Python, Benchmark, Reproducibility]
description: "复盘 Workflow Arena 的 Linux 原生引擎与 tmux 监督：协议等价、证据隔离、进程组清理和失败闭合。"
---

把一个 PowerShell 实验框架“适配 Linux”，最差的做法是逐行翻译成 Bash。脚本或许能启动，但原来的生命周期、超时、隔离、证据冻结和子进程清理很容易悄悄变形。一次 Codex 历史会话把 `workflow-arena` 迁到 Linux/tmux 时，我最关注的就是：跨平台实现可以不同，实验协议必须相同。

## 先固定生命周期，再选择语言

Workflow Arena 的核心不是 PowerShell，而是一条受控实验链：

```text
Prepare → Run → Freeze → Audit → Judge → Summarize
```

Linux 版本使用 Python 3 标准库与 Bash，Windows 保留 PowerShell。实验 manifest 必须显式声明 `linuxDriver`，Linux 原生引擎绝不回退执行 `.ps1`，也不会在 driver 缺失时把 `dry-run` 伪装成成功。

这个 fail-closed 规则很重要。`Validate` 和 `Plan` 可以读取尚未迁移的历史协议，但执行 `All` 必须有真正的 Linux driver。否则报告看起来完成，实际可能一个 treatment 都没跑。

## 平台差异应留在 manifest 边界

不同系统的安装器、路径链接、shell 和工具链获取方式不同。迁移后的设计把这些差异声明在 capability 与 experiment manifest 中，而不是让业务 driver 到处写 `if linux`。

Linux bootstrap 会验证固定 commit 与 tree identity，创建隔离 baseline，恢复平台适配的目录链接，安装任务依赖并预取 Cargo/Go 缓存。任务工具链来自带 SHA-256 的官方归档，再显式注入候选和测试进程的 `PATH`。

这解决了 tmux 场景下的一个常见问题：交互 shell 里能找到 Go，detached session 却因为没有读取相同启动文件而失败。实验环境不能依赖操作者恰好配置好的 `.zshrc`。

## tmux 是监督层，不是简单的后台符号

长实验需要跨 SSH 断开继续运行。直接在命令末尾加 `&`，无法可靠回答进程是否还活着、日志在哪里、退出码是什么，也很难终止整棵子进程树。

tmux wrapper 因此提供明确动作：

```bash
./scripts/tmux-experiment.sh start <experiment> --action All --dry-run
./scripts/tmux-experiment.sh status <experiment>
./scripts/tmux-experiment.sh logs <experiment> --follow
./scripts/tmux-experiment.sh attach <experiment>
./scripts/tmux-experiment.sh stop <experiment>
```

默认 session 名来自实验目录，也允许显式指定，避免同名冲突。生成 runner、日志、引擎进程组 ID 与最终退出码都放在被 Git 忽略的 `.scratch/tmux/`。

`status` 不只检查 tmux session。pane 被人为关闭时，引擎进程可能仍在后台；反过来 pane 还在，也可能主进程已经退出。监督逻辑需要同时观察 session、PID/process group 和 exit-code 文件。

## 停止实验必须清理整棵进程树

Coding Agent 评测可能启动候选、operator、reviewer、judge 和测试工具。只杀最外层 Python 进程，GPU、终端和网络子进程仍会继续运行，下一次实验就会受到污染。

原生引擎让模型调用注册各自的进程组。`stop` 与终止信号会先结束已跟踪的 Codex 子进程组，再关闭 tmux。这样“停止一次实验”才是完整的生命周期动作，而不是让孤儿进程留在宿主机上。

## 历史证据不可被新平台覆盖

旧 PowerShell campaign 是已经产生结论的冻结证据，不应被 Linux 版本原地改写。新 driver 写入新的 state/campaign root，旧报告保留为历史；早期 Linux v1/v2 中发现的问题也作为诊断证据保留，而不是为了页面好看直接覆盖。

实验框架的迁移成功标准不是“同一个命令在 Linux 能跑”，而是：同一输入、同一 treatment、同一隔离和同一评审协议能够生成可区分的新证据。平台实现可以重写，实验不变量不能漂移。

## 自测试覆盖边界错误

`native_selftest.py` 与 `tmux-selftest.sh` 不需要调用真实模型，就能验证 manifest 解析、缺失 driver 拒绝、平台 installer、路径逃逸、dry-run、状态文件和 tmux 生命周期。这类小而确定的测试比直接烧一次完整 API 成本更适合作为迁移门禁。

跨平台适配最终是一项协议工程。把 PowerShell 换成 Python/Bash只是表层；真正需要保持的是执行顺序、环境身份、失败语义、进程所有权和证据不可变性。

项目地址：[cyijun/workflow-arena](https://github.com/cyijun/workflow-arena)
