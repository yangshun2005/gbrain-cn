# takes-bootstrap 分类器评估（TODO-E 毕业工具）

这套 100+ 用例的评估用于把关 takes-bootstrap 自动驾驶层级
（`TODOS.md` TODO-E：在该套件的一次实跑通过并**毕业**之前，该层级保持 `manual_only`）。

## 目录内容

| 文件 | 作用 |
|---|---|
| `generate-corpus.mjs` | 确定性构建器：41 个手工编写的原型 × 3 个标签不变的占位符变体 → `corpus.jsonl`（123 个用例）。每个页面都通过生产环境的资格过滤（`briefing` 或 `writing` 类型，正文超过 200 字符：`# <title>`、原型文本，然后是一份共享的后续任务列表，标签将其视为非主张），并带有中性 slug，因此提示词从不提及标签。编辑原型后需重新生成；无密钥 CI 测试会在出现漂移时失败。 |
| `corpus.jsonl` | 已提交的带标签语料。类别：fact / take / bet / hunch / mixed，外加精度类别——empty（无可提取内容）、归因陷阱（他人的观点绝不能作为持有者的 take 出现）、对抗性（内容中的提示词注入、粘贴的 JSON 噪声、过度提取诱饵）。 |
| `scorer.ts` | 纯评分 + 毕业判定（`SCORER_VERSION 1`）：每种类型的精确率 ≥ 0.80 且召回率 ≥ 0.70，零格式错误用例（输出无法解析的用例即为失败，绝不算跳过——分母绝不静默缩小），零 forbid 违规。报告还按变体（123 行）和按原型（41 行）细分结果；当每个预期主张都匹配、每个预测都精确且没有触发 forbid 模式时，该变体通过。 |
| `run-case.ts` | 单个用例走真实生产路径：将页面种子写入 brain，将其 markdown 文件写入 brain 仓库，运行 `extractTakesFromPages`（同意门 → 资格选择器 → 提示词 → `parseClaimsJson` → md 优先的围栏写入 → DB 镜像），并从页面的围栏读回 takes。提取器未分类的用例属于 harness 错误，绝不计分。 |
| `harness.mjs` | 运行器。LIVE 模式用共享评估引导配置网关，针对一次性 PGLite brain 和临时 brain 仓库通过 `run-case.ts` 运行每个用例，并写出预测 JSONL。它先打印花费估算，当估算超过 `--max-usd`（默认 $1）时拒绝运行，并将同一上限作为硬性天花板强制执行。REPLAY 模式以 $0 重新评分已保存的预测文件。无密钥环境会大声拒绝。 |

无密钥 CI 验证位于 `test/eval-takes-bootstrap.test.ts`（语料完整性与生产资格、评分器算术、毕业边界、`--max` 重放切片，以及证明每个标签都可满足的 oracle 通过）和 `test/eval-takes-bootstrap-harness.test.ts`（用打桩分类器通过 `run-case.ts` 运行一个语料用例：分类器被调用，take 落入围栏和 takes 表）。它们守护的是工具，不是分数。

## 运行

```bash
# Live (123 calls on anthropic:claude-haiku-4-5, about $0.10-0.25; needs a chat-capable key):
bun evals/takes-bootstrap/harness.mjs --out results.jsonl [--model <provider:model>] [--max-usd 1]

# Re-score a saved run ($0):
bun evals/takes-bootstrap/harness.mjs --replay results.jsonl

# Bounded smoke (runs and scores corpus.slice(0, 10)):
bun evals/takes-bootstrap/harness.mjs --max 10
```

退出码 0 = 已毕业；退出码 1 = 打印了带失败门槛的报告；退出码 2 =
基础设施问题（无密钥、模型未定价、估算超过上限、达到花费上限，或提取器未分类的用例——部分结果绝不计分）。

## 毕业协议

1. 实跑；将预测 JSONL 与翻转自动驾驶层级的 PR 一同提交（重放模式让凭据永远可重新评分）。
2. 层级翻转 PR 必须引用通过的报告（按类型表格）并划掉 TODO-E。
3. 语料增长：添加原型（而非原始用例），以保持变体标签不变；CI 下限确保每个精度类别都有代表。

遵循北极星评估纪律：这评的是**功能**价值——分类器是否为 gbrain 用户产出正确、归因正确、权重正确的 takes 行——而非模型横评。