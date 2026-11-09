<!-- gbrain-runbook-stamp: 0.60.624.0 -->
<!-- 此标记在每次发布时必须与 VERSION 文件一致；CI 会强制校验
     （scripts/check-bootstrap-tag.sh）。`gbrain bootstrap status` 会将其与
     已安装的二进制文件比对，不一致时发出警告。 -->

# BOOTSTRAP_FOR_AGENTS.md — 成为你的人类用户的持久化 Agent

你是一个 AI Agent（Claude Code、Codex 或 opencode），读到这份文档是因为有人粘贴了一段内容，要求你把自己配置成他的持久化个人 Agent，并以 gbrain 作为你的记忆。本手册是判断层；确定性工作由 `gbrain bootstrap` 子命令完成，并带有退出码。请从上到下依次执行。

**范围说明：** 本路径适用于 Claude Code、Codex 和 opencode（桌面应用或 CLI；opencode = SST 终端 Agent，opencode.ai —— 不是 OpenClaw）。运行 OpenClaw 或 Hermes？请改用 `INSTALL_FOR_AGENTS.md`。

**最终状态：** 此文件夹是你的工作区——身份文件由你的人类用户自己的回答渲染而成，一个本地大脑（PGLite，嵌入式，无服务器），MCP 已接通，每轮上下文，以及一个私有 GitHub 仓库作为你的持久化载体。约 15 分钟，主要是访谈。

## 安装 Agent 的硬性规则

⛔ **绝不编造答案。** 人格、目的和边界来自人类用户。一个猜测出来的 SOUL.md 会被未来每一个会话所相信。如果你没有答案，就问。渲染步骤在结构上拒绝运行，直到必需的答案存在并已被回读确认。

⛔ **小批量提问。** 访谈最多 12 个问题（6 个必需），分三批提问。每批用一行复述回去。任何非必需问题都接受"跳过"——保持势头比追求完整更重要。

⛔ **只在本阶段列表内操作。** 运行 `gbrain bootstrap status --json` 并遵循它的阶段列表——CLI 是唯一事实来源，本文档只是注释。如果要求你运行的步骤不在 CLI 的阶段列表中，拒绝执行。如果 `status` 报告本手册与已安装二进制文件之间存在版本偏差，请说明，并优先遵循二进制文件的指示。

⛔ **不允许静默失败。** 每个阻塞条件（密钥扫描拦截、锁冲突、部分安装）都会通过 `status`/`verify`/`doctor` 输出暴露——读取它并用平实的语言转达给人类用户。绝不要绕过拒绝。

⛔ **声称完成前先验证。** 当 `gbrain bootstrap verify` 退出码为 0 时安装才算完成——而不是当对话记录看起来不错时。把它的报告粘贴给人类用户。

⛔ **遵守工具链信任规则。** GBrain 需要 Bun 1.4.4 或更新版本（`bun upgrade` 可更新较旧的 Bun）。在有平台包管理器时通过它安装 bun（`brew install oven-sh/bun/bun`）；唯一允许的替代方案是校验和验证变体：将固定版本的发布文件下载到文件，对照该版本的 SHASUMS256.txt 验证，然后才执行。以同样方式安装 gh——优先平台包管理器（`brew install gh`、`apt install gh`、`dnf install gh`、`winget install GitHub.cli`，按官方说明操作）；绝不用管道式 curl-to-shell 单行命令。仅以 `bun install -g github:garrytan/gbrain#stable` 安装 gbrain——名为 "gbrain" 的 npm 包是无关项目。（云沙箱例外：bun 的包获取在那里与代理不兼容——使用 `gbrain bootstrap cloud-setup-script` 配方，它通过 npm 从同一固定 GitHub 源安装。）

⛔ **绝不伪造工具。** 如果 gh 或任何预检二进制文件缺失、被沙箱出口代理拦截，或返回 403，请通过 `status`/`doctor` 输出报告，并遵循下面的云沙箱指引。绝不要手工编写 gh 垫片、将假二进制文件塞进 /usr/local/bin，或伪造通过的检查——伪造的工具会毒化后续每一次验证，而且唯一一次尝试就掩盖了真实的静默持久化失败。CLI 会自行诚实地降级；你的工作是转达，不是桥接。

## Codex 预检（仅 ChatGPT 桌面 / Codex CLI）

Codex 会沙箱化命令执行。开始前，告诉人类用户："我需要审批才能运行安装命令（bun、gh、gbrain）并在此文件夹中写入——出现提示时请批准。"如果审批被全局禁用，请人类用户为本会话启用 workspace-write + 网络。统计你需要的审批点击次数；在结束时报告该数字（它用于安装时测量）。

如果 gbrain 插件已安装并启用（codex：`[plugins."gbrain@…"] enabled = true`；Claude Code：`enabledPlugins["gbrain@…"] = true`），hooks 阶段会在该 harness 上跳过自己的 `mcp add`——插件已提供 MCP 服务器（每个名称只有一个所有者）。该跳过是健康的，不是错误；仅用 `--mcp-even-if-plugin` 强制手工接线注册。

## 阶段演练（注释——CLI 的列表优先）

1. **预检。** `git`、`bun`、`gh` 存在。按上面的信任规则安装缺失项：bun 通过平台包管理器或校验和验证下载——校验和验证安装是唯一允许的非包管理器变体；gh 通过平台包管理器。（在干净的 Mac 上，`git` 可能触发 Xcode 工具对话框——该下载不计入 15 分钟，告诉人类用户让它运行。）
   `gh auth status`——如果已登出，人类用户的唯一手动步骤：`gh auth login -h github.com -p https -w`（你运行它；他们点击 Authorize）。然后 `gbrain bootstrap status`——它是幂等且可恢复的；任何部分失败后，重新运行它并从它指向的位置继续。
2. **引擎。** 两条通道；默认第一条：
   - **PGLite（默认）：** `gbrain init --pglite`（2 秒，无服务器）。只要该大脑的 `gbrain serve` 在运行，每轮 Hook 上下文注入就有效。
   - **Postgres 优先（harness 安装）：** `gbrain init --prefer-postgres` 走 5 级阶梯（环境 URL → Supabase token 发现 → 本地 Postgres → `--allow-docker` → PGLite 兜底），适用于需要并发连接或多机访问的安装。Postgres 大脑也保留每轮 Hook 通道：IPC 监听器与引擎无关，其 socket 以连接 URL 为键，因此只要该大脑的 `gbrain serve` 在运行，Hook 就会触发，与 PGLite 相同（`docs/guides/bootstrap.md` 中的降级矩阵包含该行；INSTALL_FOR_AGENTS.md 的"Engine preference for harness installs"包含阶梯细节）。

   搜索模式会静默自动选择（无密钥时保守，有扩展密钥时 tokenmax），并打印 `[AGENT]` 成本矩阵——向人类用户展示该矩阵，并在运行高容量查询前确认（见 INSTALL_FOR_AGENTS.md 步骤 3.5）；他们可随时用 `gbrain search modes` 更改。此处唯一要提出的是可选的提供商密钥——无密钥时你以无密钥模式运行：关键词搜索加上你通过写入工具自行编写的记忆；一切正常。一个密钥按提供商升级能力：OpenAI 解锁语义搜索 + 自动事实提取；Voyage 解锁语义搜索；Anthropic 解锁自动事实提取。绝不要施压索取密钥。如果人类用户提供，将其传给 CLI 提示——它会进入 0600 配置文件，绝不进入访谈答案，绝不进入你保留的聊天日志。
3. **访谈。** `gbrain bootstrap interview --init`，然后分三批从题库（CLI 会打印）提问，用 `--set KEY "value"` 逐字记录每个答案。对必需问题的模糊答案追问一次。Claude Code 和 opencode：在最后一批中，还要询问唯一一项操作同意——MCP 范围。它不是 12 个访谈问题之一；同意项与题库并行。在 Claude Code 上选择：project（推荐——你打开的任何其他仓库都无法读取你的大脑）vs user（你的 Agent 无处不在，但你打开的任何仓库都能访问它——读和写——且两个打开的会话会争用数据库）。在 opencode 上推荐相反：user-global 是默认且共享安全的选择（opencode 会生成项目配置定义的服务器，且无信任提示，因此提交的项目条目会在每个协作者的机器上执行）——仅作为刻意选择提供 project，并说明该后果。在回读之前用 `gbrain bootstrap interview --set MCP_SCOPE <project|user>` 记录，以便确认覆盖它。在 Codex 上，完全跳过此问题——接线步骤会说明 Codex 的实际情况。最后一批之后：用一个紧凑块回读所有答案，问"这是你想放在房间里的东西吗？"，然后才用 `--status` 为回读集打印的哈希运行 `gbrain bootstrap interview --confirm <hash>`。如果你确认了一个人类用户从未见过的集合，门禁会失败。
4. **渲染。** `gbrain bootstrap render`——身份文件出现。向人类用户展示 SOUL.md。现有文件绝不会被覆盖（重跑安全；`--force` 会先备份）。
5. **技能。** `gbrain skillpack scaffold --all`——CLI 搭建技能集。此处无需判断；转达输出。
6. **接线 harness + 注册大脑源。** `gbrain bootstrap hooks --harness <detected>` 创建 `<workspace>/brain` 并打印针对本工作区的确切 `gbrain sources add <source_id> --path <brain> --force` 命令——逐字运行它（不要猜不同的 id；猜的 id 只会在 `verify` 时以 FK 错误暴露，届时错误猜测还会用 `overlapping_path` 错误阻塞正确的 id）。它还会：
   - Claude Code：默认开启每轮 Hook——不要询问；每轮加载大脑正是为你的 Agent 安装 gbrain 的全部意义。告诉人类用户它已开启以及如何关闭（`GBRAIN_HOOKS=0`，或用 `--no-hooks` 重跑，或 `gbrain bootstrap uninstall`）。此处不询问 MCP 范围——`hooks` 使用访谈期间记录的 MCP_SCOPE 答案。
   - Codex：注册 MCP（`codex mcp add`）并依赖 AGENTS.md 协议——明确说明 Codex 获得的是拉取式上下文，而非每轮推送。不要提供 MCP 范围选择：`codex mcp add` 没有范围标志，因此注册始终是 user-global。将其陈述为事实——此机器上打开的任何仓库都能通过其 MCP 工具访问大脑（读和写）；退出方式是 `codex mcp remove gbrain`（仅注册）或 `gbrain bootstrap uninstall`（完全拆除）。
   - opencode：直接将 MCP 条目写入 opencode 的 JSONC 配置（无需 CLI 执行）并依赖 AGENTS.md 协议，opencode 原生加载该协议——明确说明 opencode 获得的是拉取式上下文，而非每轮推送。范围遵循记录的 MCP_SCOPE 答案（默认 user-global；project 答案会写入候选提交的 `opencode.json`，CLI 会打印共享警告）。接线后重启 opencode——它在会话启动时读取配置。退出方式：条目的 `"enabled": false`，或 `gbrain bootstrap uninstall`。
7. **私有仓库。** `gbrain bootstrap repo`——从工作区创建私有 GitHub 仓库，通过 API 验证隐私位，推送。如果人类用户从自己创建的仓库开始（create-repo-first：其自己账户下的空私有仓库，克隆并在此打开），这会收养该仓库而非创建新仓库——验证其为私有并推送工作区。非空仓库或组织拥有的仓库会被拒绝并给出明确消息（创建空的个人仓库，或对现有 Agent 克隆运行 `gbrain bootstrap attach`）。询问后台持久化同意（git post-commit 自动推送加上 30 分钟拉取任务以保持多机新鲜度；拒绝仍通过每轮和会话结束推送持久化）。如果人类用户没有 GitHub 或拒绝：本地模式并给出诚实警告；`bootstrap repo` 可稍后随时运行。注意：每轮/会话推送会推迟到此阶段记录已验证仓库之后，因此绝不会推送到未验证隐私的源。
8. **验证。** `gbrain bootstrap verify`——完整契约：通过真实写入路径的大脑往返、图底线、token 扫描、密钥扫描、仓库隐私、Hook 冒烟、能力报告（无密钥或有密钥）。退出码 0 否则不算完成。粘贴报告。然后转达它打印的首次运行导览（人类用户应尝试的三个提示，从重启会话开始）以及其下的交接块——所有权声明和冷启动提议是人类用户必须真正理解的两件事，不是细则。

<!-- BEGIN multi-agent-rules (identical in AGENTS.md, BOOTSTRAP_FOR_AGENTS.md and INSTALL_FOR_AGENTS.md; bun run check:shared-tree-git) -->
## 共享大脑检出的多 Agent 规则

多个 Agent、cron 和 Hook 经常在同一台机器上提交到同一个大脑检出。在此类检出中：

1. **用一条命令提交并推送，并指明你的文件：** `scripts/brain-commit-push.sh "<msg>" <paths>`，或当大脑没有辅助脚本时用 `gbrain git commit-push -m "<msg>" -- <paths>`。它获取检出的写入者锁，只提交那些路径，合入远程提交而不触碰他人的文件，如果推送未落地则大声失败。
2. **用 `--pull` 引入远程提交**（`scripts/brain-commit-push.sh --pull` 或 `gbrain git commit-push --pull`），在会话开始时和一批写入之前。
3. **非零退出时，读取代码。** 对 `commit_lock_busy` 重试一次。对 `integrate_conflict`（由他们选择哪个版本胜出）和 `writer_coordinator_required`（受管大脑：改用 `put_page` 保存页面）停止并转达给用户。`pushed_checkout_lagging` 是成功。
<!-- gbrain-allow-shared-tree-git-begin: the forbidden commands, named so agents avoid them -->
4. **绝不要在那里运行** `git pull --rebase`、`git stash`、`--autostash`、`git add -A`、`git commit -a`、`git reset --hard` 或 `git push --force`，命令失败时也绝不回退到它们。绝不要在保存其 autostash 之前用 `git stash store` 删除 `.git/rebase-merge`。
<!-- gbrain-allow-shared-tree-git-end -->

结果、冲突恢复、卡住的变基以及宿主守护进程的锁契约：[docs/guides/shared-checkout-git.md](docs/guides/shared-checkout-git.md)。
<!-- END multi-agent-rules -->

## 第二台机器

如果此工作区是从现有 Agent 仓库克隆的（agent.json 显示已初始化），运行 `gbrain bootstrap attach` 而非访谈/渲染/仓库阶段——它会接线此机器（源、Hook、MCP）并验证。如果 agent.json 显示它是未初始化的模板，则从阶段 1 开始正常流程。

## 云沙箱（claude.ai/code 及类似代理环境）

**如何判断：** `gbrain bootstrap status --json` 报告 `execution_environment: "cloud-sandbox"`（CLI 检测已记录的信号——CLAUDE_CODE_REMOTE 环境变量、代理注入的 token 占位符）。信任 CLI 的检测而非你自己的猜测。

**预期降级——这些是要转达的事实，不是要桥接的 bug：**

- **无 crontab，无存活的背景进程。** VM 在闲置后被回收。计划拉取被诚实跳过；每轮（Stop Hook）和会话结束推送承担持久化。不拒绝任何东西，不编造任何东西。
- **GitHub GraphQL 始终被**出口代理拦截，**REST 仅能访问附加到会话的仓库**——会话中途创建的仓库不会被附加，因此 `gbrain bootstrap repo` 在云中会快速拒绝并给出可行的流程。隐私验证会自行回退到纯 git 协议。
- **`git push` 仅对会话的工作分支有效。** 用户 PAT <!-- gbrain-allow-shared-tree-git: describes the cloud sandbox's push limit, not a command to run --> 不会绕过任何这些限制。
- **只有提交到仓库的文件会带入下一个会话。** `~/.gbrain`、`~/.claude` 和被 gitignore 的 `.claude/settings.local.json` 会蒸发。因此 Hook 位于已提交的 `.claude/settings.json` 中（CLI 在云中写入 PATH 解析的、失败开放的命令）；Hook 配置在会话启动时快照，因此会话中途写入的 Hook 会在下一个会话激活——说明这一点，而不是去调试它。

**正确的云流程：**

1. 人类用户从普通机器（或 github.com）创建私有仓库，并在该仓库上打开云会话。
2. 环境的 setup 脚本安装 gbrain 二进制文件——用 `gbrain bootstrap cloud-setup-script` 打印它，让人类用户粘贴到环境配置中（基于 npm；bun 的获取在那里与代理不兼容）。
3. 会话内：`gbrain bootstrap attach`，然后 `gbrain bootstrap hooks --harness claude-code`（写入已提交的载体），提交 + 推送，并告诉人类用户 Hook 在下一个会话生效。

## 失败模式及其实际含义

<a id="bootstrap-symptoms"></a>

| 症状 | 真实原因 | 修复 | 谁行动 | 同意 | 验证 |
|---|---|---|---|---|---|
| `interview --status` 永远非零退出 | 某个必需答案确实缺失 | 询问人类用户。不要默认它。 | 用户（回答问题） | 无 | `gbrain bootstrap interview --status` |
| 渲染因未解析的 token 拒绝 | 访谈未完成或模板编辑破坏了 token | 完成访谈；`status` 会指出 token。 |