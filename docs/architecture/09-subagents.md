# 09 子代理

## 是什么

子代理（sub-agent）是主对话里再启动的一个独立模型循环，用于完成一个子任务，然后把结果返回给主对话。主要入口是 `AgentTool`（`tools/AgentTool/`）。[已验证]

## 为什么需要

- **隔离上下文**：子任务的大量中间输出不污染主对话的上下文窗口。[推断]
- **并行**：多个子代理可以同时探索不同方向。[推断]
- **专业化**：不同代理可以有不同的系统提示、工具集和模型。[已验证，见下文]

## 内置代理

`tools/AgentTool/built-in/` 下有六个内置代理：[已验证]

- `exploreAgent.ts`：探索代码库
- `planAgent.ts`：制定计划
- `generalPurposeAgent.ts`：通用
- `verificationAgent.ts`：验证结果
- `claudeCodeGuideAgent.ts`：回答产品使用问题
- `statuslineSetup.ts`：配置状态栏

其中探索和计划代理是否启用由特性开关和运行期灰度共同决定（`builtInAgents.ts` 中的 `areExplorePlanAgentsEnabled`）。[已验证]

用户和项目也可以通过 `loadAgentsDir.ts` 定义自己的代理，插件也能提供代理。[已验证]

## 核心执行：runAgent

`runAgent()`（`tools/AgentTool/runAgent.ts` 第 248 行）是子代理的执行体。它是一个异步生成器，接收的参数体现了子代理的配置维度：[已验证]

- `agentDefinition`：代理的定义（提示、工具、模型）
- `promptMessages` 与 `forkContextMessages`：初始消息与可选的上下文分叉
- `canUseTool`：权限回调，与主线程共用决策逻辑
- `isAsync`：是否后台运行
- `canShowPermissionPrompts`：子代理能否弹出权限确认
- `availableTools` / `allowedTools` / `useExactTools`：工具可见性
- `model`、`maxTurns`：模型与回合上限
- `worktreePath`：可在独立 git worktree 中工作
- `transcriptSubdir`：转录写入子目录，与主会话分开

**设计要点**：子代理复用同一套 `query()` 循环和权限系统，而不是另写一套。这保证了子代理的行为与主线程一致，也意味着它同样受权限约束。[推断]

## 分叉（fork）

`forkSubagent.ts` 提供"分叉"模式：子代理继承父对话的上下文，而不是从零开始。[已验证]

- `isForkSubagentEnabled()`：仅在特性开关开启、非协调模式、非无头会话时可用。[已验证]
- `buildForkedMessages()`：构造分叉后的初始消息。[已验证]
- `isInForkChild()`：识别当前是否已在分叉子代理里，防止无限嵌套。[已验证]

**设计理由**：分叉的好处是子代理立刻了解背景，但代价是上下文占用高。分叉与普通子代理并存，让用户和系统可以按任务选择。[推断]

## 缓存安全参数

`runAgent` 接收 `onCacheSafeParams`，源码中 `stopHooks.ts` 也会调用 `saveCacheSafeParams`。这是为了让分叉代理能共享父代理的提示缓存，减少重复计算。[已验证]

## 后台与恢复

- `isAsync`：子代理可以在后台运行，结果稍后取回。[已验证]
- `resumeAgent.ts`：从已保存的转录恢复代理。[已验证文件存在]
- `agentMemory.ts`、`agentMemorySnapshot.ts`：代理级别的记忆。[已验证文件存在]

## 协作：团队与消息

多代理协作还涉及 `TeamCreateTool`、`TeamDeleteTool`、`SendMessageTool` 等工具，以及 `utils/swarm/`、`utils/teammateMailbox.ts`。这些是协作层，本文不展开，见第 14 章的覆盖说明。[已验证文件存在]

## 权限冒泡

子代理无法直接弹出权限确认时，权限请求会"冒泡"到父代理，由父代理的界面处理。这对应权限模式中的内部模式 `bubble`（见第 05 章）。[推断，基于 `bubble` 模式的存在]

## 设计上的取舍

- **复用循环 vs 独立循环**：复用保证一致性，代价是子代理的行为受主循环所有改动的影响。[推断]
- **隔离 vs 共享**：默认隔离上下文，分叉时共享。隔离保护主线程，共享提高连贯性。[推断]
- **后台运行 vs 结果一致性**：异步代理结果可能晚到，需要额外的附件机制（`createAsyncAgentAttachmentsIfNeeded`）把结果带回主线程。[已验证，`compact.ts` 第 1568 行附近]
