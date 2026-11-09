# 预注册：预测日志能否正确排序和统计预测？(2026-10-17)

于 2026 年 10 月 17 日冻结，早于任何计分运行，与同一变更中的工具一同冻结：`fixtures.ts`（生成器）、`score.ts`（统计与判定规则）和 `harness.ts`（运行器）。运行之后此处不再改动；后续改动另建带日期的新文件。冻结之前，唯一运行过的是在一个不属于 fixtures 的一次性 brain 上对 op 处理器做的管道冒烟测试。它检查了带持有者允许列表的远程上下文能否返回行。未进行任何计分。

fixtures sha256：`ad271efc7fb94cc53170984619bb9a9346c9ca98fdec826e6d8e66046a9fd44a`

## 问题

W7（wave 4b）为 brain 的所有者提供私有预测日志：每条预测及其预测时间、时间范围、状态和解决结果，以及带各持有者统计的 `takes_scorecard {group_by}`。计划对 W7 的门槛是 $0 且合成（gbrain-evals `docs/plans/2026-10-07-10x-memory-advantage/PLAN.md`，wave 4，行 W7）。本次运行提出四个问题：

1. 当持有者能力不同时，按持有者的 Beta 收缩准确率能否按真实顺序对他们排序（**Spearman ρ**）？
2. 日志能否用正确的持有者集合回答"谁在 T 上说对了"（**set-F1**）？
3. 它是否保留未解决的分母（每条逾期、开放和无日期的预测都计入），标记 `low_n` 持有者，并标注每条无法认证的预测时间？
4. 这些对远程调用者是否都不可见，远程调用者已有的内容是否保持不变（**N6 式探针**）？

它不测量任何关于真人的内容。每个持有者和每个页面都是占位符（`holder-NN-example`、`companies/topic-NNN-example`）。

## Fixtures

`fixtures.ts`，种子 20261017，生成 20 个队列。每个队列运行在自己的 source 上：

- **12 个有能力的持有者**，已知能力为 0.30 到 0.90，均匀间隔。能力是一条已解决预测结果为正确的概率。每个持有者有 10 到 30 条已解决预测（均匀分布）。
- **2 个 low-n 持有者**（能力 0.6），各有 4 条已解决预测。
- **每个持有者一条逾期和一条开放预测。** 逾期的那条声明"到 2025 年底"，因此其时间范围早于 as-of 日期 2026-10-16。开放的那条声明"到 2030 年底"。
- **每个队列一条无日期 take**，无截止日期。
- **每条预测都是关于 40 个主题页面之一的押注**，均匀选择。它声明"到 YYYY 年底"（已解决的为 2021 到 2024），声明权重 0.55 到 0.95，since_date 在前一年，解决时间在时间范围之后 0 到 6 个月。80% 的解决结果引用了来源。

总计 5,622 条预测。Takes 通过引擎写入，而非页面写入路径，因此没有信念账本记录它们：每条预测时间都是 since_date，未认证。

$0 检查（`bun evals/forecast-journal/harness.ts --check`，也在无密钥测试 `test/eval-forecast-journal.test.ts` 中）验证工具：

- 生成器的输出哈希等于上面的值。
- 有 20 个队列，每个 14 个持有者。
- 声明的已解决计数与预测匹配。
- 每个持有者和主题都是占位符。

## 流程

`bun evals/forecast-journal/harness.ts --run --json summary.json --md summary.md`，Bun 1.4.3，内存 PGLite，无密钥无网络：

1. 加载每个队列：一行 `sources`，通过 `putPage` 写入 40 个主题页面，通过 `addTakesBatch` 写入 takes，并在一次更新中写入解决结果。
2. 在队列 1 上捕获四个输出。三个来自带持有者允许列表的远程上下文，该列表覆盖队列 1 的每个持有者（一个可读取它们的 N6 式授权）：`takes_list {}`、`takes_search {query: "milestone"}` 和 `takes_scorecard {}`。第四个是本地 `takes_scorecard {}`。
3. 应用声明时间范围扫描（`scanStatedHorizons {apply: true}`），如同 `gbrain takes horizon scan --apply` 所做。再次捕获相同输出。
4. 对每个队列，读取 `scorecardGroups {groupBy: holder, minN: 10}` 和 `readForecastJournal`。对每个主题，读取 `readForecastJournal {about: topic, state: resolved}`，均以 2026-10-16 为 as-of。

## 测量与判定规则

只有当每条标准都成立时，门槛才**通过**：

| 标准 | 测量 | 阈值 |
|---|---|---|
| 排序 | 每个队列，真实能力与 `accuracy_shrunk` 在 12 个 n ≥ 10 的持有者之间的 Spearman ρ（并列取平均秩）；20 个队列的中位数 | ≥ 0.80 |
| 谁在 T 上说对了 | 每个主题和队列，在该主题上有正确已解决预测的持有者（金标准）与日志中 `about: topic` 已解决条目评为 `correct` 的持有者之间的 F1；两个空集记 1 分；800 个主题-队列的宏平均 | ≥ 0.95 |
| 分母 | 日志总计（预测、已解决、未解决、开放、逾期、无日期、认证时间 = 0）以及每个持有者的 `n`、`unresolved`、`overdue`、`open` 等于金标准计数 | 精确 |
| `low_n` | `low_n` 恰好对已解决预测少于 10 条的持有者为 true | 精确 |
| 标注 | 每条预测的时间为 `{at: since_date, basis: since_date, certified: false}` | 精确 |
| 远程探针 | 远程 `takes_list` 和 `takes_search` 非空，不受时间范围扫描影响，且不携带时间范围、预测或日志字段。远程 `takes_scorecard {}` 统计押注，保持不变，且没有 `groups` 或 `journal`。本地 `takes_scorecard {}` 保持不变。远程 `{group_by: holder}`、`{group_by: source}` 和 `{min_n: 5}` 被拒绝并返回 `scorecard_group_by_local_only`。没有 MCP op 名称包含 `journal` 或 `horizon`。 | 全部通过 |

报告但不作判定：各队列 ρ、ρ ≥ 0.80 的队列占比，以及原始准确率与收缩准确率对比的 ρ。计划将 ρ 标准称为诊断：它测试收缩能否在采样噪声中保持真实顺序，而非任何真实持有者是否被良好排序。对真人的排序仍推迟在独立裁决的门槛之后（计划，挑战 6）。

## 本次不改变什么

fixtures、读取器和统计在运行之后不再编辑。如果门槛失败，失败连同其原因一并报告。修复随后进入新变更，在任何重跑之前有新的带日期预注册。