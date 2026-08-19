---
title: "把私有仓库转公开之前，我会做哪几层安全审计"
date: 2026-08-06 20:00:00 +0800
categories: [工程实践, 安全]
tags: [GitHub, Git, Secrets, Open-Source, Supply-Chain]
description: "一次真实仓库公开操作后的复盘：当前文件无密钥只是起点，历史、Actions、制品、许可证和身份信息都要检查。"
---

将 GitHub 仓库从 private 改成 public 只需要一条命令，判断“是否应该公开”却不是一个布尔检查。一次 Codex 历史会话中，我先做了本地扫描，再用 `gh` 查看和修改可见性。事后回看，最值得沉淀的不是命令，而是一套公开前审计顺序。

本文只讲通用方法，不复述当时项目的文件、身份或业务内容。

## 第一层：当前工作树与已跟踪文件

先确认 Git 实际会公开什么，而不是只看文件管理器：

```bash
git status --short --branch
git ls-files
git submodule status
git lfs ls-files
```

未跟踪的本地 `.env` 不会随仓库可见性变化自动上传，但它暴露了一个风险：同类文件是否曾经被提交过？`.gitignore` 也不是安全边界，它只影响未跟踪文件，不会让已经进入历史的秘密消失。

当前树应重点检查：

- API key、token、密码、私钥和证书；
- `.env`、云配置、SSH 配置和浏览器 session；
- 内网域名、IP、用户名、绝对路径与组织标识；
- 数据库、日志、聊天记录、客户或员工数据；
- 模型权重、数据集和不允许再分发的二进制；
- 构建产物、缓存、测试截图和调试 dump。

`rg --hidden -g '!.git'` 适合快速文本筛查，但真正的 secret scanner 还应识别编码、熵值和常见凭据格式。扫描结果需要人工确认，不能把“没有正则命中”当成安全证明。

## 第二层：完整 Git 历史

公开仓库会公开可达历史中的旧版本。当前文件删掉密码，如果旧 commit 仍含密码，攻击者几秒内就能找到。

审计应覆盖全部 refs、tag 和大对象：

```bash
git log --all --stat
git rev-list --objects --all
git tag --list
```

发现真实凭据时，优先立即吊销和轮换，再讨论历史重写。历史清理无法让已经被复制的秘密重新保密；强推也会改变协作者 commit，必须单独获得授权并制定迁移方案。

还要检查分支保护之外的 tag、release asset、GitHub Actions artifact、Packages、Pages 和 wiki。它们不一定出现在默认分支工作树中，却可能随项目一起被外界发现。

## 第三层：CI/CD 与 GitHub 权限

Workflow 文件本身可能泄露部署架构，也可能在 fork 或 pull request 场景获得过宽权限。重点看：

- `permissions` 是否最小化；
- 是否在不可信 PR 上使用 secrets；
- `pull_request_target` 是否 checkout 了攻击者代码；
- 第三方 Action 是否固定到可信版本或 commit；
- 发布 token、云角色和环境审批是否合理；
- cache、artifact 和日志会不会包含敏感配置。

把仓库设为 public 还会改变 Dependabot、fork、Actions 运行与外部贡献者的交互面。只扫描源码，不审 Workflow，是不完整的。

## 第四层：许可证与可再分发权

“这是我机器上的文件”不等于“我有权公开”。需要逐类确认：

- 第三方源码许可证与 NOTICE；
- 图片、字体、音频和文档授权；
- 数据集、行情、论文附件和模型权重条款；
- 复制进仓库的 SDK、wheel、固件和厂商手册；
- submodule 与 vendored dependency 的来源。

如果项目代码可以 MIT 开源，但数据不能再分发，就应只公开获取脚本、schema 和空目录占位，并在 README 说明用户自行取得授权。

## 第五层：从外部访问者视角检查

公开前最好在一个没有本机配置的临时目录重新 clone，验证构建是否依赖未提交文件，同时观察 README、issue 模板和示例是否暴露内部上下文。

还应通过 GitHub API 查看当前可见性、默认分支、Pages、release、Actions 和分支列表。修改可见性前明确目标仓库，避免在错误 remote 或 fork 上执行操作。

```bash
gh repo view OWNER/REPO --json nameWithOwner,isPrivate,visibility,url
```

真正切换 public 是一个有外部影响的不可逆式动作：即使以后改回 private，期间产生的 clone 和 fork 也无法收回。命令必须与“已审计的确切仓库”绑定，并在执行后重新查询状态。

## 公开之后仍要继续观察

上线后检查首次 Actions、Pages、依赖告警和 secret scanning；搜索 GitHub 页面确认没有意外文件；为安全问题提供私密报告渠道。公开不是审计结束，而是威胁模型发生变化的起点。

我的最终清单可以浓缩为五个问题：现在有什么、历史有什么、自动化能做什么、我有权发布什么、外部人实际能看到什么。只有这五项都得到证据支持，`gh repo edit --visibility public` 才应该成为最后一步。
