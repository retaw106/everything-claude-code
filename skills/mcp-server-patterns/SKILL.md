---
name: mcp-server-patterns
description: 使用 Node/TypeScript SDK 构建 MCP 服务器 — 工具、资源、提示、Zod 验证、stdio 与 Streamable HTTP。使用 Context7 或官方 MCP 文档获取最新 API。
origin: ECC
---

# MCP Server 模式

Model Context Protocol (MCP) 让 AI 助手可以从你的服务器调用工具、读取资源和使用提示。在构建或维护 MCP 服务器时使用此技能。SDK API 会演进；检查 Context7（查询 "MCP"）或官方 MCP 文档以获取当前方法名称和签名。

## 何时使用

使用场景：实现新的 MCP 服务器、添加工具或资源、选择 stdio 还是 HTTP、升级 SDK，或调试 MCP 注册和传输问题。

## 工作原理

### 核心概念

- **工具 (Tools)**：模型可以调用的操作（例如搜索、运行命令）。根据 SDK 版本使用 `registerTool()` 或 `tool()` 注册。
- **资源 (Resources)**：模型可以获取的只读数据（例如文件内容、API 响应）。使用 `registerResource()` 或 `resource()` 注册。处理程序通常接收 `uri` 参数。
- **提示 (Prompts)**：客户端可以展示的可重用、参数化提示模板（例如在 Claude Desktop 中）。使用 `registerPrompt()` 或等效方法注册。
- **传输 (Transport)**：stdio 用于本地客户端（例如 Claude Desktop）；Streamable HTTP 是远程的首选（Cursor、云）。传统 HTTP/SSE 用于向后兼容。

Node/TypeScript SDK 可能暴露 `tool()` / `resource()` 或 `registerTool()` / `registerResource()`；官方 SDK 随时间有变化。始终根据当前 [MCP 文档](https://modelcontextprotocol.io) 或 Context7 进行验证。

### 使用 stdio 连接

对于本地客户端，创建 stdio 传输并将其传递给服务器的连接方法。确切的 API 因 SDK 版本而异（例如构造函数与工厂）。查看官方 MCP 文档或查询 Context7 的 "MCP stdio server" 以获取当前模式。

保持服务器逻辑（工具 + 资源）独立于传输，以便你可以在入口点插入 stdio 或 HTTP。

### 远程 (Streamable HTTP)

对于 Cursor、云或其他远程客户端，使用 **Streamable HTTP**（当前规范中每个 MCP 单一 HTTP 端点）。仅在需要向后兼容性时支持传统 HTTP/SSE。

## 示例

### 安装和服务器设置

```bash
npm install @modelcontextprotocol/sdk zod
```

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

const server = new McpServer({ name: "my-server", version: "1.0.0" });
```

使用你的 SDK 版本提供的 API 注册工具和资源：某些版本使用 `server.tool(name, description, schema, handler)`（位置参数），其他使用 `server.tool({ name, description, inputSchema }, handler)` 或 `registerTool()`。资源同理 — 当 API 提供时在处理程序中包含 `uri`。查看官方 MCP 文档或 Context7 获取当前 `@modelcontextprotocol/sdk` 签名，以避免复制粘贴错误。

使用 **Zod**（或 SDK 首选的模式格式）进行输入验证。

## 最佳实践

- **模式优先**：为每个工具定义输入模式；记录参数和返回格式。
- **错误处理**：返回模型可以解释的结构化错误或消息；避免原始堆栈跟踪。
- **幂等性**：尽可能优先使用幂等工具，以便重试是安全的。
- **速率和成本**：对于调用外部 API 的工具，考虑速率限制和成本；在工具描述中记录。
- **版本控制**：在 package.json 中锁定 SDK 版本；升级时查看发行说明。

## 官方 SDK 和文档

- **JavaScript/TypeScript**：`@modelcontextprotocol/sdk` (npm)。使用 Context7 查询库名 "MCP" 获取当前注册和传输模式。
- **Go**：GitHub 上的官方 Go SDK (`modelcontextprotocol/go-sdk`)。
- **C#**：.NET 的官方 C# SDK。
