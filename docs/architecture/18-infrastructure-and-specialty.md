# 18 基础设施与特色模块

这一章覆盖遥测、配置远程下发、插件治理等基础设施，以及一些特色模块。大部分只做到结构和职责的层面。

## 一、遥测与分析（services/analytics/）

### 设计原则

`services/analytics/index.ts` 文件头部注释明确写出设计约束：[已验证]

- **模块无依赖**：为了避免循环引用，本模块不依赖其他业务模块。
- **事件先排队**：在 `attachAnalyticsSink()` 被调用前，事件进入队列；启动完成后，排队事件统一派发。
- **路由由 sink 负责**：实际发送到哪里（Datadog、第一方事件日志）由 sink 决定。

**设计理由**：在启动早期就可能产生事件，而那时发送通道还没准备好。排队加延迟派发，既不丢事件，也不引入循环依赖。[已验证，由注释与 `setup.ts` 的 `initSinks()` 调用共同支持]

### 相关文件

- `datadog.ts`、`firstPartyEventLogger.ts`、`firstPartyEventLoggingExporter.ts`：两条发送路径。[已验证文件存在]
- `sinkKillswitch.ts`：紧急关闭开关，可以停止发送。[已验证文件存在]
- `growthbook.ts`：功能开关与灰度实验。`getFeatureValue_CACHED_MAY_BE_STALE` 是源码中大量使用的接口，名字中的 "CACHED_MAY_BE_STALE" 直接说明它可能返回过期值。[已验证]
- `metadata.ts`：附加到事件上的元数据。
- `config.ts`：配置。

**隐私约束**：事件类型名中有 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 这样的类型标记，要求开发者显式确认某个字段不包含代码或文件路径。这是一种类型层面的隐私防护。[已验证，多处使用]

**设计理由**：这种标记把隐私审查变成编译期的显式动作，比事后审计更可靠。[推断]

### 功能开关的两套机制

源码里有两种开关，设计上要区分：[已验证]

1. **构建期**：`feature('XXX')`（来自 `bun:bundle`），约 196 个文件使用。
2. **运行期**：`getFeatureValue_*`（来自 GrowthBook）或 `checkStatsigFeatureGate_*`，用于灰度。

## 二、远程托管配置（services/remoteManagedSettings/、services/policyLimits/）

组织可以通过远程下发托管设置，这些设置位于设置来源的最高层 `policySettings`（见第 11 章）。`policyLimits/` 处理策略限制。[已验证目录存在，具体协议未读]

`services/settingsSync/` 负责设置的同步。[已验证目录存在]

## 三、插件治理

插件的分发和治理在第 08 章已说明。补充两点：

- `pluginBlocklist.ts`：黑名单，可以禁止某些插件。
- `managedPlugins.ts`：组织托管的插件。[已验证文件存在]

## 四、费用与限流

- `cost-tracker.ts`、`costHook.ts`：费用追踪。[已验证文件存在]
- `services/claudeAiLimits.ts`、`rateLimitMessages.ts`：限流状态与提示。[已验证文件存在]
- `commands/extra-usage`、`commands/rate-limit-options`、`commands/usage`：用户可见的用量与限额命令。[已验证目录存在]

## 五、诊断与自检

- `commands/doctor/`、`screens/Doctor.tsx`：诊断页面，检查环境与配置。[已验证]
- `utils/doctorDiagnostic.ts`、`utils/doctorContextWarnings.ts`：诊断逻辑。[已验证文件存在]
- `utils/preflightChecks.tsx`：启动前的检查。[已验证文件存在]

## 六、自动更新

`utils/autoUpdater.ts`、`components/AutoUpdater.tsx` 处理自动更新。`utils/nativeInstaller/` 处理原生安装。[已验证文件存在，更新策略未读]

## 七、会话的其他能力

- `utils/sessionTitle.ts`：会话标题生成。
- `services/awaySummary.ts`：用户离开后回来时的摘要。[已验证文件存在]
- `services/tips/`：使用提示。
- `services/PromptSuggestion/`：提示建议（见第 06 章）。[已验证目录存在]

## 八、特色模块

以下模块在快照中存在，但本文只说明结构，不解释功能细节。

| 模块 | 观察到的内容 | 深度 |
|---|---|---|
| `buddy/` | 一个伴随式的字符组件：有物种、帽子、眼睛等属性，由用户 ID 通过伪随机方式确定（`rollWithSeed`），并通过系统提示让模型知道它的存在，但要求模型"不要替它说话" | 结构已读，属于界面与提示的组合 |
| `voice/`、`services/voice*.ts` | 语音输入，依赖 OAuth 登录状态与灰度开关 | 仅见入口 |
| `assistant/` | 历史会话列表的 API 访问，分页大小 100 | 仅见入口 |
| `moreright/` | 外部版本中的桩文件，注释说明真实实现仅内部可用 | 已确认是桩 |
| `upstreamproxy/` | 运行在云端容器内的代理配置，涉及会话令牌与证书 | 仅见入口 |
| `native-ts/` | 布局引擎的纯 TypeScript 实现（见第 16 章） | 已读头部 |
| `context/`、`state/`、`bootstrap/` | 全局状态与启动引导 | 仅见目录 |
| `migrations/` | 配置版本迁移 | 仅见目录 |
| `schemas/` | 数据模式定义 | 仅见目录 |
| `outputStyles/` | 输出样式 | 仅见目录 |
| `skills/bundled/` | 内置技能 | 仅见目录 |

**关于 `buddy/` 的设计观察**：它通过系统提示告诉模型"有一个独立的观察者存在，不要替它回答"。这是一个用提示词协调界面元素与模型行为的例子。[已验证，`buddy/prompt.ts`]

**关于 `moreright/` 的观察**：外部版本使用桩文件替代内部实现。这是一种常见的"同一代码库，多个发行版本"的做法。[已验证，文件头注释]

## 设计上的取舍

- **遥测的隐私防护 vs 开发效率**：类型标记增加了每次记录事件的心智负担，换来隐私风险的前置发现。[推断]
- **外部桩 vs 完整代码**：外部版本用桩替代内部模块，保护内部实现，但要求桩与真实接口保持一致。[推断]
