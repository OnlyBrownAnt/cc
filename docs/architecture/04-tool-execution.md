# 04 工具执行：并发、流式与结果处理

## 两个执行器

模型返回的一个回合里可能包含多个 `tool_use` 块。执行方式有两种：

| 方式 | 入口 | 触发条件 | 特点 |
|---|---|---|---|
| 批量执行 | `runTools()`（`services/tools/toolOrchestration.ts`） | 模型输出完毕后 | 实现简单，逻辑集中 |
| 流式执行 | `StreamingToolExecutor`（`services/tools/StreamingToolExecutor.ts`） | `config.gates.streamingToolExecution` 为真 | 模型还在输出时就开始执行，降低延迟 |

切换由 `query.ts` 第 561 行附近决定。[已验证]

## 批量执行的分批规则

`runTools()` 的核心是 `partitionToolCalls()`（`toolOrchestration.ts`）。它把一批工具调用切成若干"批"，规则是：

1. 每个工具调用先用 `inputSchema.safeParse` 校验，校验失败按非并发安全处理。[已验证]
2. 调用 `tool.isConcurrencySafe(parsedInput.data)` 判断，**按输入**决定。如果判断函数抛异常（例如 shell 解析失败），保守地视为非并发安全。[已验证]
3. **连续的**并发安全工具合并为一批，非并发安全工具单独成批。[已验证]

执行时：

- 并发安全的批：`runToolsConcurrently`，并发上限默认 10，可用环境变量 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` 覆盖。[已验证，`toolOrchestration.ts` 第 8–11 行与第 175 行]
- 非并发安全的批：`runToolsSerially`，逐个执行。[已验证]

**设计要点**：读操作可以并行，写操作必须串行，这是一个简单而正确的近似。"能否并行"由工具在具体输入下自己判断，而不是一刀切。

### 上下文修改器

并发批里，工具可能想修改后续调用共享的上下文（例如切换当前目录）。源码的做法是：

- 收集每个工具产生的 `contextModifier`，按 `toolUseID` 归组。
- 等整批结束后，按原始顺序依次应用。

这样即使并发执行，上下文的变化仍然是确定的、按顺序的。[已验证]

## 流式执行器

`StreamingToolExecutor` 的注释概括了它的规则：[已验证]

- 并发安全的工具可以与其他并发安全工具并行。
- 非并发安全的工具必须独占执行。
- **结果按工具接收的顺序输出**，即使执行完成的顺序不同。这保证了模型看到的结果顺序稳定。

它还处理几类取消：

| 原因 | 含义 |
|---|---|
| `sibling_error` | 同批某个 Bash 工具出错，兄弟子进程应被杀掉，避免白跑 |
| `user_interrupted` | 用户新消息到来，且工具的 `interruptBehavior` 是 `cancel` |
| `streaming_fallback` | 流式执行中途回退到批量路径，丢弃已执行的结果 |

其中 `sibling_error` 通过一个子 `AbortController` 实现，它是主取消信号的子节点。这样兄弟工具可以被取消，而主控制器本身不受影响。[已验证，`StreamingToolExecutor.ts` 第 45–48 行、第 294–308 行]

**设计理由**：流式执行降低延迟，但要保证结果顺序、错误传播和取消语义都正确。把这三件事封在一个类里，主循环只需要调用 `addTool` 和 `getRemainingResults`。[推断]

## 单个工具的执行管线

`services/tools/toolExecution.ts`（约 1745 行）的 `runToolUse` 与 `checkPermissionsAndCallTool` 定义了单次调用的完整步骤。按源码中的顺序：[已验证]

1. **Zod 输入校验**：失败则返回 `InputValidationError` 类型的 `tool_result`，`is_error: true`。失败时还会附加提示，说明某些延迟加载的工具 schema 没有发送给模型。
2. **`validateInput`**：业务层校验，失败返回错误结果。
3. **PreToolUse 钩子**：可以修改输入、直接给出权限决定、或阻止执行。见第 06 章。
4. **权限决策**：`resolveHookPermissionDecision` 结合钩子结果与 `canUseTool`，得到最终决定。见第 05 章。
5. **执行**：调用 `tool.call(...)`。
6. **PostToolUse 钩子**：可以替换工具输出（MCP 工具可以返回 `updatedMCPToolOutput`）。
7. **结果映射与截断**：应用 `maxResultSizeChars`，超长内容落盘。

**关键顺序**：权限检查在 PreToolUse 钩子**之后**。这意味着钩子可以修改输入，权限检查看到的是修改后的输入，从而防止"先过权限、后改参数"的绕过。源码注释在第 775–781 行附近专门说明了这一点。[已验证]

## 错误的表达方式

所有错误最终都变成一条 `tool_result`（`is_error: true`），而不是抛出异常中断整个回合。模型可以看到错误并自行修正。这是一个重要的设计选择：**工具失败是对话的一部分，不是系统故障。** [推断]

## 结果预算

`applyToolResultBudget`（`query.ts` 中调用）在每次模型调用前限制消息里工具结果的总量。它与 `maxResultSizeChars` 配合：单个工具结果有上限，整体也有上限。被截断或落盘的内容，模型拿到的是预览和路径，需要时可以再读。[已验证]
