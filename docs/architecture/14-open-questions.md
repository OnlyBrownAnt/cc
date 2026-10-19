# 14 待确认问题与覆盖情况

## 一、覆盖情况

快照约 1,900 个文件。下表说明本文档对各子系统的覆盖程度。

| 子系统 | 目录 | 覆盖程度 | 所在章节 |
|---|---|---|---|
| 主线：查询循环 | `query.ts`、`QueryEngine.ts`、`query/` | 深入 | 02 |
| 工具契约与执行 | `Tool.ts`、`services/tools/` | 深入 | 03、04 |
| 权限 | `utils/permissions/`、`types/permissions.ts` | 深入（决策链与模式），分类器未深入 | 05 |
| 钩子 | `utils/hooks/`、`services/tools/toolHooks.ts` | 中等（事件与执行位置），各执行器细节未读 | 06 |
| 压缩与上下文 | `services/compact/` | 中等（阈值与流程），摘要提示词未读 | 07 |
| 扩展（MCP/插件/技能） | `services/mcp/`、`utils/plugins/`、`skills/` | 中等（结构与模式），协议细节未读 | 08 |
| 子代理 | `tools/AgentTool/` | 中等 | 09 |
| 会话与记忆 | `utils/sessionStorage.ts`、`memdir/`、`services/SessionMemory/`、`services/extractMemories/` | 中等 | 10 |
| 启动与模式 | `main.tsx`、`setup.ts`、`utils/settings/` | 中等（阶段与模式），细节很多未读 | 11 |
| 输入与命令 | `utils/processUserInput/`、`commands.ts`、`commands/` | 中等（分流与类型），单个命令未读 | 12 |
| 终端界面 | `ink/`、`components/`、`screens/`、`hooks/`、`vim/` | 中等（结构与主界面），单个组件未读 | 16 |
| 模型 API 层 | `services/api/` | 中等（请求组装、重试、降级） | 15 |
| 遥测与分析 | `services/analytics/` | 中等（设计约束与开关） | 18 |
| 远程与桥接 | `remote/`、`bridge/`、`server/` | 结构层面（入口与职责），协议细节未读 | 17 |
| 协作与团队 | `coordinator/`、`utils/swarm/`、`tools/Team*` | 结构层面，消息语义待确认 | 17 |
| 任务系统 | `tasks/`、`tools/Task*` | 中等（任务类型与后台判定） | 17 |
| 语音 | `voice/`、`services/voice*` | 仅见入口 | 18 |
| 其他特色模块 | `buddy/`、`assistant/`、`moreright/`、`upstreamproxy/`、`native-ts/` | 仅见入口，见第 18 章逐项说明 | 18 |
| 具体工具实现 | `tools/*/` | 只覆盖契约，未逐个解释 | 03 |
| 具体命令实现 | `commands/*/` | 只覆盖分类，未逐个解释 | 12 |

**结论**：本文档讲清了**主线与核心机制**，能回答"一次请求怎么流转、为什么这样分层"。但它没有覆盖界面、远程、协作、任务、语音等子系统，也没有逐个解释工具和命令。如果要"完整"地覆盖，需要按上表的"未覆盖"项补充章节。

## 二、需要运行时验证的问题

源码能说明"代码怎么写"，但以下问题只能靠运行观察确认：

1. 流式工具执行在真实环境中是否默认开启？`streamingToolExecution` 的灰度比例未知。
2. 各压缩层的实际触发频率与效果，需要长会话的观察数据。
3. 权限分类器在 `auto` 模式下的判定质量与误判率。
4. 钩子在失败、超时情况下的行为（源码中有超时常量，但实际表现待测）。
5. MCP 服务器断线重连的完整状态流转。

## 三、需要读源码才能确认的问题

以下问题文档中标注为 [待确认] 或 [推断]，需要进一步阅读：

1. `services/api/claude.ts` 如何组装请求、处理重试与降级（`withRetry.ts`、`fallbackModel`）。
2. `permissions.ts` 中 `handleDenialLimitExceeded` 的具体动作。
3. `PermissionUpdate` 如何写入不同来源的设置文件。
4. `services/compact/prompt.ts` 中摘要提示词的结构。
5. `utils/hooks.ts` 中命令型钩子的执行与超时逻辑。
6. `skills/loadSkillsDir.ts` 的加载优先级与冲突处理。
7. `memdir/findRelevantMemories.ts` 的相关性算法。
8. `setup.ts` 中 worktree 创建失败的回退路径。

## 四、文档本身的局限

- 文档基于源码静态阅读，没有运行验证。
- 行号以快照为准，同一快照内有效。
- "设计理由"部分多为推断，已用标签标出。读者应把它们当作假设，而不是定论。
- 快照是非官方来源，其内容可能与正式版本不同。
