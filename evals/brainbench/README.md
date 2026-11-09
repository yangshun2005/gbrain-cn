# BrainBench — 跨 harness 记忆一致性测试套件

BrainBench 按 harness 接缝测量 agent 记忆的四种失效模式，外加四个记忆信任套件（#5575）：

| 套件 | 回答的问题 | 核心指标 |
|---|---|---|
| `know-to-ask` | 记忆是否在应当主动浮现时浮现——在不应浮现时保持沉默？ | `know_to_ask_failure_rate`、`false_fire_rate` |
| `push` | 当上下文被主动提供时，提供的是否是正确的上下文？ | `push_precision`、`push_recall` |
| `write-back` | 对话中的事实是否带着正确的来源信息留存到存储中？ | `write_back_fidelity`、`provenance_accuracy` |
| `continuity` | 在 harness A 中做出的决策——能否在 harness B 中被召回？ | `continuity_rate` |
| `trust` | 每次写入是否都按其通道应得的层级存储和读回——没有向上洗白，也没有自我提升？ | `trust_label_accuracy`、`laundering_violations`、`self_promotion_violations` |
| `state-resolution` | 更新之后，当前事实是否为当前、旧事实是否归为历史、所有者的事实是否免受低层级写入者影响？ | `current_fact_accuracy`、`stale_surfaced_as_current`、`history_preserved`、`lower_tier_supersede_violations` |
| `poisoning` | 来自连接器、工具输出或 agent 的类指令文本，是否会被排除在后续会话的主动上下文之外？ | `poison_persist_rate`、`flagged_and_labeled_rate`、`agent_relayed_activation_rate`、`poison_activation_rate`、`benign_retention`、`false_quarantine_rate`（防护开启）；`default_persist_unlabeled_rate`、`default_activation_unlabeled_rate`、`default_benign_retention`（出厂默认） |
| `deletion` | `forget --purge` 是否从每个活跃存储中移除某条声明、在回执中逐项记录，并阻止其回归？ | `residual_after_purge`、`receipt_completeness`、`resurrection_after_resync` |

此外还有横切指标：`source_isolation_violations`（门禁为零——跨源注入是 gbrain 的数据泄漏不变量）和 `avg_injected_tokens`（入侵诊断，不参与门禁）。

运行方式：

```bash
gbrain eval brainbench --harness all --suite all              # 记分板
gbrain eval brainbench --json --out /tmp/bb.json              # 机器可读
gbrain eval brainbench --compare evals/brainbench/baselines/main.json
gbrain eval brainbench --update-baseline                      # 认可有意的变动
```

默认封闭运行：内存 PGLite、`noEmbed` 种子数据、零 API 密钥、零 LLM 调用。`--llm` 让 write-back 套件接入真实提取器（有预算保护）。

**记忆信任套件通过真实通道写入，绝不使用种子器。** 其 fixture 携带 `trust_steps` 而非 `seed_pages`：`owner` 在 fixture 自有的 git 支持源中写文件并同步（你的笔记），在终端上以本地 CLI 身份确认和清除；`remote_agent` 以 MCP 连接身份调用 `put_page`、`capture`、`remember`、`forget`、`confirm_memory` 和 `purge_fact`；`local_agent` 以无终端 CLI 身份调用它们；`connector` 是 GitHub 连接器，导入合成的 issue 流；`raw` 尝试直接提升层级。一个启用持久化的 brain 运行所有信任 fixture，每个 fixture 一个源（及仓库），每一步都等待其提交后效果。poisoning fixture 的轮次随后作为后续会话，通过每个 harness 接缝和 `context_pack` 重放。Gold（`trust.items`）指明每一步应产生什么；每个指标的公式见 `docs/eval/BRAINBENCH.md`。

## 目录结构

```
fixtures/   *.fixture.json   adapter 可见的对话（schema/fixture.schema.json）
gold/       *.gold.json      封存的 gold 标注（schema/gold.schema.json）
schema/     JSON Schemas     发布的交换契约（fixture/gold/result/baseline）
baselines/  main.json        提交的 CI 门禁基线（diff 稳定，仅指标）
generator/  gen.ts           确定性语料生成器（种子 42）
_ledger.json                 语料元数据：计数、种子、重建命令
```

**封存的 gold。** fixture 文件只包含 adapter 可见的内容。Gold 位于 `gold/`，通过 `fixture_id` 关联；fixture 轮次内出现 `gold` 键是校验错误。harness 向 adapter 传递经过净化的 `PublicTurn`。

**留出集。** 约 15% 的 fixture 带有 `holdout: true`——排除在 CI 门禁之外，仅在发布运行中评分（`--include-holdout`）。用于抗博弈。

## 语料

生成而非手写：`bun evals/brainbench/generator/gen.ts` 可逐字节重建已提交的语料（Mulberry32 PRNG，种子 42）。虚构宇宙（约 40 人、约 30 家公司、约 12 只基金）完全由精选的合成名称池凭空创造——不映射任何真实人物、公司、基金、交易形态或时间线（场景隐私规则；见下文"Fixture 编写"）。

**文本由模板配合 PRNG 选定的变体合成——刻意不经 LLM 处理。** 一致性基准的难度必须可控，而非偶然：模板精确布置大小写模式、近似别名、停用词碰撞、有歧义的共享别名，以及超出预算的实体数量。若干 know-to-ask 变体刻意触发已记录的 v1 反射限制（小写提及、仅姓氏引用——见 `src/core/context/entity-salience.ts`）；gold 记录应当发生什么，基线记录当前系统实际做什么，差距就是被测量的路线图。可选的 Opus 文本润色层（缓存键控、成本上限）是已记录的未来扩展。

手写的尖刺 fixture（`kta-001`、`kta-002`、`ms-001`、`wb-001`、`cont-001-*`）在生成器扩展之前冻结了 schema；它们仍是语料的一部分。`ms-002-nondefault-active` 也是手写的：它是唯一带有非 `default` 的 `active_source` 的 fixture（生成器将 gen-ms fixture 固定为 `active_source: default`），用于测试跨源泄漏检测器的另一条分支——一个孪生 slug 同时种入 `default` 和 `teambrain`、一个仅 teambrain 的页面，以及一个仅 default 的泄漏金丝雀，以 `active_source: teambrain` 重放。

## Fixture 编写（欢迎贡献）

1. 符合 `schema/fixture.schema.json` + `schema/gold.schema.json`（`bun test test/brainbench-fixtures.test.ts` 校验语料）。
2. **字符串隐私：** 仅使用占位/合成名称；日期在 2024–2026 内；用 `$X` 代替真实金额（`scripts/check-synthetic-corpus-privacy.sh` 在 `bun run verify` 中强制执行）。
3. **场景隐私：** 凭空创造场景。不要对真实情境做匿名化——不得有真实交易形态、金额模式、时间线映射或可识别的基金行为。如果换个名字你人脉中的某人仍能认出该情境，请重写。
4. write-back fixture（及 continuity 写入者）需要每个轮次都有 `ts`——分段基于时间。超过 30 分钟的间隔会切分段落。
5. Continuity 对：每个 `pair_id` 恰好一个 `writer` + 一个 `reader`；reader 的 gold 携带决策探针。
6. 记忆信任 fixture 只运行一个信任套件，只通过 `trust_steps` 写入（加载器在此拒绝 `seed_pages`/`seed_facts`），任何地址都使用保留域名（`.invalid`、`*-example`）。每个 poisoning 载荷和良性条目都携带一个语料唯一的金丝雀令牌，且恰好出现在一个步骤中，以便评分器能在每个存储和表面中追踪它。不要在门禁捕获之前调优载荷：门禁漏掉的载荷正是该套件存在的意义所在。

### Gold 约定（经盲双标注验证，一致率 96.4%）

- **事实必须被陈述。** 每个 `gold_facts[].fact` 都是对轮次文本中实际存在的声明的忠实复述——绝不是推断的理由或模板残留。
- **重提是降级，不是检索。** 某轮次重提一个其指针刚刚浮现过的实体，为 `should_retrieve: false`（抑制是正确行为）；当轮次混合新实体与重提时，新实体是 gold，重提的进入 `acceptable_slugs`。
- **承载信息的页面是 gold；被提及但次要的页面是可接受的。** 在多实体 push 轮次中，承载答案的人物/基金页面是 `gold_slugs`；被逐字提及但次要于提问的公司可放在 `acceptable_slugs`（注入 = 可以，遗漏 = 不扣分）。
- **"刚开完电话会"式开场白不携带 gold。** 会议发生过低于显著性门槛；其中的声明才是事实。

## 从其他仓库复用 BrainBench（gbrain-evals）

完整的外部运行器接口是子进程 CLI 契约：

```bash
gbrain eval brainbench --fixtures <DIR> --gold <DIR> --json --out result.json
```

指向任何符合规范的语料；按 `schema/result.schema.json` 解析 `result.json`。退出码：0 通过，1 回归（配合 `--compare`），2 错误/不确定。兄弟仓库 gbrain-evals 将其接入为 `eval/runner/brainbench-memory.ts`。

## 基线治理（CI 门禁如何工作）

CI 将 HEAD 的运行与 **main 的** `baselines/main.json` 副本比较（通过 `git show origin/master:...` 获取——PR 无法改写它所比较的对象）。相同 `fixtures_hash` ⇒ 计数感知门禁（任何新失败的 gold 项即失败）。不同哈希（你改了 fixture）⇒ 语料认可模式：门禁验证你提交的基线与 HEAD 的实际运行完全一致，fixture diff 由审查者判断。任何相对 main 基线的指标回归都要求在更新的基线中提供 `justification` 字符串——在你的 PR diff 中可见。方法论：`docs/eval/BRAINBENCH.md`。