# 为 GBrain 做贡献

## 环境搭建

```bash
git clone https://github.com/garrytan/gbrain.git
cd gbrain
bun install --frozen-lockfile && bun run test && bun run verify
```

`bun run test` 是并行单元测试循环；`bun run verify` 是 CI 的守卫检查集。[`docs/TESTING.md`](docs/TESTING.md#quick-start) 涵盖 E2E、失败日志和各测试通道。

需要 Bun 1.4.4 或更新版本，与 `package.json` 一致。

### Windows

`bun run test`、`verify`、`ci:local` 和 `test:e2e` 都通过 bash 调度，因此 `scripts/` 下的 shell 脚本必须以 Unix 换行符检出。根目录的 `.gitattributes` 固定了 `*.sh text eol=lf`，覆盖 Git for Windows 默认安装的 `core.autocrlf=true`。全新克隆无需额外步骤即可正确工作。

`.gitattributes` 出于同样原因固定了 `*.md text eol=lf`。frontmatter 读取器以 `---` 围栏加 Unix 换行符为锚点，因此 CRLF 检出会让格式良好的文档被解析为没有 frontmatter。这种失败是静默的：没有报错，字段只是返回空值。

如果你在任一固定规则存在之前克隆，工作副本仍带有旧的 Windows 换行符。Bash 会报 `$'\r': command not found`，frontmatter 会被读取为缺失。从仓库根目录刷新一次。`git reset --hard` 会丢弃未提交的更改，所以先提交或运行 `git stash -u`。

```bash
git rm --cached -r . -q
git reset --hard
bash -n scripts/run-unit-parallel.sh          # 无输出表示 bash 能读取脚本
git ls-files --eol -- '*.md' | grep -cE 'w/(crlf|mixed)' # 0 表示 Markdown 干净
```

`package.json` 中每个 `check:*` 条目都以 `bash scripts/<name>.sh` 调用其脚本，而不是依赖 shebang，因为 Windows 上的 bun 无法直接执行 `.sh`。添加新的 shell 脚本检查时保留该前缀。

## 项目结构

```
src/
  cli.ts                  CLI 入口：op 调度 + 有序的 CLI-only 流水线（handleCliOnly）
  cli/
    command-table.ts      每个 CLI-only 命令一条记录（阶段、瘦客户端模式、懒加载器）
    commands/             每条记录加载的按命令调度模块（仅胶水代码）
  commands/               CLI-only 命令实现（init、upgrade、import、export 等）
    doctor.ts             gbrain doctor 门面（标志解析、注册表运行、输出）
    doctor/               registry.ts（有序检查注册表）+ checks/<topic>.ts 条目
    sync.ts               gbrain sync 门面（重新导出 performSync、performFullSync 等）
    sync/                 同步实现：SyncRun 状态 + 各阶段（preflight、deletes、renames、imports、finalize）
    jobs.ts, jobs/        gbrain jobs 门面（子命令表、registerBuiltinHandlers）+ 每个子命令一个模块
    serve-http.ts         HTTP 服务器：buildServeHttpApp（共享上下文、挂载顺序）+ listen/shutdown
    serve-http-<area>.ts  mount<Area>(app, ctx) 路由模块（oauth、metrics、admin-api、spa、mcp、webhooks）
    autopilot*.ts         Autopilot 模式表、守护进程 tick 步骤、调度、探测
    migrations/           版本化升级编排器（vX.Y.Z、gbrain apply-migrations）
  core/
    operations.ts         操作契约组装（ops/ 的门面）
    ops/                  契约类型 + 安全围栏 + op 领域模块
    engine.ts             BrainEngine 接口
    engine-factory.ts     引擎工厂（动态导入配置的引擎）
    postgres-engine.ts    Postgres + pgvector 实现（门面；已迁移的领域委托给 engine-sql/）
    postgres-engine/      仅 Postgres 的模块（取消、init-schema 锁）
    pglite-engine.ts      PGLite（通过 WASM 嵌入的 Postgres）实现（门面）
    pglite-engine/        仅 PGLite 的模块（检查点守卫、方言特定读取）
    engine-sql/           每个已迁移存储域一个 SQL 实现：executor、方言适配器、
                          sqlFragment、RLS 读取品牌、行规范化器、前向引用引导
    migrate.ts            Schema 迁移运行器（从生成的注册表重新导出 MIGRATIONS）
    schema-migrations/    每个 schema 迁移一个文件（v<NNN>-<name>.ts）+ registry.generated.ts
    pglite-schema.ts      生成的 PGLite schema 模板（pglite-schema.generated.ts）的门面
    doctor-categories.ts  唯一的 doctor 检查类别权威
    page-state/           规范快照、修订、版本和受守卫的投影
    persistence/          持久请求、所有者协调、恢复和写入者强制
    db.ts                 连接管理 + schema 加载器
    import-file.ts        导入流水线（分块 + 嵌入 + 标签）
    sync-*.ts             可复用的同步集群（成本门、git、锚点、锁、协调、状态报告等）
    minions/              作业队列 + worker；handlers/ 为每个内置作业处理器保存一个模块
    types.ts              TypeScript 类型
    markdown.ts           Frontmatter 解析
    config.ts             配置文件管理
    storage.ts            可插拔 blob 存储接口
    storage/              Blob 存储后端（S3、Supabase、本地）
    supabase-admin.ts     Supabase 管理 API
    file-resolver.ts      MIME 检测 + 内容哈希
    sql-query.ts          用于 OAuth/admin/auth 表的仅标量标记 SQL（sqlQueryForEngine）
    bootstrap/            Agent 引导流程（面试、钩子、仓库、验证）
    yaml-lite.ts          轻量级 YAML 解析器
    chunkers/             3 层分块（递归、语义、llm）
    search/               混合搜索：hybrid.ts 运行 hybrid/ 中的命名阶段（request、arms、rank、cache）
    embedding.ts          嵌入服务（按提供商路由；默认 Voyage）
  mcp/
    server.ts             MCP stdio 服务器（从 operations 生成）
    http-transport.ts     HTTP MCP 传输（OAuth、请求体上限）
    dispatch.ts           Op 调度 + 作用域强制 + 参数脱敏
    rate-limit.ts         速率限制
  schema.sql              Postgres DDL，手工编辑（片段区域 + 两个引擎 blob 由 bun run build:schema 生成）
skills/                   面向 AI agent 的胖 Markdown 技能
test/                     单元测试（bun test，无需数据库）
test/e2e/                 E2E 测试（需要 DATABASE_URL、真实 Postgres+pgvector）
  fixtures/               微型真实大脑语料库（16 个文件）
  helpers.ts              DB 生命周期、fixture 导入、诊断
  mechanical.test.ts      所有操作针对真实 DB
  skills.test.ts          第 2 层技能测试（需要 OpenClaw + API 密钥）
docs/                     架构文档
```

门面（`operations.ts`、`doctor.ts`、`sync.ts`、`jobs.ts`、`serve-http.ts`、`hybrid.ts`、`migrate.ts`、`pglite-schema.ts`、两个引擎）继续导出它们一直导出的所有内容；新代码放入模块目录，绝不回到门面。如果你在移植重构第 1 波之前编写的分支，[移植指南](docs/architecture/wave-1-porting.md)将每个移动的符号映射到其新位置。

每个文件的不变量位于 `docs/architecture/KEY_FILES.md`——编辑文件前先阅读其条目。

## 运行测试

测试层级、隔离规则、计时和 E2E 生命周期的规范参考是 [`docs/TESTING.md`](docs/TESTING.md)。简版：

```bash
# 内层编辑循环（Mac 开发机上完整套件约 8 分钟；单文件几秒）
bun run test                      # 并行 4 分片扇出（内存自适应）+ 串行后处理；PGLite 快照默认开启
bun test test/markdown.test.ts    # 特定单元测试

# 推送前门禁（50+ 并行检查 + 类型检查）
bun run verify

# 合并前本地套件（平台/持久化矩阵单独运行）
bun run test:full                 # verify + 并行单元 + 慢速 + 智能 e2e

# 单独运行慢速 / 串行 / e2e
bun run test:slow                 # 仅 *.slow.test.ts（冷路径正确性）
bun run test:serial               # 仅 *.serial.test.ts（按文件池化进程，最重的优先）
bun run test:e2e                  # 真实 Postgres E2E（需要 DATABASE_URL）

# E2E 设置（带 pgvector 的 Postgres）
docker compose -f docker-compose.test.yml up -d
DATABASE_URL=postgresql://postgres:postgres@localhost:5434/gbrain_test bun run test:e2e

# 或使用你自己的 Postgres / Supabase
DATABASE_URL=postgresql://... bun run test:e2e
```

注意：当环境中设置了 `DATABASE_URL` 或 `GBRAIN_DATABASE_URL` 时，裸 `bun test` 会拒绝启动——某些测试会对这些 URL 指向的任何内容运行破坏性 SQL。单元测试运行时取消设置该变量（它们不需要数据库），或使用包装器：单元/慢速运行器在其边界处剥离这些变量，而 `bun run test:e2e` 在自己的边界处选择加入。拒绝消息会引导你完成操作；详情见 [`docs/TESTING.md`](docs/TESTING.md)（"Database-URL run guard"）。如果你将 `bun run test:e2e` 指向自己的 Postgres 或 Supabase，还有第二层限制：数据库名称必须包含 "test" 作为单词段（如上面的 `gbrain_test`），否则破坏性测试拒绝运行——用 `GBRAIN_E2E_ALLOW_DB=<name>` 一次性选择不同名称的数据库。

对持久持久化的更改还需要 [`scripts/persistence/README.md`](scripts/persistence/README.md#test-suites) 中的原生/运行时、进程崩溃、浸泡、部署矩阵和读取延迟门禁。仅 `test:full` 不会执行那些完整的平台和运行时矩阵。将每个结果与其测试的修订绑定，并披露跳过的单元格。

推送前使用 `bun run verify`。它并行运行 50+ 守卫检查（`scripts/run-verify-parallel.sh`），包括：禁止的 fork 名称泄漏（`scripts/check-privacy.sh`）、`JSON.stringify(x)::jsonb` 插值模式（`scripts/check-jsonb-pattern.sh`）、`\r` 进度泄漏到 stdout（`scripts/check-progress-to-stdout.sh`）、测试隔离规则违规（`scripts/check-test-isolation.sh`——见下文"编写能在并行循环中存活的测试"）、编译二进制中静默回退到递归分块（`scripts/check-wasm-embedded.sh`）、过期的管理仪表板构建产物（`scripts/check-admin-build.sh`）、捆绑技能的解析器漂移（`bun run check:resolver`）以及类型检查。守卫 REGISTRY 是 `scripts/guards-manifest.tsv`，`scripts/guard-self-test.sh`（也在 `verify` 中）通过针对已知坏 fixture 运行每个自测试扫描器守卫（清单中 `selftest=yes`；覆盖率从 `todo` 行逐步提升）来证明它确实能失败——新的 `scripts/check-*` 守卫必须在清单中注册，否则构建失败。没有 `check:all` 脚本；尾随换行和 no-legacy-getconnection 检查在 `verify` 中与其他所有检查一起运行，`test/public-exports.test.ts` 拥有包导出映射。

### 编写能在并行循环中存活的测试

`bun run test` 将 1000+ 单元测试文件分片到最多 4 个工作进程，将总并发（分片 × 分片内文件）限制在可用内存内，并在将 OOM 杀死或外部杀死的文件称为失败之前串行重新运行它们（救援通道详情和旋钮见 `docs/TESTING.md`）。同一分片中的文件共享一个进程，因此进程全局状态会在它们之间泄漏。四条 lint 规则（`scripts/check-test-isolation.sh`，R1–R4）强制隔离：禁止直接修改 `process.env`（使用 `test/helpers/with-env.ts` 中的 `withEnv()`），禁止在 `*.serial.test.ts` 之外使用 `mock.module(...)`，每个 `new PGLiteEngine(` 都必须位于规范的 `beforeAll` 块内，并配对 `afterAll(disconnect)`。

**完整规则、规范的 PGLite 块、`withEnv` 模式和 `*.serial.test.ts` 隔离策略位于 [`docs/TESTING.md`](docs/TESTING.md#test-isolation-lint-and-helpers)——编写新测试文件前先阅读。** 早于规则的文件列在 `scripts/check-test-isolation.allowlist` 中；允许列表必须随时间缩小——绝不要添加新条目。

### 区分性测试——每个修复都必须（#3665）

修复的测试只有在**没有修复时失败**才有价值。两种方式都通过的测试比没有测试更糟：它膨胀审查者的信心，永远被加权到 CI 分片中，并在未来重构静默破坏行为后继续通过。（一次对抗性审查发现，某些 PR 中 8 个新测试有 7 个在 master 上通过。）

每个修复行为的 PR 必须在 PR 模板中填写**区分性测试**字段，附上检查此事的实际结果：

> 区分性测试：将 `<源文件>` 回退到 `<ref>`，运行 `<测试文件>` → `N 通过 / M 失败`。恢复 → 全部通过。

辅助脚本用一条命令完成整个流程：

```bash
bash scripts/check-test-discriminates.sh <test-file> <source-file> [<source-file>...]
```

它将源文件回退到修复前状态（纯文件复制，无 git stash），运行测试文件，要求至少一个已执行的测试失败——仅退出码 ≠ 0 不算，因为文件缺失或导入崩溃也会以非零退出（退出码 3，空洞失败类）——恢复，并打印可粘贴的字段行。退出码 1 表示测试在修复回退后通过：在请求审查前收紧断言。

要避免的空洞断言形态（它们反复出现）：
- 仅断言 `exit ≠ 0`（缺失的二进制也会以非零退出）；
- 断言属于覆盖所有可达值的集合（当 `['warn','fail']` 是唯一输出时）；
- 断言也会出现在损坏输出中的子串——改为断言解析后的结构。

对于整个范围的已合并更改或开放 PR，贡献者审计对每个更改运行相同检查，而不使用辅助脚本的整文件回退：它仅在审计头处反转该更改的产品 hunk，因此后续修复保持原位。先在离线 fixture 上尝试：

```bash
bun scripts/contributor-audit-fixture.ts /tmp/audit-demo   # 打印 fixture 范围的审计命令
bun run audit:contributors <base>..<head> [--prs <manifest>] [--json]
```

详情、退出码和清理过的沙箱：[贡献者审计](docs/TESTING.md#contributor-audit)。

添加测试前，回答[编写门禁](docs/TESTING.md#authoring-gate)中的四个问题；删除测试前，遵循[退役测试](docs/TESTING.md#retiring-a-test)并在 PR 正文中记录其证据表。

相关地：仅断言源代码文本上的正则的测试固定的是拼写，不是行为。读取 `src/` 文本的测试（`readFileSync`、`readFile` 或对 `src/` 路径的 `Bun.file`，直接或通过路径常量）需要在读取处或紧上方有标记：`// test-reads-source-ok[<category>]: <why>`，类别为 `prompt-byte`、`trust-boundary`、`generated-artifact`、`structural` 或 `raw-bytes` 之一。`test/test-reads-source-smell.test.ts` 强制执行此规则，并按未论证读取点的确切数量对预先存在的文件进行棘轮。它只计算读取点，因此对现有源绑定的新断言仍需要编写门禁。见[测试中的源读取](docs/TESTING.md#source-reads-in-tests)。

### 本地 CI 门禁（推送前推荐）

```bash
bun run ci:local         # 完整门禁：gitleaks + guards/typecheck + 4 分片并行单元 + E2E
bun run ci:local:diff    # 仅文档差异：gitleaks + 文档检查；否则完整门禁
bun run ci:select-e2e    # 打印差异选择的 E2E 文件（仅文档则无，否则全部）
bun run ci:ubicloud      # 相同门禁扇出到临时 Ubicloud VM（约 5 分钟）
bun run ci:ubicloud:diff # Ubicloud 门禁，带相同的仅文档快速路径
```

`ci:local` 通过 `docker-compose.ci.yml` 启动四个 pgvector 服务加一个事务模式 PgBouncer，运行 PR CI 运行的所有内容加上完整 E2E 套件（4 路并行分片），然后拆除。命名卷在多次运行间保持安装热。需要 Docker（Docker Desktop、OrbStack 或 Colima）和主机上的 `gitleaks`（`brew install gitleaks`）。如果 5434 冲突，用 `GBRAIN_CI_PG_PORT=5435 bun run ci:local`