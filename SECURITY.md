# 安全

## 报告漏洞

如果你在 GBrain 中发现安全问题，请通过 GitHub 上的[私密安全公告](https://github.com/garrytan/gbrain/security/advisories/new)私下报告。

不要为安全漏洞创建公开 Issue。

## 自动化安全扫描

CI 在密钥扫描（Gitleaks）之外还运行三项自动化安全检查：

- **依赖漏洞** — OSV-Scanner（`.github/workflows/osv-scanner.yml`）每周运行一次，并在任何触及 `package.json` 或 `bun.lock` 的 PR 上运行。
- **静态分析（SAST）** — Semgrep CE（`.github/workflows/semgrep.yml`）在每个 PR 上运行，并每周运行一次。在 PR 上，它**会阻断自 PR 基线以来新增的发现**（`--baseline-commit`），因此净新增问题会使检查失败，而既有发现永远不会阻断无关的 PR。定时/手动触发的运行执行全树仅报告扫描。
- **发布二进制来源证明** — 发布构建（`.github/workflows/release.yml`）使用 [GitHub 构建产物来源证明](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations)为每个编译后的二进制出具证明，并在发布时从 `admin/src` 重新构建管理界面，使发布的二进制内嵌可追溯到源码的打包产物（而非已提交的 `admin/dist` 字节）。手动验证下载的发布二进制：

  ```bash
  gh attestation verify ./gbrain-darwin-arm64 -R garrytan/gbrain
  gh attestation verify ./gbrain-linux-x64 -R garrytan/gbrain
  ```

所有安全工作流都使用 SHA 固定的 action 和最小权限，并由 actionlint 在每次工作流变更时从结构上强制执行。

### 安装路径信任模型

- **编译二进制的自更新（在 `darwin-arm64` / `linux-x64` 上执行 `gbrain upgrade`）** 在安装前自动验证完整性：它计算下载二进制的 SHA-256，并与从 GitHub REST API 获取的构建来源证明比对——该 API 与资源 CDN 来源不同——确认证明中的摘要以及证明的构建者 id 是本仓库的发布工作流。验证为失败关闭：一旦不匹配或无法获取证明，下载即被丢弃，正在运行的二进制保持不变。它还会拒绝所报告版本与获取该二进制时的发布版本不符的二进制（降级重放防护）。这项无依赖检查依赖 GitHub 账户信任、来源分离，以及通过 TLS 获取的证明的摘要/身份匹配；它**不会**独立验证证明的 Sigstore 签名（Fulcio 证书链或 Rekor 包含证明）。
- **从源码安装和固定标签安装仍为首次使用即信任。** `bun install -g github:garrytan/gbrain#stable` 跟随一个可被强制移动的标签，`codex-plugin` 分支/模板仓库也是强制发布的；这些路径信任 TLS + GitHub，没有独立的完整性检查。从源码安装还会提供已提交的 `admin/dist` 打包产物（全局安装不会安装重新构建管理界面所需的 devDeps），因此该打包产物在此路径上为首次使用即信任。若需最强保证，请安装带证明的发布二进制并按上述方式运行 `gh attestation verify`。

## 环境变量与 cwd 中的 `.env` 文件

gbrain 通常是全局安装的 CLI（或编译后的二进制），在任意工作目录下运行——你碰巧所在的仓库、agent hook 触发的工作区。Bun 会在任何 gbrain 代码运行之前，从该工作目录自动加载 `.env`、`.env.local` 以及 `.env.<NODE_ENV>[.local]` 变体到进程环境，`bun run` 和编译后的二进制皆如此，并会展开这些文件中的 `${VAR}` 引用。因此，提交到克隆仓库中的 `.env` 是**不可信输入**——它由仓库作者写入，而非你。

编译后的 `gbrain` 二进制以 `--no-compile-autoload-bunfig` 构建，因此工作目录中的 `bunfig.toml` 无效——它无法在 gbrain 启动前将代码预加载进二进制（CI 守卫会为每次构建调用加上该标志）。开发运行时 `bun src/cli.ts` 保持 Bun 原生；贡献者自己的工作目录是可信的。脚本模式安装（`bun install -g github:garrytan/gbrain`、`git clone` + `bun link`）以普通 Bun 脚本运行 `src/cli.ts`，Bun 会在任何 gbrain 代码运行前应用当前目录的 `bunfig.toml`——gbrain 发现顶层 `preload` 时会打印一行警告，但无法撤销它。只有编译后的二进制会忽略 cwd 中的 `bunfig.toml`；请从你控制的目录运行脚本模式的 gbrain，或在不可信的检出中使用发布二进制。

Hook 是 gbrain 自行封闭的例外，因为 harness 会在会话目录中运行它们，而你无从批准。`gbrain setup claude-code|codex` 写入的每个 hook（读取上下文和捕获 hook，以及 `--instructions-only`、`gbrain connect --install` 和 `gbrain bootstrap harness` 添加的每轮规则 hook）都通过 `<gbrain config dir>/hook-launcher/gbrain-hook` 启动 gbrain。该启动器是一个小型 POSIX `sh` 脚本：它将会话目录记录到 `GBRAIN_HOOK_CWD`，切换到自身目录并在那里启动 gbrain，因此 Bun 永远不会读取会话目录的 `bunfig.toml` 或 `.env`。其目录中只存放启动器；若出现其他任何内容，启动器会拒绝启动 gbrain 并向 stderr 打印一行。`gbrain post-upgrade` 会将启动器之前写入的 hook 迁移到启动器上（对于 `gbrain setup codex` 的 hook，它改为打印 setup 命令，因为该重写会重新信任它们），`gbrain setup` 会在下次运行时重写它们，`gbrain doctor`（`harness_wiring`，代码 `hook_session_cwd`）会标记任何遗留项。stdio MCP 服务器也得到同样处理，因为 harness 也会在会话目录中启动它（#8426）：`gbrain setup` 和 `gbrain bootstrap hooks` 注册 `command: <hook launcher>`、`args: [<gbrain>, serve, …]`，而 `gbrain serve` 从 `GBRAIN_HOOK_CWD` 读取 `.gbrain-source` 和上传根目录，而不改变自身目录。`gbrain bootstrap hooks`、`gbrain bootstrap harness` 及其 Codex `SessionEnd` 捕获的 hook，以及 gbrain 插件的每轮 hook（其 `.agents/gbrain-launcher` 会先切换到插件目录，#8428）也是如此。`gbrain post-upgrade` 只重写 setup 仍拥有的用户级 `gbrain setup claude-code` MCP 条目，且在 setup 的锁下进行；每个 Codex 条目和每个 `gbrain bootstrap` 条目都以重写它的命令命名，因为这些写入者会重新信任 Codex hook 或从工作区运行。项目级 `.mcp.json` 和已提交的 `.claude/settings.json` hook 属于仓库，保持 PATH 解析。

在引导工作区中，hook 自身的 git 探测以 `-c core.fsmonitor=false` 运行，而在安装回执所指工作区之外的任何会话中完全不运行 git（#8427）：否则来自解压归档或共享文件夹的 `.git/config` 可能指定 `git status` 运行的命令。

**信任边界。** 与安全相关的变量从你的 shell 环境、服务的 `EnvironmentFile` 以及 `~/.gbrain/.env`（运维者拥有的密钥文件）中生效。它们**绝不**从当前目录的 `.env` 文件中生效。启动时，每个 gbrain 进程都会检查工作目录的 `.env` 系列文件，对于其中任一文件*赋值*的每个受保护变量（无论值是什么——展开后的 `${PWD}/…` 到达进程时看起来就像普通绝对路径），丢弃该变量并向 stderr 打印一行：

```
[env] Ignoring GBRAIN_PLUGIN_PATH because a .env file in the current directory assigns it — cwd .env files are untrusted for security settings. Export it from your shell or set it in ~/.gbrain/.env.
```

当当前目录的 `.env` 文件赋值任何受保护变量时（`GBRAIN_GUARDRAILS_MODULE` 是唯一例外——它会拒绝运行，见下文），gbrain 丢弃它，打印一行 `[env] Ignoring …`，然后从空临时目录以净化后的环境重新运行自身一次，切回你的目录，并以该次运行的状态退出。重新运行的环境只是缺少被丢弃的变量（它们绝不会以空字符串形式携带，因为 git 和动态加载器会将其视为已设置），因此 gbrain 启动的每个程序——git、claude CLI、worker——都继承干净的视图。内部变量 `GBRAIN_CWD_ENV_QUARANTINED` 标记重新运行——它不是设置项，gbrain 仅在可证明其启动于标记所指的空目录时才认可它。被信号杀死的重新运行映射为退出码 128+信号；包装器的退出状态始终是重新运行的退出状态（它只转发子进程，别无其他），且重新运行随其包装器一同终止：若包装器被强制杀死（SIGKILL 无法转发），重新运行会在大约一秒内察觉，删除其中性目录并退出，因此跟踪包装器 pid 的 supervisor 永远不会留下孤立的 worker。有控制终端时，Ctrl-C 从该终端直接到达重新运行，包装器不转发它；无控制终端时（supervisor、cron、分离的 harness），仅到达包装器的 SIGINT 会被转发一次，以免重新运行成为孤儿；发送给包装器的 SIGTERM/SIGHUP 始终转发。当被丢弃的变量是被植入的 `HOME` 时，重新运行会收到你真实的主目录（源自启动环境，绝不来自文件），使 git 身份和 `~/` 相对路径继续工作。gbrain 自身从该目录启动的 gbrain 进程（受监督的 worker、后台推送）会为其自身子树重复该步骤一次。当工作目录没有 `.env`，或那里未赋值任何受保护变量时，这没有成本。

**受保护变量。** 两组，一个谓词（`src/core/env-trust.ts` 中的 `isCwdDotenvProtectedKey`）：

- 与安全相关的 `GBRAIN_*` 键（`CWD_DOTENV_PROTECTED_KEYS`）：代码加载 `GBRAIN_GUARDRAILS_MODULE`、`GBRAIN_PLUGIN_PATH`；执行目标 `GBRAIN_CLAUDE_CLI_BIN`、`GBRAIN_CLAUDE_CLI_HERMETIC_CONFIG`、`GBRAIN_JOB_CHILD_CLI`、`GBRAIN_BIN_OVERRIDE`；根/注册表重定向 `GBRAIN_HOME`、`GBRAIN_MOUNTS_PATH`；大脑连接字符串 `GBRAIN_DATABASE_URL`（cwd 的 `.env` 会将每个 hook、查询和写入重定向到它指定的数据库——`DATABASE_URL` 的值守卫只覆盖裸名称）以及 OAuth 中继 `GBRAIN_OAUTH_RELAY_URL`；姿态放宽 `GBRAIN_ALLOW_SHELL_JOBS`、`GBRAIN_ALLOW_PRIVATE_REMOTES`、`GBRAIN_ALLOW_UNVERIFIED_REMOTE`、`GBRAIN_GIT_ALLOW_FILE_TRANSPORT`、`GBRAIN_ALLOW_MASS_RECONCILE`、`GBRAIN_ALLOW_DEFAULT_WRITE`、`GBRAIN_NO_SANITY`、`GBRAIN_REMOTE_PRIVATE_PAGES`。
- 检出可能借此劫持 gbrain 启动的程序（而非 gbrain 本身）的变量族（`CWD_DOTENV_PROTECTED_PREFIXES`、`CWD_DOTENV_PROTECTED_TOOLCHAIN_KEYS`）：`LD_*`/`DYLD_*` 动态加载器族，git 配置/hook/执行注入（`GIT_*`），Bun 和 npm 运行时配置（`BUN_*`、`NPM_CONFIG_*`/`npm_config_*`），Node 预加载和 TLS 旋钮（`NODE_OPTIONS`、`NODE_PATH`、`NODE_EXTRA_CA_CERTS`、`NODE_TLS_REJECT_UNAUTHORIZED`），git 和其他工具作为全局配置读取的 `XDG_CONFIG_HOME`/`XDG_DATA_HOME`/`XDG_CACHE_HOME` 和 `GNUPGHOME` 根，`SSL_CERT_FILE`/`SSL_CERT_DIR`/`CURL_CA_BUNDLE`/`REQUESTS_CA_BUNDLE` 信任库，OpenSSL provider/engine 加载器（`OPENSSL_CONF`、`OPENSSL_ENGINES`、`OPENSSL_MODULES`——被植入的配置会将共享对象加载进每个链接 OpenSSL 的子进程，例如通过 https 的 git），净化后重新运行的中性目录及每次临时写入所解析的临时根（`TMPDIR`、`TMP`、`TEMP`），`HOME` 和 `USERPROFILE`（导出时无效，但在 `HOME` 未设置时，被植入的值会将 `~/.gitconfig`、`~/.ssh` 和 `~/.gbrain` 迁移到检出中），gbrain 子进程启动的非交互式 shell 会读取的 shell 启动和解释器主目录族（`BASH_ENV`、`SHELLOPTS`/`PS4`、`BASHOPTS`、`PROMPT_COMMAND`、`ZDOTDIR`、`PYTHONHOME`、`PERLLIB`、`GCONV_PATH`，以及 `BASH_FUNC_*` 前缀下的导出函数），`EDITOR`/`VISUAL`/`PAGER`，ssh askpass 程序，Python/Perl/Ruby 辅助程序的解释器预加载，两种拼写的 HTTP(S)/ALL 代理变量，以及 gbrain 网关从环境读取的每个 AI 提供商端点或凭据变量（`CLAUDE_CONFIG_DIR`、`ANTHROPIC_BASE_URL`、`ANTHROPIC_AUTH_TOKEN`、`OPENAI_BASE_URL`、`OPENROUTER_BASE_URL`、`LITELLM_BASE_URL`、`OLLAMA_BASE_URL`、`LMSTUDIO_BASE_URL`、`LLAMA_SERVER_BASE_URL`、`LLAMA_SERVER_RERANKER_BASE_URL`）。端点重定向的 `GBRAIN_*` 键（`GBRAIN_REMOTE_MCP_URL`、`GBRAIN_REMOTE_ISSUER_URL`、`GBRAIN_DIRECT_DATABASE_URL`）以及重新运行标记本身属于第一组。这是已知劫持族的拒绝列表；随着新族的识别而增长。

`GBRAIN_GUARDRAILS_MODULE` 还只接受绝对路径或 `~/` 路径：拒绝 cwd 相对规格和裸包名（包名会从当前目录的 `node_modules` 解析）。它也是在 cwd 边界失败关闭的唯一键：当 cwd 的 `.env` 赋值它且非空值被丢弃时，gbrain 不会在没有它的情况下重新运行——它会拒绝启动（退出码 1，`guardrails: GBRAIN_GUARDRAILS_MODULE is assigned by a .env file in the current directory; refusing to run without the operator's firewall …`），因为 Bun 在 gbrain 启动前合并该文件，你导出的值无法与文件中的值区分。良性仓库永远不会赋值这个 gbrain 专属键；你自己的设置应放在 shell 或 `~/.gbrain/.env` 中。

**值得了解的语义。**

- *逐进程，然后继承。* 每个 gbrain 进程在启动时重新应用检查；上述一次性净化重新运行使结果到达 git、agent CLI 和 worker。supervisor 将 shell 作业选择加入以 `--allow-shell-jobs` 标志传递给 worker——worker 再传递给每作业子进程——以免隔离静默禁用它。
- *键存在，而非值。* 如果 cwd 的 `.env` 赋值了受保护键，你从 shell 导出的值在你停留于该目录期间也会被丢弃——警告会告知你。将设置移到 `~/.gbrain/.env` 或从其他目录运行。你导出的族成员（例如 `GIT_AUTHOR_NAME`）不受影响，除非 cwd 的 `.env` 赋值了同名变量。
- *其他一切仍会加载。* 路由和调优的 `GBRAIN_*` 变量以及上文未列出的每个变量仍像以前一样从 cwd 的 `.env` 加载。`DATABASE_URL` 保留自己的值匹配守卫（见 `docs/ENGINES.md`）。从 shell 导出的 `GBRAIN_DATABASE_URL` 始终生效，但*由 cwd 的 `.env` 赋值*的 `GBRAIN_DATABASE_URL` 会像其他受保护键一样被丢弃——项目目录永远不能选择你写入哪个大脑。将其移到 shell 或 `~/.gbrain/.env`。
- *你自己的配置目录。* 从 `~/.gbrain`（或 `$GBRAIN_HOME/.gbrain`）内部运行 gbrain 会使其 `.env` 成为"cwd 的 `.env`"。该文件属于你，因此无需警告即生效——除非 cwd 的 `.env` 赋值了配置目录所派生的任何键（`GBRAIN_HOME`、`HOME`、`USERPROFILE`）；被植入的 home 永远不会被视为你的配置目录。
- *服务器部署。* `GBRAIN_ADMIN_BOOTSTRAP_TOKEN`、`GBRAIN_HTTP_CORS_ORIGIN` 和 `GBRAIN_HTTP_TRUST_PROXY` 故意不在列表中，以便将它们并置的已记录容器部署继续工作——请从你控制的目录启动 `gbrain serve --http