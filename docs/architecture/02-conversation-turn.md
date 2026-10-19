# 02 主线：一次对话回合

这一章沿着一次用户输入，走一遍系统的主要步骤。读完它，后面各章都能找到自己的位置。

## 1. 入口：submitMessage

`QueryEngine.submitMessage()`（`QueryEngine.ts` 约 209 行）是一次回合的入口，无论交互式 REPL、无头 `-p` 模式还是 SDK 调用，最终都走到这里。[已验证]

它做的事情按顺序是：

1. 从配置读取工具列表、MCP 客户端、权限回调 `canUseTool`、最大回合数、预算上限等。[已验证]
2. 清空本轮发现的技能名、设置当前工作目录。[已验证]
3. 调用 `processUserInput` 解析输入。[已验证]

## 2. 输入分流：processUserInput

`utils/processUserInput/processUserInput.ts` 把输入分成四类，文件分别是：

- `processSlashCommand.tsx`：斜杠命令，如 `/compact`
- `processBashCommand.tsx`：Bash 模式（直接执行 shell）
- `processTextPrompt.ts`：普通文本，要发给模型
- 其余情况：直接返回结果

它的返回值包含 `shouldQuery` 布尔值。[已验证]

**关键设计**：`shouldQuery` 决定本次输入是否需要进入模型循环。斜杠命令很多只需要本地执行，执行完直接结束，不消耗模型调用。`QueryEngine.ts` 第 556 行附近有 `if (!shouldQuery)` 分支直接返回。[已验证]

## 3. 进入查询循环：query()

只有 `shouldQuery` 为真时，才调用 `query()`（`QueryEngine.ts` 约 675 行，实现在 `query.ts` 第 219 行，内部是 `queryLoop`）。[已验证]

`query()` 是一个 **异步生成器**（`async function*`），边执行边 `yield` 事件（流式文本、工具进度、系统消息等）。`QueryEngine` 用 `for await` 消费这些事件，转换成 SDK 消息并写入转录。这种设计让 UI 和 SDK 都能实时看到进度，不需要等整轮结束。[已验证]

## 4. 循环体

`queryLoop` 的核心是一个 `while (true)` 循环（`query.ts` 第 307 行开始，第 1728 行结束）。每一轮迭代：

### 4.1 准备上下文

- 从上一个压缩边界之后取消息：`getMessagesAfterCompactBoundary`。[已验证]
- 应用工具结果预算：`applyToolResultBudget`，限制单条消息里工具结果的总大小，超出部分落盘。[已验证]
- 依次执行 `snip`（可选）、`microcompact`、`autocompact`。压缩的细节见第 07 章。[已验证]

### 4.2 调用模型

通过依赖注入的 `deps.callModel`（默认是 `services/api/claude.ts` 的 `queryModelWithStreaming`）发起流式请求。[已验证]

流式返回期间，每当出现 `tool_use` 块，就把 `needsFollowUp` 置为真。注释特别说明：**不依赖 `stop_reason === 'tool_use'`**，因为它不总是可靠，而是以流中是否真的出现了工具调用块为准。[已验证，`query.ts` 第 554–558 行附近]

### 4.3 工具调用的两种执行方式

- 若 `config.gates.streamingToolExecution` 开启，使用 `StreamingToolExecutor`：模型还在流式输出时，就可以开始执行已完整的工具调用。[已验证]
- 否则，等模型输出完再调用 `runTools` 批量执行。[已验证]

两种方式的详细对比在第 04 章。

### 4.4 决定是否继续循环

这是循环的分叉点：

- **没有工具调用（`needsFollowUp` 为假）**：进入结束检查。依次看停止钩子（`handleStopHooks`）、token 预算续写（`token_budget_continuation`）等。全部通过则返回 `completed`。[已验证，第 1062–1357 行区间]
- **有工具调用**：执行工具，把 `tool_result` 追加到消息，进入下一轮。下一轮的 `transition` 标记为 `next_turn`。[已验证，第 1725 行]

### 4.5 恢复路径

循环内有多种"出错后继续"的分支，每种都通过设置 `transition` 字段记录原因，便于调试和遥测：

| transition 原因 | 触发场景 | 位置 |
|---|---|---|
| `reactive_compact_retry` | 上下文超长，触发一次即时压缩后重试 | `query.ts` 约 1162 行 |
| `collapse_drain_retry` | 上下文折叠后重试 | 约 1110 行 |
| `max_output_tokens_escalate` | 输出 token 超限，提高上限后重试 | 约 1217 行 |
| `max_output_tokens_recovery` | 输出超限的恢复续写 | 约 1246 行 |
| `stop_hook_blocking` | 停止钩子要求继续 | 约 1302 行 |
| `token_budget_continuation` | 预算未用完，自动续写 | 约 1338 行 |

**设计要点**：模型输出超长、上下文超长这类"可恢复错误"，系统优先尝试自我修复（压缩、提高上限、续写），而不是直接报错。只有修复失败才以错误终止。[推断，基于上面的分支结构]

## 5. 终止原因

循环以 `Terminal` 类型返回终止原因（`query/transitions.ts`）。已在源码中看到的终止原因包括：

- `completed`：正常完成
- `max_turns`：超过最大回合数，并产生 `max_turns_reached` 附件消息（`query.ts` 约 1705 行）
- `model_error`：模型调用出错（约 996 行）
- `prompt_too_long`：上下文超长且无法恢复（约 1182 行）
- `image_error`：图片相关错误（约 977 行）
- `aborted_streaming` / `aborted_tools`：用户中断（约 1051、1515 行）
- `stop_hook_prevented` / `hook_stopped`：钩子阻止继续（约 1279、1520 行）
- `blocking_limit`：达到阻塞性上限（约 646 行）

## 6. 回合结束后的工作

- 记录转录：`recordTranscript`，见第 10 章。[已验证]
- 任务摘要：`maybeGenerateTaskSummary`，在满足条件时异步生成（`query.ts` 约 1690 行）。[已验证]
- 排队命令的附件：在回合中途注入用户的排队输入，见 `query.ts` 第 1547 行附近的注释。[已验证]

## 7. 一图回顾

```
submitMessage
  └─ processUserInput ──(shouldQuery=false)──► 直接返回
        │
        └─(shouldQuery=true)
              ▼
        query() 生成器
          └─ while(true)
               ├─ 压缩准备（budget / snip / microcompact / autocompact）
               ├─ callModel（流式）
               ├─ 工具调用？
               │    ├─ 否 → 停止钩子 / 预算续写 → completed
               │    └─ 是 → runTools → tool_result 回写 → next_turn
               └─ 可恢复错误 → 对应 transition 重试
```
