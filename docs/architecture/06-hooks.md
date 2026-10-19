# 06 钩子系统

## 是什么

钩子（hooks）让用户或项目在关键事件上插入自己的逻辑，而不需要修改源码。事件包括工具执行前后、会话开始结束、停止、子代理结束、压缩前后、权限请求等。

## 事件清单

`entrypoints/sdk/coreTypes.ts` 中的 `HOOK_EVENTS` 数组列出了全部事件（共 27 个）。[已验证]

按生命周期分组：

| 阶段 | 事件 |
|---|---|
| 会话 | `SessionStart`、`SessionEnd`、`Setup` |
| 用户输入 | `UserPromptSubmit`、`Notification` |
| 工具 | `PreToolUse`、`PostToolUse`、`PostToolUseFailure`、`PermissionRequest`、`PermissionDenied` |
| 回合结束 | `Stop`、`StopFailure` |
| 子代理与团队 | `SubagentStart`、`SubagentStop`、`TeammateIdle`、`TaskCreated`、`TaskCompleted` |
| 压缩 | `PreCompact`、`PostCompact` |
| 其他 | `Elicitation` 等 |

## 钩子的四种执行形式

`utils/hooks/` 下有四种执行器：[已验证]

- `execHttpHook.ts`：向指定地址发送请求
- `execPromptHook.ts`：用模型对输入做一次判断
- `execAgentHook.ts`：启动一个代理执行检查
- 命令形式：执行一条 shell 命令（具体实现位置[待确认]，`utils/hooks.ts` 中有大量钩子相关类型和逻辑）

**设计理由**：不同检查的成本和确定性不同。Shell 命令最便宜、最确定；提示词钩子和代理钩子能处理语义判断，但更慢、更贵。同一个接口支持多种形式，让用户按需选择。[推断]

## 工具钩子的位置

工具钩子是第 04 章管线的一部分，执行顺序是：[已验证，`services/tools/toolHooks.ts`、`toolExecution.ts`]

1. `runPreToolUseHooks`：在权限检查之前运行。钩子的输出可能是：
   - 修改后的输入（`hookUpdatedInput`）
   - 直接的权限决定（`hookPermissionResult`）
   - 阻止执行的信号
2. 权限检查（`resolveHookPermissionDecision`）：若钩子已给出决定，直接采用；否则进入正常决策链。
3. `runPostToolUseHooks`：工具执行后运行。可以替换输出（MCP 工具的 `updatedMCPToolOutput`）或附加消息。
4. 失败时运行 `runPostToolUseFailureHooks`。

**关键设计**：PreToolUse 钩子可以**修改输入**，且权限检查看的是修改后的输入。这保证钩子不会成为绕过权限的漏洞。[已验证，`toolExecution.ts` 第 775–790 行]

## 停止钩子与后台任务

`query/stopHooks.ts` 的 `handleStopHooks` 在模型给出最终回答（没有工具调用）时运行。它除了执行用户配置的 `Stop` 钩子，还会触发几个后台任务。非精简模式（`--bare` 之外）下，它们的触发条件如下：[已验证]

- `executePromptSuggestion`：提示建议，由环境变量控制，不限于主线程
- `executeExtractMemories`：从对话中提取记忆（见第 10 章），需要编译期开关且只在主线程
- `executeAutoDream`：后台整理，只在主线程

这些任务用"即发即忘"（`void`）的方式启动，不阻塞用户。

**设计理由**：收尾工作不应该拖慢用户看到回答。把它们放在 `Stop` 阶段并异步执行，是一种很自然的分离。[推断]

## 钩子的配置来源

钩子和其他配置一样，来自多个设置源（见第 11 章）。插件也可以提供钩子（`utils/plugins/loadPluginHooks.ts`）。[已验证]

## 设计上的取舍

- **钩子能力强，安全面大**：钩子可以执行任意命令。因此它们受到和工具同样的约束：权限检查不被绕过，托管设置可以限制。[推断]
- **多种执行形式**：统一接口背后是四种实现，增加了测试和调试成本。[推断]
