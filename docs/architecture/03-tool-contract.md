# 03 工具契约：为什么 Tool 是一个统一接口

## 问题

系统需要让模型调用几十种能力：读文件、写文件、执行 shell、搜索、抓网页、调用 MCP 服务器、启动子代理……它们的输入格式、副作用、并发安全性、输出大小都不同。如果每种能力各写一套调度和权限逻辑，代码会迅速失控。

## 方案：一个 Tool 类型描述所有能力

`Tool.ts` 定义了泛型类型 `Tool<Input, Output, Progress>`（约第 362 行）。每个工具实现同一组成员。[已验证]

契约可以分成五组：

### 1. 身份与发现

- `name`：模型看到的名称
- `aliases`：改名后的兼容别名
- `searchHint`：3–10 个词的能力短语，供 ToolSearch 做关键词匹配
- `isEnabled()`：运行时是否可用
- `isMcp` / `mcpInfo` / `isLsp`：标记来源，便于区分内置工具与外部工具
- [已验证]

### 2. 输入与输出

- `inputSchema`：Zod 模式，用于校验模型生成的参数。源码注释直言"模型生成的输入并不总是合法"，因此每次调用前都要校验。[已验证]
- `inputJSONSchema`：可选的 JSON Schema 形式，MCP 工具直接提供这种形式，避免从 Zod 转换。[已验证]
- `call(args, context, canUseTool, parentMessage, onProgress)`：真正的执行体，返回 `ToolResult<Output>`。[已验证]
- `description(input, options)`：生成给模型看的工具描述，可以依赖当前权限上下文和可用工具集。[已验证]
- `outputSchema`：可选，描述输出结构。[已验证]

### 3. 行为标签（决定调度方式）

这些标签不是文档性质的注释，而是直接被调度器读取：

| 成员 | 作用 | 被谁使用 |
|---|---|---|
| `isConcurrencySafe(input)` | 能否与其他工具并行 | `toolOrchestration.ts` 分批 |
| `isReadOnly(input)` | 是否只读 | 权限与展示 |
| `isDestructive?(input)` | 是否不可逆（删除、覆盖、发送） | 权限提示 |
| `interruptBehavior()` | 用户新消息到来时，是 `cancel` 还是 `block` | `StreamingToolExecutor` |
| `isSearchOrReadCommand?()` | 是否应折叠成简略展示 | UI |
| `shouldDefer` / `alwaysLoad` | 是否延迟加载 schema | ToolSearch |

[已验证，字段定义见 `Tool.ts`]

注意 `isConcurrencySafe` 是**按输入**判断的，而不是按工具类型。同一个 Bash 工具，`ls` 可以并发，`rm` 不可以。这一点在第 04 章展开。

### 4. 结果控制

- `maxResultSizeChars`：工具结果超过该字节数后，内容落盘，模型只拿到预览和路径。源码注释说明，`Read` 工具必须设为 `Infinity`，否则会出现"读文件 → 结果落盘 → 再读落盘文件"的循环。[已验证]

### 5. 扩展点

- `checkPermissions` 等权限相关方法（见第 05 章）
- `validateInput`：业务校验，早于权限检查执行
- `inputsEquivalent`：判断两次输入是否等价，用于去重

## 设计理由

1. **统一入口，统一管线**：不论内置还是 MCP 工具，都经过同一套校验、权限、钩子、结果处理，减少分支。[推断]
2. **行为由声明驱动**：并发、中断、展示都由工具自己声明，调度器不需要知道每个工具的细节。这是开闭原则在工具层的应用。[推断]
3. **输入不可信**：Zod 校验是第一道门，`validateInput` 是第二道。模型生成的参数被当作外部输入对待。[已验证]
4. **为 MCP 留口子**：`inputJSONSchema` 和 `mcpInfo` 的存在，说明外部工具与内置工具共用同一个接口。[已验证]

## 设计上的取舍

- **接口很宽**：Tool 有二三十个成员，大部分可选。好处是灵活，代价是新人难以一眼看全。源码用大量注释补偿这一点。[推断]
- **Zod 与 JSON Schema 并存**：内置工具用 Zod 便于类型推导，MCP 工具直接给 JSON Schema 避免转换成本。两条路径并存增加了复杂度。[推断]
