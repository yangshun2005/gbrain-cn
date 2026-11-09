

> **[中文化] gbrain**
>
> 此项目是 [garrytan/gbrain](https://github.com/garrytan/gbrain) 的中文翻译版本。
> - 原项目 Stars: 0
> - 主语言: 
> - 许可证: 
> - 翻译日期: 2026-11-10
> - 原始 README: [README_en.md](README_en.md)
>
> 说明：下方 README 正文为**原项目文档的中文翻译**，力求忠于原文；本页顶部的「中文快速上手」为**面向中文用户的补充导读**（非原文内容），目的是让你不用读完整篇 README 就能先把项目跑起来。
>
> 如有翻译不准确之处，欢迎提 Issue 或 PR。

---


## 🚀 中文快速上手（补充导读，非原文）

### 这是什么 / 解决什么问题

GBrain 是一个 Postgres 原生的个人知识大脑（personal knowledge brain），带混合 RAG 检索能力。它给你已经在用的 AI agent 提供一份「你自己掌控的记忆」：存储带来源的明确事实，支持更正与撤回，并让同一份记忆在多个 agent 之间共享。它解决的是 agent 记不住上下文、每次都要重新读资料的问题——你可以直接问「明天见 Alice 前我需要知道什么」，而不是拿回一堆相关页面。

### 快速上手（最短路径）

以下为材料中给出的**托管版（gbrain.io）**最短路径命令，照抄自 `docs/install.md`：

1. 启动托管版 GBrain：
   ```bash
   curl -sSL https://gbrainio.terminalwire.sh | bash
   ```
2. 在新终端中连接：
   ```bash
   gbrainio terminal connect
   ```
3. 在浏览器中选择套餐并创建工作区：
   ```bash
   gbrainio workspace new
   ```
4. 安装 Bun（若已装 Bun 1.4.4+ 可跳过）并加入 PATH：
   ```bash
   curl -fsSL https://bun.sh/install | bash
   export PATH="$HOME/.bun/bin:$PATH"
   ```
5. 安装 gbrain CLI（仅 CLI，不含本地 brain）：
   ```bash
   bun install -g github:garrytan/gbrain#stable
   ```
6. 接入你的 agent（以 Claude Code 为例）：
   ```bash
   (claude mcp get gbrain >/dev/null 2>&1 || claude mcp add --transport http --scope user gbrain https://gbrain.io/mcp) && gbrain setup claude-code --instructions-only
   ```

> 若想自托管本地 brain，原文提到「Install in 30 minutes」，但材料中未给出本地自托管的完整安装命令，请参考 README 正文与 `docs/install.md`。

### 环境要求

- **操作系统**：原文未明确说明支持哪些操作系统。
- **硬件**：原文未说明具体硬件要求；提到「runs on your hardware, your DB, your keys」，即自托管时使用你自己的硬件与数据库。
- **依赖**：
  - Bun（材料中出现 `bun install`、`bun.lock`、`bunfig.toml`；托管路径中注明 Bun 1.4.4+ 可跳过安装）。
  - Postgres（项目描述为「Postgres-native」，并含 `docker-compose` 相关文件；具体版本要求原文未说明）。
- **版本要求**：除 Bun 1.4.4+ 外，其余版本要求原文未说明。
- **网络提示**：材料未提及国内镜像源；若在国内网络环境下载 Bun 或 GitHub 包较慢，可能需要自行配置镜像/加速（原文未说明具体做法）。

### 常见坑与注意事项

- **托管与自托管是两套成本**：原文明确「Your harness subscription and any separately configured model API usage are different costs」，且「always-on enrichment needs its own compute and API configuration」——持续运行的富化功能需要额外的算力与 API 配置。
- **可以先无密钥起步**：原文强调「Start keyless」，先用关键词检索，语义搜索、综合、后台富化按需再加。
- **数据边界需注意**：配置的云端 embedding、rerank、抽取、综合服务提供方可能收到文本内容；即使无密钥模式，harness 模型也能看到被召回的记忆。Markdown 导出**不是**完整的数据库备份。
- **共享访问有信任边界**：原文说明本地文件与共享数据库凭据属于不同的信任边界，仅靠 source 无法隔离这些调用方；设置共享访问前应先阅读 sharing boundaries 与 company-brain 教程。

---



---

<h1><picture><source media="(prefers-color-scheme: dark)" srcset="docs/images/brand/gbrain-wordmark-light.svg"><img src="docs/images/brand/gbrain-wordmark.svg" alt="GBrain" height="52"></picture></h1>

**为你已在使用的 agent 赋予你可控的记忆。** GBrain 存储带来源的显式事实，支持更正与撤回，并让同一份记忆在你的多个 agent 之间共享。从无密钥记忆和关键词检索起步；需要时再添加语义搜索、综合与后台增强。

GBrain 采用 MIT 许可，可免费自托管；gbrain.io 是可选付费托管。

> [!TIP]
> **需要托管？第一次运行 agent？** 试试 **[gbrain.io](https://gbrain.io/?ref=readme)**，即托管版 GBrain，体验最省心：你和团队共用一个大脑，连接 ChatGPT、Claude 和 Cursor，无需安装，也无需保持在线。遇到问题？发邮件至 support@gbrain.io，会有专人免费帮你完成设置。

**让技能与知识并存。** 新的本地大脑会在其内容根目录中包含记忆技能。Agent 可以加入已授权的目录；只有经批准的编辑者才能发布。托管式编码 agent 安装会添加一个自有的原生路由器；重启和实际使用验证是分开的步骤。分阶段迁移会保留共享选择和个人编辑。参见[共享大脑技能](docs/guides/shared-brain-skills.md)。

## 选择你的配置

1. **将 GBrain 添加到现有 agent——推荐。** 保留 agent 的身份，并将记忆保存在其环境中。无需新的个人 agent 身份或私有仓库。从 **[Grok Bot](docs/guides/grok-bot.md)**、**[Muse](docs/guides/muse.md)** 或 **[Codex / Claude Code](docs/tutorials/connect-coding-agent.md)** 的指南开始。[其他 harness](#connect-gbrain-to-your-ai-client-mcp)。
2. **在我自己的电脑上随处使用大脑。** 一条命令即可通过 MCP 在你的 Tailscale tailnet 上发布它并保持运行：`gbrain mcp expose`（为 Grok Bot、Muse 和 ChatGPT 等云端 agent 添加 `--funnel`）。遵循 **[通过 MCP 随处使用你的大脑](docs/guides/remote-mcp.md)**。**对你的 agent 说：** *"use my brain over mcp"* — *"put my brain on tailscale"*。
3. **连接我现有的托管大脑。** 遵循 **[托管 harness 访问](docs/guides/hosted-harness-access.md)** 选择原生 OAuth 或私有机器连接。仪表盘登录、客户端和权限请使用 **[MCP 管理](docs/mcp/ADMIN.md)**，并使用服务器的独立所有者凭据。委派是显式选择。

Grok **Bot** 和 Grok **Build** 是不同产品。Muse 的个人 agent 和 **Muse Code** 也是不同产品。Muse 已有原生可编辑记忆；GBrain 增加了一份带来源和共享访问的显式可移植记录。参见各指南中标注日期的证据和剩余验证步骤。

我是 Garry Tan，Y Combinator 的总裁兼 CEO。我构建 GBrain 来运行自己的 AI agent。它是我 OpenClaw 和 Hermes 部署背后的生产大脑：**155,795 个页面、24,589 个人、5,340 家公司**，66 个 cron 任务自主运行。我的 agent 在我睡觉时摄取会议、邮件、推文、语音通话和原创想法。它会增强遇到的每个人和每家公司。它会在夜间修正自己的引用并整合记忆。我醒来时比入睡时更聪明——你也会如此。

**它也能作为公司大脑。** 经过认证的远程客户端受来源和操作授权以及可见性过滤器约束。本地文件和共享数据库凭据属于不同的信任边界：仅靠来源无法隔离这些调用方。授权测试只覆盖特定访问路径，并非通用的无泄漏保证。设置共享访问前，请阅读[共享边界](docs/architecture/brains-and-sources.md#what-confines-remote-callers-and-what-does-not)和[公司大脑教程](docs/tutorials/company-brain.md)。

除了关键词和语义检索，GBrain 还提供两种可选方式来使用你存储的知识：

- **综合层，为你提供真正的答案。** 跨人物、公司、交易和想法的综合、引用充分的文字。不是"这里有 10 个提到你查询的片段"，而是带引用的实际答案，并明确说明大脑还不知道什么。差距分析正是改变你使用大脑方式的部分。
- **类型化知识图谱。** 启用自动链接时，受信任的本地页面写入无需 LLM 调用即可提取受支持的引用。远程 `put_page` 边稍后通过扫描、维护或 `add_link` 到达。问"谁在 acme-example 工作？"：遍历为广度优先，在密集集群上只需毫秒（[如何实现](docs/architecture/RETRIEVAL.md#breadth-first-graph-walks)）。在 BrainBench 关系问题上，关系检索将生产环境的 **R@5 从 0.6626 提升到 0.7241**（[基准](docs/architecture/RETRIEVAL.md#the-benchmark)）；0.9791 是 `graph-oracle-parse` 的上限，不是产品得分。

构建 15 万页大脑的意义在于将其用作战略护城河。永不丢失上下文。无需重读即可查询你头脑中的内容。大脑层让护城河可用。7×24 小时的梦境循环让它保持敏锐。两者都运行在你的硬件、你的数据库、你的密钥上。

发布一个 7×24 小时运行、负责摄取、增强和整合的守护进程，比让聊天中的 agent 持续努力工作要容易。GBrain 就是那个守护进程的泛化版本。30 分钟即可安装。你的 agent 负责干活。我的个人 agent 越聪明，你的也越聪明。

> **从无密钥开始。** 你的 harness 订阅和任何单独配置的模型 API 用量是不同成本。可选的个人 agent 引导约需 15 分钟；常开增强需要自己的计算和 API 配置。

持久的偏好和事实可在 agent 之间共享；瞬时的任务和 harness 配置状态保留在本地。已配置的云端嵌入、重排序、提取和综合提供方可以接收文本，即使在无密钥模式下，harness 模型也能看到召回的记忆。Markdown 导出不是完整的数据库备份。阅读[记忆边界](docs/guides/memory-boundaries.md)。

> **LLM：** 获取 [`llms.txt`](llms.txt) 查看文档地图，或获取 [`llms-full.txt`](llms-full.txt) 在一次请求中获取同一地图及内联的核心文档。**Agent：** 从 [`AGENTS.md`](AGENTS.md) 开始（如果你使用 Claude Code，则用 [`CLAUDE.md`](CLAUDE.md)）。

## 实际效果

假设你明天要和 Alice 开会。你希望走进会议室时记得她做什么、你们上次何时交谈、以及你们之间还有哪些未了事项。以下是你输入的内容，以及你得到的回复。

**你问：**

> "明天和 Alice 开会前我需要知道什么？"

**大多数个人知识工具会返回一个页面列表。** 类似这样：

```
1. people/alice — Alice runs engineering at Acme...
2. meetings/2026-03-15-alice-q1 — Q1 product review with Alice...
3. meetings/2026-01-08-acme-kickoff — Kickoff meeting with Acme team...
4. customers/acme — Acme is a series-B fintech we work with...
5. notes/2026-04-22 — Quick chat with Alice about pricing...
```

五个页面，你现在得自己打开阅读才能真正做好准备。工具找到了正确的材料，但没有完成工作。

**GBrain 返回的是答案，并附来源：**

```
Alice runs engineering at Acme (a series-B fintech). You last spoke
on April 22 in a quick pricing chat. Three things are still open
from that conversation:

1. She owes you the security review for the new tier
   (deadline was May 1; no update since).
2. You committed to pricing for a 500-seat tier
   (you sent it April 25; no response yet).
3. She mentioned they're hiring a CISO; you said you'd intro
   someone from your network.

Heads up: nothing's been added to the brain about Alice or Acme
since April 22, six weeks ago. She may have replied through email
or Slack DM, channels the brain doesn't see. Worth asking her to
catch up before assuming any of this is still current.
```

每条陈述背后都有来源页面。末尾的"提醒"告诉你大脑还不知道什么，这样你可以直接问 Alice，而不是被意外打个措手不及。大脑刚刚帮你完成了会议准备。

这就是搜索引擎和大脑的区别。搜索找到页面。大脑替你阅读并写出答案。

## 安装

> [!TIP]
> **最快：使用 [gbrain.io](https://gbrain.io/cli?ref=readme) 上的托管 GBrain，适用于任何 harness。** 一条命令即可启动托管在 gbrain.io 上的 GBrain；Claude Code、Codex、ChatGPT、Cursor 以及你使用的其他所有 AI 都能连接它，无需保持任何东西在线。
>
> ```bash
> curl -sSL https://gbrainio.terminalwire.sh | bash
> gbrainio terminal connect      # in a new terminal
> gbrainio workspace new         # pick a plan in the browser
> curl -fsSL https://bun.sh/install | bash   # skip if Bun 1.4.4+ is installed
> export PATH="$HOME/.bun/bin:$PATH"
> bun install -g github:garrytan/gbrain#stable   # the gbrain CLI only; no local brain
> # Claude Code (an existing gbrain entry is kept):
> (claude mcp get gbrain >/dev/null 2>&1 || claude mcp add --transport http --scope user gbrain https://gbrain.io/mcp) && gbrain setup claude-code --instructions-only
> # Codex (an existing gbrain entry is kept):
> (codex mcp get gbrain >/dev/null 2>&1 || codex mcp add gbrain --url https://gbrain.io/mcp) && gbrain setup codex --instructions-only
> ```
>
> `gbrain setup <harness> --instructions-only` 会告知 agent 大脑的存在；agent 很少会调用没人告诉过它的 MCP 服务器。它会将[大脑优先块](docs/guides/setup.md#brain-first-instructions)写入 `~/.claude/CLAUDE.md` 或 Codex 的 `AGENTS.md`（当 Codex 有非空的 `AGENTS.override.md` 时则写入该文件），并附带每轮规则 Hook，同时不动 MCP 条目；同一命令加上 `--no-instructions` 则会移除它们。
>
> 安装程序是适用于 macOS 和 Linux 的 shell 脚本；运行前请先阅读。

### 自托管方法

GBrain 采用 MIT 许可，可免费自托管。它需要 **Bun 1.4.4 或更新版本**；现有 worker 安装在此版本上重启前，请阅读[授权与队列升级指南](docs/guides/authorization-upgrade.md)。

> [!WARNING]
> **GBrain 不在 npm 上分发。** npm 上的 `gbrain` 包与此无关，且可能遮蔽真正的二进制文件，因此切勿运行 `npm install -g gbrain` 或 `bun add -g gbrain`。只能通过 `bun install -g github:garrytan/gbrain#stable`（经过沉淀的稳定通道；`#latest-stable` 是包含每个版本的 alpha 线）或 `git clone --branch stable` + `bun install && bun link` 安装。误装了？`gbrain doctor` 会给出修复方法。
> Schema 未迁移（[#218](https://github.com/garrytan/gbrain/issues/218)）？运行 `gbrain apply-migrations --yes --no-autopilot-install`；如果退出码为 1，运行 `gbrain doctor --json`，而不是再次 `--yes`（`wedged`：转达其修复方法）。

```bash
bun install -g github:garrytan/gbrain#stable
gbrain setup claude-code              # or: gbrain setup codex
gbrain init --pglite --no-embedding   # any other harness, or no agent
```

`gbrain setup` 会先显示将要写入的内容（`--dry-run` 预览，`--remove` 撤销），仅在不存在时创建无密钥大脑，配置 MCP、Hook 和 agent 指令文件中的大脑优先块（`--no-instructions` 跳过），并对它们进行冒烟测试（[设置指南](docs/guides/setup.md)）。对于其他编码 agent，粘贴：

```text
Add GBrain memory to this existing agent. Read and follow:
https://raw.githubusercontent.com/garrytan/gbrain/master/INSTALL_FOR_AGENTS.md
Keep my current identity and instructions, start keyless, and show me the
search-mode choice. Verify a remember/recall round trip with real GBrain calls.
```

Postgres、瘦客户端、API 密钥和升级：[docs/INSTALL.md](docs/INSTALL.md)。Agent：[INSTALL_FOR_AGENTS.md](INSTALL_FOR_AGENTS.md)。纯记忆演练：[为你的编码 agent 赋予记忆](docs/tutorials/connect-coding-agent.md)。

### CLI 独立使用（无 agent）

```bash
gbrain init --pglite --no-embedding   # keyless; no server, no Docker
gbrain import ~/notes/ --no-embed     # index your markdown
gbrain search "a phrase from a note" --json
gbrain doctor                         # verify health
```

### 可选：新建个人 agent

要让 GBrain 创建一个**新的持久个人 agent**（含身份文件和自有私有 GitHub 仓库），请在一个新的空文件夹中打开 Codex（CLI 或 ChatGPT 桌面版）或 Claude Code，然后粘贴：

```
Read and follow every step of:
https://raw.githubusercontent.com/garrytan/gbrain/stable/BOOTSTRAP_FOR_AGENTS.md
Goal: set yourself up as my persistent personal agent in this folder, with gbrain
as your memory. Interview me before writing any identity file — never invent
answers. Ask before anything destructive. You are not done until
`gbrain bootstrap verify` exits 0.
```

该仓库不是完整备份；参见[引导指南](docs/guides/bootstrap.md)。对于常开版本（在具有 8GB+ 内存和自有 API 用量的服务器上进行 7×24 小时摄取和夜间梦境循环），请在 [OpenClaw](https://github.com/openclaw/openclaw) 或 [Hermes](https://github.com/NousResearch/hermes-agent) 下运行 GBrain，并将其指向 `INSTALL_FOR_AGENTS.md`；[个人大脑教程](docs/tutorials/personal-brain.md)会端到端带你完成。

### 将 GBrain 连接到你的 AI 客户端（MCP）

- **本地 stdio 客户端：** `gbrain setup claude-code` 或 `gbrain setup codex` 会在 gbrain 的启动器后面注册 `gbrain serve`（列出记忆动词；`--surface full` 列出所有工具），因此在克隆的仓库中打开的会话永远不会加载该仓库的 `bunfig.toml` 或 `.env`，并添加大脑优先指令：agent 很少会调用没人告诉过它的大脑。Cursor、Windsurf 和其他 stdio 客户端使用 `"$(command -v gbrain)" serve --surface starter --advertise verbs`。
- **带精选技能的插件：** Codex `codex plugin marketplace add garrytan/gbrain@codex-plugin` + `codex plugin add gbrain@gbrain`；Claude Code `/plugin marketplace add garrytan/gbrain` + `/plugin install gbrain@gbrain`。
- **随处使用此大脑：** `gbrain mcp expose` 将其作为用户服务发布在你的 Tailscale tailnet 上（为 Grok Bot、Muse 和 ChatGPT 等云端 agent 添加 `--funnel`）。[指南](docs/guides/remote-mcp.md)。
- **另一台主机上的大脑：** `gbrain connect https://your-host/mcp --token gbrain_xxx --install`（为 Codex 添加 `--agent codex`）。托管大脑使用[原生 OAuth 或私有交接](docs/guides/hosted-harness-access.md)；仪表盘、客户端和权限见 [MCP 管理](docs/mcp/ADMIN.md)。

各客户端指南：[Claude Code](docs/mcp/CLAUDE_CODE.md) · [Codex](docs/mcp/CODEX.md) · [ChatGPT](docs/mcp/CHATGPT.md) · [Claude Desktop](docs/mcp/CLAUDE_DESKTOP.md) · [Claude Cowork](docs/mcp/CLAUDE_COWORK.md) · [Perplexity](docs/mcp/PERPLEXITY.md) · [Hermes](docs/mcp/HERMES.md) · [OpenClaw](docs/mcp/OPENCLAW.md) · [opencode](docs/mcp/OPENCODE.md) · [Grok Build](docs/mcp/GROK.md) · [Grok Bot](docs/guides/grok-bot.md) · [Muse](docs/guides/muse.md) · [所有 MCP](docs/mcp/README.md)。

## 查询大脑的两种方式

原始检索（大多数个人知识工具提供的功能）和能给你实际答案的综合层。它们服务于不同任务。

```bash
# raw retrieval: top pages by hybrid score, no answer-generation call
gbrain search "who's working on AI agents at portfolio companies?"

# brain layer: synthesized answer with citations and gap analysis
gbrain think "who's working on AI agents at portfolio companies?"
```

**`gbrain search`** 返回排名靠前的检索页面，按混合评分排序（向量 + 关键词 + RRF + 来源层级加权 + 重排序器）。当你想要浏览原始材料时使用它：agent 上下文窗口、引用查找、查找特定引文。

已配置的嵌入和重排序提供方仍可接收文本并收取检索费用。无密钥关键词搜索不需要模型 API；综合需要已配置的聊天能力。参见[记忆边界](docs/guides/memory-boundaries.md#