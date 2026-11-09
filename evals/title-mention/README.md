# 标题提及语料库

用于标题-主题加权的检索语料库：`corpus/` 下有 72 个占位页面（事件、财务、招聘、租赁等），`queries.jsonl` 中有 144 个问题，每页四个：

| `family` | 查询形式 |
|---|---|
| `title-substring` | 仅页面标题 |
| `question-wrapped-title` | 标题嵌入简短问题中 |
| `long-form-wrapper` | 标题嵌入长自然语言请求中 |
| `mention-other-gold` | 查询提及某页面，但标准答案是另一页面 |

没有测试运行器或 CI 任务读取此目录。加权逻辑本身由 `test/search/title-mention-boost.test.ts` 和 `test/search/general-title-mention-boost.test.ts` 负责。

针对临时 brain 运行：

```bash
gbrain import evals/title-mention/corpus
gbrain eval retrieval-quality evals/title-mention/queries.jsonl --json
```

`gbrain eval retrieval-quality` 会报告所有 family，但其默认门禁（`src/eval/retrieval-quality/harness.ts` 中的 `DEFAULT_GATE`）仅对其中 `title-substring` 设下限（Hit@1 >= 0.95）；其余三个 family 只报告，不设门禁。