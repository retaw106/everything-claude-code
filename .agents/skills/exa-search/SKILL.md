---
name: exa-search
description: 通过 Exa MCP 进行神经搜索，用于网络、代码和公司研究。当用户需要网络搜索、代码示例、公司情报、人员查找，或使用 Exa 神经搜索引擎进行 AI 驱动的深度研究时使用。
origin: ECC
---

# Exa 搜索

通过 Exa MCP 服务器进行网络内容、代码、公司和人员的神经搜索。

## 何时启用

- 用户需要当前的网络信息或新闻
- 搜索代码示例、API 文档或技术参考
- 研究公司、竞争对手或市场参与者
- 查找专业资料或领域内的人员
- 为任何开发任务进行背景研究
- 用户说"搜索"、"查找"、"找到"或"最新的情况是"

## MCP 要求

必须配置 Exa MCP 服务器。添加到 `~/.claude.json`：

```json
"exa-web-search": {
  "command": "npx",
  "args": ["-y", "exa-mcp-server"],
  "env": { "EXA_API_KEY": "YOUR_EXA_API_KEY_HERE" }
}
```

在 [exa.ai](https://exa.ai) 获取 API 密钥。

## 核心工具

### web_search_exa
通用网络搜索，用于当前信息、新闻或事实。

```
web_search_exa(query: "2026年最新AI发展", numResults: 5)
```

**参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `query` | string | 必需 | 搜索查询 |
| `numResults` | number | 8 | 结果数量 |

### web_search_advanced_exa
带域名和日期约束的过滤搜索。

```
web_search_advanced_exa(
  query: "React Server Components 最佳实践",
  numResults: 5,
  includeDomains: ["github.com", "react.dev"],
  startPublishedDate: "2025-01-01"
)
```

**参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `query` | string | 必需 | 搜索查询 |
| `numResults` | number | 8 | 结果数量 |
| `includeDomains` | string[] | 无 | 限制到特定域名 |
| `excludeDomains` | string[] | 无 | 排除特定域名 |
| `startPublishedDate` | string | 无 | ISO 日期过滤（开始） |
| `endPublishedDate` | string | 无 | ISO 日期过滤（结束） |

### get_code_context_exa
从 GitHub、Stack Overflow 和文档站点查找代码示例和文档。

```
get_code_context_exa(query: "Python asyncio 模式", tokensNum: 3000)
```

**参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `query` | string | 必需 | 代码或 API 搜索查询 |
| `tokensNum` | number | 5000 | 内容 token 数 (1000-50000) |

### company_research_exa
为公司情报和新闻研究公司。

```
company_research_exa(companyName: "Anthropic", numResults: 5)
```

**参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `query` | string | 必需 | 公司名称 |
| `numResults` | number | 5 | 结果数量 |

### people_search_exa
查找专业资料和简介。

```
people_search_exa(query: "Anthropic 的 AI 安全研究员", numResults: 5)
```

### crawling_exa
从 URL 提取完整页面内容。

```
crawling_exa(url: "https://example.com/article", tokensNum: 5000)
```

**参数：**

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `url` | string | 必需 | 要提取的 URL |
| `tokensNum` | number | 5000 | 内容 token 数 |

### deep_researcher_start / deep_researcher_check
启动异步运行的 AI 研究代理。

```
# 启动研究
deep_researcher_start(query: "2026年AI代码编辑器的全面分析")

# 检查状态（完成时返回结果）
deep_researcher_check(researchId: "<start返回的id>")
```

## 使用模式

### 快速查询
```
web_search_exa(query: "Node.js 22 新功能", numResults: 3)
```

### 代码研究
```
get_code_context_exa(query: "Rust 错误处理模式 Result 类型", tokensNum: 3000)
```

### 公司尽职调查
```
company_research_exa(companyName: "Vercel", numResults: 5)
web_search_advanced_exa(query: "Vercel 融资估值 2026", numResults: 3)
```

### 技术深入分析
```
# 启动异步研究
deep_researcher_start(query: "WebAssembly 组件模型状态和采用情况")
# ... 做其他工作 ...
deep_researcher_check(researchId: "<id>")
```

## 提示

- 广泛查询使用 `web_search_exa`，过滤结果使用 `web_search_advanced_exa`
- 较低的 `tokensNum` (1000-2000) 用于聚焦的代码片段，较高 (5000+) 用于全面上下文
- 结合 `company_research_exa` 和 `web_search_advanced_exa` 进行彻底的公司分析
- 使用 `crawling_exa` 从搜索结果中找到的特定 URL 获取完整内容
- `deep_researcher_start` 最适合需要 AI 综合的全面主题

## 相关技能

- `deep-research` — 使用 firecrawl + exa 的完整研究工作流
- `market-research` — 带决策框架的商业导向研究
