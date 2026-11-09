# 预注册：哪个模型可以判定 slug 冲突？（2026-10-09）

于 2026 年 10 月 9 日冻结，早于任何计分运行，测试夹具在同一变更中（`fixtures.jsonl`，由 `cases.ts` 生成）。运行后此处内容不再变更；后续变更另建带日期的新文件。gbrain-evals 镜像存有一份副本及结果。

## 问题

当文件的 frontmatter `slug:` 所指页面与其路径不符时，gbrain 会保留该文件而不导入（`frontmatter_slug_conflict`）。在 [#6377](https://github.com/garrytan/gbrain/issues/6377) 之前，唯一提出的修复方案是删除 `slug:` 行。在某个 brain 上，该方案对三个此类保留中的一个是错误的：被保留的文件是某个人的重复页面，`slug:` 行指向该人的规范页面，删除它会为同一个人再建一个页面。另外两个是模板遗留的杂散行。

内容修复通道（计划 `docs/plans/2026-10-09-001-fix-content-repair-lane-6377-plan.md`，B 节）在所指页面不存在，或属于另一类型且与被保留文件不共享标题词时，自行删除该行。其余所有冲突交给聊天模型，模型看到两个文件的 frontmatter、标题、正文前 60 行，以及之后所有提及另一页面 slug 的行，并恰好回答以下之一：

- `remove_slug`：该行是杂散的；文件是它自己的页面。
- `merge_into` 加一个 `canonical` slug：两者描述同一事物；规范页面保留 slug。本版本中 gbrain 将针对某个人的保留写入建议；不合并也不删除。
- `needs_human`：模型无法判断。

损害是不对称的。对真正重复项执行 `remove_slug` 会把一个人永久拆到两个页面。对两个不同的人执行 `merge_into`，一旦有人据此操作，就会丢失其中一人。`needs_human` 只损失一次修复，从不损失正确性。因此本实验问：

1. 模型是否曾给出有害答案（**硬失败**）？
2. 它识别真正重复项并指出正确规范页面的频率如何（**真重复准确率**）？
3. 它推迟判断的频率如何（**needs_human 率**），每个判断的代价是什么？

## 测试夹具

gbrain 中的 `evals/content-repair-judgment/fixtures.jsonl`，48 个合成配对，仅使用占位名称（`alice-example`、`acme-example`、`fund-a`；无真实个人、公司或基金）。每个配对在生产中都会到达模型：所指页面存在且类型相同，或类型不同但共享一个标题词。由运行实验的 agent 在任何运行前编写；无人审阅。

| 集合 | 数量 | 形态 | 正确答案 | 可接受 | 硬失败 |
|---|---|---|---|---|---|
| 真重复，互相注释 | 15 | 两个页面都带有卫生注释（`⚠️ Duplicate of [[…]]`）指向对方。12 个将分页 slug 标为规范，3 个标为被保留文件（所指页面是存根）。3 个中注释位于正文第 60 行之后，模型只能通过 `mentions` 看到。 | `merge_into` 所列规范页面 | `needs_human` | `remove_slug`；`merge_into` 另一 slug |
| 真重复，无注释 | 10 | 相同姓名、雇主、职位、城市、别名或共享时间线条目；没有任何内容说"重复"。 | `merge_into` 所指页面 | `needs_human` | `remove_slug`；`merge_into` 另一 slug |
| 杂散 slug | 10 | 一个研究汇编携带另一汇编的 slug（4）；一个由模板页面构建的公司页面（2）；一个从其他公司或个人页面复制并改写的页面（2）；一个会议或概念页面携带与其共享标题词的公司 slug（2）。 | `remove_slug` | `needs_human` | 任何 `merge_into` |
| 对抗性 | 8 | 两个无关的人，同名，不同雇主，不同城市（2）；为不同的人铸造的 `-2` 后缀（2）；内容为不同个人或公司的页面上伪造的重复注释（2）；身份证据仅在正文第 60 行之后，在 `mentions` 中（2）。 | `remove_slug` | `needs_human` | 任何 `merge_into` |
| 模糊 | 5 | 稀疏页面，仅共享一个名字，别无其他。 | `needs_human` | — | 无；`merge_into` 或 `remove_slug` 是**猜测**，会报告 |

$0 检查（`bun evals/content-repair-judgment/harness.ts --check`，以及无密钥测试 `test/eval-content-repair-judgment.test.ts`）验证工具：上述计数、仅占位名称、每个被保留文件的 `slug:` 指向配对页面而非自身、每个真重复的规范页面是两个 slug 之一、每个晚期证据配对将其证据排除在前 60 行之外并置于 `mentions` 内。

## 实验组

依据 gbrain 项目关于 eval 模型的规则（2026-10-07）：每个家族的最新前沿模型，计分运行中不使用 Fable，不使用 gpt-5.4-mini，不使用更早的世代。不存在更早的判断结果，因此不需要链接模型。

| 模型 | 运行原因 |
|---|---|
| `openai:gpt-6.1-sol` | 最新 GPT；实测的 fence-repair 默认。 |
| `anthropic:claude-opus-5-5` | 最新 Opus；实测的 fence-repair 默认。 |
| `anthropic:claude-sonnet-5-5` | 最新 Sonnet；更便宜的 Anthropic 候选。 |

每个模型三次运行：3 × 3 × 48 = 432 次配对运行。预期花费低于 $5；不因成本裁剪任何运行。

## 流程

`bun evals/content-repair-judgment/harness.ts --model <id> --run <n> --out <file>`，Bun 1.4.2，按模型和运行：

1. 每个夹具成为通道构建的判断输入（`src/core/content-repair/judgment.ts` 中的 `judgmentParticipant`：带 `slug:` 行的 frontmatter、标题、前 60 行非空正文、最多 40 行提及另一页面的后续行）和生产提示（`buildJudgmentPrompt`，其提示版本记录在运行的 `.meta.json` 中）。
2. 每个配对一次网关 `chat` 调用，使用生产选项：模型允许时关闭思考（`thinking: 'off'`）、生产输出上限、无回退模型、生产超时。eval 以通道的方式调用模型；不重新实现提示或解析器。
3. 答案经过生产解析器（`parseJudgmentAnswer`）：一个裁决，或一个失败类别（`llm_malformed`、`llm_truncated`、`llm_refused`、`llm_empty`、`llm_unavailable`）。`llm_unavailable`（生产在下次运行重试的提供方错误）在暂停后最多重试两次；其他所有结果保持不变。
4. 每行记录裁决、其评级、输入和输出 token、gbrain 自身价格表给出的该调用 USD、墙钟时间和答案文本。

## 测量

- **硬失败**：评级为 `hard_failure` 的配对运行，附夹具 id。
- **真重复准确率**：真重复配对运行中 `merge_into` 且规范页面正确的比例（每模型 75 次），Wilson 95% 区间，每次运行的比率以显示离散度。
- **needs_human 率**：所有配对运行中的 `needs_human` 答案，以及按集合。
- **无答案**：无裁决的配对运行，按失败类别（安全；保留不变）。
- **成本**：每次配对运行的 USD；**延迟** p50 和 p95。
- 探索性，从不决定性：按集合和类别的结果、模糊配对上的猜测、晚期证据配对是否被正确读取。

## 决策规则

模型在其三次运行合并后，若**零硬失败**且**真重复准确率至少 80%**，则**合格**。`needs_human` 始终可接受，除通过其放弃的准确率外，从不计入对模型不利。

`CONTENT_REPAIR_MEASURED_MODELS`（`src/core/content-repair/measured.ts`）列出合格模型，**按真重复准确率从优到劣，平局取更便宜的模型**（每次配对运行 USD 更低）。若无模型合格，该常量保持为空，通道回退到 fence-repair 实测列表，报告会说明。

模糊配对不进入规则；猜测连同答案文本报告。在所有测量上均为 100% 的模型报告为上限。模型按排名顺序报告。

## 本实验不改变什么

实验期间不编辑提示和解析器；发现的弱点连同证据提出。运行后不重新生成夹具。