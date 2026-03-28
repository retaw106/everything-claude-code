---
description: Look up current documentation for a library or topic via Context7.
---

# /docs

## 用途

通过 Context7 查找库、框架或 API 的最新文档，并返回带有相关代码片段的摘要答案。使用 Context7 MCP（resolve-library-id 和 query-docs），因此答案反映的是当前文档而非训练数据。

## 使用方法

```
/docs [library name] [question]
```

对多词参数使用引号，以便它们被解析为单个标记。示例：`/docs "Next.js" "How do I configure middleware?"`

如果省略库或问题，提示用户提供：
1. 库或产品名称（例如 Next.js、Prisma、Supabase）。
2. 具体问题或任务（例如 "How do I set up middleware?"、"Auth methods"）。

## 工作流程

1. **解析库 ID** — 使用库名称和用户的问题调用 Context7 工具 `resolve-library-id`，以获取 Context7 兼容的库 ID（例如 `/vercel/next.js`）。
2. **查询文档** — 使用该库 ID 和用户的问题调用 `query-docs`。
3. **摘要** — 返回简明的答案，并包含从获取的文档中提取的相关代码示例。如果相关，提及库（和版本）。

## 输出

用户获得一个简明、准确的答案，基于当前文档和有用的代码片段。如果 Context7 不可用，请说明这一点，并从训练数据中回答，同时注明文档可能已过时。
