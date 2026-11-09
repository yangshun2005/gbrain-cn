# Forecast-journal 门禁（W7）

私有 forecast journal 的 $0 合成门禁（`src/core/forecast-journal.ts`、`gbrain takes journal`、`gbrain takes scorecard --group-by`）。它在已知技能的占位持有者和占位主题页上检查三件事：

- 每个持有者的统计是否按真实顺序排列持有者；
- journal 能否回答"谁在 T 上说对了"，并保留未解决的分母；
- 远程调用者是否完全看不到这些，同时其现有输出保持不变。

预注册见 [`PREREGISTRATION.md`](PREREGISTRATION.md)。

## 当前结果

**通过。** 测量于 2026-10-17，gbrain `65ed40ec6`（分支 `capy/w7-forecast-journal`），使用该提交中冻结的测量工具和预注册。这是全部 20 个队列、5,622 条预测的一次运行。花费 $0，在内存 PGLite 上耗时 7.4 秒。

| 标准 | 阈值 | 结果 |
|---|---|---|
| 收缩准确率与真实技能的中位 Spearman ρ（12 个持有者，n ≥ 10） | ≥ 0.80 | **0.905**（20 个队列中 18 个 ≥ 0.80；最低 0.741 和 0.748） |
| Who-was-right 集合 F1，800 个主题队列的宏平均 | ≥ 0.95 | **1.000** |
| 分母（journal 总计；每个持有者的 n、未解决、逾期、开放） | 精确 | 0 处不匹配 |
| `low_n` 标记 | 精确 | 0 处不匹配 |
| 标记为 `since_date`、未认证的预测时间 | 精确 | 0 处不匹配 |
| 远程探测 | 全部通过 | **8/8** |

原始准确率的中位 ρ 也是 0.905，因此在这些队列上，收缩对中位排名顺序既无帮助也无损害。它在 20 个队列中的 13 个里将 ρ 改变 −0.084 到 +0.035。n 在 10 到 30 之间的队列，其样本量差异比真实大脑小。收缩对已解决预测很少的持有者影响最大，这些持有者被标记为 `low_n` 并排除在 ρ 之外。

集合 F1 为 1.000 以及精确计数是上限。journal 读取的是给它的解决结果，因此这些标准防范的是回归，而非衡量困难的东西。排名标准是唯一带有抽样噪声的标准。

本次运行的 `summary.json` 和 `summary.md` 未提交在此处。W7 发布时它们会镜像到 gbrain-evals。

## 目录内容

| 文件 | 作用 |
|---|---|
| `fixtures.ts` | 确定性生成器：20 个队列。每个队列有 12 个技能 0.30 到 0.90 的持有者（每人 10 到 30 条已解决预测）、2 个低 n 持有者、每个持有者一条逾期和一条开放预测、一条无日期 take，覆盖 40 个主题页。仅使用占位名称。输出哈希冻结在预注册中。 |
| `score.ts` | 纯统计：Spearman ρ（并列取平均秩）、集合 F1、中位数，以及预注册的判定规则。 |
| `harness.ts` | `--check`（$0 工具检查）和 `--run`（$0，内存 PGLite：加载队列、应用时间范围扫描、通过生产读取器和 op 处理器测量，并应用规则）。 |

无密钥测试是 `test/eval-forecast-journal.test.ts`。它守护的是工具，不是分数。

## 运行

```bash
bun evals/forecast-journal/harness.ts --check
bun evals/forecast-journal/harness.ts --run --json summary.json --md summary.md
```

两者都花费 $0 且无需密钥。`--run` 在门禁通过时退出码为 0，失败时为 1。