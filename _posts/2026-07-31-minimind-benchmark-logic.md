---
title: "Benchmark 不是让模型自由答题：MiniMind 评测逻辑拆解"
date: 2026-07-31 20:00:00 +0800
categories: [大模型, 评测]
tags: [MiniMind, lm-evaluation-harness, C-Eval, MMLU, VLM]
description: "解释多选 benchmark 的 teacher forcing、acc 与 acc_norm，以及 MiniMind-V 示例生成为什么不等同于量化评测。"
---

“让模型跑一下 benchmark”听起来像把题目发给聊天接口，再判断最终回答。实际查看 MiniMind 的评测脚本后，会发现标准多选评测更接近一次受控概率比较：候选答案都被强制送进模型，程序比较它们在相同题干下的条件概率。

这也是一次 Codex 会话里最容易混淆的地方：自由生成、teacher forcing、多选准确率和图片示例展示，测量的并不是同一件事。

## 多选题如何变成 log-likelihood

假设问题有两个候选：

```text
A：太阳从东方升起
B：太阳从西方升起
```

评测器不会先让模型随便生成一段话，再用正则找 A 或 B。它分别构造“题干 + 候选 A”和“题干 + 候选 B”，在 teacher forcing 下计算候选 token 的对数概率。

对于候选序列 `y1...yn`：

```text
log P(y | x) = Σ log P(yi | x, y1...y(i-1))
```

每一步都喂入正确的候选前缀，而不是模型上一时刻自由生成的 token。Transformer 的因果 mask 允许这些位置在一次 forward 中并行得到 logits。最终选择 log-likelihood 更高、等价于负对数似然更低的候选。

所以 teacher forcing 不是“把正确答案告诉模型然后看它会不会复述”，而是在固定候选集合中测量：模型认为哪个完整续写更合理。

## `acc` 与 `acc_norm` 测的是什么

MiniMind 的本地任务定义把 ARC、HellaSwag、PIQA、OpenBookQA 等声明为 `multiple_choice`，同时输出 `acc` 与 `acc_norm`。

- `acc`：直接比较每个候选的总 log-likelihood；
- `acc_norm`：按候选长度进行归一化，减少长答案因为 token 更多而天然累积更多负值的偏差。

两者都合理，但回答的问题略有不同。候选长度接近时差别可能很小；长度差异明显时，只报一个数容易掩盖模型究竟是在利用知识，还是受长度偏置影响。

C-Eval 与 CMMLU 又包含多个学科子任务。仓库中的 group YAML 按样本量加权汇总 `acc` 和 `acc_norm`，避免一个只有少量题目的学科与大规模学科拥有完全相同的总权重。报告总分时，也应该保留学科分项，否则强弱项会被平均数抹平。

## 一键脚本真正固定了哪些变量

`run_benchmark.sh` 使用 `lm-evaluation-harness` 的 Hugging Face adapter，默认运行 C-Eval、CMMLU、ARC Easy、HellaSwag、Social IQA、PIQA 与 OpenBookQA。脚本固定了：

- 本地模型路径与任务 YAML；
- batch size 与设备；
- chat template 是否应用；
- 本地数据集加载器；
- 逐样本日志与带时间戳的输出目录。

`--limit` 只适合冒烟，不能把少量样本得分当成完整 benchmark。正式比较还应固定模型 revision、tokenizer、lm-eval 版本、dtype、模板和任务版本。对小模型来说，prompt 格式差异就可能造成显著波动。

## MiniMind-V 的 `eval_vlm.py` 不是量化 benchmark

MiniMind-V 的评估脚本会遍历目录中的几张测试图，使用统一提示“请描述图中的主要物体和场景”，然后打印自由生成结果。这非常适合快速检查视觉编码器、projection、占位 token 和生成链路有没有跑通，但它没有：

- 标准答案与自动评分；
- 多样本统计；
- 对象、OCR、关系、幻觉等分项指标；
- 固定解码与重复试验；
- 与基线模型相同的数据协议。

因此几张图片“看起来回答不错”只能叫定性 smoke test，不能与 C-Eval 准确率放在同一张雷达图上解释。

MiniMind-V 把图像编码成视觉 token，经 projector 映射到语言模型隐藏空间，再替换文本中的图像占位符。这个结构是否能运行，可以用示例图验证；它在视觉问答、图表理解或幻觉控制上有多强，则需要专门的数据集和评测器。

## 评测结果的正确读法

一份可信报告至少应写明：任务、split、样本数、模型与 tokenizer revision、模板、dtype、batch、评分指标和 harness 版本。多选 log-likelihood 衡量固定候选偏好，不完全等价于聊天时自由生成；定性图片展示衡量可用性，也不等价于泛化能力。

知道 benchmark 的执行逻辑后，分数才不只是一个装饰数字。真正可比较的不是“都跑过 C-Eval”，而是双方是否使用了相同的数据、提示、候选评分与汇总协议。

项目地址：[MiniMind](https://github.com/jingyaogong/minimind) · [MiniMind-V](https://github.com/jingyaogong/minimind-v)
