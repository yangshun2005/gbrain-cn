# 预注册：哪个模型可以给 take 评分？（2026-10-16）

于 2026 年 10 月 16 日冻结，早于任何计入统计的运行，fixtures 与本文在同一变更中（`fixtures.jsonl`，由 `cases.ts` 生成）。计入统计的运行之后，本文不再改动；后续改动另建带日期的新文件。gbrain-evals 镜像保留一份副本及结果。

## 问题

dream cycle 的 `grade_takes` 阶段（`src/core/cycle/grade-takes.ts`）会遍历尚未解决、且声明已足够久、结果应已见分晓的预测 take，从 brain 中检索带日期的证据，然后要求聊天模型给出一个判定：`correct`、`incorrect`、`partial` 或 `unresolvable`，并附带置信度。判定缓存在 `take_grade_cache` 中。自动解决（把判定写回 take）代码已存在但处于关闭状态，cycle 从不开启。在任何人开启之前，计划（wave 4b，W6）要求先做一次测量：当生产规则会应用某个判定时（不是 `unresolvable`，置信度达到或超过 `GRADE_TAKES_AUTO_RESOLVE_THRESHOLD`，即 0.95），它出错的频率有多高？

损害是不对称的。错误的已应用判定会悄悄改写一个人的历史记录，并影响校准档案。`unresolvable` 或低置信度判定只多一次复核，不会损害正确性。因此本实验问：

1. 在规则会应用的判定中，错误占比是多少，上界有多紧（**已应用判定的错误占比**）？
2. 一个模型能正确且置信度足够高、可被应用地评出多少个可解决 take（**覆盖率**）？
3. 对于证据无法定论的 take，它有多经常选择不判定（**弃权率**），每个判定的成本是多少？

## Fixtures

gbrain 中的 `evals/take-grading/fixtures.jsonl`：908 个结果已知的合成 take，每个都带有评判者看到的带日期证据命中，格式与生产检索器一致（`slug`、`title`、`chunk_text`、`effective_date`）。公司和人物均为占位符（`*-example`）；唯一真实世界主体是 `world_*` 集合中的公共命题（价格、利率、事件），不涉及任何人物或公司。由运行实验的 agent 从固定种子确定性生成；无人审阅。

两个划分。**dev**（118 个 fixture）用于调优 prompt 和解析器，从不计入统计。**counted**（790 个 fixture）用于决策；它先构建，dev 避开其声明。每个公司 take 都声明一个时间范围（`by the end of 2023`、`by Q3 2024`、`within 18 months`、`before 2025`），每个命中都相对于 take 的 `since_date` 和该时间范围标注日期。除 `no_evidence` 外，每个公司 fixture 还带有零到两条关于其他公司的噪声命中（会议改期、租约续签）。

| 集合 | 计入数 | 形态 | 金标准 | 是否在规则内 |
|---|---|---|---|---|
| `resolved_correct` | 250 | 时间范围之后报告结果的带日期命中（60%），或时间范围之前的提前达成（40%）；部分带有更早的进展命中，总在达成之前。 | `correct` | 是 |
| `resolved_incorrect` | 250 | 时间范围之后报告未达成的带日期命中；部分带有更早的进展。 | `incorrect` | 是 |
| `pre_horizon` | 60 | 仅有时间范围之前的进展命中。 | `unresolvable` | 是 |
| `pre_claim` | 40 | 仅有 take 做出之前的背景命中。 | `unresolvable` | 是 |
| `no_evidence` | 40 | 完全没有命中。 | `unresolvable` | 是 |
| `restatement` | 40 | 时间范围之后的命中，持有者重述了声明；无结果。 | `unresolvable` | 是 |
| `distractor` | 40 | 看似决定性的命中实际关于名称相似的另一家公司（`<word>-<suffix>-example`）。 | `unresolvable` | 是 |
| `world_no_evidence` | 30 | 公共真实世界命题，brain 中无证据。 | `unresolvable` | 是 |
| `partial` | 20 | 报告缺口的命中（目标部分达成）。 | `partial` | 否，探索性 |
| `world_counterfactual` | 20 | 已知真实结果的公共命题；brain 的带日期记录说反话。 | brain 一方 | 否，探索性 |

因此规则看到 750 个计入 fixture，其中 500 个可解决。两个探索性集合仅报告、从不决策：`partial` 没有公认的评分标准，`world_counterfactual` 测量模型是遵循 brain 还是自身世界知识，这是本实验不解决的政策问题。

$0 检查（`bun evals/take-grading/harness.ts --check`，以及无密钥测试 `test/eval-take-grading.test.ts`）验证工具：`fixtures.jsonl` 与 `cases.ts` 一致、上述计数、仅占位符名称、各集合承诺的日期（时间范围之后的结果命中，或声明之后、进展在前的提前达成；声明与时间范围之间的 pre-horizon 命中；`since_date` 之前的背景；distractor 的决定性命中关于近名公司且无关于主体的命中；无证据集合无命中），以及每个 fixture 都能通过生产证据块渲染。

## 实验组

依据 gbrain 项目关于 eval 模型的规则（2026-10-07）：每个家族的最新前沿模型，计入统计的运行中不用 Fable，不用 gpt-5.4-mini，不用更早世代。此前无 take-grading 结果，故无需链接模型。

| 模型 | 运行理由 |
|---|---|
| `openai:gpt-6.1-sol` | 最新 GPT。 |
| `anthropic:claude-opus-5-5` | 最新 Opus；计入统计运行中的顶级 Anthropic 模型。 |
| `anthropic:claude-sonnet-5-5` | 最新 Sonnet；更便宜的 Anthropic 候选。 |

每个模型对 790 个计入 fixture 运行一次：共 2,370 个判定。每模型上限（`--max-usd`）：gpt-6.1-sol 为 $3，Sonnet 5.5 为 $5，Opus 5.5 为 $10，合计 $18，低于 $25 的计入预算；dev 轮预计约 $1.20、$2.60 和 $5.80。

## 流程

`bun evals/take-grading/harness.ts --model <id> --split counted --out <file> --max-usd <cap> --concurrency 4 --preregistration <commit>`，Bun 1.4.x，逐模型运行，其中 `<commit>` 是冻结本文件的 gbrain 提交：

1. 每个 fixture 转为生产 take 行和生产证据块（`formatEvidenceBlock`）；评判者不得看到的字段（`decisive`、`about`）先被剥离。
2. 判定来自 `grade-takes.ts` 中的 `defaultJudge`：生产 prompt（`GRADE_TAKES_PROMPT_VERSION` 为 `hybrid-evidence-v2`）、一次使用生产选项的网关 `chat` 调用，以及生产解析器（`parseJudgeOutput`）。harness 仅包装 chat 调用以读取 token 用量。解析失败记为无答案（生产在下一 cycle 重试）；提供方错误在暂停后最多重试两次。
3. 每行记录判定、置信度、规则是否应用、是否错误、tokens、按 gbrain 自身价格表计算的 USD、墙钟时间、评判者推理，以及解析失败时的原始答案。运行的 `.meta.json` 记录 gbrain 提交、本预注册提交、prompt 版本与哈希、阈值，以及 fixture 和 harness 哈希。
4. 达到上限的运行停止，报告为不完整，其模型不合格。计入统计的运行不重复、不延长。

证据检索不测量：命中是给定的，因此这测量的是评判者在它将看到的证据上的表现，而非检索器能否找到。

## 测量

- **已应用判定的错误占比**：规则内 750 个 fixture 上的 错误 / 已应用，附单侧 95% Clopper-Pearson 上界。在金标准为 `unresolvable` 的 fixture 上，每个已应用判定都是错误的。
- **覆盖率**：500 个可解决 fixture 中，已应用且正确的可解决 fixture 占比。
- **弃权率**：250 个不可解决 fixture 中，未应用的不可解决 fixture 占比。
- **无答案**：无判定的 fixture，按失败类型（`parse`、`unavailable`）；安全，take 保持未评分。
- **成本**：每判定 USD；**延迟** p50 和 p95。
- 探索性、从不决策：各集合和类别的结果、`partial` 上的判定，以及 `world_counterfactual` 上判定遵循 brain 还是真实结果。

## 决策规则

当模型至少应用了一个判定，且 错误 / 已应用 的单侧 95% Clopper-Pearson 上界 **至多 1%**（`score.ts` 中的 `MAX_WRONG_UPPER`）时，该模型**合格**。这需要至少 299 个已应用判定且零错误，或 473 个且一个错误。弃权率和覆盖率不进入规则，仅通过已应用计数间接影响。

`GRADE_TAKES_MEASURED_MODELS`（`src/core/cycle/grade-takes-measured.ts`）列出合格模型，**按覆盖率从优到劣排序，并列时取更便宜的模型**（每判定 USD 更低）。若无模型合格，该常量保持为空，报告如实说明。合格不会开启自动解决：本发布中，该列表仅在 `gbrain takes grades` 中标记哪些缓存判定来自已测量的评判者。自动解决是否接入、以何种真实样本检查接入，是另行决定。

覆盖率和弃权率均为 100% 的模型报告为上限。模型按排名顺序报告。

## Dev 结果（背景，不计入统计）

冻结前跑了两轮 dev，用掉 $4 dev 预算中的 $2.87。第一轮使用 prompt 版本 1；gpt-6.1-sol 应用了 63 个判定，4 个错误，全部在 `distractor` fixture 上，它把近名公司的结果评了进去。prompt 增加一段（关于名称仅与声明相似的不同实体的证据不算数）并成为 `hybrid-evidence-v2`；解析器学会读取 JSON 周围有散文的答案，或重复且一致的答案，此前 Sonnet 5.5 在 9 个 fixture 上无可解析答案，多数是先给 JSON 答案再给更正答案。第二轮，prompt v2：

| 模型 | 已应用 | 错误 | 上界 | 覆盖率 | 弃权率 | 无答案 | 每判定 USD |
|---|---|---|---|---|---|---|---|
| `openai:gpt-6.1-sol` | 60 | 0 | 4.9% | 100% | 100% | 0 | $0.0015 |
| `anthropic:claude-sonnet-5-5` | 12 | 0 | 22.1% | 20% | 100% | 1 | $0.0033 |
| `anthropic:claude-opus-5-5` | 0 | 0 | — | 0% | 100% | 1 | $0.0073 |

Opus 5.5 在多数可解决 dev fixture 上给出了金标准判定，但置信度从未达到 0.95 或以上，因此规则一个也未应用。Sonnet 5.5 在其中五分之一上达到 0.95。按此规则，这两者在计入划分上不太可能合格；它们仍运行，因为项目规则要求每个家族的最新模型，且其计入结果将是后续阈值或 prompt 决策的起点。第二轮后 fixtures 重新生成一次，将进展命中置于提前达成之前（一个 dev `resolved_correct` fixture 的进展在其导致的达成之后）；检查现已强制此点。

## 本文不改变什么

计入统计的运行期间不编辑 prompt、解析器、阈值和 fixtures；发现的弱点连同证据提交给后续带日期的预注册。自动解决保持关闭。