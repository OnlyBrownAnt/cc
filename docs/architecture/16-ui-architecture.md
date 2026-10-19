# 16 终端界面架构

## 技术选型

界面基于 React，渲染到终端。快照中的 `ink/` 目录是对 Ink 的内部实现，包含自己的 reconciler、布局、输出与键盘事件处理。[已验证，`ink/reconciler.ts`、`ink/renderer.ts`、`ink/output.ts` 等文件存在]

布局计算使用 `native-ts/` 中的纯 TypeScript 实现，它是 Meta 的 yoga-layout（flexbox 引擎）的简化移植，只覆盖 Ink 实际用到的特性（如 flex-direction 的 row/column 及 reverse）。[已验证，`native-ts/` 文件头部注释]

**设计理由**：终端不是浏览器，没有现成的 DOM 与 CSS。用 React 描述界面，再用 flexbox 布局、再渲染到字符网格，是在终端里获得组件化开发体验的路径。纯 TypeScript 的布局引擎避免了原生依赖，便于在 Bun 下分发。[推断]

## 分层

| 层 | 位置 | 职责 |
|---|---|---|
| 渲染基础 | `ink/` | reconciler、布局、输出、事件 [已验证] |
| 布局引擎 | `native-ts/` | flexbox 计算 [已验证] |
| 通用组件 | `components/`（约 389 个文件） | 消息、对话框、输入框、进度等 [已验证] |
| 页面 | `screens/` | REPL（主界面）、Doctor（诊断）、ResumeConversation（恢复会话）[已验证] |
| 状态与钩子 | `state/`、`hooks/`、`context/` | 应用状态与副作用 [已验证目录存在] |

## 主界面：REPL

`screens/REPL.tsx` 约 5,000 行，是最大的单个界面文件。它组合了消息列表、输入框、权限对话框、进度显示、后台任务指示器等。[已验证行数]

**观察**：如此大的界面组件说明主界面承载了大量状态和交互。这是一个值得注意的设计债务来源，也解释了为什么源码中有大量与界面状态相关的注释。[推断]

## 权限对话框

权限询问以对话框的形式出现在 REPL 中，对应多种对话框组件，例如 `BypassPermissionsModeDialog`、`AutoModeOptInDialog`、`ChannelDowngradeDialog`。它们把第 05 章的决策结果呈现给用户。[已验证组件存在]

## 工具结果的展示

每种工具有对应的展示组件，例如 `FileEditToolDiff.tsx`、`FileEditToolUpdatedMessage.tsx`、`FileEditToolUseRejectedMessage.tsx`。被拒绝、出错、回退的工具调用各有独立组件，如 `FallbackToolUseErrorMessage.tsx`。[已验证]

工具可以声明 `isSearchOrReadCommand`（第 03 章），让连续的搜索或读取操作在界面上折叠为一行，减少视觉噪音。[已验证，`Tool.ts`]

## 全屏与普通模式

`FullscreenLayout.tsx` 与 `utils/fullscreen.ts` 处理全屏布局。`components/` 下还有 `CtrlOToExpand.tsx` 等组件，允许用户展开折叠的内容。[已验证文件存在]

## 键盘与快捷键

- `keybindings/`：加载默认绑定与用户绑定，解析与匹配按键。[已验证]
- `reservedShortcuts.ts`：保留键，防止覆盖关键操作。[已验证文件存在]
- `vim/`：Vim 模式。[已验证目录存在]

## 历史与搜索

`HistorySearchDialog.tsx` 与 `GlobalSearchDialog.tsx` 提供历史输入与全局搜索。[已验证组件存在]

## 其他

- `buddy/`：一个伴随式的字符动画组件（见第 18 章）。
- `ClaudeCodeHint`、`EffortIndicator`、`ContextVisualization` 等组件展示提示与状态。[已验证组件存在]

## 界面与核心的边界

界面不直接调用模型或工具。它通过 `QueryEngine` 的 `submitMessage` 提交输入，通过回调接收消息流和权限请求。权限请求通过 `canUseTool` 回调，在界面里弹出对话框，用户的选择回到决策链。[已验证，见第 02、05 章]

**设计理由**：把界面和核心分开，使同一套核心可以驱动 REPL、无头模式和 SDK。界面只是多个"外壳"之一。[推断]

## 设计上的取舍

- **React 的组件化 vs 终端的性能**：终端刷新成本高，需要 reconciler 与输出层的优化。`ink/optimizer.ts`、`frame.ts`、`log-update.ts` 等文件体现了这类优化。[已验证文件存在，具体算法未读]
- **自研渲染层 vs 通用库**：自研可以精确控制行为，代价是维护成本。[推断]
