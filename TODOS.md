# TODOS

## 本地编辑采纳后续事项（2026-11-07 提交，GBRA-341，#8755）

- [ ] **P2 — 在任何源启用前加固 `adopt_local_edits`（#8755）。** **内容：** 采纳过程可能导入合并冲突标记、用编辑器的过期保存覆盖 gbrain 已提交的写入、整读超大文件、在静默窗口期间占用工作树队列、在发布事务回滚时遗留其 Git 提交，以及将嵌套检出的不可读 `HEAD` 视为无仓库。**修复：** 按 #8755 中的逐项处理。**工作量：** M。**优先级：** P2。

## 记忆列车后续事项（2026-11-04 提交，GBRA-286，#8481）

- [ ] **P3 — `extraction_pending` 的本地重链接 argv 可能携带围栏事实文本（#8597）。** **内容：** 运行所有者的重链接操作会将引用围栏存储为事实文本。**修复：** 在构建 argv 前去除引号。**工作量：** S。**优先级：** P3。
- [ ] **P3 — `sync_write_through_off` 的遗忘拒绝没有类型化修复，且读起来像临时错误（#8599）。** **内容：** 代理可能重试而不是询问用户。**修复：** 一个 `ask_user` 修复，指明配置变更。**工作量：** S。**优先级：** P3。
- [ ] **P3 — `forget_fact` 的描述仍说当写穿关闭时撤回会留在数据库中（#8600）。** **修复：** 改写为拒绝。**工作量：** S。**优先级：** P3。
- [ ] **P3 — `checkFactAgainstSaved` 在 #7809 移除后已导出但无调用者（#8601）。** **修复：** 删除它。**工作量：** S。**优先级：** P3。

## SkillOpt E1/B1 后续事项（2026-11-01 提交，GBRA-274，#8408）

- [ ] **P2 — SkillOpt 审查后续事项（#8415）。** **内容：** 中止时的留出延迟和收据的待定冠军分数；未变更技能的最终测试评估；预检将测试评估定价为每任务 2 次运行而非 6 次；恢复不固定 Hook、标准或成本范围；`--resume` 与不可加载的检查点会重新开始；一次性输出上限检查在基线支出后运行；一个被接受的一次性步骤可以替换整个正文；cat31 配置 C 改变了含义；过时的文档和注释。**修复：** 按 #8415 中的逐项处理。**工作量：** 每项 S-M。**优先级：** P2。

## 写容量楔子后续事项（2026-11-01 提交，GBRA-271，#8368）

- [ ] **P3 — 当另一个进程看到漂移的 APFS 创建时间时，`owner_marker_missing` 可能命名普通的自我转移（#8381）。** **内容：** 拒绝在拒绝进程中精确比较预留的创建时间，而自我转移在 CLI 进程中精确比较；在 APFS 上两者可能读取相差数十纳秒（#6501），因此一个长期运行的进程创建了检出并记录了其预留，可能推荐一个自我转移，而 CLI 随后以 `recovery_required` 拒绝。两种方式都不会写入任何内容。**修复：** 双方使用一个身份规则。**工作量：** S。**优先级：** P3。
- [ ] **P3 — 在 `failRefusedClaims` 内报告连接丢失，并在公平份额措辞中命名页面批次（#8383）。** **内容：** 组拒绝路径在临时数据库故障时释放未完成的成员并返回而不重新抛出，因此消费者的错误报告会遗漏它；并且主体公平份额也整体接纳 `put_pages` 或导入批次，而注释和 `write-refusals.md` 没有说明这一点。**修复：** 释放后重新抛出；改写措辞。**工作量：** S。**优先级：** P3。

## RLS 回填后续事项（2026-10-26 提交，GBRA-95，#7625）

- [ ] **P1 — 在无查找表策略的作用域写角色下，守卫触发器会失败开放（#7632）。**
  **内容：** 通过新迁移和 Schema blob，将 `gbrain_refuse_purging_derivation`、`gbrain_mark_derivation_scope_entry`、`gbrain_preserve_fact_withdrawal` 和 `gbrain_guard_inserted_facts` 设为 `SECURITY DEFINER`，由 Schema 所有者拥有并固定 `search_path`。**原因：** 它们以 SECURITY INVOKER 运行，因此一个既不拥有表也不绕过 RLS 的写角色，在受保护表上有写策略但在 `page_segment_derivation_purges` / `page_segment_derivation_purge_pages` / `fact_withdrawals` 上没有策略时，在那里读不到任何行：已清除的派生声明或已撤回的事实可能被再次存储。全新安装已在这些表上启用 RLS；v275 将其扩展到仅迁移的大脑。**测试：** 在作用域 NOBYPASSRLS 非所有者写角色下进行 e2e 红/绿测试（master 上红：写入成功；绿：守卫拒绝）。**工作量：** M。**优先级：** P1。

## 运行子进程堆后续事项（2026-10-24 提交，GBRA-133）

- [ ] **P3 — 历史重写后托管同步/lint 运行子进程的堆增长。** **内容：** #7467 的每子进程 `--max-rss` 上限可以遏制它，但持有者未知；合成 lint --fix 含 15k 持有、并发 60k 文件同步排空加 lint、以及强制推送重写含 15k `file_drift` 持有均保持平稳，且每个模块级缓存都有界。**下一步：** 一个跨多个周期的长期运行子进程，每次发布做堆快照差异。如果 `ChildRssLimitError` 触发，用该作业的数据重新打开。**工作量：** M。**优先级：** P3。

## LHF 第 2 波后续事项（2026-10-15 提交，GBRA-107）

背景：gbrain-evals `docs/plans/2026-10-15-lhf-wave-2/PLAN.md`（"不在范围内"和"第二轮处置"），审计在 `docs/plans/2026-10-15-lhf-wave-2/audit/`。审查者发现位于 `docs/plans/2026-10-15-lhf-wave-2/reviews/`：第二轮 id（`ceo-native-N`、`eng-outside-N`、...）在 `findings.jsonl` 和匹配的 `<phase>-<voice>.md` 中；第一轮发现在 `*.r1.md` 文件中。

- [ ] **P2 — 工具结果大小：一个载荷计划（旧第 8 项和更大的杠杆）。** **内容：** 代理工具结果是剩余的成本杠杆（gbrain 自己的提供商调用约占 Cat 40 账单的 1%；只有 GPT-6.1 Sol 在 Hard world M 上每成功任务比纯文件更贵，0.32 对 0.24 美元）。`chunk_text` 占搜索输出的 59%，重复搜索占 6.5%。精简行默认省略需要结构化的每大脑 `row_defaults`，`query` 可以机器恢复，并保留 `[gbrain dates]` 标签和 `create_safety` 守卫；`context_pack` 压缩需要选择加入的格式或 v2 协议决策，因为 `MEMORY_VERBS_v1` 冻结了卡片字段和 `budget_used`。**门禁：** 在任何支出前决定的、有功效的、预注册的 Cat 40 规则。**发现：** `eng-native.r1.md` F3、F11；`eng-outside.r1.md` F9、F10、F15；`dx-outside.r1.md` F1、F2、F4；`ceo-outside.r1.md` F2、F3；`audit/cost.md`。**工作量：** L。**优先级：** P2。
- [ ] **P2 — 在页面阈值以上跳过 PGLite 关键词计数（旧第 4 项）。** **内容：** 计数在 PGLite 的单一连接上与搜索串行运行，因此推迟它节省约 0 ms；节省在于跳过它。需要一个独立的 `skipped` 状态（不是 `unavailable`）、一个诚实的分母，并保留原始宽限截止时间。**门禁：** "在规模上测量关键词计数"中的规则（`keyword_count_unavailable` 低于 1%，在 55k 槽位上增加的 p50 低于 100 ms）。**发现：** `eng-native.r1.md` F4；`eng-outside.r1.md` F4；`dx-outside.r1.md` F8。**工作量：** S。**优先级：** P2。
- [ ] **P2 — 扩展模型和重写计数（旧第 10 项）。** **内容：** 用于查询扩展的 Haiku 5.5 需要一个由服务和 `gbrain models` 共享的解析器、一个不变的 OCR 回退（扩展模型也是 OCR 回退）以及不变的 init 自动固定规则。自第 0 项以来扩展默认关闭，因此节省很小。与 TODO-D 一起（`TIER_DEFAULTS`、`synthesize`/`think` 在 Opus 4.7 上每次调用 $0.094、其他实用层调用者）；#6335 作为 TODO-D 输入保持开放。**发现：** `eng-native.r1.md` F14、F15；`eng-outside.r1.md` F14；`ceo-outside.r1.md` F6、F7；`dx-outside.r1.md` F6。**工作量：** M。**优先级：** P2。
- [ ] **P2 — 融合后读取扇出、融合预取和 CLI 配置快照（来自第 5 项）。** **内容：** 第 5 项保留了 MCP 每请求配置快照、每次搜索一个设置语句、并发响应元数据读取和并发实体解析。这三者各自需要自己的失败语义和过期设计：一个 `Promise.all` 不能成为融合后排序的一个失败域，扇出需要池-2 检查，CLI 快照需要长期运行命令的过期规则。**发现：** `eng-native.r1.md` F8、F9；`eng-outside.r1.md` F3、F11。**工作量：** M。**优先级：** P2。
- [ ] **P2 — 回复外的 `mcp_request_log` 插入（从第 5 项中移除）。** **内容：** 在回复后写入请求日志行将插入从每个 MCP 调用的回复路径上移开，但有界队列可能丢弃行，且 `serve --http` 中的内存丢弃计数器对 `gbrain doctor`（另一个进程）不可见并在重启时重置，而管理仪表板、每客户端使用量和 `last_used_at` 读取该表。**修复：** 持久化丢失（一个摘要行或 doctor 读取的计数器行）、当操作员需要完整审计行时阻止插入的配置开关、在不完整窗口上限定使用报告、以及等待进行中插入的关闭；测试强制背压、插入失败和从新进程重启。**发现：** `dx-native-1`、`dx-native-2`、`eng-native-9`、`eng-outside-9`、`dx-outside-3`。**工作量：** M。**优先级：** P2。
- [ ] **P3 — 已保存事实阈值移动和门控 OR 重试（未从 #6734 采纳，@troyhoffman-oss）。** **内容：** #6734 将已保存事实匹配阈值移入 SQL，返回旧 200 行上限遗漏的事实，并为严格关键词未命中添加了门控 OR 重试（55dc07cb）。两者都改变搜索返回的结果，因此都不适合一个保持对等性的延迟波。**修复：** 在下面的 trigram 索引条目决定后，各自用检索质量门禁采纳（上限边界的已保存事实召回；Cat 13 和 Cat 40 开发数据上的 OR 重试精度）。**来源：** `audit/latency.md`（#6734 审查）、第 5 项车道报告。**工作量：** 每项 S-M。**优先级：** P3。
- [ ] **P2 — Postgres 上自动表达式统计恢复（第 6 项更重的修复）。** **内容：** 第 6 项在每次索引构建后分析，并在 PGLite 上自动恢复已存在的表达式索引；在 Postgres 上 doctor `planner_stats_stale` 命名它，操作员运行 `gbrain repair planner-stats --apply`，因为 autovacuum 等待表变化 10%。**修复：** 一个有界维护所有者（例如 `post-upgrade` 或常驻 `serve` 为标记的表运行一次 `repairPlannerTable`），首次 ANALYZE 失败后重试，受限角色保持未知，且不向任何搜索添加语句。**发现：** `eng-native-7`、`ceo-outside-4`、`dx-outside-4`、`eng-outside-6`。**工作量：** S。**优先级：** P2。
- [ ] **P2 — 改述的出席问题（第 9 项，被其预注册门禁丢弃）。** **内容：** 收窄的词典（"were present at"、"was in the room for"、"took part in"、"participated in"、"sat in on"、"participants" 名词框架，带资金对象守卫）在 world-v1-attendees 上触发了开发改述拆分 150/150（之前为 0；recall@5 从 0.530 到 0.707），但 30 个独立编写的留出问题集在 90 次运行中触发了 18 次（20%；基线 6/90），低于 80% 的下限：30 个留出问题中有 6 个解析为出席（基线 2）。留出作者使用了诸如 "showed up"、"made it to"、"turned out for"、名册和宾客名单名词等框架。一个竞争意图问题（学校出席）在 master 上已路由到出席。"joined"、"was at" 和 "went to" 会路由就业、教育和位置问题，除非由锚点的类型守卫，而仅解析器守卫仍会改变关键词权重和标题提及排序（`relational-plan.ts` `isRelationalQuery` 馈入 `hybrid/rank.ts` 和 `hybrid/request.ts`）；守卫必须遵循会议类型的 Schema 包别名。**门禁：** 一个新的留出集（2026-10-15 的已用完）、所选关系上零错误路由和返回的证据。**收据：** gbrain-evals `docs/benchmarks/2026-10-16-lhf-wave-2-attendance-preregistration.md` 及其结果。**发现：** `eng-outside-1`、`eng-native-13`、`ceo-native-11`、`ceo-outside.r1.md` F5。**工作量：** M。**优先级：** P2。
- [ ] **P3 — 已保存事实 trigram 索引（第 2 项，被其插入门禁丢弃；来自 #6734，@troyhoffman-oss）。** **内容：** 已保存事实匹配表达式上的 GIN trigram 索引在两个引擎上 ANALYZE 后服务 master 不变的语句（PGLite 在 10k/50k/100k 事实时 p50 19.6 到 3.4 / 94 到 12 / 167 到 23 ms；Postgres 上 32 到 4.9 / 87 到 12 / 83 到 21 ms），但 `fastupdate=on` 时待处理列表使规划器选择 seq scan，`fastupdate=off` 成本 +27-40% 插入 p50 和 -45% 批吞吐量，而 256k-1MB 待处理列表保持 p50/p95/p99 平稳但成本 -11-37% 批吞吐量和 15-30 ms 刷新尖峰，因此每种配置都未通过 +25% 插入门禁。一个搁置的补丁（1MB 待处理列表、自己的重建命令、EXPLAIN 测试）在 GBRA-107 线程中。**修复：** 更便宜的写入路径（例如单独的窄匹配表或仅对事实文本建索引）或单独权衡批量导入的门禁。**工作量：** M。**优先级：** P3。
- [ ] **P3 — 规划器锚点中的年份标记（旧第 9b 项）。** **内容：** 规划器在年份规则运行前拒绝带引号的锚点，因此年份标题例外从不适用于它；修复需要锚点跨度通过解析器保留。**发现：** `eng-native.r1.md` F16；`eng-outside.r1.md` F13。**工作量：** S。**优先级：** P3。
- [ ] **P2 — 通过索引遍历的类型过滤向量搜索。** **内容：** `type`/`types`/日期过滤器将向量语句标记为收窄并禁用遍历（`vector-statement.ts`），因此规划器 seq-scan `content_chunks` 并排序每个候选：Postgres 20k 上 `types: ['email']` 的 `searchVector` p50 为 462 ms 对未过滤 56 ms，PGLite 10k 上 195 对 58 ms。"在真实语料槽位上分析电子邮件类型搜索"中电子邮件类型症状的后半部分。**修复：** 让遍历服务类型过滤器，根据 `