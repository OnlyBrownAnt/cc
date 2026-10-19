# 05 权限系统

## 目标

模型能执行 shell、改文件、访问网络，权限系统是这些能力的安全边界。设计上的核心问题是：**如何在"每次都问，烦死用户"和"全部放行，不安全"之间找到平衡，并且让规则可配置、可解释。**

## 三个层次

权限分成三层，各自在 `utils/permissions/` 下有对应文件：

1. **模式（mode）**：一个全局的"宽松程度"档位
2. **规则（rule）**：针对具体工具和内容的允许、拒绝、询问规则
3. **工具自检（checkPermissions）**：工具根据具体输入给出的判断

## 一、权限模式

`types/permissions.ts` 定义了模式（第 13–36 行）。[已验证]

| 模式 | 含义（推断） |
|---|---|
| `default` | 默认：需要询问的操作弹出确认 |
| `acceptEdits` | 自动接受文件编辑，其他仍询问 |
| `plan` | 规划模式：只允许读和规划，不执行修改 |
| `bypassPermissions` | 跳过大部分询问，但不跳过安全检查（见下文） |
| `dontAsk` | 不询问，需要询问的操作直接拒绝 |
| `auto` | 由分类器自动判断（构建期开关 `TRANSCRIPT_CLASSIFIER` 控制） |

另有内部模式 `bubble`，用于子代理把权限请求冒泡给父代理。[已验证]

模式还有三个配套概念：

- `isBypassPermissionsModeAvailable`：记录用户是否从绕过模式开始，用于从 plan 模式返回时正确恢复。[已验证]
- `prePlanMode`：进入规划模式前的模式，退出时恢复。[已验证]
- `dontAsk` 的转换：在 `permissions.ts` 约第 503–513 行，把"询问"转换成"拒绝"。[已验证]

## 二、规则

规则由 `permissionRuleParser.ts` 解析，类型是 allow / deny / ask 三种行为，来源有多个（用户设置、项目设置、托管设置、命令行、会话内授予等）。`permissions.ts` 中的 `getAllowRules`、`getDenyRules`、`getAskRules` 分别按行为取规则。[已验证]

规则可以精确到工具内容。例如对 Bash 工具，规则可以匹配某个命令前缀，而不是整个工具。`shellRuleMatching.ts` 负责这类匹配。[已验证]

## 三、决策链（核心）

`hasPermissionsToUseToolInner()`（`permissions.ts` 约第 1158 行开始）是决策的主体。按源码顺序，判断依次进行：[已验证]

1. **中断检查**：已中止则直接返回。
2. **1a. 整个工具被 deny 规则禁止** → 拒绝
3. **1b. 整个工具被 ask 规则要求询问** → 询问（沙箱自动放行的情况例外）
4. **1c. 工具自检** `tool.checkPermissions`，结果可能是 allow、deny、ask 或 passthrough
5. **1d. 工具自检明确拒绝** → 拒绝
6. **1e. 工具要求用户交互** → 询问，即使在绕过模式下
7. **1f. 内容级 ask 规则** → 询问，即使在绕过模式下
8. **1g. 安全检查** → 询问，**绕过模式也不能跳过**
9. **2a. 模式放行**：`bypassPermissions` 或符合条件的计划模式 → 允许
10. **2b. 整个工具被 allow 规则允许** → 允许
11. **3. passthrough 转为 ask** → 询问

**设计要点（第 1g 步）**：`.git/`、`.claude/`、`.vscode/`、shell 配置文件等路径的修改，即使在 `bypassPermissions` 模式下也要询问。源码注释直接说明这些路径是"bypass-immune"。这是一个重要的安全底线：宽松模式只放宽常规操作，不放宽对关键配置的修改。[已验证]

**另一个要点（1d、1f）**：拒绝规则和内容级询问规则的优先级高于绕过模式。用户明确配置的规则不会被模式覆盖。[已验证]

## 四、询问与用户交互

当决策为 ask 时，交互层弹出确认。用户的选择可以是一次性允许、长期允许（写入规则）或拒绝。这些选择会生成 `PermissionUpdate`，持久化到对应来源。[推断，基于 `PermissionUpdate.ts` 的存在和类型名]

在无人值守场景（SDK、后台代理），还有 `runPermissionRequestHooksForHeadlessAgent`（`permissions.ts` 约第 400 行），允许通过 PermissionRequest 钩子来决定，而不依赖界面。[已验证]

## 五、拒绝追踪与熔断

`utils/permissions/denialTracking.ts` 定义了拒绝的上限（第 12–15 行）：[已验证]

- 连续拒绝达到 3 次，或累计拒绝达到 20 次，触发限制。
- 触发后调用 `handleDenialLimitExceeded`（约第 984 行）。

**设计理由**：自动分类器或用户反复拒绝时，模型可能陷入"不断尝试被拒绝的操作"的循环，浪费 token 也干扰用户。拒绝计数是一种熔断机制。[推断]

## 六、自动模式（auto）与分类器

`auto` 模式在构建期开关打开时可用。它用分类器（`bashClassifier.ts`、`yoloClassifier.ts`、`classifierDecision.ts`）判断一个操作是否安全，代替人工询问。[已验证文件存在，具体判定逻辑待确认]

`PermissionDenied` 钩子可以在分类器拒绝后被调用，钩子返回 `retry: true` 时，系统告诉模型该操作可以重试。这是在拒绝和重试之间留出的一个受控出口。[已验证，`toolExecution.ts` 约第 1073–1096 行]

## 七、设计上的取舍

- **规则可配置 vs 默认安全**：规则越多越灵活，但规则之间可能冲突。`shadowedRuleDetection.ts` 专门检测被遮蔽的规则，说明团队遇到过这类问题。[推断]
- **自动化 vs 可解释**：分类器省去询问，但用户需要知道为什么某个操作被允许或拒绝。`permissionExplainer.ts` 承担解释的职责。[推断]
- **宽松模式的边界**：绕过模式不是"关掉权限"，而是跳过常规询问，保留安全底线。这是一个被明确设计出来的分层。[已验证]
