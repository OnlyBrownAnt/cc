# 17 远程会话、桥接、任务与协作

这一章覆盖"会话不在本地运行"或"多个执行单元协作"的几类机制。深度低于前面的主线章节，重点是结构和边界。

## 一、远程会话（remote/）

`remote/` 目录让本地界面连接到一个远程运行的会话。[已验证目录与文件]

- `RemoteSessionManager`（`remote/RemoteSessionManager.ts` 第 95 行）：管理远程会话的生命周期，类型 `RemoteSessionConfig`（第 50 行）与 `RemoteSessionCallbacks`（第 64 行）定义输入与回调。
- `SessionsWebSocket`（`remote/SessionsWebSocket.ts` 第 82 行）：WebSocket 连接，接收远程会话的消息。
- `sdkMessageAdapter.ts`：把远程的 SDK 消息转换为本地界面能处理的格式。
- `remotePermissionBridge.ts`：把远程的权限请求转到本地界面，由本地用户决定，再把结果回送。
- `RemotePermissionResponse`（第 40 行）：权限响应的类型。

**设计要点**：远程会话中，工具实际在远端执行，但权限决定可以在本地做。这保持了第 05 章的权限语义：用户始终能在本地看到并批准敏感操作。[推断，基于 `remotePermissionBridge` 的命名与职责]

**远程模式的命令过滤**：只有 `REMOTE_SAFE_COMMANDS` 中的命令可用（见第 12 章）。[已验证]

## 二、桥接（bridge/）

`bridge/` 目录有 31 个文件，是最大的子目录之一。它的主入口是 `runBridgeLoop`（`bridge/bridgeMain.ts` 第 141 行）。[已验证]

- `bridgeEnabled.ts`：判断桥接是否可用。源码注释说明，**Remote Control 需要 claude.ai 订阅**，桥接使用订阅身份连接云端会话。[已验证，第 19 行注释]
- `bridgeConfig.ts`、`envLessBridgeConfig.ts`：配置。
- `bridgeApi.ts`、`codeSessionApi.ts`、`createSession.ts`：与服务端交互，创建和管理会话。
- `bridgeMessaging.ts`、`inboundMessages.ts`、`inboundAttachments.ts`：收发消息与附件。
- `bridgePermissionCallbacks.ts`：权限回调。
- `bridgeUI.ts`、`bridgeStatusUtil.ts`：界面状态。
- `capacityWake.ts`、`flushGate.ts`：容量与刷新控制。[已验证文件存在]
- `BackoffConfig`（`bridgeMain.ts` 第 59 行）与 `isConnectionError` / `isServerError`（第 1590、1604 行）：断线与服务端错误的退避处理。[已验证]

**设计理由**：桥接让用户在网页或其他设备上控制本地的 Claude Code，因此需要稳定的长连接、断线重连和权限回传。大量的退避与容量控制代码说明这是一个对稳定性要求高的子系统。[推断]

## 三、直连服务（server/）

`server/` 有三个文件，`DirectConnectSessionManager`（`server/directConnectManager.ts` 第 40 行）管理直连会话，`createDirectConnectSession.ts` 负责创建。[已验证]

直连与桥接的区别：桥接经过云端中转，直连由客户端直接连接到一个已运行的服务。[推断，基于命名与配置类型，未读完整链路]

## 四、任务系统（tasks/）

任务是"可以在后台运行、需要被跟踪的工作单元"。`tasks/types.ts` 定义了任务状态的联合类型 `TaskState`，共 7 种：[已验证]

| 任务类型 | 含义 |
|---|---|
| `LocalShellTask` | 本地 shell 命令（如后台运行的脚本） |
| `LocalAgentTask` | 本地子代理（见第 09 章） |
| `RemoteAgentTask` | 远程子代理 |
| `InProcessTeammateTask` | 同进程的协作成员 |
| `LocalWorkflowTask` | 工作流 |
| `MonitorMcpTask` | 监控 MCP 服务器 |
| `DreamTask` | 后台整理（与停止阶段的 auto-dream 相关，见第 06 章） |

**后台任务的判定**：`isBackgroundTask`（`tasks/types.ts` 第 37 行）要求任务状态为运行中或待执行，并且是显式后台化的，而不是前台任务。这决定了界面底部的后台任务指示器显示什么。[已验证]

**任务工具**：模型可以通过 `TaskCreateTool`、`TaskGetTool`、`TaskListTool`、`TaskUpdateTool`、`TaskOutputTool`、`TaskStopTool` 管理任务。[已验证目录存在]

**设计理由**：把不同来源的后台工作统一成任务模型，界面和管理工具就只需要处理一种抽象。[推断]

## 五、协调者模式（coordinator/）

`coordinator/coordinatorMode.ts` 实现了一种多代理协作模式，由环境变量 `CLAUDE_CODE_COORDINATOR_MODE` 开启。[已验证，第 38、66 行]

- `isCoordinatorMode()`（第 36 行）：判断是否处于协调者模式。
- `matchSessionMode()`（第 49 行）：恢复会话时匹配模式。
- `getCoordinatorSystemPrompt()`（第 111 行）：协调者使用专门的系统提示。
- `getCoordinatorUserContext()`（第 80 行）：协调者的用户上下文，包括 scratchpad 目录。

协调者模式与分叉子代理互斥：`isForkSubagentEnabled()` 在协调者模式下返回 false（见第 09 章）。[已验证]

**设计理由**：协调者负责拆分任务、分派给子代理、汇总结果，它的系统提示和上下文都与普通模式不同。把它做成模式而不是另一个程序，可以复用大部分核心逻辑。[推断]

## 六、团队与消息

- `TeamCreateTool`、`TeamDeleteTool`：创建和删除团队。
- `SendMessageTool`：代理之间发消息。
- `utils/swarm/`、`utils/teammateMailbox.ts`、`utils/inProcessTeammateHelpers.ts`：团队协作的底层实现。[已验证文件存在]
- `--team-name`、`--agent-id`、`--teammate-mode` 等命令行选项（`main.tsx`）：这些选项为协作成员启动服务。[已验证，选项定义存在，`--teammate-mode` 可选 `auto`、`tmux`、`in-process`]

**待确认**：协作的消息投递语义（可靠性、顺序、重试）未读完整实现，此处不做结论。[待确认]

## 设计上的取舍

- **远程与本地的权限一致性**：远程执行但本地批准，需要两端都理解权限语义，增加了协议复杂度。[推断]
- **任务抽象的通用性 vs 类型差异**：7 种任务状态差异很大，统一抽象需要联合类型和守卫函数。[推断]
