# 固定问题收益门禁（B5）

衡量固定问题是否值得：在单一固定种子工作负载下，按冻结的读写比（1 和 10），比较三种方案每得到一个正确、新鲜答案的生命周期总成本（美元）。

| 方案 | 计入的生命周期成本 |
|---|---|
| `pinned` | 初始固定、每次刷新尝试（每批写入后的 `standing_questions` 阶段），以及针对交付的新鲜句子调用读取器 |
| `query_reader` | 按固定答案的交付 token 预算进行 `query` 检索，然后每次读取调用一次读取器 |
| `think` | 每次读取按需调用一次 `think` |

报告给出每种方案的读取次数、正确回答数、`stale_wrong`（答案陈述了已被取代的值）、美元成本及每个正确回答的美元成本、固定成本拆分（固定 + 刷新 vs 读取），以及盈亏平衡读取次数：超过该读取次数后，`pinned` 比 `query_reader` 更便宜。

工作负载（`generateWorkload(seed)`）为每个实体分配一个出厂城市，并在后续写入批次中修正其中一部分；金标准答案是当前城市。对同一种子，工作负载逐字节一致（`workload_hash`）。

## 模式

```bash
bun evals/pinned-questions/benefit-gate.ts --plan --json      # 规模与成本估算，不产生费用
bun evals/pinned-questions/benefit-gate.ts --offline --json   # 桩模型：仅检查管道，不作为证据
bun evals/pinned-questions/benefit-gate.ts --run --yes --max-usd <cap> \
  --answer-model <provider:model> --reader-model <provider:model>   # 付费
```

`--run` 在缺少 `--yes` 或 `--max-usd` 低于计划估算时拒绝执行（退出码 3）。`--entities N` 和 `--batches N` 设定工作负载规模；`--seed` 固定种子。付费运行遵循 gbrain 评测的模型选择规则（各家族最新前沿模型），由当轮负责人调度；`test/pinned-questions-benefit-gate.test.ts` 覆盖离线管道。

仅当此门禁显示收益，且 B5 安全门禁（`test/helpers/pinned-questions-scenarios.ts`）以零泄漏通过时，固定问题才默认开启；仅通过安全门禁时，以可选方式开启。