# 11 启动流程与运行模式

## 启动的几个阶段

从 `main()`（`main.tsx` 第 585 行）开始，启动可以分成五个阶段：[已验证]

1. **早期准备**：设置安全环境变量、警告处理器、信号处理（`main.tsx` 约第 585–600 行）。Windows 下禁止从当前目录执行命令，以防 PATH 劫持。[已验证]
2. **命令行解析**：`run()` 中使用 Commander 定义命令和选项。`preAction` 钩子只在真正执行命令时才初始化，显示帮助时不触发。[已验证，约第 884–910 行]
3. **模式判定**：根据参数判断是否非交互。`-p/--print`、`--init-only`、`--sdk-url`，或标准输出不是 TTY，都视为非交互。[已验证，第 797–803 行]
4. **setup**：`setup()`（`setup.ts` 第 56 行）处理工作目录、git worktree、会话 ID、消息套接字等。[已验证]
5. **启动界面或无头执行**：交互模式调用 `launchRepl`；无头模式调用 `runHeadless`。[已验证，第 2829 行与多处 `launchRepl`]

## 运行模式

从入口看，主要有四种模式：[已验证]

| 模式 | 触发 | 特点 |
|---|---|---|
| 交互式 REPL | 默认，TTY 环境 | React + Ink 界面，有信任对话框 |
| 无头（`-p`） | `--print` 或非 TTY | 不显示界面，单次输出，跳过信任对话框 |
| SDK / 流式 JSON | `--input-format stream-json` 或 `--sdk-url` | 通过标准输入输出或 WebSocket 与外部程序交互 |
| 远程 / 桥接 | `--remote`、`--remote-control` 等 | 会话在远程运行，本地只做显示和控制 |

另有若干子命令和特殊入口（`doctor`、`mcp`、`plugin`、`auth` 等），它们不需要完整初始化。[已验证，`main.tsx` 第 3894 行起注册 `mcp` 子命令]

### 无头模式的信任

源码注释说明，无头模式跳过信任对话框，因为它没有界面可以交互。[已验证，`main.tsx` 第 361–366 行]

**设计理由**：信任对话框保护用户免于在不可信目录中执行代码。无头模式下调用者是程序而不是人，信任的前提由调用者承担。这个取舍需要使用者清楚理解。[推断]

## 信任检查

交互模式下，`checkHasTrustDialogAccepted()`（`main.tsx` 第 372 行附近）判断当前目录是否已被信任。未信任时显示 `showSetupScreens()`。注释提到，在显示信任对话框之前，程序会运行约 1000 行初始化代码，目的是尽量早地完成与信任无关的工作。[已验证，`main.tsx` 第 1044 行附近注释]

## 设置来源

设置（settings）按五个来源加载，`utils/settings/constants.ts` 中的 `SETTING_SOURCES` 定义了顺序：[已验证]

1. `userSettings`：用户全局
2. `projectSettings`：项目共享（可提交到仓库）
3. `localSettings`：项目本地（不提交）
4. `flagSettings`：命令行 `--settings` 指定
5. `policySettings`：组织托管（`managed-settings.json` 或远程下发）

`settings.ts` 的 `getSettingsForSource`（第 309 行）读取单个来源，`getInitialSettings`（第 812 行）返回合并后的结果。[已验证]

**设计理由**：分层让个人偏好、团队约定、组织强制策略各得其所。托管策略位于最高层，用户无法通过本地设置覆盖。[推断，基于来源顺序]

## 工作目录与 worktree

`setup()` 可以创建 git worktree，并通过 `process.chdir` 切换到新目录。注释特别说明，`setup()` 的 chdir 会影响后续所有依赖当前目录的代码，因此很多初始化必须等它完成。[已验证，`setup.ts` 第 216–271 行，`main.tsx` 第 1903–1927 行注释]

**设计要点**：启动顺序之所以重要，是因为"当前工作目录"是全局状态。这是一个容易出错的地方，源码在多处用注释提醒。[推断]

## 并行化

启动过程大量使用 `Promise.all` 和预取（`startDeferredPrefetches`，`main.tsx` 第 388 行）。例如 `setup()` 与命令、代理加载并行执行，以减少启动时间。[已验证]

## 遥测与日志

`initSinks()`（`setup.ts` 约第 371 行）在设置完成后挂载错误日志和分析事件的接收器，之前的事件先排队。[已验证]

`--hard-fail` 选项让 `logError` 直接崩溃而不是静默记录，用于开发调试。[已验证，`main.tsx` 第 3871 行]
