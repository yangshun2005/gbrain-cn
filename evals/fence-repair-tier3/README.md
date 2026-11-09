# Tier 3 围栏修复评估（#6188 T4）

衡量聊天模型通过生产 Tier 3 路径，修复自由修复规则（Tier 1）遗留的格式错误围栏的事实与围栏处理能力，以及哪些模型足够准确、可作默认。预注册、其第 2 轮修订、逐项原始结果与判定存于 gbrain-evals（`docs/benchmarks/2026-10-06-fence-repair-tier3*`）。

## 当前结果

第 2 轮，2026-10-06 在 gbrain `7d75e08c` 测得（提示词版本 2：模型可回答 HOLD；仅在门 (a) 或 (e) 之后纠正性重问；Tier 1 `stray_empty_cell`；额外文本单元格与拆分声明在模型之前被扣留）。每个模型三次运行。门通过率与误接受率统计到达模型的围栏：40 个留出可修复围栏中的 33 个（每模型 99 次尝试），以及 66 个第 1 轮可修复围栏中的 55 个（165 次）。预注册门槛为门通过率至少 80%、误接受率至多 1%，模型须在两组上同时达标才算合格；留出集在第 2 轮代码之前编写，由它决定。

| 模型 | 留出门通过 | 留出误接受 | 第 1 轮门通过 | 第 1 轮误接受 | 每次修复 USD | 延迟 p50 / p95 | 合格 |
|---|---|---|---|---|---|---|---|
| `openai:gpt-6.1-sol` | 95/99 (96.0%) | 0/99 | 156/165 (94.5%) | 0/165 | $0.0026 | 3.7 s / 7.1 s | 是 |
| `anthropic:claude-opus-5-5` | 95/99 (96.0%) | 0/99 | 157/165 (95.2%) | 0/165 | $0.0081 | 3.9 s / 10.9 s | 是 |
| `anthropic:claude-fable-5-1` | 93/99 (93.9%) | 0/99 | 154/165 (93.3%) | 0/165 | $0.0210 | 6.6 s / 17.7 s | 是 |
| `anthropic:claude-opus-4-7` | 96/99 (97.0%) | 0/99 | 162/165 (98.2%) | 3/165 (1.8%) | $0.0077 | 1.8 s / 4.5 s | 否 |
| `anthropic:claude-sonnet-5-5` | 96/99 (97.0%) | 3/99 (3.0%) | 144/165 (87.3%) | 3/165 (1.8%) | $0.0031 | 1.7 s / 3.0 s | 否 |

显示的是留出集的 USD 与延迟。因此 `FENCE_REPAIR_MEASURED_MODELS`（`src/core/fence-repair/measured.ts`）为 `openai:gpt-6.1-sol`、`anthropic:claude-opus-5-5`（Fable 5.1 也合格但被排除：Fable 只在冒烟测试中运行，从不作为计数或默认模型）：当 `models.fence_repair` 未设置时，OpenAI 密钥得到 `gpt-6.1-sol`，Anthropic 密钥得到 `claude-opus-5-5`（它在留出门通过率上与 `gpt-6.1-sol` 持平，在门通过率、成本和速度上优于 Fable 5.1）。三个合格模型的每次留出可修复尝试都是 HOLD；每次误接受都是错误列中的自由文本单元格。自由层精确修复了每个 stray-empty-cell 围栏（每模型 18 次留出和 30 次第 1 轮尝试）。没有回答耗尽输出 token，也没有发生重问。

第 1 轮（提示词 v1，2026-10-06 在 `171a7e24`）发现当时的默认模型 `claude-opus-4-7` 在 198 次修复中有 8 次写错单元格（4.0%）；其发现促成了第 2 轮代码。

## 目录内容

| 文件 | 作用 |
|---|---|
| `cases.ts` | 第 1 轮：78 个手写用例。每个是一页，其围栏在第 1 轮 Tier 1 后留有模型可清除的残留原因（`short_row`、`no_header`、`row_before_header`、`extra_cells`、`header_unmapped`），整个修复段落为手写。集合：66 个 `repairable`（存在正确修复且通过所有门）、9 个 `adversarial`（唯一正确结果是保持扣留：6 个 `ambiguous` 有两种门无法区分的读法，3 个 `unrecoverable` 缺少必需值）和 3 个 `gate_limited`（人会修复，但门禁止正确表格；诊断用）。自预注册运行以来未变；自由层现在决定其中 17 个（`oracle.ts` 中的 `FREE_TIER_PATH`）。仅占位名称。 |
| `heldout-cases.ts` | 第 2 轮留出集：52 个用例，写于第 1 轮之后、任何第 2 轮代码结果之前，在新页面上有相同类别组合（40 个可修复）外加 6 个 ambiguous、4 个 split-claim（声明被未转义管道切成两半；正确结果是保持扣留）和 2 个 unrecoverable 对抗用例。自由层决定其中 12 个。 |
| `generate-fixtures.ts` | 确定性构建器：`cases.ts` → `fixtures.jsonl` 和 `heldout-cases.ts` → `heldout.jsonl`（均已提交）。编辑用例后重新生成；无密钥测试在漂移时失败。 |
| `run-case.ts` | 一个 fixture 走生产路径：页面成为存储页目标，`analyzeFences` 运行自由层，`runTier3` 用大脑的真实每日账本和尝试存储发起模型调用（提示词 v2，模型可回答 HOLD，记为 `llm_declined`；每围栏一次调用；仅在门 (a) 或 (e) 之后纠正性重问；门 (a)-(g)；Tier 1 不动点）。自由层修复或扣留的围栏按该层报告。不重新实现任何层或门。 |
| `oracle.ts` | $0 标签检查：脚本化模型以每个真值（或对抗探针）作答，真实门必须与标签一致；自由层必须按所列决定 `FREE_TIER_PATH` fixtures；对每个 Tier 3 fixture 的 HOLD 回答必须记为 `llm_declined` 扣留。 |
| `score.ts` | 纯评分：单元格级匹配规则、预注册 Tier 3 比率、全路径比率（自由层修复计为已修复）、每层错误写入、对抗围栏如何被扣留、Wilson 区间与决策规则。 |
| `harness.ts` | 运行器：`--oracle`、实时（`--model`、`--run`、`--out`）和 `--score`；每种模式接受 `--fixtures fixtures.jsonl`（默认）或 `--fixtures heldout.jsonl`。 |

无密钥测试是 `test/eval-fence-repair-tier3.test.ts`（两组 fixture 新鲜度与覆盖下限、两组 oracle、评分器算术）。它守护仪器，而非分数。

## 运行

```bash
# Prove every label against the production gates ($0, no key):
bun evals/fence-repair-tier3/harness.ts --oracle
bun evals/fence-repair-tier3/harness.ts --oracle --fixtures heldout.jsonl

# One live run of one model on a fresh throwaway brain (needs the provider key):
bun evals/fence-repair-tier3/harness.ts --fixtures heldout.jsonl --model anthropic:claude-opus-5-5 --run 1 --out results/heldout-opus-5-5-run1.jsonl --max-usd 10
bun evals/fence-repair-tier3/harness.ts --model default --run 1 --out results/default-run1.jsonl   # gbrain's own models.fence_repair resolution

# Score saved runs ($0), one set at a time:
bun evals/fence-repair-tier3/harness.ts --score results/heldout-*.jsonl --json summary-heldout.json
```

实时运行会在其大脑上设置 `models.fence_repair`（`default` 时未设置，解析为第一个有提供商密钥的已测量模型，无则拒绝运行），在 `pricing.overrides` 中注册 gbrain 价格表缺失的标价（如同 `gbrain pricing set` 所为；目前无），并将 `fences.repair.max_usd_per_day` 设为 `--max-usd`，每日账本将其作为硬上限执行。每页上限保持生产默认，因此上限无法覆盖的调用会像生产中一样被拒绝。提供商错误在暂停后重试两次，如同下一次维护运行会重试。每行结果记录层与结果（已修复，或以其门或失败类别扣留）、与真值的匹配、每次调用 token、账本计价 USD、延迟和模型回答文本。

退出码：0 完成，1 oracle 违规，2 基础设施（无密钥或无已测量模型、达到上限、fixture 已编译）。

## 指标

- **门通过率**（预注册）：到达 Tier 3 且修复通过所有门的可修复围栏，除以到达 Tier 3 的可修复围栏。HOLD 计为未修复。
- **误接受率**（预注册）：其中与真值不匹配的修复。匹配指相同列中相同非空单元格文本（忽略空格）且围栏外文本相同。
- **全路径**（报告）：对每个可修复围栏的相同比率，自由层修复计为已修复；**端到端**另加每层错误写入。
- **正确扣留**（报告）：保持扣留的对抗围栏，按种类和方式（模型之前、HOLD、门、输出预算）。
- **每次修复 USD**：到达模型的可修复围栏上按账本计价的花费除以修复数。**延迟**：发起调用的每围栏 Tier 3 步骤墙钟时间（p50、p95）。