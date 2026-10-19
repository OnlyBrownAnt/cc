# 08 扩展机制：MCP、插件、技能与自定义命令

## 总体思路

系统不把所有功能都写死在核心里，而是提供几个扩展入口，让外部能力以统一方式接入。设计上的关键是：**外部能力最终都要变成核心已经认识的类型**（工具、命令、钩子、代理），这样核心逻辑不需要为每种扩展单独处理。

## 一、MCP（Model Context Protocol）

MCP 是一种让外部服务器提供工具和资源的协议。`services/mcp/` 负责接入。[已验证]

### 连接

- `connectToServer`（`services/mcp/client.ts` 第 595 行）：建立与某个 MCP 服务器的连接，结果被缓存（`memoize`）。[已验证]
- `fetchToolsForClient`（约第 1743 行）：获取服务器提供的工具列表，使用 LRU 缓存。[已验证]
- `fetchResourcesForClient`（约第 2000 行）：获取资源列表。[已验证]
- 传输层有多种实现：`InProcessTransport.ts`（进程内）、`SdkControlTransport.ts`（SDK 控制）、`utils/mcpWebSocketTransport.ts`（WebSocket）等。[已验证]

### 配置来源（作用域）

MCP 服务器的配置有七种作用域，定义在 `services/mcp/types.ts` 的 `ConfigScopeSchema`：[已验证]

`local`、`user`、`project`、`dynamic`、`enterprise`、`claudeai`、`managed`

不同作用域的优先级和可见性不同。例如 `managed` 来自组织托管的配置，`project` 可以随代码仓库共享。[推断，基于命名和设置来源的对应关系]

### 工具命名

MCP 工具在内部以 `mcp__<服务器名>__<工具名>` 的形式命名，源码中的 `Tool.ts` 注释说明，`mcpInfo` 保留了服务器返回的原始名称，即使使用了无前缀模式。[已验证]

### 认证与审批

- `auth.ts`、`oauth` 相关文件处理 OAuth 认证。[已验证]
- `mcpServerApproval.tsx`：首次接入某个服务器时需要用户批准。[已验证]
- `channelAllowlist.ts`、`channelPermissions.ts`：限制哪些服务器可以推送通道消息。[已验证]

**设计理由**：外部服务器是不可信的，接入时需要显式批准，并且其工具与内置工具一样经过权限系统。[推断]

## 二、插件（plugins）

插件打包了多种扩展：命令、代理、钩子、输出样式、MCP 服务器、LSP 集成。`utils/plugins/` 下的加载器按类型分开处理：[已验证]

- `loadPluginCommands.ts`
- `loadPluginAgents.ts`
- `loadPluginHooks.ts`
- `loadPluginOutputStyles.ts`
- `mcpPluginIntegration.ts`
- `lspPluginIntegration.ts`

插件还有一套市场体系：`marketplaceManager.ts`、`officialMarketplace.ts`、`pluginBlocklist.ts`（黑名单）、`dependencyResolver.ts`（依赖解析）、`pluginAutoupdate.ts`（自动更新）。[已验证文件存在]

**设计理由**：插件把一组相关扩展作为一个单位分发和管理，比让用户逐个配置更易用。黑名单和依赖解析是分发机制必需的治理能力。[推断]

## 三、技能（skills）

技能是一组说明和资源，模型可以按需加载。`skills/` 目录包括：[已验证]

- `bundledSkills.ts`：随产品内置的技能
- `loadSkillsDir.ts`：从目录加载用户或项目技能
- `mcpSkillBuilders.ts`：从 MCP 服务器构建技能

`tools/SkillTool/` 让模型调用技能。`commands.ts` 中的 `getSkillToolCommands` 和 `getSlashCommandToolSkills` 把技能同时暴露给模型（作为工具）和用户（作为斜杠命令）。[已验证]

**设计理由**：技能比工具更"软"，它是给模型读的指导，而不是直接执行的函数。把它做成可按需加载的单元，可以减少常驻上下文。[推断]

## 四、自定义斜杠命令

命令有多个来源：内置（`commands/`）、插件、MCP、用户目录。`commands.ts` 的 `getCommands(cwd)`（第 476 行）汇总它们。`Command` 类型的 `source` 字段（`types/command.ts` 第 32 行）标注来源：设置文件、内置、MCP、插件或内置包。[已验证]

`meetsAvailabilityRequirement`（第 417 行）检查命令是否满足运行条件，例如仅在某些认证方式下可用。[已验证]

**远程模式过滤**：`REMOTE_SAFE_COMMANDS` 与 `BRIDGE_SAFE_COMMANDS`（`commands.ts` 第 619、651 行）限定哪些命令可以在远程或桥接会话里执行，这是一种按场景的能力收窄。[已验证]

## 五、输出样式

`outputStyles/` 与插件的输出样式加载器配合，允许自定义模型回复的呈现方式。[已验证文件存在，具体机制待确认]

## 六、LSP

`services/lsp/` 与 `tools/LSPTool/` 提供语言服务器能力，让模型可以查询符号定义、引用等。[已验证文件存在]

## 扩展机制的共同模式

| 扩展类型 | 进入核心的形式 | 治理手段 |
|---|---|---|
| MCP 服务器 | Tool、Resource | 作用域、审批、通道白名单 |
| 插件 | 命令、代理、钩子、样式等 | 市场、黑名单、依赖解析 |
| 技能 | 技能工具与斜杠命令 | 来源标注、按需加载 |
| 自定义命令 | Command | 来源标注、可用性检查、远程过滤 |
| 钩子 | 事件回调 | 托管设置、权限不被绕过 |

这个表格体现了一个统一思路：**扩展越外部，治理越严格**。内置能力可以直接信任，插件和 MCP 需要审批，托管设置可以覆盖一切。
