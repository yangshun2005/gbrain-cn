# Take 评分评估（W6，wave 4b）

衡量哪些聊天模型可以为预测 take 评分：当 `grade_takes` 周期阶段的自动解析规则会应用裁决时（非 `unresolvable`，置信度达到或超过 `GRADE_TAKES_AUTO_RESOLVE_THRESHOLD`，即 0.95），它出错的频率有多高？结果决定 `GRADE_TAKES_MEASURED_MODELS`（`src/core/cycle/grade-takes-measured.ts`），`gbrain takes grades` 用它标记没有测量支撑的裁决。自动解析保持关闭。预注册见 [`PREREGISTRATION.md`](PREREGISTRATION.md)；原始结果和结论镜像到 gbrain-evals（`docs/benchmarks/2026-10-16-take-grading*`）。

## 当前结果

测量于 2026-10-16，gbrain `d99b84694`（分支 `capy/w6-take-grading`，prompt `hybrid-evidence-v2`，阈值 0.95），即冻结预注册的那个提交；每个模型在 790 个计入 take 上运行一次，共花费 $9.25。只有 gpt-6.1-sol 合格。

| 模型 | 应用数 | 错误应用 | 95% 上界 | 覆盖率 | 弃权率 | 无答案 | 每次裁决 USD | 延迟 p50 / p95 | 是否合格 |
|---|---|---|---|---|---|---|---|---|---|
| `openai:gpt-6.1-sol` | 496 | 0 | 0.6% | 99.2% | 100% | 1 | $0.0016 | 2.4 s / 3.2 s | 是 |
| `anthropic:claude-sonnet-5-5` | 121 | 1 | 3.9% | 24.0% | 99.6% | 14 | $0.0032 | 1.7 s / 4.5 s | 否 |
| `anthropic:claude-opus-5-5` | 2 | 0 | 77.6% | 0.4% | 100% | 3 | $0.0069 | 2.7 s / 4.9 s | 否 |

因此 `GRADE_TAKES_MEASURED_MODELS` 为 `openai:gpt-6.1-sol`，适用于以 prompt `hybrid-evidence-v2`（`GRADE_TAKES_MEASURED_PROMPT_VERSION`）请求的裁决。gpt-6.1-sol 对规则内全部 750 个 take 都给出了金标准裁决，并对每个 unresolvable 的 take 都弃权；它接近这些合成 fixture 的上限，所以该结果说明评判器能正确读取干净证据，而非它在真实大脑更杂乱证据上的表现。Opus 5.5 对全部 500 个可解析 take 都给出了金标准裁决，但除两个外置信度均低于 0.95，因此规则几乎不应用它的任何裁决；它在阈值以下将 5 个 pre-horizon take 判为 `incorrect`。Sonnet 5.5 唯一一次错误应用（`counted-world_no_evidence-013`）用自身知识而非证据给一个真实世界命题评分；它还在阈值以下将 13 个 pre-horizon take 判为 `incorrect`。

探索性集合（规则之外）：在 `partial`（声明目标差一定幅度未达成）上，三个模型都回答 `incorrect`（gpt-6.1-sol 20 个中 18 个，以 0.96 或以上应用），因此差额应判 `partial` 还是 `incorrect` 是自动解析前需确定的评分策略。在 `world_counterfactual` 上，gpt-6.1-sol 基于大脑证据对全部 20 个裁决都予以应用，而非真实结果；Sonnet 5.5 和 Opus 5.5 一个都没应用。

18 个无裁决答案中，12 个（Sonnet 5.5 9 个，Opus 5.5 3 个）用尽了评判器 600 token 的完整输出上限，只留下简短或空的可见答案，说明 Anthropic 模型把上限花在了思考上；评判器调用未传思考设置。4 个是 Sonnet 5.5 的 JSON 带有多余的尾部 `,"}`，1 个是 gpt-6.1-sol 的 JSON 缺少右花括号，1 个是 Sonnet 5.5 的答案从 `incorrect` 自我更正为 `correct`，解析器按设计拒绝。为评判器设置思考参数或提高上限，以及对尾部逗号做容错解析，提议留待后续带日期的预注册，此处不改动；每项都会降低覆盖率，但绝不降低正确性。原始行、`summary.json`、`summary.md` 和 `verdict.json` 位于 gbrain-evals（`docs/benchmarks/2026-10-16-take-grading/`）；此处 `results/` 下未提交任何内容。

## 目录内容

| 文件 | 作用 |
|---|---|
| `cases.ts` | 带日期证据的 908 个合成 take 的种子构建器：dev 118，计入 790，覆盖规则内八个集合（resolved correct 和 incorrect、pre-horizon、pre-claim、no evidence、restatement、near-name distractor、无证据的真实世界命题）和两个探索性集合（partial、world counterfactual）。仅使用占位名称。 |
| `generate-fixtures.ts` | `cases.ts` → `fixtures.jsonl`（已提交）。先构建计入集，dev 避开其声明。编辑 case 后需重新生成；无密钥测试会在漂移时失败。 |
| `check.ts` | $0 的仪器检查（计数、占位符、各集合承诺的日期）以及每个 fixture 变成的生产 take 行和证据块。 |
| `score.ts` | 纯评分与汇总：每个裁决的应用与错误、覆盖率、弃权率、Clopper-Pearson 上界、决策规则和排名。 |
| `report.ts` | JSON 报告和 Markdown 表格，包括提议的 `GRADE_TAKES_MEASURED_MODELS`。 |
| `harness.ts` | 运行器：`--check`（$0）、实时（`--model`、`--split`、`--out`、`--max-usd`：每个 fixture 调用生产 `defaultJudge`；计入集需要 `--preregistration <commit>`）和 `--score`。 |

无密钥测试是 `test/eval-take-grading.test.ts`。它守护仪器，而非分数。

## 运行

```bash
# Prove the instrument ($0, no key):
bun evals/take-grading/harness.ts --check

# One live dev run (needs the provider key; stops at the cap):
bun evals/take-grading/harness.ts --model openai:gpt-6.1-sol --split dev --out results/gpt-dev.jsonl --max-usd 1

# A counted run, only after PREREGISTRATION.md is frozen:
bun evals/take-grading/harness.ts --model openai:gpt-6.1-sol --split counted --out results/gpt.jsonl --max-usd 3 --preregistration <commit>

# Score saved runs ($0):
bun evals/take-grading/harness.ts --score results/*.jsonl --json summary.json --md summary.md
```

## 指标

- **应用裁决的错误占比**（预注册）：单侧 95% Clopper-Pearson 上界至多 1%，即至少 299 个干净的应用裁决。
- **覆盖率**：规则内可解析 take 中已应用且正确的比例。
- **弃权率**：规则内 unresolvable take 中未应用的比例。
- **无答案**：无可用裁决（解析失败、提供方不可用）；安全，单独报告。
- **成本与延迟**：每次裁决的 USD 来自 gbrain 价格表；墙钟时间 p50 和 p95。

排名：合格模型优先，按覆盖率排序，并列时取更便宜的模型。