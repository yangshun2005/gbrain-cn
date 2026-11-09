# functional-area-resolver A/B 评估

`functional-area-resolver` 技能的维护者侧评估证据。刻意放在 `skills/` 之外——技能包打包器会递归遍历 `skills/<skill>/`，放在里面的评估内容会随每次下游 `gbrain skillpack install` 一起发布。本目录不打包。模式（在 SKILL.md 中）随包发布到各处；评估证据留在 gbrain 仓库，供维护者重新基线化。

## 证明了什么

在三个 Anthropic 前沿模型上测试了三种解析器形态。`skills/functional-area-resolver/SKILL.md` 中的模式（带 `(dispatcher for: ...)` 子句的功能区调度器）**在训练集上比冗长的项目符号列表基线高 +13 到 +17pp，同时体积只有 48%**，并且在 Sonnet 上**彻底击败不带调度器子句的压缩版**（训练集 100% vs 41.7%，宽松评分）。

## 方法

### 变体

- `variants/baseline.md` — 从真实生产 AGENTS.md 的 git 提交 `93848ff3b^`（压缩前状态）提取的 270 行冗长项目符号列表形态，已清除所有者 PII。约 25KB。
- `variants/functional-areas.md` — git 提交 `93848ff3b`（提交信息为 "AGENTS.md: functional-area resolver — 25KB→13KB, 100% routing accuracy"）的调度器模式。约 13KB。
- `variants/resolver-of-resolvers.md` — 从 functional-areas 机械地剥离 `(dispatcher for: ...)` 子句得到。消融用例：结构相同，无子技能可见性。约 10KB。

### 语料

- `fixtures.jsonl` — 20 条手工编写的训练夹具，用于开发变体。训练集上的头条准确率有参考价值，但不是结论（同作者过拟合风险）。
- `fixtures-held-out.jsonl` — 5 条在变体之前编写、之后未调整的夹具。留出集才是正式结论，但样本量小意味着大多数单元格接近 100% 饱和。

### 评分

每条输出行带两个分数：

- **STRICT**（`correct`）— 预测 slug 与期望完全相等。
- **LENIENT**（`correct_lenient`）— 按变体的 `(dispatcher for: ...)` 子句，预测与期望属于同一调度器区域。对于无调度器子句的变体（baseline、resolver-of-resolvers），LENIENT 退化为 STRICT。

两者都重要：
- STRICT 衡量"LLM 是否返回精确 slug？"
- LENIENT 衡量"LLM 是否落在正确区域，即使选了更具体的子技能？"这反映生产 agent 行为——邮件意图落在 `gmail` 即成功，即使解析器写的是 `executive-assistant`。

### 重复 + 统计

- 每个（夹具、变体、模型）n=3 次带种子的重复。
- 95% 置信区间，基于 3 个种子均值的 t 分布（df=2 时 t 临界值=4.303）。
- 模型：`claude-opus-4-7`、`claude-sonnet-4-6`、`claude-haiku-4-5-20251001`。

### 回执格式

每次运行写一个 JSONL：
- 表头行：`{kind:'receipt', model, prompt_template_hash, fixtures_hash, fixtures_held_out_hash, harness_sha, ts, cmd_args}` — 将运行绑定到特定 harness 版本和输入，使重跑可审计。
- 每个（夹具 × 变体 × 种子）一行：完整行 schema 见 `harness-runner.ts`。

v0.32.3.0 重新基线化后的基线回执提交在 `baseline-runs/`。

## 结果（2026-05-11）

训练语料（n=20，3 种子，LENIENT 评分）：

| 变体 | Opus 4.7 | Sonnet 4.6 | Haiku 4.5 | 体积 |
|---|---|---|---|---|
| baseline | 81.7% ± 7.2% | 86.7% ± 7.2% | 73.3% ± 7.2% | 25KB |
| **functional-areas** | **98.3% ± 7.2%** | **100% ± 0%** | **88.3% ± 7.2%** | **13KB** |
| resolver-of-resolvers | 63.3% ± 14.3% | 41.7% ± 7.2% | 65.0% ± 12.4% | 10KB |

留出语料（n=5，3 种子，LENIENT 评分）：

| 变体 | Opus 4.7 | Sonnet 4.6 | Haiku 4.5 |
|---|---|---|---|
| baseline | 100% ± 0% | 100% ± 0% | 100% ± 0% |
| **functional-areas** | **100% ± 0%** | **100% ± 0%** | **100% ± 0%** |
| resolver-of-resolvers | 100% ± 0% | **73.3% ± 28.7%** | 100% ± 0% |

严格评分数字和逐夹具失败轨迹见回执。

## 如何复现

在 gbrain 仓库根目录，设置 `ANTHROPIC_API_KEY`：

```bash
cd evals/functional-area-resolver

# 冒烟测试（1 次调用，约 $0.01）
node harness.mjs --limit 1 --yes

# Opus 4.7 完整运行（225 次调用，约 $1.70）
node harness.mjs --model opus --parallel 3 --yes

# 跨模型
node harness.mjs --model sonnet --parallel 3 --yes      # 约 $1.00
node harness.mjs --model haiku --parallel 3 --yes       # 约 $0.30

# 重新评分已有运行，不额外消耗 API 预算
node rescore.mjs baseline-runs/2026-05-11-opus-4-7.jsonl

# 单元测试（无需 API key）
bun test harness-runner.test.ts
```

harness 通过 gbrain 网关路由，因此继承 gbrain 的认证、速率租约和成本计量行为。没有 `ANTHROPIC_API_KEY` 时会以明确错误退出。

## 重要警告：提示词是关键

harness 使用调度器感知的提示词（见 `harness-runner.ts:PROMPT_TEMPLATE`），明确告诉 LLM：

> 有些条目是功能区调度器，形如：
>   "**Area name**: triggers... → `dispatcher-skill` (dispatcher for: subskill-a, subskill-b, ...)"
> 当用户意图匹配某区域时，从该区域的 "dispatcher for" 列表中返回最具体的子技能，而非调度器本身。

**没有这条指令，所有压缩变体在训练集上都跌到约 30-60%。** 朴素的"返回技能 slug"提示词会让 LLM 选区域主技能，而不深入调度器列表。这是 run-1（合成变体 + 朴素提示词）的失败模式，之后才用真实变体 + 调度器感知提示词重新基线化。

如果你在自己的 agent 中采用该模式，SKILL.md 的指导同样适用于你的 harness 提示词。从本 harness 提取 PROMPT_TEMPLATE，或自行编写解释调度器列表的指令。

## 局限与 v0.33.x 后续

1. 留出语料小（n=5）。大多数单元格 100% 饱和。v0.33.x 扩到 >=20。
2. 单一厂商（Anthropic）。跨厂商（Gemini、GPT）是 v0.33.x。
3. 尚无描述长度扫描。Anthropic Agent Skills 的 frontmatter 中位数约 80 token；我们尚未测量逐行描述长度的最佳点。v0.33.x。
4. 同作者训练语料 + 变体。留出集部分缓解。
5. 无对抗性夹具（例如"我想做点跟大脑相关的事"而不说明具体做什么）。v0.33.x。

完整列表见 `TODOS.md`。

## 相关工作

本评估实现了分层 agent 路由的**静态提示词类比**，这是 2024-2025 年的研究方向。已发表的分层方案在运行时通过第二次 LLM 调用解析层级；本技能将层级内联为单次 LLM 通过的调度器列表。

- AnyTool（[arXiv:2402.04253](https://arxiv.org/abs/2402.04253)）— 元 agent → 类别 → 工具层级，在 16K API 上比扁平检索高 +35.4pp。
- RAG-MCP（[arXiv:2505.03275](https://arxiv.org/html/2505.03275v1)）— 基于嵌入的预检索，token 减少 49.2%，准确率提升 3.2×。
- Anthropic Agent Skills（[工程博客](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)）— 渐进式披露（启动时加载约 80 token 的 frontmatter；匹配时加载正文）。

## 文件列表

```
evals/functional-area-resolver/
├── README.md                                       # 本文件
├── fixtures.jsonl                                  # 20 条训练夹具
├── fixtures-held-out.jsonl                         # 5 条留出盲测夹具
├── variants/
│   ├── baseline.md                                 # 25KB，从生产清除 PII
│   ├── functional-areas.md                         # 13KB，从生产清除 PII
│   └── resolver-of-resolvers.md                    # 10KB，派生的消融版
├── harness.mjs                                     # 轻量 Node CLI 封装
├── harness-runner.ts                               # 经 gbrain 网关的 TS 运行器
├── harness-runner.test.ts                          # 45 个单元测试（无需 API key）
├── rescore.mjs                                     # 零成本宽松重评分
└── baseline-runs/
    ├── 2026-05-11-opus-4-7.jsonl                   # 225 行 Opus 基线
    ├── 2026-05-11-sonnet-4-6.jsonl                 # 225 行 Sonnet 基线
    └── 2026-05-11-haiku-4-5.jsonl                  # 225 行 Haiku 基线
```