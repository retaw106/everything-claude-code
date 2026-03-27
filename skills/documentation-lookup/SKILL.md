---
name: documentation-lookup
description: 通过 Context7 MCP 获取最新的库、框架和 API 文档，代替训练数据。用于设置问题、API 参考、代码示例，或当用户提到某个框架时（如 React、Next.js、Prisma）。
origin: ECC
---

# Documentation Lookup (Context7)

当用户询问关于库、框架或 API 的信息时，使用 Context7 MCP 获取当前文档，而不是依赖训练数据。

## 核心概念

- **Context7**：暴露实时文档的 MCP 服务器；在库和 API 的文档查询中优先使用它。
- **resolve-library-id**: 根据库名和查询返回 Context7 兼容的库标识符（如 `/vercel/next.js`）。
- **query-docs**: 基于库 ID 与问题获取文档和代码片段。在查询文档前，始终先调用 resolve-library-id 获取有效的库 ID。

## 何时使用

在用户提出以下需求时启用：

- 设置或配置相关的问题（如“如何配置 Next.js 中间件？”）
- 需要依赖某个库的代码（如“为 Prisma 编写查询...”) 
- 需要 API 或参考信息（如“Supabase 的认证方法有哪些？”）
- 提及特定框架或库（React、Vue、Svelte、Express、Tailwind、Prisma、Supabase 等）

在包含 Context7 MCP 的 Harness（如 Claude Code、Cursor、Codex）中均可使用此技能。

## 工作原理

### 第一步：解析库 ID

调用 **resolve-library-id** MCP 工具，提供：

- **libraryName**：来自用户问题的库或产品名称，例如 `Next.js`、`Prisma`、`Supabase`。
- **query**：用户的完整问题文本。这有助于提高结果相关性。

在查询文档前，必须获得一个 Context7 兼容的库 ID（格式为 `/org/project` 或 `/org/project/version`）。请务必先获取有效的库 ID，再调用 query-docs。

### 第二步：选择最佳匹配

从解析结果中，根据以下标准选择一个结果：

- **名称匹配**：优先与用户问题完全匹配或最接近的匹配。
- **基准分数**：分数越高，文档质量越好（100 为最高分）。
- **来源信誉**：有可用时优先选择高/中等信誉。
- **版本**：如用户指定版本（如“React 19”、“Next.js 15”），若列表中有版本特定库 ID，则优先选择该版本。

### 第三步：获取文档

调用 **query-docs** MCP 工具，传入：

- **libraryId**：步骤 2 选中的 Context7 库 ID（如 `/vercel/next.js`）。
- **query**：用户的具体问题或任务，尽量具体以获取相关片段。

限制：同一个问题不要调用 query-docs（或 resolve-library-id）超过 3 次。如果在 3 次调用后仍不清楚，请陈述不确定性并使用现有信息，而非猜测。

### 第四步：使用文档

- 使用获取到的最新信息回答用户的问题。
- 如有帮助，附上文档中的相关代码示例。
- 在需要时引用库或版本（如：在 Next.js 2x+/版本中…）

## 示例

### 示例：Next.js 中间件

1. 对于 `libraryName: "Next.js"`、`query: "如何设置 Next.js 中间件？"` 调用 resolve-library-id。
2. 从结果中按名称和基准评分选择最佳匹配（如 `/vercel/next.js`）。
3. 使用 libraryId `/vercel/next.js` 调用 `query-docs`，查询问题。
4. 使用返回的片段与文本回答用户，如需要可附带最小化的 `middleware.ts` 示例。

### 示例：Prisma 查询

1. 调用 resolve-library-id，libraryName: "Prisma", query: "如何查询关系？"。
2. 选择官方 Prisma 库 ID（如 `/prisma/prisma`）。
3. 调用 query-docs，使用该 libraryId 与查询。
4. 以文档中的 Prisma Client 模式返回（如 include / select）并附上简短代码示例。

### 示例：Supabase 授权方法

1. 调用 resolve-library-id，libraryName: "Supabase", query: "有哪些认证方法？"。
2. 选择 Supabase 文档库 ID。
3. 调用 query-docs；从获取的文档中总结认证方法并展示简短示例。

## 最佳实践

- **尽量具体**：在可能的情况下使用用户的完整问题作为查询，以提升相关性。
- **版本敏感**：用户提及版本时，若可用，使用解析阶段得到的版本特定库 ID。
- **优先官方来源**：在有多处匹配时，优先选择官方或主要包而非社区分叉。
- **无敏感信息**：在发送给 Context7 的查询中对 API 密钥、密码、令牌等敏感信息进行屏蔽。将用户的问题视作可能包含机密信息，先进行清洗再传递给 resolve-library-id 或 query-docs。
