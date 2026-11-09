# DESIGN.md

gbrain 的设计系统事实标准。源自 v0.26.0 管理后台 SPA 开发期间落地于 `admin/src/index.css` 的事实令牌，并在 v0.36.1.0 Hindsight 校准波次的设计评审中正式确立。

本文档是 `/plan-design-review` 和 `/design-review` 的校准目标。当问题为"这个 UI 是否符合系统？"时，答案在此。

## 语气

GBrain 说话像一个了解你过往的聪明朋友，而非冷冰冰的评分系统。每条面向用户的字符串都经过此过滤器：

- 第二人称，允许缩略形式。
- 基于用户可验证的具体数据（"3 次中错过 2 次"优于"Brier 0.31"）。
- 绝不说教。绝不"我们建议"。绝不"根据你的数据"。
- 简短。叙述性文字不超过 25 词；状态不超过一行。
- 数字基于真实结果，绝不使用未经翻译的抽象指标。

五个界面使用此语气（v0.36.1.0+）：
`pattern_statement`、`nudge`、`forecast_blurb`、`dashboard_caption`、
`morning_pulse`。五者均通过 `src/core/calibration/voice-gate.ts` 中的 `gateVoice()`，按模式使用不同评分标准。Haiku 评判器拒绝学术腔候选；最多重生成 2 次；之后回退到 `src/core/calibration/templates.ts` 中的手写模板。

## 颜色令牌

CSS 变量位于 `admin/src/index.css`。SVG 渲染器内联与这些令牌匹配的字面量（`src/core/calibration/svg-renderer.ts`）。

| 令牌               | 值        | 用途                                      |
|--------------------|-----------|-------------------------------------------|
| `--bg-primary`     | `#0a0a0f` | 页面背景                                  |
| `--bg-secondary`   | `#14141f` | 侧边栏、卡片                              |
| `--bg-tertiary`    | `#1e1e2e` | 细微表面、边框                            |
| `--text-primary`   | `#e0e0e0` | 正文文本                                  |
| `--text-secondary` | `#888`    | 标题、标签                                |
| `--text-muted`     | `#777`    | 三级文本 — TD2 从 #555 提升以满足 WCAG AA 对比度（约 5.5:1） |
| `--accent`         | `#3b82f6` | 激活状态、链接、主要 CTA                  |
| `--success`        | `#22c55e` | 健康 / 正常状态                           |
| `--warning`        | `#f59e0b` | Doctor 警告                               |
| `--error`          | `#ef4444` | 失败、破坏性确认                          |

深色主题是唯一主题。不计划浅色模式切换 — 管理后台是运维工具，不是营销界面。用户本就生活在深色主题的终端中。

WCAG 对比度：
- 正文文本（#e0e0e0 于 #0a0a0f）→ 约 14:1，AAA
- 弱化文本（#777 于 #0a0a0f）→ 约 5.5:1，AA（TD2 前为 4.0 / 不合格）
- 强调链接（#3b82f6 于 #0a0a0f）→ 约 5.7:1，AA

## 排版

| 变量               | 值                          | 用途                            |
|--------------------|-----------------------------|---------------------------------|
| `--font-sans`      | `Inter, system-ui, sans-serif` | UI 文本、标题、正文             |
| `--font-mono`      | `JetBrains Mono, monospace` | 数字、slug、代码、终端风格数据  |

字号阶梯（事实标准，尚未正式化）：
- 18px：侧边栏 logo / 页面标题
- 14px：正文
- 13px：导航项
- 12px：图表说明、次级标签
- 11px：密集图表中的三级标签

表格和指标中的数字使用 JetBrains Mono，使列对齐机械化。避免在同一行混用 Inter 和 JetBrains Mono。

## 间距阶梯

4 / 8 / 16 / 24 / 32px。Linear 应用风格的密度：主要区块之间 24-32px，行组之间 16px，行内 8px。Calibration 标签页（已批准的 variant-B 原型）是标准范例。

## 布局

- 左侧边栏 200px。激活项获得 3px `--accent` 左边框。
- 主内容区使用剩余宽度。
- 最大内容宽度：文本密集页面（Calibration）720px，数据表格（Request Log）960px。
- 无三列功能网格。无彩色圆圈图标。无装饰性色块。
- 卡片需证明其存在价值 — 大多数情况下标题 + 内容无需卡片边框即可工作。

## 图表

通过 `src/core/calibration/svg-renderer.ts` 服务端渲染 SVG。纯函数：数据 → SVG 字符串。无 DOM、无 React 组件、无图表库。

XSS 防护：对每个调用方控制的字符串执行服务端 `escapeXml()`。数值输入经 `.toFixed()` 强制转换。管理后台 SPA 通过 `<TrustedSVG>` 包装器以 `dangerouslySetInnerHTML` 渲染。端点由 `requireAdmin` 中间件保护。

为何服务端渲染 SVG（依据 D23）：
- 图表逻辑贴近数据计算。
- 零新增客户端图表库依赖。
- SVG 可访问（文本标签）、可缩放、便于复制粘贴到 PR 描述和文档。
- 为未来管理后台图表（矛盾趋势、观点记分卡等）树立先例。

v0.36.1.0 中的四个图表渲染器：
- `renderBrierTrend({ series })` — 迷你折线图 + 0.25 基线参考
- `renderDomainBars({ bars })` — 水平准确率条形图
- `renderAbandonedThreadsCard(threads)` — 文本行 + "立即回顾"链接
- `renderPatternStatementsCard(statements)` — 可点击下钻锚点

## 交互模式

- 所有 CLI 交互界面必须支持键盘导航。提议队列审查使用 J/K/space/u/q 快捷键（gmail 风格）。
- 加载状态："Loading..."。200ms 以内的操作不显示加载指示器。
- 空状态本身就是功能：温暖感 + 主要操作 + 上下文。冷启动的 Calibration 页面明确告诉用户如何建立画像，而非"无可用数据"。
- 错误状态：说明什么失败了 + 说明下一步。绝不"发生错误 — 请重试"。

## 尚未包含（v0.37+ 路线图）

- 字号阶梯正式化（当前值为事实标准，未强制执行）
- 动画令牌（管理后台 SPA 刻意零动画；v0.37 可能添加细微的进度 / 加载过渡）
- 打印样式表
- 浅色模式（不计划 — 见上文"深色主题是唯一主题"）
- 组件库提取（React 组件内联于 admin/src/pages/；尚无 `<Button>` / `<Card>` 抽象层）

## 如何使用本文档

向 gbrain 添加新 UI 界面时：

1. 优先选用现有令牌，再引入新令牌。新令牌需经 `/plan-design-review`。
2. 遵循语气规则。在校准界面发布任何面向用户的字符串前，先通过 `gateVoice()` 运行候选。
3. 遵循间距阶梯和密度。Linear 式平静清晰优于仪表盘卡片马赛克。
4. 遵循排版：UI 用 Inter，数字用 JetBrains Mono。

更新本文档时：它是活的目标，不是冻结的规范。重大变更需经 `/plan-design-review` 以保持系统一致。