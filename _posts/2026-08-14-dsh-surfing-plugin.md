---
title: "给 DeepSeek Harness 接上自托管搜索：SearXNG 与 Crawl4AI Provider 设计"
date: 2026-08-14 20:00:00 +0800
categories: [AI Agent, 工具集成]
tags: [DeepSeek-Harness, SearXNG, Crawl4AI, TypeScript, Plugin]
description: "设计 dsh-surfing-plugin：搜索与抓取解耦、配置优先级、错误映射和安全边界。"
---

Agent 的“联网”其实包含两个不同问题：先找到可能相关的页面，再读取其中一页的正文。前者是搜索，后者是抓取。把二者塞进同一个后端，不但难以替换，也很容易让结果格式、超时和错误语义混在一起。

我在 `dsh-surfing-plugin` 中把这两个职责拆开：`web_search` 交给自托管 SearXNG，`web_fetch` 交给 Crawl4AI。插件接入 DeepSeek Harness 的 `ctx.web` Provider 接口，所以模型看到的仍然是 DSH 原生工具；工具名称、参数、取消和结果展示不需要重新发明。

## Provider 是适配层，不是第二套工具系统

插件分别注册 `surfing-searxng` 和 `surfing-crawl4ai`。随包提供的 `cordis.patch.yml` 负责挂载插件，并将 DSH 的搜索与抓取 Provider 指向它们。

近期修复还增加了一个只注册原生 `web_fetch` 的独立 Consumer，使 headless profile 与 Web UI 中不同的 Agent Preset 都能获得抓取能力。这里的经验是：功能代码正确还不够，插件必须进入宿主实际使用的组装路径。只在一个 profile 中注册，另一个界面就会表现成“搜索可用、打开网页不可用”。

Provider 的职责应该保持克制：把宿主协议转换为后端协议，再把结果映射回来。搜索排序、抓取浏览器隔离和 Agent 工具交互仍分别由 SearXNG、Crawl4AI 与 DSH 负责。

## SearXNG：清洗而不是搬运

搜索 Provider 向 `/search` 发送表单编码 POST，并固定请求 `format=json`。SearXNG 端必须在 `search.formats` 中启用 JSON，否则服务会拒绝请求。

返回结果不会原样透传。插件只保留绝对 HTTP(S) URL，按 URL 去重，再将标题、摘要和发布日期映射为 DSH source。`maxResults` 在 Provider 与 DSH web service 两层执行，避免上游返回过多内容后继续占用模型上下文。

如果 SearXNG 返回 `answers`，它们可以成为搜索结果的摘要内容；但普通结果仍保留来源列表，避免 Agent 得到一个没有出处的合成答案。

## Crawl4AI：限制模型能控制的参数

抓取端只接受 HTTP(S) 目标，并向 `/crawl` 发送最小请求：

```json
{ "urls": ["https://example.com/page"] }
```

模型不能借工具参数向 Crawl4AI 注入任意浏览器或 crawler 配置。这个限制缩小了攻击面，也让抓取行为更可预测。目标站点的 HTTP 状态会进入 DSH 结果；Crawl4AI API 自身的非 2xx、抓取失败或无法识别的响应则转换成结构化 `WebError`。

正文可以选择 `raw`、`fit` 或 `citations` markdown。后两者为空时回退到 raw；如果没有 markdown，才使用 `cleaned_html` 或原始 HTML。最大返回字符数在 Provider 侧截断，防止单页吞掉整个上下文窗口。

## 配置优先级要明确

服务地址既可写根地址，也可写完整 `/search` 或 `/crawl` 端点。显式插件配置优先于环境变量；密钥字段又优先于 `apiKeyEnv` 指向的环境变量。没有密钥时不发送认证头，适配无认证的本地服务。

生产环境更推荐只配置密钥所在的环境变量名：

```yaml
crawl4ai:
  url: https://crawl.example.com
  apiKeyEnv: CRAWL4AI_API_TOKEN
  authHeader: Authorization
  authScheme: Bearer
```

这样 profile 配置可以进入版本控制，真实凭据仍留在运行环境。对非本机服务应使用 HTTPS；Provider 禁止跟随重定向，避免认证头被带到另一个后端地址。

需要注意，插件只能控制它到 Crawl4AI 的请求。浏览器隔离、目标网络访问和 SSRF 策略仍属于 Crawl4AI 部署本身，公开服务前必须单独加固。

## 自托管的真正价值是可替换

自托管搜索并不自动意味着更准确。它的价值在于数据路径、语言、类别、安全搜索、时间范围和抓取策略都可由使用者控制，而且搜索与抓取可以分别扩容或替换。

对 Agent 框架来说，最稳的集成也不是让模型知道每个后端的细节，而是保持原生 `web_search` / `web_fetch` 契约稳定，把变化限制在 Provider。这样以后替换 SearXNG 实例、升级 Crawl4AI 或增加认证方式，都不需要重写上层 prompt 和工具逻辑。

项目地址：[cyijun/dsh-surfing-plugin](https://github.com/cyijun/dsh-surfing-plugin)

