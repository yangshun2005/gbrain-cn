# 内容修复判定评估（#6377）

衡量哪些聊天模型可以判定确定性层级无法裁决的 `frontmatter_slug_conflict` 挂起：被挂起的文件是其 `slug:` 行所指页面的重复（`merge_into` 到规范页）、多余行（`remove_slug`），还是需要人工处理（`needs_human`）？结果决定 `CONTENT_REPAIR_MEASURED_MODELS`（`src/core/content-repair/measured.ts`）。预注册见 [`PREREGISTRATION.md`](PREREGISTRATION.md)；原始结果和结论镜像到 gbrain-evals（`docs/benchmarks/2026-10-09-content-repair-judgment*`）。

## 当前结果

测量于 2026-10-09，gbrain `ac6e0868`（分支 `capy/gbra72-content-repair`，提示词版本 1），使用同一变更中的 harness；每个模型三次运行，共 432 次配对运行，总计 $2.05。所有模型均合格：各自 144 次配对运行中无硬失败、无不可用回答、所有多余 slug 均被移除、所有模糊配对均被推迟、所有对抗性配对均回答 `remove_slug` 或 `needs_human`。差异在于合并而非推迟的真实重复数量。

| 模型 | 硬失败 | 真实重复准确率 | needs_human 率 | 每对 USD | 延迟 p50 / p95 | 合格 |
|---|---|---|---|---|---|---|
| `anthropic:claude-opus-5-5` | 0/144 | 69/75 (92.0%) [83.6, 96.3] | 18.8% | $0.0088 | 3.0 s / 6.4 s | 是 |
| `openai:gpt-6.1-sol` | 0/144 | 64/75 (85.3%) [75.6, 91.6] | 20.1% | $0.0022 | 2.7 s / 7.3 s | 是 |
| `anthropic:claude-sonnet-5-5` | 0/144 | 60/75 (80.0%) [69.6, 87.5] | 24.3% | $0.0033 | 1.6 s / 2.4 s | 是 |

因此 `CONTENT_REPAIR_MEASURED_MODELS`（`src/core/content-repair/measured.ts`）为 `anthropic:claude-opus-5-5`、`openai:gpt-6.1-sol`、`anthropic:claude-sonnet-5-5`，按真实重复准确率从优到劣排列。Sonnet 5.5 恰好卡在 80% 门槛上（各次运行 84%、80%、76%），gpt-6.1-sol 和 Sonnet 5.5 的区间下限低于该门槛，因此两个 Anthropic 模型的顺序已定，其余仅差一个真实重复。模型未合并的每个真实重复都是 `needs_human`，几乎都出现在两条卫生备注各自写着“Duplicate of the other”的配对上：模型将配对读作同一事物，但将备注读作对哪个页面为主存在分歧（见镜像结果行中的 `why` 字段）。身份证据位于正文第 60 行之后的五个配对通过 `mentions` 读取：三个晚备注重复除 `td-n-14` 外均被合并，`td-n-14` 被 gpt-6.1-sol 在三次运行中全部推迟、被 Opus 5.5 在两次运行中推迟；两个晚证据对抗性配对从未被合并（Opus 5.5 在三次运行中全部推迟 `adv-08`）。原始行、`summary.json`、`summary.md` 和 `verdict.json` 位于 gbrain-evals（`docs/benchmarks/2026-10-09-content-repair-judgment/`）；此处 `results/` 下未提交任何内容。

## 目录内容

| 文件 | 作用 |
|---|---|
| `cases.ts` | 48 个手写配对：25 个真实重复（15 个带相互卫生备注，其中 3 个在第 60 行之后；3 个以被挂起文件为规范页；10 个无备注）、10 个多余 slug（汇编页、模板页、复制页、跨类型标题词元）、8 个对抗性（不同雇主的同名、不同人的 `-2` 后缀、伪造的重复备注、身份证据仅在第 60 行之后）和 5 个模糊（共享名字的稀疏页面）。仅使用占位名。 |
| `generate-fixtures.ts` | 确定性构建器：`cases.ts` → `fixtures.jsonl`（已提交）：每个文件的路径、slug 和内容、规范页，以及预期判定集（`best`、`acceptable`、`hard`）。编辑用例后重新生成；无密钥测试在漂移时失败。 |
| `input.ts` | 作为判定输入的 fixture 配对，通过生产参与者构建器（`judgmentParticipant`）：带 `slug:` 行的 frontmatter、标题、前 60 个非空正文行，以及之后提及另一页面的行（`mentions`）。 |
| `check.ts` | $0 仪器检查：集合计数、占位符、slug 行、规范页、`mentions` 内的晚证据。 |
| `score.ts` | 纯评分与汇总：判定等级（`best`、`acceptable`、`guess`、`hard_failure`、`no_answer`）、按模型和按集合计数、Wilson 区间、决策规则和排名。 |
| `report.ts` | JSON 报告和 Markdown 表格，包括提议的 `CONTENT_REPAIR_MEASURED_MODELS`。 |
| `harness.ts` | 运行器：`--check`（$0）、实时（`--model`、`--run`、`--out`：每对调用生产 `askJudgment`；无账本、无备忘）和 `--score`。 |

无密钥测试为 `test/eval-content-repair-judgment.test.ts`。它守护仪器，而非分数。

## 运行

```bash
# Prove the instrument ($0, no key):
bun evals/content-repair-judgment/harness.ts --check

# One live run of one model (needs the provider key):
bun evals/content-repair-judgment/harness.ts --model anthropic:claude-opus-5-5 --run 1 --out results/opus-5-5-run1.jsonl

# Score saved runs ($0):
bun evals/content-repair-judgment/harness.ts --score results/*.jsonl --json summary.json --md summary.md
```

## 指标

- **硬失败**（预注册）：对真实重复执行 `remove_slug`、对多余或对抗性配对执行 `merge_into`，或 `merge_into` 到错误的规范页。要求为零。
- **真实重复准确率**（预注册）：在真实重复配对运行中 `merge_into` 到正确规范页的比例。要求至少 80%。
- **needs_human 率**：所有配对运行中的推迟比例；始终可接受。
- **无回答**：无可用判定（格式错误、截断、拒绝、不可用）；安全，单独报告。
- **成本和延迟**：每对运行的 USD 来自 gbrain 价格表；墙钟时间 p50 和 p95。

排名：合格模型优先，按真实重复准确率排序，并列时取更便宜的模型。