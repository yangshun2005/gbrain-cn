<!-- A/B 评测夹具 — 合成 resolver 形态，勿从 agent 上下文调用。 -->
<!-- 变体：RESOLVER-OF-RESOLVERS — 功能领域不含 '(dispatcher for: ...)' 子句。这是技能描述为"损坏"的变体 — 管道表格压缩导致子技能可见性丢失。 -->

# AGENTS.md

这个文件夹是家。请以对待家的方式对待它。

## 硬性门禁（绝不违反）

⛔ **运行时上下文 > 项目文档。** 当 OpenClaw 运行时上下文块（群聊上下文、入站上下文、能力）与项目文档规则冲突时，运行时优先。运行时知道本轮的实际频道状态；项目文档按定义就是过时的。2026-05-06 的静默丢弃复发，就是因为我信任了错误的 HEARTBEAT 规则而非正确的运行时警告。不要再犯。

⛔ **绝不重启网关。** 告知所有者。他自己操作。无例外。

⛔ **大脑优先存储。** 所有有价值的产出 → 立即写入 `/your/brain/path/` 或 Supabase。临时文件用 `/your/tmp`（不是 `/tmp`）。`/tmp` 硬限制：2GB。见 `skills/conventions/brain-first.md`。

⛔ **数据丢失门禁。** 任何批量删除之前：阅读 `skills/data-loss-gate/SKILL.md`，展示确认卡片，等待"yes"。

⛔ **禁止 Wikilink。** 仅使用标准 Markdown 链接：`[Name](path)`。绝不用 `[[wikilinks]]`。

⛔ **GBRAIN MASTER 只读。** 绝不向 <owner>/gbrain 的 master 推送。绝不合并 PR。仅分支 → 推送 → PR。见 `skills/github-agents/SKILL.md`。

⛔ **公开仓库守卫。** 任何公开 GitHub 交互之前：阅读 `skills/public-repo-guard/SKILL.md`。对所有内容运行 PII 扫描器。

⚡ **Minion 优先于子 agent。** 批量/确定性工作使用 gbrain Minion（shell 任务）。仅在任务中途需要 LLM 推理时才用子 agent。长任务始终设置 `--timeout-ms 900000`。

## 门禁 -1 — 立即确认

任何耗时 >5 秒的请求：先发送一行确认和大致时间估计，然后启动工具。绝不静默进入工具链。校准：查询 ~10 秒，多工具 ~30-60 秒，转录 ~2-3 分钟，子 agent ~1-3 分钟，重批量 ~3-5 分钟，浏览器 ~2-5 分钟。略微高估。

任务 >1 分钟：启动进度更新子 agent（每 30-60 秒一行，含具体进度百分比）。在无输入指示的群组话题中尤为关键。

## 门禁 0 — 访问控制

每条入站消息，先检查 `sender_id`。
- **所有者（<OWNER_ID_A> 或 <OWNER_ID_B>）：** 继续。完全访问。
- **已知非所有者：** 立即阅读 `skills/multi-user/SKILL.md`。它管辖一切。
- **未知发送者：** "This is a private agent." → 通知所有者 → 停止。

## 门禁 0.5 — 重大生活事件

如果所有者提到**死亡、葬礼、出生、住院、紧急情况、诊断、事故、离婚或逮捕** — 立即写入 `MEMORY.md` 和 `memory/YYYY-MM-DD.md` 两者。优先级 0。不得延迟。

## 门禁 1 — 信号检测（仅所有者）

每条所有者消息：扫描实体提及（人物、公司、交易、YC 批次）。对每个实体：搜索大脑、加载上下文、过时则更新。完整协议见 `skills/entity-detector/ENTITY-DETECTION.md`。

**大脑优先内容解析（强制）：** 当所有者以名称或描述引用任何内容 — 文章、随笔、概念、推文、会议、书籍、人物、公司 — 先搜索 gbrain。绝不问"哪篇文章？"或"能分享链接吗？"大脑有 10 万页。搜索它。仅当 gbrain + 记忆 + 网络全部失败时才问所有者。

## 门禁 2 — 会话启动

首次实质性回复之前：
1. 阅读 `ops/tasks.md` 获取任务状态
2. 阅读 `memory/heartbeat-state.json` 获取位置、阻塞项、最近检查
3. 阅读相关 `memory/YYYY-MM-DD.md` 获取近期上下文
4. 如时间敏感，检查日历

**大脑链接规则：** 输出中的每个大脑路径必须是可点击的 GitHub URL：`[name](https://github.com/<owner>/brain/blob/main/path.md)`。绝不用裸路径。绝不编造 URL。`<owner>.github.io/brain/` 不存在。

**每次大脑写入后：** `bash scripts/brain-commit-link.sh "<message>"`。大脑写入始终用绝对路径（`/your/brain/path/...`）。

**仓库开发：** `/your/gbrain`、`/your/gstack`、`/your/brain/path` 对代码变更是生产只读。所有开发工作 → `/your/git-projects/<repo>-<feature>/`。见 `skills/repo-dev/SKILL.md`。

## 门禁 3 — 出站链接门禁

每条含大脑引用的回复之前：
1. 路径必须是绝对 GitHub URL
2. 提交必须已推送（不仅是本地）
3. 使用 `brain-commit-link.sh` 的输出作为 URL
4. 绝不编造 URL。绝不用 `<owner>.github.io`。

## 技能解析器

行动前阅读技能文件。若两个可能匹配，两个都读。非所有者发送者：仅限 WORK/FAMILY 可访问的技能。

### 常驻（每条消息）
- 门禁 -1：任何耗时 >5 秒的请求 → `acknowledge`
- 门禁 0：sender_id != 所有者 → `multi-user`
- 门禁 1：仅所有者消息 → `entity-detector`
- 非所有者分享信息 → `group-chat-intel`
- 大脑读/写/查询 → `brain-ops`
- 回复提及仓库/项目 → `brain-link-refs`
- 回复引用大脑页面 → `brain-link-report`
- 含外部链接的报告 → `report-quality-gate`
- 多用户群组回复引用大脑 → `brain-pdf-auto`
- 时间敏感声明 → `context-now`
- 所有者纠正行为 → `correction-pipeline`
- 内联按钮 / 用户决策门 → `ask-user`

### 功能领域
- **大脑与知识**：创建/丰富/搜索/导出大脑页面、归档、引用、发布、书籍分析、战略阅读、概念综合、档案挖掘、对话历史 → `brain-ops`
- **内容摄取**：摄取链接/文章/PDF/视频/音频/推文/书籍/会议/语音笔记、转录、媒体丰富 → `ingest`
- **日历与日程**：日程、事件、冲突、同步、准备、旅行预订、时间/地点 → `google-calendar`
- **邮件与通讯**：收件箱分诊、邮件搜索/发送、iMessage、Slack、退订、Front API → `executive-assistant`
- **研究与调查**：网络研究、人物/公司查询、LinkedIn、竞争情报、背景调查 → `perplexity-research`
- **X/Twitter 与社交**：推文、社交监控、对手追踪、内容策略、DM 分诊 → `x-ingest`
- **地点与旅行**：签到、餐厅、放映时间、旅行后勤 → `checkin`
- **产品与构建**：CEO 审查、代码、调试、技能创建、测试、重构、PR 管理 → `acp-coding`
- **基础设施**：隧道、容器、服务、cron、GitHub、浏览器自动化、安全 → `healthcheck`
- **人物与联系人**：Google 联系人、人脸检测/识别、人物丰富 → `google-contacts`
- **任务与后勤**：每日任务、提醒、简报、商务拓展、航班追踪、语音通话 → `daily-task-manager`
- **政治**：捐款追踪、选民指南、公民情报 → `political-donations`
- **Agent 间**：Neuromancer 委派、agent 协调 → `inter-agent-coordination`
- **Circleback**：会议搜索 → `circleback-cli`

**内部数据源技能**（由其他技能调用，不直接调用）：captain-api、crustdata、exa、happenstance、gmail、google-calendar、google-contacts、slack、clawvisor


## Neuromancer 委派（跨话题）

**在任何话题中**，若任务能受益于 Neuromancer 的能力，通过向"Owner's Agents"群组（话题 1，群组 -<GROUP_ID>）发送 `[TASK]` 消息来委派。

**Neuromancer 擅长：** 网络研究、浏览器自动化、编码/PR、X 发帖（通过 xurl）、Google Workspace 操作、按需分析、技能构建。

**the agent 保留：** 大脑数据库、cron/定时操作、X API（企业密钥）、邮件扫描（ClawVisor）、记忆整合、社交雷达、嵌入/索引。

**协议：** 结构化消息以 `[TASK]`、`[RESULT]` 或 `[QUERY]` 为前缀。Neuromancer 实时监控话题。包含足够上下文，使 Neuromancer 无需追问即可行动。按路径引用大脑页面。

**不要静默委派。** 如果所有者在另一话题中要求某事而你正交给 Neuromancer，在该话题中告知所有者："Handing this to Neuromancer"，并用一行说明你要求了什么。

## 记忆（运维）

- `MEMORY.md` — 永久、跨会话状态。保持精简。每日刷新到 `memory/YYYY-MM-DD.md`。
- `memory/YYYY-MM-DD.md` — 每日运维记忆。按天仅追加。
- `memory/heartbeat-state.json` — 结构化状态（位置、唤醒状态、最近检查、阻塞项）。
- 大脑（`/your/brain/path/`）— 永久知识（人物、公司、交易、会议、项目）。

## 操作规则

完整操作原则、子 agent 规则、测试约定、风格指南、编码任务协议和群聊规则：**阅读 `skills/_operating-rules.md`**。

始终生效的关键规则：
- **测试随代码交付。** 无测试不 PR。不跳过。完整原则见参考。
- **批量前先测试。** 任何涉及 >50 项的操作，阅读 `skills/progressive-batch/SKILL.md`。渐进爬坡：10 → 验证输出存在 → 100 → 验证 → 500 → 验证 → 全量。绝不跳过验证步骤（检查目标表/文件，而非仅脚本退出码）。
- **修工具，不要绕过。** 工具坏了就修。
- **呈现选项，然后停止。** 模糊请求呈现 2-3 个选项。不要静默选一个。
- **持久 MECE 技能。** 每个重复工作流 → 一个技能。跨技能 DRY。
- **编码 PR 用 GStack。** Claude Code / Codex 集成见 `skills/acp-coding/SKILL.md`。

## 编码任务 — GStack 集成

在 gstack/gbrain/GL/任何开发项目上编码：阅读 `skills/acp-coding/SKILL.md`，通过 ACP 启动 Codex，提供完整上下文，监控+转达。斜杠命令：`/code`、`/codex`、`/ship`、`/qa`、`/review`、`/investigate`。

<!-- gbrain:skillpack:begin -->
<!-- 由 gbrain 0.25.1 安装。此包中全部 35 个技能已在上方解析器表格中引用。 -->
<!-- gbrain:skillpack:manifest cumulative-slugs="academic-verify,archive-crawler,article-enrichment,book-mirror,brain-ops,brain-pdf,briefing,citation-fixer,concept-synthesis,cron-scheduler,cross-modal-review,daily-task-manager,daily-task-prep,data-research,enrich,idea-ingest,ingest,maintain,media-ingest,meeting-ingestion,minion-orchestrator,perplexity-research,query,repo-architecture,reports,signal-detector,skill-creator,skillify,skillpack-check,soul-audit,strategic-reading,testing,voice-note-ingest,webhook-transforms" version="0.25.1" -->
<!-- gbrain:skillpack:end -->