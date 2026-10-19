# 15 模型 API 层

## 位置与职责

模型调用集中在 `services/api/`。上层（`query.ts`）只通过依赖注入的 `deps.callModel` 调用，不直接接触 HTTP 细节。这样测试可以替换模型调用，而不必 mock 整个网络层。[已验证，`query/deps.ts`]

## 两个入口

| 函数 | 位置 | 用途 |
|---|---|---|
| `queryModelWithStreaming` | `services/api/claude.ts` 第 752 行 | 主循环使用，流式返回 |
| `queryModelWithoutStreaming` | 同文件第 709 行 | 非流式，用于不需要逐字输出的场景 |

另有 `queryHaiku`（第 3241 行）与 `queryWithModel`（第 3300 行），用于调用较小模型做辅助任务，例如生成标题或摘要。[已验证]

## 请求组装

`claude.ts` 中与请求组装相关的函数：[已验证]

- `getExtraBodyParams`（第 272 行）：额外的请求体参数，按 beta 头决定。
- `getPromptCachingEnabled`（第 333 行）：按模型判断是否启用提示缓存。
- `getCacheControl`（第 358 行）：生成缓存断点标记。
- `addCacheBreakpoints`（第 3063 行）与 `buildSystemPromptBlocks`（第 3213 行）：把系统提示拆成带缓存标记的块。
- `configureTaskBudgetParams`（第 479 行）：任务级别的预算参数。
- `userMessageToMessageParam` / `assistantMessageToMessageParam`（第 588、633 行）：内部消息格式到 API 格式的转换。

**设计要点**：系统提示被拆成多个块，每块可以有独立的缓存标记。稳定的部分（如基础指令）放在前面，变化的部分（如当前目录、日期）放在后面，这样变化不会使前面的缓存失效。这是成本设计的关键。[推断，基于 `buildSystemPromptBlocks` 与缓存断点的存在；具体切分规则待确认]

## 重试与降级

`services/api/withRetry.ts` 是重试层，核心是 `withRetry`（第 170 行）。[已验证]

- 基础退避 `BASE_DELAY_MS = 500`（第 55 行）。
- 对 529（过载）错误最多重试 `MAX_529_RETRIES = 3` 次（第 54 行）。
- **只有前台查询才重试 529**。源码注释说明：摘要、标题、建议、分类器等后台请求直接放弃，因为在容量紧张时，每次重试都会放大网关压力，而用户看不到这些失败。`FOREGROUND_529_RETRY_SOURCES` 这个集合决定了哪些来源允许重试。[已验证，第 58–87 行]
- 可选的无人值守重试（`CLAUDE_CODE_UNATTENDED_RETRY`，仅内部版本）：对 429 和 529 持续重试。[已验证，第 91 行附近]
- 模型降级：配置了 `fallbackModel` 时，触发 `FallbackTriggeredError`（第 160 行），由上层切换模型。[已验证]

**设计理由**：重试策略不是越多越好。对后台任务盲目重试会加剧拥塞，这是一个针对系统整体的考量，而不仅是单个请求的可靠性。[推断，但注释本身即是这一理由]

## 错误分类

`services/api/errors.ts` 与 `errorUtils.ts` 把 API 错误分类，供上层决定是重试、压缩还是终止。例如 `is529Error`（`withRetry.ts` 第 610 行）。上层 `query.ts` 对"上下文超长"错误会走反应式压缩，见第 07 章。[已验证]

## 用量与计费

- `usage.ts`、`accumulateUsage`（第 2993 行）、`updateUsage`（第 2924 行）：累计 token 用量。
- `emptyUsage.ts`：用量的零值。
- 额外用量、限流相关逻辑见 `utils/extraUsage.ts`、`services/claudeAiLimits.ts`。[已验证文件存在，细节未读]

## 其他服务端交互

`services/api/` 下还有：`bootstrap.ts`（启动时获取配置）、`filesApi.ts`（文件 API）、`grove.ts`、`referral.ts`、`ultrareviewQuota.ts`、`sessionIngress.ts` 等。这些属于产品的服务端功能，本文未逐个阅读。[已验证文件存在]

## 认证

`verifyApiKey`（`claude.ts` 第 530 行）用于校验密钥。OAuth 相关逻辑在 `services/oauth/` 与 `utils/auth.ts`。[已验证]

## 设计上的取舍

- **流式与非流式并存**：流式体验好，但解析复杂。非流式用于特定场景，减少这些场景的复杂度。[推断]
- **降级与重试分开**：重试针对瞬时错误，降级针对模型不可用。两者的触发条件和代价不同，分开处理更清楚。[推断]
