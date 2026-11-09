# CLAUDE.md

GBrain 是面向 agent 平台的个人知识大脑与 GStack 模组。引擎可插拔：PGLite（通过 WASM 嵌入的 Postgres，零配置默认）或托管 Supabase 实例中的 Postgres + pgvector + 混合搜索。`gbrain init` 默认使用 PGLite；文件超过 1000 个时建议 Supabase。GStack 教 agent 如何写代码。GBrain 教 agent 其他一切：大脑运维、信号检测、内容摄取、富化、cron 调度、报告、身份与访问控制。

## 北极星

gbrain 的目标是成为**记忆领域的下一个 Postgres**：为公司大脑和个人 AI 打造测试最充分、覆盖最广、以最少成本服务最多场景的检索 + agent 记忆系统，服务十亿人。每个功能和每次评估都以这一标准衡量。"gbrain 最好"是一个**全系统**主张——需通过完整的 BrainBench 套件（检索、longmemeval、校准……）证明——而非任何单一功能。划定评估范围时，要证明**该功能**为 gbrain 用户带来价值；不要浪费精力去证明 gbrain 的特定算法优于其他算法（那是研究对比，偏离使命）。

## 两条组织轴（先读这里）

GBrain 知识沿两条正交轴组织。用户和 agent 都必须理解这两条轴，否则查询会静默路由错误。

- **Brain**——哪个数据库。你的个人大脑是 `host`。可通过 `gbrain mounts add`（v0.19+）挂载额外大脑（团队发布，各有自己的 DB 和访问策略）。路由：`--brain`、`GBRAIN_BRAIN_ID`、`.gbrain-mount` dotfile。
- **Source**——数据库内的哪个仓库。一个大脑可容纳多个 source（wiki、gstack、openclaw、essays）。Slug 按 source 划分作用域。路由：`--source`、`GBRAIN_SOURCE`、`.gbrain-source` dotfile。

两条轴遵循相同的 6 层解析模式。拓扑图（个人、团队挂载、多团队大脑的 CEO 级）见 `docs/architecture/brains-and-sources.md`，面向 agent 的决策表见 `skills/conventions/brain-routing.md`。

## 架构

契约优先：`src/core/operations.ts` 定义 100+ 共享操作（包括 `volunteer_context`——推送式上下文，见 `docs/guides/push-context.md`——以及七个冻结的 MEMORY_VERBS `recall`/`remember`/`entity`/`synthesize`/`forget`/`context_pack`/`delta`——后两个是 v0.45.7 的环境召回边界动词（预算打包的 pack + "自那以后发生了什么"），七个都标记 `protocol_version: 1`，可通过 `gbrain serve --surface verbs` 单独提供，见 `docs/protocol/MEMORY_VERBS_v1.md` + `docs/guides/ambient-recall.md`）。CLI 和 MCP 服务器均由这一单一来源生成。引擎工厂（`src/core/engine-factory.ts`）动态导入配置的引擎（`'pglite'` 或 `'postgres'`）。Skill 是胖 markdown 文件（工具无关，CLI 和插件上下文均可用）。

**信任边界：** `OperationContext.remote` 区分受信任的本地 CLI 调用方（`remote: false`，由 `src/cli.ts` 设置）与不受信任的面向 agent 的调用方（`remote: true`，由 `src/mcp/server.ts` 设置）。`file_upload` 等安全敏感操作在 `remote=true` 时收紧文件系统限制，未设置时默认严格行为。

**横切不变量（无论改哪个文件都绝不能违反）。** 逐文件细节见 `docs/architecture/KEY_FILES.md`。

- **信任是失败关闭的。** `OperationContext.remote` 在类型上是必需的。任何非严格 `false` 的值都视为远程/不受信任（仅受信任站点用 `ctx.remote === false`；除非显式 false 否则不受信任用 `ctx.remote !== false`）。不要默认设为假值。
- **Source 隔离。** 每个读侧操作都通过 `sourceScopeOpts(ctx)` 路由；优先级为联邦数组（`ctx.auth.allowedSources`）> 标量（`ctx.sourceId`）> 无。不要手写 source 过滤——漏掉一个线程就是跨 source 数据泄露。推论（无作用域检查/有作用域写入）：无 opts 的 `engine.getPage` 匹配任意 source，而 `putPage` 默认 `'default'`——存在性检查 + 写入对必须把读取限定到写入的 source（`getPage(slug, { sourceId: x ?? 'default' })`）。由 `scripts/check-getpage-scoped-write.mjs` 守护（只读首次匹配站点可用退出标记 `gbrain-allow-unscoped-getpage`）。
- **JSONB：绝不 `JSON.stringify` 后转 `::jsonb`。** postgres.js 会双重编码（变成 jsonb 字符串标量）；PGLite 会掩盖此 bug。两种写法都会中招——模板形式（`${JSON.stringify(x)}::jsonb`）和位置形式（`executeRaw(\`…$N::jsonb\`, [JSON.stringify(x)])`，即 #2339 类，曾导致每次同步中止）。修复：向 `engine.executeRaw` 传原始对象 / 用 `executeRawJsonb` / `sql.json()`；或位置路径通过 `$N::text::jsonb` 绑定（按 text 绑定，cast 负责解析）。由 `scripts/check-jsonb-pattern.sh`（模板 grep）+ `scripts/check-jsonb-params.mjs`（位置 AST 扫描器）守护；真正的兜底是 DATABASE_URL 门控的 e2e 一致性测试，因为 PGLite 无法暴露此 bug。完整规则见 `docs/ENGINES.md`。
- **引擎活跃路径的辅助依赖避免运行时动态 `import()`。** 两个引擎中，`src/core/migrate.ts`、`src/core/engine-sql/` 和 `src/core/schema-migrations/` 的辅助模块使用静态顶层导入。例外：`pglite-engine.ts:tryLoadSnapshot` 中快照加载器的惰性 `require()` 簇（惰性是为了让生产构建跳过测试夹具路径；快照哈希读取 schema 导入闭包的文件字节，`src/core/snapshot-schema-inputs.ts`，从不读取已加载模块，因此覆盖率不会歪曲它）以及两个引擎 `initSchema()` 和 `_upsertChunksOnce()` 中的四处 `ai/gateway.ts` 查找，每处都在本地 `try/catch` 内惰性执行，因为急切求值 gateway 庞大的 provider/config 闭包可能把可恢复的默认/config 行回退变成模块加载失败。每个例外在导入行上带 `engine-dynamic-import-ok`；`scripts/check-engine-dynamic-import.sh` 强制执行该规则（也匹配 `require()`）。查历史用 `git log -G'await[[:space:]]+import\\('`，不要用 `-S`：动态转静态的重写可能保留 token 但改变上下文。
- **引擎一致性。** 两个引擎实现每个 `BrainEngine` 方法。已迁移存储域的 SQL 只存在于 `src/core/engine-sql/<domain>.ts`，两个引擎都委托给它（域：`scripts/engine-sql-baseline.tsv` 的 `migrated` 行；`check:engine-sql-ratchet` 在引擎出现新 SQL 时失败；真正方言特定的代码标记 `// engine-sql-ok: <reason>`）。执行器契约、RLS 读取品牌、能力：`docs/ENGINES.md`（"Storage domains and engine-sql"）。由 `test/e2e/engine-parity.test.ts` 固定。前向引用的列/索引放在 `src/core/engine-sql/bootstrap.ts`（由 `test/schema-bootstrap-coverage.test.ts` 守护）。
- **契约优先。** `src/core/operations.ts` 是单一来源；CLI + MCP 由它生成。每个操作带 `scope: 'read'|'write'|'admin'` + 可选 `localOnly`。HTTP 分发在处理器运行前强制执行 scope/localOnly。
- **迁移 + schema 文本。** `src/core/schema-migrations/` 中每个 schema 迁移一个文件（`bun run new:migration <name>`；`migrate.ts` 是运行器；重新生成 `registry.generated.ts`，绝不手工合并）。全新安装 DDL 有一份手工编辑副本，`src/schema.sql` + 其 TS 片段；`bun run build:schema` 生成两个引擎 blob。`CREATE INDEX CONCURRENTLY` 需要 `transaction: false`（Postgres 上先删除无效残留；PGLite 上通过 `sqlFor.pglite` 用普通 `CREATE INDEX`）。
- **多 source。** Slug 唯一性是 `(source_id, slug)`，不是 slug。关键批量操作和反向写入基于复合键；任何 `source_id` 路径拼接前先 `validateSourceId`。
- **一张规范聊天定价表。** 所有付费云聊天/补全价格只存在于 `src/core/model-pricing.ts`（`CANONICAL_PRICING` + `canonicalLookup`）。其他所有表（`anthropic-pricing.ts` 的 `ANTHROPIC_PRICING`、contradictions/cross-modal/skillopt 成本视图）都是派生视图，绝不是手工复制的副本——因此跨表价格漂移在结构上不可能（takes-quality 读 `pricing.overrides`，然后读规范表）。只在 `model-pricing.ts` 更新价格；每个消费者保留自己的未命中策略，而非自己的数字。由 `test/model-pricing.test.ts` 固定（漂移守卫断言每个视图等于规范表）。Embedding 在 `embedding-pricing.ts` 单独定价（单位不同）。
- **模块大小棘轮。** `scripts/module-size-limits.tsv` 固定每文件行数上限（verify 中的 `check:module-size`）：超过上限增长、收缩后超过 50 行陈旧余量、已删除文件仍有行、以及任何未列出的 src 文件超过 1,500 行，都会失败。只能在同一次提交中通过评审可见的 TSV 编辑提高上限；任何剥离都在同一次提交中降低上限。`check:function-size` 以同样方式棘轮函数（`scripts/function-size-baseline.tsv`）：不允许新函数超过 300 行。
- **剥离后的门面保留其表面。** operations.ts（`src/core/ops/*`）、两个引擎（`src/core/engine-sql/*`、`src/core/{postgres,pglite}-engine/*`）、migrate.ts（`src/core/schema-migrations/*`）、doctor.ts（`src/commands/doctor/registry.ts` + `checks/*`）、sync.ts（`src/commands/sync/*`；可复用簇在 `src/core/sync-*`）、serve-http.ts（`serve-http-<area>.ts` 的 `mount<Area>(app, ctx)` 模块）、cli.ts（`src/cli/command-table.ts` + `src/cli/commands/*`）、jobs.ts（`src/commands/jobs/*`，处理器在 `src/core/minions/handlers/*`）、hybrid.ts（`src/core/search/hybrid/*` 阶段）、autopilot.ts 和 skillpack.ts（`src/commands/skillpack/*`）重新导出它们一直导出的所有内容——导入点和已发布包导出从不追逐剥离。新代码放在模块目录，不放回门面（CONTRIBUTING.md "Where does my change go?"）。引擎模块接受窄的显式依赖（绝不是引擎形状的袋子）；源文本守卫读 `test/helpers/source-surface.ts`，flag 注册表生成器的 `facadeExpansion` 让剥离后的 flag 文本保留在每个命令的扫描表面中。
- **覆盖率是诚实测量的。** CI 每次运行都把各 lane 的 lcov（`scripts/merge-lcov.ts`）合并成 PR 语料报告（在 diff 门通过 `COVERAGE_GATE_ENFORCE` 毕业前为建议性）以及包含完整 e2e glob 的 nightly fullCorpus 数字。bun 事实：每个进程唯一 `--coverage-dir`（复用会覆盖 lcov.info）、只有行记录（JSC 省略函数名）、无子进程覆盖率（cli.ts 作为有记录的少算豁免）、从未加载的文件是计数+列表，绝不伪造全文件数学。


## 参考地图（按需加载）

CLAUDE.md 是始终加载的定位 + 分发器。详细参考按需加载——在该领域工作前先读链接文档。

| 你在处理…… | 先读 |
|---|---|
| `src/` 中任何文件（它做什么 + 其不变量） | `docs/architecture/KEY_FILES.md`——选择子系统，再找该文件的条目 |
| 搜索 / 排序 / 混合 / 检索 | `docs/architecture/RETRIEVAL.md` + `KEY_FILES.md` 中的 `search/*` 条目 |
| 搜索模式 / 成本旋钮 | `docs/guides/search-modes.md` |
| 引擎检测 / Postgres 采用 / DB 访问修复 / 降级 serve（`engine status`、`db-repair`、`init --prefer-postgres`、`GBRAIN_DB_ACCESS`） | `docs/ENGINES.md`（"Engine detection and access repair" + "Local Postgres"） |
| embedding 支出门 / 成本门 / `spend.posture` / 关闭开关 | `docs/operations/spend-controls.md` |
| 每月备份覆盖检查（`gbrain backup`、渲染通道、nag 预算） | `docs/operations/backup-check.md` + `KEY_FILES.md` 中的 `backup/*` 条目 |
| 推送式上下文（volunteer/watch/reflex 窗口） | `docs/guides/push-context.md` |
| checkpoint 压缩 / 编译上下文文件（`gbrain compile-context`） | `docs/guides/checkpoint-compaction.md` + `docs/guides/ambient-recall.md` |
| 环境记忆回写（可选加入的无提示事实捕获——`memory.auto_writeback`、harness 指令块、Stop-hook 兜底、读取时 TTL） | `docs/guides/ambient-writeback.md` + `KEY_FILES.md` 中的 ambient-writeback 簇 |
| Memorable 集成 / 会话回执 / relay 同意（`integrations.memorable.*`） | `docs/memorable-agents.md` + `KEY_FILES.md` 中的 hook-heartbeat/capture-spec/codex-hooks 条目 |
| 聊天连接器（实时 ChatGPT/Claude 历史同步——`gbrain connectors`） | `docs/guides/chat-connectors.md` + `KEY_FILES.md` 中的 `src/core/connectors/*` 条目 |
| schema 包 / 页面类型 / 提取 | `docs/architecture/schema-packs.md`、`type-taxonomy.md`、`lens-packs.md` |
| 瘦客户端 / 远程 MCP / 跨模态 | `docs/architecture/thin-client.md` |
| 将大脑的 MCP 服务器发布到其他设备和 agent（`gbrain mcp expose`、默认 Tailscale、Grok Bot / Muse 托管路径） | `docs/guides/remote-mcp.md` + `docs/mcp/DEPLOY.md` + `remote-mcp` skill |
| 记忆动词 / MCP 工具表面（`--surface`）/ 一致性 | `docs/protocol/MEMORY_VERBS_v1.md` + `KEY_FILES.md` 中的 `verbs*`/`surface.ts`/`protocol.ts` 条目 |
| CLI 表面（命令 + flag） | `gbrain --help` / `gbrain --tools-json`，加上相关 `KEY_FILES.md` 条目 |
| 添加存储方法、迁移、doctor 检查、仅 CLI 命令、HTTP 路由或同步阶段 | `CONTRIBUTING.md`（"Where does my change go?"）；重构波 1 之前的分支：`docs/architecture/wave-1-porting.md` |
| 运行、编写或退役测试 | `docs/TESTING.md`（[authoring gate](docs/TESTING.md#authoring-gate)、[retiring](docs/TESTING.md#retiring-a-test)） |
| 批量命令进度接线 | `docs/progress-events.md` |
| 评估方法 / 指标 | `docs/eval/` |
| 大脑 vs source / 拓扑 | `docs/architecture/brains-and-sources.md`、`topologies.md` |
| google 连接器（Gmail/Calendar/Contacts、OAuth）/ 凭据保险库 | `docs/guides/google-connect.md` + `KEY_FILES.md` 中的 `creds/*` + `google/*` 条目 |
| 开放循环 / `gbrain waiting` / 承诺提取 | `docs/guides/open-loops.md` + `KEY_FILES.md` 中的 `loops*` 条目 |
| skill 路由 | `skills/RESOLVER.md` |
| agent 引导（粘贴安装、hook、`gbrain bootstrap`、sweep、无密钥） | `docs/guides/bootstrap.md` + `docs/designs/AGENT_BOOTSTRAP_PLAN.md` + KEY_FILES bootstrap 簇 |
| 发布版本 / CHANGELOG / PR 约定 | `docs/RELEASING.md`（发布 IRON RULES 保留在下方内联） |

## 维护 CLAUDE.md 和参考文档

一个仅追加的逐文件索引曾把 CLAUDE.md 增长到约 592KB。防止复发的规则：

- **CLAUDE.md 是定位，不是实现规范。** 它承载北极星、两条轴、架构 + 横切不变量、解析器以及内联 IRON RULES。逐文件/