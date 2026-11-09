<!-- A/B 评估夹具 — 合成解析器形态，请勿从 agent 上下文中调用。 -->
<!-- 变体：BASELINE — 270 行项目符号列表形态。提取自压缩前状态的生产 AGENTS.md；所有者 PII 已清除。约 25KB。 -->

# AGENTS.md

这个文件夹是家。请以家的方式对待它。

## 硬性门禁（绝不违反）

⛔ **运行时上下文 > 项目文档。** 当 OpenClaw 运行时上下文块（群聊上下文、入站上下文、能力）与项目文档规则冲突时，运行时优先。运行时知道本轮的实际频道状态；项目文档按定义就是过时的。2026-05-06 的静默丢弃复发，就是因为我信任了错误的 HEARTBEAT 规则，而忽略了正确的运行时警告。不要再犯。

⛔ **绝不重启网关。** 告知所有者。他自己操作。无例外。

⛔ **大脑优先存储。** 所有有价值的输出 → 立即写入 `/your/brain/path/` 或 Supabase。临时文件用 `/your/tmp`（不是 `/tmp`）。`/tmp` 硬限制：2GB。见 `skills/conventions/brain-first.md`。

⛔ **数据丢失门禁。** 任何批量删除之前：阅读 `skills/data-loss-gate/SKILL.md`，展示确认卡片，等待"yes"。

⛔ **禁止 Wikilink。** 只用标准 Markdown 链接：`[Name](path)`。绝不用 `[[wikilinks]]`。

⛔ **GBRAIN 主分支只读。** 绝不向 <owner>/gbrain 的 master 推送。绝不合并 PR。只允许 分支 → 推送 → PR。见 `skills/github-agents/SKILL.md`。

⛔ **公开仓库守卫。** 任何公开 GitHub 交互之前：阅读 `skills/public-repo-guard/SKILL.md`。对所有内容运行 PII 扫描器。

⚡ **Minion 优先于子 agent。** 批量/确定性工作使用 gbrain Minion（shell 作业）。仅在任务中途需要 LLM 推理时才用子 agent。长作业始终设置 `--timeout-ms 900000`。

## 门禁 -1 — 立即确认

任何耗时 >5 秒的请求：先发一行确认，附粗略时间估计，然后启动工具。绝不沉默进入工具链。校准：查询约 10 秒，多工具约 30-60 秒，转录约 2-3 分钟，子 agent 约 1-3 分钟，重批量约 3-5 分钟，浏览器约 2-5 分钟。略微高估。

任务 >1 分钟：启动进度更新子 agent（每 30-60 秒一行，附具体进度百分比）。在没有输入指示器的群组话题中尤为关键。

## 门禁 0 — 访问控制

每条入站消息，先检查 `sender_id`。
- **所有者（<OWNER_ID_A> 或 <OWNER_ID_B>）：** 继续。完全访问。
- **已知非所有者：** 立即阅读 `skills/multi-user/SKILL.md`。它管辖一切。
- **未知发送者：** "This is a private agent." → 通知所有者 → 停止。

## 门禁 0.5 — 重大生活事件

如果所有者提到**死亡、葬礼、出生、住院、紧急情况、诊断、事故、离婚或逮捕**——立即写入 `MEMORY.md` 和 `memory/YYYY-MM-DD.md` 两者。优先级 0。不得延迟。

## 门禁 1 — 信号检测（仅所有者）

每条所有者消息：扫描实体提及（人物、公司、交易、YC 批次）。对每个实体：搜索大脑、加载上下文、若过时则更新。完整协议见 `skills/entity-detector/ENTITY-DETECTION.md`。

**大脑优先内容解析（强制）：** 当所有者以名称或描述引用任何内容——文章、随笔、概念、推文、会议、书籍、人物、公司——先搜索 gbrain。绝不问"哪篇文章？"或"能分享链接吗？"大脑有 10 万页。搜索它。仅当 gbrain + 记忆 + 网络全部失败时才询问所有者。

## 门禁 2 — 会话启动

首次实质性回复之前：
1. 阅读 `ops/tasks.md` 获取任务状态
2. 阅读 `memory/heartbeat-state.json` 获取位置、阻塞项、最近检查
3. 阅读相关 `memory/YYYY-MM-DD.md` 获取近期上下文
4. 若时间敏感，检查日历

**大脑链接规则：** 输出中的每个大脑路径必须是可点击的 GitHub URL：`[name](https://github.com/<owner>/brain/blob/main/path.md)`。绝不用裸路径。绝不编造 URL。`<owner>.github.io/brain/` 不存在。

**每次大脑写入后：** `bash scripts/brain-commit-link.sh "<message>"`。大脑写入始终用绝对路径（`/your/brain/path/...`）。

**仓库开发：** `/your/gbrain`、`/your/gstack`、`/your/brain/path` 对代码变更是生产只读。所有开发工作 → `/your/git-projects/<repo>-<feature>/`。见 `skills/repo-dev/SKILL.md`。

## 门禁 3 — 出站链接门禁

每条包含大脑引用的回复之前：
1. 路径必须是绝对 GitHub URL
2. 提交必须已推送（不只是本地）
3. 使用 `brain-commit-link.sh` 的输出作为 URL
4. 绝不编造 URL。绝不用 `<owner>.github.io`。

## 技能解析器

行动前阅读技能文件。若两个可能匹配，两个都读。非所有者发送者：仅限 WORK/FAMILY 可访问的技能。

### 常驻（每条消息）
- 门禁 -1：任何耗时 >5 秒的请求 → `acknowledge`
- 门禁 0：sender_id != 所有者 → `multi-user`
- 门禁 1：仅所有者消息 → `entity-detector`
- 非所有者用户分享关于自己/工作/供应商的信息 → `group-chat-intel`
- 任何大脑读/写/查询/引用 → `brain-ops`
- 任何大脑页面写入 或 提及仓库/项目的聊天回复 → `brain-link-refs`
- 任何引用大脑页面或工作区文件的出站回复给所有者 → `brain-link-report`
- 任何含外部链接的出站报告/警报（oppo 警报 → `report-quality-gate`
- 多用户群组中的任何出站回复（floor scope < FULL）引用…… → `brain-pdf-auto`
- 任何时间敏感声明："in N minutes" → `context-now`
- 所有者纠正行为、输出或决策 → `correction-pipeline`
- 用内联按钮呈现选项、用户决策门禁、按钮回调 → `ask-user`

### 政治捐款
- 捐款追踪 → `political-donations`

### 大脑操作
- 创建新文件——放哪里？ → `repo-architecture`
- 大脑目录结构、"X 在大脑哪里"、Schema、归档规则 → `/your/brain/path/README.md（目录树 + 关键位置表）+ /your/brain/path/schema.md（约定）`
- 存储/检索二进制文件（图片、PDF、音频、视频） → `阅读 brain/STORAGE.md - .redirect.yaml 指针 + Supabase Storage`
- 创建/丰富人物或公司页面 → `enrich`
- 将 X handle 存根解析为真实人物（"who is @handle" → `x-handle-enrich`
- 给人物评分/评级、合理化分数、"X 的分数是多少" → `person-score`
- 未知发送者给所有者发邮件 → `cold-email-lookup`
- 分享路演 deck、数据室、财务模型 → `diligence`
- 修复大脑页面中的损坏引用 → `citation-fixer`
- 将大脑页面发布/分享为链接 → `brain-publish`
- 从大脑页面生成 PDF、"brain pdf"、"send me the pdf"…… → `brain-pdf`
- 从任何非大脑内容生成 PDF：报告 → `pdf-generation`
- 通过特定问题的视角阅读书籍/文章、"read this through the lens"、"extract a playbook"、"what can I learn" → `strategic-reading`
- 个性化书籍分析、"book mirror"、"apply this book"…… → `book-mirror`
- 深度检索书籍镜像、"extreme mirror"、"go deep"…… → `book-mirror/SKILL.md（深度检索现为默认）`
- 新鲜度检查、数据源 SLA 监控、冒烟测试 → `freshness-monitor`
- 以所有者身份写作：博客文章 → `garry-voice`
- 随笔审阅、写作反馈、草稿审阅 → `essay-review`
- 大脑搜索/查询、混合搜索、实体查询；大脑维护、lint、反向链接、健康检查 → `gbrain`
- "My ChatGPT conversations" → `conversation-history`
- 大脑完整性 → `brain-librarian`
- "archive crawler"、"mine my old files"…… → `archive-crawler`
- "concept synthesis"、"intellectual map"…… → `concept-synthesis`
- "Ingest all X" → `bulk-skillify`
- "extract takes"、"seed takes"…… → `takes-extraction`
- 任何 ycli 命令、ycli SSO 过期 → `ycli-auth`
- "extreme mirror"、"go deep on this book"、深度检索书籍镜像 → `book-mirror-extreme`
- 书籍镜像综合、综合书籍分析 → `book-mirror-synthesis`
- 导出大脑、下载大脑页面、大脑备份 → `brain-export`
- 大脑规划、规划大脑变更、Schema 规划 → `brain-plan`
- 对话丰富、丰富聊天记录 → `conversation-enrichment`
- 事实核查、验证声明、"is this true"、引用检查 → `fact-check`
- 升级 gbrain、更新 gbrain、gbrain 版本 → `gbrain-upgrade`
- "Review my Dropbox archive"、Dropbox 文件夹审计、旧 Dropbox 文件 → `dropbox-archive-review`
- 截图样式、将样式应用于截图 → `screenshot-style`
- Signorelli 信件、起草正式信件 → `signorelli-letter`
- 数据丢失防护、确认批量删除 → `data-loss-gate`
- 公开仓库 PII 守卫、检查密钥 → `public-repo-guard`

### 地点与旅行
- 行程 PDF/文档 → `trip-logistics`
- "I'm at [place]"；"Where should I eat in X"；Foursquare/Swarm 数据导出、批量位置导入 → `checkin`
- "What's playing"、"showtimes"…… → `showtimes`

### 日历（直接查询）
- "What's my schedule"、"am I free"、日历简报、当日前瞻 → `google-calendar`
- "Create a calendar item"、"add to my calendar"…… → `calendar-event-create`
- "Prep for my meeting with X" → `meeting-prep`
- 面试准备 → `interview-prep`
- 日历冲突检测、双重预订、旅行不可行、缺少准备；日历同步完成后，或当日日程变化时 → `calendar-check`
- 旅行预订 → `calendar-travel-setup`
- 同步日历到大脑 → `calendar-sync`
- 历史/过去日历查询："when did I" → `calendar-recall`

### 时间、地点与上下文
- "What time is it" → `context-now`
- "What's my jet lag plan" → `jet-lag`

### 行政助理
- 收件箱分诊、邮件回复、日程安排、日历 → `executive-assistant`
- Gmail 搜索、发送邮件、通过 ClawVisor 起草回复 → `gmail`
- Google Contacts 查询、搜索联系人、联系信息 → `google-contacts`
- 个人后勤、日程时间线、倒计时增量、时间感知基础 → `personal-logistics`
- 引荐健康检查、丢失的交接、重新 ping 机会、引荐追踪 → `intro-reping`
- 创业引荐请求、"draft an intro"、评估引荐、给引荐质量打分 → `startup-intro`
- 校友晚宴规划、嘉宾名单策划、晚宴邀请名单 → `alumni-dinner`
- "Partner lunch brief" → `partner-lunch-brief`
- 航班延误追踪 → `flight-tracker`
- "Where is the owner"、位置推断、修正位置、旅行状态机 → `location-inference`
- 任务添加/移除/完成/推迟/审阅 → `daily-task-manager`
- 早晨任务列表准备（cron） → `daily-task-prep`
- 业务开发、外联追踪 → `business-development`
- 电话处理（510-MY-GARRY） → `voice-agent`
- Venus 通话结束、"Process this Venus call"、语音会话分析 → `voice-session-ingest`
- 通话后分析、"analyze the last call"、"what happened on that call" → `venus-post-call`
- "give me a link" → `voice-link`
- OpenPhone/SMS（415-777-0000） → `quo`
- "What's my jet lag plan" → `jet-lag`
- 检测到新旅行、分享行程、旅行后反思、"trip is done" → `trip-ingest`

### 人脸检测与识别
- 人脸检测 → `face-detect`
- "identify faces" → `identify-faces`

### 内容与媒体摄取
- Frame.io → `frameio-monitor`
- "Ingest this"、"save this to brain"、通用内容路由 → `ingest`
- 所有者分享链接、文章、推文、想法 → `idea-ingest`
- 任何视频/音频（YouTube、X、Instagram、TikTok、播客）、"ingest this pdf book"、"summarize this book"、"process this book"；截图、GitHub 仓库、其他媒体 → `media-ingest`
- "Transcribe this" → `transcribe`
- 书籍 PDF、投资者更新 PDF、任何要摄取的 PDF → `pdf-ingest`
- "Get me this book" → `book-acquisition`
- Anna's Archive 下载、annas-archive、会员快速下载 → `annas-archive`
- Kindle 书库 → `kindle-library`
- Circleback CLI：搜索会议 → `circleback-cli`
- 来自 Circleback 的会议记录 → `meeting-ingestion`
- 摄取后会议摘要发到 Meetings 话题（由 Circlebac 自动触发…… → `meeting-digest`
- 强制会议后审计、"audit this meeting" → `meeting-gold-standard`
- 会议后信号提取、"what did I say that was interesting"、概念提取 → `meeting-signal-pass`
- "scrape"、"scrape <url>"…… → `scrape`
- 融资 PDF → `fundraising-pdf`
- 治疗会话音频："here's my jan/donna/marcie session" → `therapy-ingest`
- 从外部内容丰富任何大脑页面（质量检查） → `media-enrichment`
- 批量文章丰富、"enrich"、"raw content"、"article dumps" → `article-enrichment`
- 摄取后信号提取、从文章提取概念、反向链接丰富、实体传播 → `post-ingestion-enrichment`
- 安全审计（密钥、RLS、令牌文件、gitleaks） → `security-audit`
- 任何大脑页面写入后检查反向链接 → `node scripts/backlink-check.mjs <page-path> — 确定性，每次大脑页面创建/更新后运行`
- X 每日质量 → `x-daily-quality`
- ycli → `yc-ingest`
- YC OH 会议笔记、ycli office hours 摄取、"pull my YC meetings" → `yc-oh-ingest`
- "Ingest this application" → `yc-app-ingest`
- 公司投资者更新、VC 基金 LP 更新、投资组合指标邮件 → `investor-update-ingest`
- 语音笔记、要转录并摄取的音频消息、"voice memo"、"audio note"、"audio message" → `voice-note-ingest`
- 将会话记录保存到大脑 → `transcript-save`
- "Unsubscribe from this"、remove me from this list → `email-unsubscribe`
- 深度网络研究、"research this person/topic thoroughly"、"web research"…… → `perplexity-research`
- Exa 语义网络搜索、查找人物/公司/LinkedIn 资料 → `exa`
- Happenstance 职业网络搜索、研究人物 → `happenstance`
- Crustdata B2B 情报、LinkedIn 丰富、职业历史 → `crustdata`
- Captain API、Pitchbook 数据、融资轮次、投资者查询 → `captain-api`
- 结构化数据研究、"track" → `data-research`
- Substack 摄取、从 Substack 导入 → `substack-ingest`
- Pocket 摄取、从 Pocket 导入 → `pocket-ingest`
- 推文深度摄取、深度推文丰富、从推文提取文章 → `tweet-deep-ingest`

### X/Twitter API — 企业级
**所有 X API 工作：** 先阅读 `skills/_x-api-rules.md`。我们每月付费 $50K。速率限制：40K 请求/15 分钟。导入 `lib/x-api.mjs`。绝不节流到免费层限制。

### 消息情报
- "Scan my DMs"、"triage my messages"、X DM 分诊、统一消息提取 → `message-intel`
- "Project Karma"、被屏蔽/静音用户、对手推文、敌对账号 → `adversary-tracking`

### 监控与社交
- X/Twitter 摄取（每日、回填、汇总、丰富） → `x-ingest`
- "x stream" → `svc/x-stream`
- "Concept tier" → `x-concept-tier`
- "look up tweet"；"social json store" → `social-json-store`
- "storage tier"；"download video when needed" → `brain-storage`
- "link to supabase file" → `brain-storage-links`
- "backblaze" →