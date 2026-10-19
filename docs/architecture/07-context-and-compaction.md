# 07 上下文管理与压缩

## 问题

模型的上下文窗口有限，而一次长会话的消息、工具结果和文件内容会不断增长。如果不管理，会话会在某一刻因超长而失败。系统的目标是：**尽量保留有用信息，在接近上限前主动释放空间，并且在失败时能恢复。**

## 分层策略

压缩不是一个动作，而是几层由轻到重的手段。在 `query.ts` 中按顺序执行：[已验证]

| 层级 | 手段 | 代价 | 位置 |
|---|---|---|---|
| 1 | 工具结果预算：超长结果落盘 | 很低 | `applyToolResultBudget` |
| 2 | snip：裁剪历史片段（构建期开关 `HISTORY_SNIP`） | 低 | `snipCompactIfNeeded` |
| 3 | microcompact：清理旧工具结果内容 | 低 | `services/compact/microCompact.ts` |
| 4 | autocompact：整体摘要 | 高（需要一次模型调用） | `services/compact/autoCompact.ts` |
| 5 | 反应式压缩：模型报错"上下文超长"后再压缩并重试 | 高，但只在失败时发生 | `query.ts` 中的 `reactive_compact_retry` |

**设计理由**：便宜的手段先做，昂贵的手段后做。大多数情况下，清理旧工具结果就够了，不需要整体摘要。[推断]

## microcompact

`microCompact.ts` 中的关键点：[已验证]

- 常量 `TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'`：被清理的旧工具结果替换为这段固定文字。
- `estimateMessageTokens` 用于估算消息的 token 数。
- 存在"缓存编辑"（cache editing）机制：`consumePendingCacheEdits`、`pinCacheEdits`。这是为了在修改历史时保持提示缓存（prompt cache）有效，避免修改历史导致缓存失效。

**设计要点**：microcompact 操作的是工具结果，而不是对话本身。它的注释说明它"只按 `tool_use_id` 操作，不检查内容"，因此与前面的结果预算可以安全组合。[已验证，`query.ts` 第 368–372 行附近]

## autocompact：何时触发

`services/compact/autoCompact.ts` 中的阈值计算：[已验证]

- 有效上下文窗口 = 模型的上下文窗口 − 为摘要预留的输出 token（上限 20,000）
- 自动压缩阈值 = 有效窗口 − 13,000（`AUTOCOMPACT_BUFFER_TOKENS`）
- 警告阈值 = 阈值 − 20,000；错误阈值 = 阈值 − 20,000
- 手动压缩的缓冲是 3,000

可以用环境变量覆盖：
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`：按百分比设置阈值，取较小值
- `CLAUDE_CODE_AUTO_COMPACT_WINDOW`：限制有效窗口大小

**设计理由**：阈值预留了缓冲，让压缩有足够空间完成。注释和常量命名清楚地区分了自动、手动、警告三种阈值，方便用户理解何时会发生什么。[推断]

## 熔断

`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3`（`autoCompact.ts` 第 70 行）。连续三次压缩失败后，系统停止自动尝试。[已验证]

**设计理由**：压缩本身需要模型调用。如果压缩持续失败，每轮都尝试只会浪费 token 且拖慢会话。熔断是对这类"自我消耗"的防护。[推断]

## 压缩的实际执行

`services/compact/compact.ts` 提供两个入口：

- `compactConversation()`（第 387 行）：整体压缩
- `partialCompactConversation()`（第 772 行）：只压缩一部分（例如某个边界之前的消息）

压缩流程的主要步骤：[已验证]

1. 用 `streamCompactSummary()`（第 1136 行）请求模型生成摘要。
2. 压缩后恢复关键上下文，由 `createPostCompactFileAttachments()` 把最近读取的文件以附件形式补回。恢复有上限：最多 5 个文件，文件总 token 预算 50,000，单文件 5,000，技能 25,000。
3. 保留计划模式、异步代理等状态的附件。
4. 运行 `PreCompact` 与 `PostCompact` 钩子。

**设计要点**：压缩不是"丢掉历史"，而是"用摘要加关键附件代替历史"。补回的文件和技能是为了让模型在压缩后仍能继续工作，而不必重新读取。[推断]

## 会话记忆压缩

`services/compact/sessionMemoryCompact.ts` 定义了另一种压缩配置：[已验证]

- `minTokens = 10,000`：压缩后至少保留的 token
- `minTextBlockMessages = 5`：至少保留的带文本消息数
- `maxTokens`：硬上限

它利用会话记忆（见第 10 章）作为压缩的依据，而不是临时生成摘要。[推断，基于文件名和配置项]

## 压缩后的一致性

压缩会改变消息结构，可能破坏工具调用与结果的配对。源码在 `query.ts` 中有专门的修复：`yieldMissingToolResultBlocks` 会为缺失结果的工具调用补上错误结果，避免 API 报错。[已验证，`query.ts` 第 123 行附近]

同时，`query.ts` 注释提到，思考块（thinking block）必须在一个助手轨迹内保留，不能被压缩丢掉。[已验证，第 158 行附近]

## 设计上的取舍

- **多层策略 vs 简单策略**：多层意味着代码路径多，调试难。收益是大多数会话几乎不需要昂贵的摘要。[推断]
- **缓存友好 vs 实现简单**：缓存编辑让实现复杂很多，但避免了每次压缩都让提示缓存失效。[推断]
