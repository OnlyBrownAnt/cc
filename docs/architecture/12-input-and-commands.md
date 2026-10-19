# 12 输入处理与斜杠命令

## 一次输入的四种去向

用户在终端输入的内容，经过 `processUserInput`（`utils/processUserInput/processUserInput.ts`）分流：[已验证]

| 类型 | 处理函数 | 是否调用模型 |
|---|---|---|
| 斜杠命令（`/xxx`） | `processSlashCommand`（`processSlashCommand.tsx` 第 309 行） | 视命令类型而定 |
| Bash 模式（以 `!` 开头等） | `processBashCommand`（`processBashCommand.tsx`） | 否，直接执行 shell |
| 普通文本 | `processTextPrompt.ts` | 是 |
| 其他（如空输入） | 直接返回 | 否 |

`processUserInput` 的返回值包含 `shouldQuery`，决定是否进入模型循环（见第 02 章）。[已验证]

## 命令的三种类型

`types/command.ts` 定义了命令的类型字段 `type`，有三种取值：[已验证]

| 类型 | 含义 | 是否调用模型 | 例子 |
|---|---|---|---|
| `prompt` | 把命令内容展开为一段提示词，交给模型处理 | 是 | 自定义命令、技能 |
| `local` | 在本地执行，返回文本结果 | 否 | `/cost`、`/version` |
| `local-jsx` | 本地执行，并渲染一个界面组件 | 否 | `/config`、`/resume` |

**设计理由**：不是所有命令都需要模型。把"本地可以完成的事"与"需要模型推理的事"区分开，既省 token，又让响应更快。[推断]

`prompt` 类型的处理函数是 `processPromptSlashCommand`（`processSlashCommand.tsx` 第 817 行）。[已验证]

## 命令的注册与汇总

`commands.ts` 的 `getCommands(cwd)`（第 476 行）汇总所有来源的命令：内置、技能、插件、MCP、用户与项目目录。[已验证]

几个关键点：

- `builtInCommandNames`（第 348 行）是记忆化的内置命令名集合，用于判断冲突。[已验证]
- `INTERNAL_ONLY_COMMANDS`（第 225 行）列出仅内部使用的命令。[已验证]
- `meetsAvailabilityRequirement`（第 417 行）按认证方式、平台等条件过滤命令。[已验证]
- 缓存清理函数（`clearCommandMemoizationCaches`、`clearCommandsCache`）存在，说明命令列表可以在运行时刷新，例如插件重载后。[已验证]

## 远程与桥接的命令过滤

远程会话和桥接会话只允许部分命令。`REMOTE_SAFE_COMMANDS`（第 619 行）与 `BRIDGE_SAFE_COMMANDS`（第 651 行）是白名单，`filterCommandsForRemoteMode`（第 684 行）据此过滤。[已验证]

**设计理由**：远程场景下，命令可能访问本地不该暴露的资源。白名单比黑名单更安全，新增命令默认不可远程使用。[推断]

## 命令与工具的边界

- 命令是**用户**入口，斜杠命令由用户主动输入。
- 工具是**模型**入口，由模型在回合中调用。
- 技能（skills）同时出现在两边：模型可以通过 `SkillTool` 调用，用户也可以用斜杠命令调用。[已验证，见第 08 章]

这条边界是系统权限模型的一部分：模型不能随意调用只属于用户的命令（例如清空会话），因此这些命令不是工具。[推断]

## 键位与交互

- `keybindings/` 处理键位映射，支持用户自定义（`loadUserBindings.ts`）。[已验证]
- `reservedShortcuts.ts` 列出不允许覆盖的保留键，防止用户把关键操作（如中断）绑定到别处。[已验证文件存在]
- `vim/` 提供 Vim 模式。[已验证目录存在]
- `history.ts` 记录输入历史。[已验证文件存在]

## 中途输入

回合执行期间，用户的新输入会进入队列。`query.ts` 在工具执行后会把排队的命令作为附件注入（第 1547 行附近）。但斜杠命令**不会**在回合中途注入给模型，而是等回合结束后通过 `processSlashCommand` 处理。[已验证，`query.ts` 注释]

**设计理由**：斜杠命令有本地副作用（如切换模式），如果混进模型上下文，模型会把它当成普通文字误读。等回合结束再执行更可控。[推断]

工具的 `interruptBehavior()` 决定新输入到来时的行为：`cancel` 停止当前工具，`block` 让新输入等待（第 03 章）。[已验证]
