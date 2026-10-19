# 01 全局视图

## 分层

按职责可以分成六层。每层只依赖下层的接口，不反向依赖。

| 层 | 主要目录/文件 | 职责 |
|---|---|---|
| 入口层 | `main.tsx`, `entrypoints/`, `replLauncher.tsx` | 解析命令行，决定交互式、无头（`-p`）、SDK、远程等模式 [已验证] |
| 启动与配置层 | `setup.ts`, `bootstrap/`, `utils/settings/`, `migrations/` | 初始化工作目录、设置来源（用户/项目/托管）、迁移旧配置 [已验证] |
| 交互与界面层 | `screens/`, `components/`, `ink/`, `hooks/`, `keybindings/`, `vim/` | 基于 React + Ink 的终端 UI [已验证] |
| 核心编排层 | `QueryEngine.ts`, `query.ts`, `query/`, `utils/processUserInput/` | 管理一次会话的消息流，驱动模型调用循环 [已验证] |
| 工具与执行层 | `Tool.ts`, `tools.ts`, `tools/`, `services/tools/` | 定义工具契约，调度与执行工具调用 [已验证] |
| 服务与基础设施层 | `services/api/`, `services/mcp/`, `services/compact/`, `services/analytics/`, `utils/permissions/`, `utils/hooks.ts` | 模型 API、MCP 连接、压缩、遥测、权限、钩子 [已验证] |

## 两套入口：命令与工具

这是读懂整个系统最关键的一点。

- **命令（commands）** 面向**用户**，即斜杠命令，如 `/config`、`/compact`、`/mcp`。注册在 `commands.ts`，目录 `commands/` 下约 50 个实现。[已验证]
- **工具（tools）** 面向**模型**，即模型在回合中可以发起的函数调用，如读文件、执行 Bash、编辑文件。注册在 `tools.ts`，目录 `tools/` 下约 40 个实现。[已验证]

两者分开的设计理由：用户操作和模型能力的权限边界不同，暴露方式也不同。用户的 `/clear` 不应该被模型随意调用，模型的 `FileEdit` 也不应该作为用户命令直接暴露。[推断]

## 构建期开关与运行期开关

源码里有两类开关，设计上区分得很清楚：

1. **构建期开关**：`feature('XXX')`，来自 `bun:bundle`，在构建时决定代码是否被打包。约 196 个文件使用。未启用的分支会被死代码消除。[已验证]
2. **运行期开关**：如 `config.gates.streamingToolExecution`，通过 Statsig 等服务在运行时取值，允许灰度发布。[已验证]

设计含义：构建期开关用于裁剪功能（例如内部版本才有的工具），运行期开关用于控制已发布功能的逐步放量。两者不要混用。

## 主线与周边

```
用户输入
  │
  ▼
入口层 main.tsx ──► setup ──► 选择模式（交互 / 无头 / SDK）
  │
  ▼
核心编排 QueryEngine.submitMessage
  │   ├─ processUserInput：区分斜杠命令、Bash 模式、普通文本
  │   ▼
  ▼
query() 循环 ──► 调用模型（services/api）
  │                 │
  │                 ▼
  │          模型返回文本 或 工具调用
  │                 │
  │                 ▼
  │       工具执行层 runTools ──► 权限决策 ──► 钩子 ──► Tool.call
  │                 │
  │                 ▼
  └────── 工具结果回写消息，进入下一轮循环
```

周边子系统（压缩、MCP、记忆、子代理、会话持久化）都挂在这条主线的某一步上，详见后续章节。
