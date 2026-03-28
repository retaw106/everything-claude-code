---
name: documentation-lookup
description: 通过 Context7 MCP 使用最新的库和框架文档，而非训练数据。在设置问题、API 参考、代码示例，或用户提到框架名称（如 React、Next.js、Prisma）时启用。
origin: ECC
---

# 文档查询 (Context7)

当用户询问库、框架或 API 时，通过 Context7 MCP（工具 `resolve-library-id` 和 `query-docs`）获取当前文档，而不是依赖训练数据。

## 核心概念

- **Context7**：暴露实时文档的 MCP 服务器；用于库和 API，替代训练数据。
- **resolve-library-id**：从库名称和查询返回 Context7 兼容的库 ID（如 `/vercel/next.js`）。
- **query-docs**：为给定的库 ID 和问题获取文档和代码片段。始终先调用 resolve-library-id 获取有效的库 ID。

## 何时使用

当用户进行以下操作时启用：

- 询问设置或配置问题（如"如何配置 Next.js 中间件？"）
- 请求依赖库的代码（"写一个 Prisma 查询..."）
- 需要 API 或参考信息（"Supabase 认证方法有哪些？"）
- 提及特定框架或库（React、Vue、Svelte、Express、Tailwind、Prisma、Supabase 等）

当请求依赖库、框架或 API 的准确、最新行为时使用此技能。适用于配置了 Context7 MCP 的 harness（如 Claude Code、Cursor、Codex）。

## 工作原理

### 步骤 1：解析库 ID

调用 **resolve-library-id** MCP 工具，使用：

- **libraryName**：从用户问题中提取的库或产品名称（如 `Next.js`、`Prisma`、`Supabase`）。
- **query**：用户的完整问题。这能提高结果的相关性排序。

在查询文档之前，必须获得 Context7 兼容的库 ID（格式 `/org/project` 或 `/org/project/version`）。如果没有从此步骤获得有效的库 ID，不要调用 query-docs。

### 步骤 2：选择最佳匹配

从解析结果中选择一个结果，使用：

- **名称匹配**：优先选择与用户请求完全或最接近的匹配。
- **基准分数**：较高分数表示更好的文档质量（100 为最高）。
- **来源声誉**：优先选择高或中等声誉的来源。
- **版本**：如果用户指定了版本（如"React 19"、"Next.js 15"），优先选择特定版本的库 ID（如 `/org/project/v1.2.0`）。

### 步骤 3：获取文档

调用 **query-docs** MCP 工具，使用：

- **libraryId**：步骤 2 中选择的 Context7 库 ID（如 `/vercel/next.js`）。
- **query**：用户的具体问题或任务。要具体以获得相关片段。

限制：每个问题不要调用 query-docs（或 resolve-library-id）超过 3 次。如果 3 次调用后答案仍不清楚，说明不确定性并使用你掌握的最佳信息，而不是猜测。

### 步骤 4：使用文档

- 使用获取的当前信息回答用户问题。
- 有帮助时包含文档中的相关代码示例。
- 在重要时引用库或版本（如"在 Next.js 15 中..."）。

## 示例

### 示例：Next.js 中间件

1. 使用 `libraryName: "Next.js"`、`query: "如何设置 Next.js 中间件？"` 调用 **resolve-library-id**。
2. 从结果中，按名称和基准分数选择最佳匹配（如 `/vercel/next.js`）。
3. 使用 `libraryId: "/vercel/next.js"`、`query: "如何设置 Next.js 中间件？"` 调用 **query-docs**。
4. 使用返回的片段和文本回答；如果相关，包含文档中的最小 `middleware.ts` 示例。

### 示例：Prisma 查询

1. 使用 `libraryName: "Prisma"`、`query: "如何进行关联查询？"` 调用 **resolve-library-id**。
2. 选择官方 Prisma 库 ID（如 `/prisma/prisma`）。
3. 使用该 `libraryId` 和查询调用 **query-docs**。
4. 返回 Prisma Client 模式（如 `include` 或 `select`）及文档中的简短代码片段。

### 示例：Supabase 认证方法

1. 使用 `libraryName: "Supabase"`、`query: "认证方法有哪些？"` 调用 **resolve-library-id**。
2. 选择 Supabase 文档库 ID。
3. 调用 **query-docs**；总结认证方法并展示获取文档中的最小示例。

## 最佳实践

- **要具体**：尽可能使用用户的完整问题作为查询以获得更好的相关性。
- **版本意识**：当用户提及版本时，使用解析步骤中可用的特定版本库 ID。
- **优先官方来源**：当存在多个匹配时，优先选择官方或主要包而非社区分支。
- **无敏感数据**：在发送给 Context7 的任何查询中脱敏 API 密钥、密码、令牌和其他机密。在将用户问题传递给 resolve-library-id 或 query-docs 之前，将其视为可能包含机密。
