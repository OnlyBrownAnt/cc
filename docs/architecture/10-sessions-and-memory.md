# 10 会话、转录与记忆

这一章说明三类持久化数据：**会话转录**（发生过什么）、**会话记忆**（当前会话的摘要笔记）、**自动记忆**（跨会话保留的知识）。它们的存储位置、写入时机和用途各不相同。

## 一、会话转录（transcript）

### 存储

转录以 JSONL 格式（每行一个 JSON 条目）保存在用户配置目录下：[已验证]

- 目录：`<配置主目录>/projects/<项目目录>/`（`utils/sessionStorage.ts` 中的 `getProjectsDir`）
- 文件：`<会话 ID>.jsonl`（`getTranscriptPath`）
- 子代理的转录写入子目录，与主会话分开（`setAgentTranscriptSubdir`）[已验证]
- 读取有大小上限：`MAX_TRANSCRIPT_READ_BYTES = 50 MB`[已验证]

### 写入

`recordTranscript()`（`utils/sessionStorage.ts` 第 1408 行）负责写入。它的行为：[已验证]

1. 清洗消息（`cleanMessagesForLogging`），去掉不需要持久化的内容。
2. 读取已有消息的集合，只追加新消息，避免重复。
3. 维护父子关系（`parentUuid`），让消息形成链，支持分支和回溯。

`QueryEngine` 在多个位置调用它（如约第 451、609、712、728、780、834 行），既有同步写入也有 `void` 的异步写入。[已验证]

**设计理由**：转录是唯一的真相来源。恢复、回溯（`/rewind`）、导出、遥测都依赖它。用追加式 JSONL 而不是整体覆盖，是为了在崩溃时尽量少丢数据，也便于增量写入。[推断]

### 恢复与回溯

- `/resume`（`commands/resume/`）：从转录恢复历史会话。
- `/rewind`（`commands/rewind/`）：回到之前的某个点。
- `/session`（`commands/session/`）：会话管理。[已验证命令目录存在]

`isChainParticipant`、`isTranscriptMessage` 等函数区分哪些条目参与对话链，哪些只是进度或元数据，是恢复逻辑的基础。[已验证]

### 会话持久化开关

`QueryEngine.submitMessage` 开头调用 `isSessionPersistenceDisabled()`，决定本次会话是否写入转录。无头模式等场景可以关闭持久化。[已验证]

## 二、会话记忆（session memory）

`services/SessionMemory/sessionMemory.ts` 的注释说明：它在后台定期运行，用一个**分叉子代理**从当前对话中提取关键信息，写入一份 markdown 笔记，而不打断主对话。[已验证]

**用途**：当会话很长需要压缩时，会话记忆可以作为压缩的依据（见第 07 章的 `sessionMemoryCompact.ts`），比临时生成摘要更稳定。[推断]

## 三、自动记忆（auto memory）

### 位置与开关

自动记忆保存在 `~/.claude/projects/<项目路径>/memory/` 下，由 `memdir/` 管理。[已验证，见 `extractMemories.ts` 头部注释]

开关：环境变量 `CLAUDE_CODE_DISABLE_AUTO_MEMORY` 可以关闭。[已验证，`memdir/paths.ts` 第 30–36 行]

### 记忆类型

`memdir/memoryTypes.ts` 定义了四种记忆类型：`user`、`feedback`、`project`、`reference`。[已验证]

源码注释明确列出了**不应该保存为记忆的内容**：代码模式、架构、文件路径、git 历史、调试方案等。这些应该从代码和 git 中直接推导，而不是存进记忆。[已验证，`memoryTypes.ts` 头部注释]

**设计理由**：记忆如果收录了可以从代码推导的东西，会随代码变化而过时，反而误导。限制记忆范围，是保持它可信的前提。[推断]

### 提取时机

`services/extractMemories/extractMemories.ts` 的注释说明：它在每轮查询循环**结束**时（模型给出无工具调用的最终回答后）运行，通过 `handleStopHooks` 触发，使用分叉代理共享父代理的提示缓存。[已验证]

它只在主线程运行（子代理不触发），非交互模式下也会触发，但 `--bare` 模式会跳过。[已验证，见第 06 章]

### 相关预取

`startRelevantMemoryPrefetch`（`query.ts` 循环开始前调用）在每个用户回合开始时预取相关记忆，结果在工具执行期间异步消费，尽量不增加首字延迟。[已验证]

`memdir/findRelevantMemories.ts` 负责查找相关记忆。[已验证文件存在]

## 四、三类数据的对比

| 数据 | 范围 | 写入者 | 生命周期 | 用途 |
|---|---|---|---|---|
| 转录 | 单个会话 | 主循环（每回合） | 长期保留，可恢复 | 恢复、回溯、遥测 |
| 会话记忆 | 单个会话 | 后台分叉代理（定期） | 会话内 | 压缩依据、续接上下文 |
| 自动记忆 | 跨会话、按项目 | 停止阶段的提取代理 | 长期 | 跨会话的稳定知识 |

## 五、团队记忆

`memdir/teamMemPaths.ts`、`services/teamMemorySync/` 处理团队共享的记忆，构建期开关 `TEAMMEM` 控制。[已验证文件存在，具体同步协议待确认]

## 设计上的取舍

- **转录追加 vs 覆盖**：追加安全，但文件会增长，需要读取上限和恢复时的过滤。[推断]
- **自动提取 vs 手动记忆**：自动减少用户负担，但可能收录不准确的内容。用记忆类型和排除规则约束它。[推断]
- **记忆的范围**：按项目隔离，避免不同项目的知识相互污染。[推断]
