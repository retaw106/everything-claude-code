---
name: exa-search
description: 通过 Exa MCP 进行神经搜索,用于网页、代码和公司研究。当用户需要网页搜索、代码示例、公司情报、人物查找,或使用 Exa 神经搜索引擎进行 AI 驱动的深度研究时使用。
origin: ECC
---

# Exa 搜索

通过 Exa MCP 服务器对网页内容、代码、公司和人物进行神经搜索。

## 何时激活

- 用户需要当前的网络信息或新闻
- 搜索代码示例、API 文档或技术参考
- 研究公司、竞争对手或市场参与者
- 查找领域内的专业资料或人物
- 为任何开发任务进行背景研究
- 用户说"搜索"、"查找"、"找到"或"最新的...是什么"

## MCP 要求

需要配置 Exa MCP 服务器。添加到 `~/.claude.json`:

```json
"exa-web-search": {
  "command": "npx",
  "args": ["-y", "exa-mcp-server"],
  "env": { "EXA_API_KEY": "YOUR_EXA_API_KEY_HERE" }
}
```

在 [exa.ai](https://exa.ai) 获取 API 密钥。
此仓库当前的 Exa 设置暴露的工具在此记录: `web_search_exa` 和 `get_code_context_exa`。
如果你的 Exa 服务器暴露了其他工具,请在使用前在文档或提示中验证其确切名称。

## 核心工具

### web_search_exa
用于当前信息、新闻或事实的通用网页搜索。

```
web_search_exa(query: "2026年最新的AI发展", numResults: 5)
```

**参数:**

| 参数 | 类型 | 默认值 | 说明 |
|-------|------|---------|-------|
| `query` | string | 必需 | 搜索查询 |
| `numResults` | number | 8 | 结果数量 |
| `type` | string | `auto` | 搜索模式 |
| `livecrawl` | string | `fallback` | 需要时优先实时抓取 |
| `category` | string | none | 可选过滤,如 `company` 或 `research paper` |

### get_code_context_exa
从 GitHub、Stack Overflow 和文档站点查找代码示例和文档。

```
get_code_context_exa(query: "Python asyncio 模式", tokensNum: 3000)
```

**参数:**

| 参数 | 类型 | 默认值 | 说明 |
|-------|------|---------|-------|
| `query` | string | 必需 | 代码或 API 搜索查询 |
| `tokensNum` | number | 5000 | 内容令牌(1000-50000) |

## 使用模式

### 快速查找
```
web_search_exa(query: "Node.js 22 新功能", numResults: 3)
```

### 代码研究
```
get_code_context_exa(query: "Rust 错误处理模式 Result 类型", tokensNum: 3000)
```

### 公司或人物研究
```
web_search_exa(query: "Vercel 2026年融资估值", numResults: 3, category: "company")
web_search_exa(query: "site:linkedin.com/in AI 安全研究人员 Anthropic", numResults: 5)
```

### 技术深度研究
```
web_search_exa(query: "WebAssembly 组件模型状态和采用情况", numResults: 5)
get_code_context_exa(query: "WebAssembly 组件模型示例", tokensNum: 4000)
```

## 技巧

- 使用 `web_search_exa` 获取当前信息、公司查询和广泛探索
- 使用搜索运算符如 `site:`、引号短语和 `intitle:` 来缩小结果范围
- 聚焦代码片段时降低 `tokensNum` (1000-2000),需要全面上下文时提高 (5000+)
- 需要 API 用法或代码示例而非一般网页时使用 `get_code_context_exa`

## 相关技能

- `deep-research` — 使用 firecrawl 和 exa 的完整研究工作流
- `market-research` — 带有决策框架的商业导向研究
