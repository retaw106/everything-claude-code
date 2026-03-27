---
name: team-builder
description: 交互式 Agent 选择器，用于组合和分派并行团队
origin: community
---

# Team Builder

按需浏览和组合 Agent 团队的交互式菜单。适用于扁平或领域子目录的 Agent 集合。

## 何时使用

- 你有多个 Agent 角色（markdown 文件），想选择用于任务的 Agent
- 你想从不同领域组合临时团队（例如：安全 + SEO + 架构）
- 你想在决定之前浏览可用的 Agent

## 前提条件

Agent 文件必须是包含角色提示（身份、规则、工作流、交付物）的 markdown 文件。第一个 `# 标题` 用作 Agent 名称，第一个段落用作描述。

支持扁平布局和子目录布局：

**子目录布局** — 从文件夹名称推断领域：

```
agents/
├── engineering/
│   ├── security-engineer.md
│   └── software-architect.md
├── marketing/
│   └── seo-specialist.md
└── sales/
    └── discovery-coach.md
```

**扁平布局** — 从共享的文件名前缀推断领域。当 2+ 个文件共享前缀时，该前缀算作领域。具有唯一前缀的文件归入"通用"。注意：算法在第一个 `-` 处分割，因此多词领域（如 `product-management`）应使用子目录布局：

```
agents/
├── engineering-security-engineer.md
├── engineering-software-architect.md
├── marketing-seo-specialist.md
├── marketing-content-strategist.md
├── sales-discovery-coach.md
└── sales-outbound-strategist.md
```

## 配置

Agent 目录按顺序探测，结果合并：

1. `./agents/**/*.md` + `./agents/*.md` — 项目本地 Agent（两种深度）
2. `~/.claude/agents/**/*.md` + `~/.claude/agents/*.md` — 全局 Agent（两种深度）

所有位置的结果合并并按 Agent 名称去重。项目本地 Agent 优先于同名的全局 Agent。如果用户指定，可以使用自定义路径。

## 工作原理

### 步骤 1：发现可用 Agent

使用上述探测顺序对 Agent 目录进行 Glob。排除 README 文件。对于找到的每个文件：
- **子目录布局**：从父文件夹名称提取领域
- **扁平布局**：收集所有文件名前缀（第一个 `-` 之前的文本）。只有当前缀出现在 2 个或更多文件名中时才成为领域（例如，`engineering-security-engineer.md` 和 `engineering-software-architect.md` 都以 `engineering` 开头 → Engineering 领域）。具有唯一前缀的文件（如 `code-reviewer.md`、`tdd-guide.md`）归入"通用"
- 从第一个 `# 标题` 提取 Agent 名称。如果没有标题，从文件名派生名称（去掉 `.md`，将连字符替换为空格，标题大小写）
- 从标题后的第一个段落提取一行摘要

如果在探测所有位置后未找到 Agent 文件，通知用户："未找到 Agent 文件。已检查：[列出的探测路径]。期望：这些目录之一的 markdown 文件。"然后停止。

### 步骤 2：呈现领域菜单

```
可用的 Agent 领域：
1. Engineering — Software Architect, Security Engineer
2. Marketing — SEO Specialist
3. Sales — Discovery Coach, Outbound Strategist

选择领域或命名特定的 Agent（例如 "1,3" 或 "security + seo"）：
```

- 跳过零 Agent 的领域（空目录）
- 显示每个领域的 Agent 数量

### 步骤 3：处理选择

接受灵活的输入：
- 数字："1,3" 选择 Engineering 和 Sales 的所有 Agent
- 名称："security + seo" 模糊匹配发现的 Agent
- "all from engineering" 选择该领域的所有 Agent

如果选择了超过 5 个 Agent，按字母顺序列出并要求用户缩小范围："你选择了 N 个 Agent（最多 5 个）。选择要保留的，或者说 'first 5' 使用按字母顺序的前五个。"

确认选择：
```
已选择：Security Engineer + SEO Specialist
他们应该做什么？（描述任务）：
```

### 步骤 4：并行生成 Agent

1. 读取每个选定 Agent 的 markdown 文件
2. 如果尚未提供，提示输入任务描述
3. 使用 Agent 工具并行生成所有 Agent：
   - `subagent_type: "general-purpose"`
   - `prompt: "{agent 文件内容}\n\n任务：{任务描述}"`
   - 每个 Agent 独立运行 — 不需要 Agent 间通信
4. 如果 Agent 失败（错误、超时或空输出），内联记录失败（例如，"Security Engineer：失败 — [原因]"）并继续使用成功的 Agent 的结果

### 步骤 5：综合结果

收集所有输出并呈现统一报告：
- 按 Agent 分组的结果
- 综合部分突出：
  - Agent 间的共识
  - 建议之间的冲突或紧张
  - 推荐的下一步

如果只选择了 1 个 Agent，跳过综合并直接呈现输出。

## 规则

- **仅动态发现。** 永远不要硬编码 Agent 列表。目录中的新文件自动出现在菜单中。
- **每个团队最多 5 个 Agent。** 超过 5 个会产生边际收益递减和过多的 token 使用。在选择时强制执行。
- **并行分派。** 所有 Agent 同时运行 — 使用 Agent 工具的并行调用模式。
- **并行 Agent 调用，不是 TeamCreate。** 此技能使用并行 Agent 工具调用进行独立工作。只有当 Agent 必须辩论或相互响应时才需要 TeamCreate（用于多 Agent 对话的 Claude Code 工具）。

## 示例

```
用户：team builder

Claude：
可用的 Agent 领域：
1. Engineering (2) — Software Architect, Security Engineer
2. Marketing (1) — SEO Specialist
3. Sales (4) — Discovery Coach, Outbound Strategist, Proposal Strategist, Sales Engineer
4. Support (1) — Executive Summary

选择领域或命名特定的 Agent：

用户：security + seo

Claude：
已选择：Security Engineer + SEO Specialist
他们应该做什么？

用户：在启动前审查我的 Next.js 电商网站

[两个 Agent 并行生成，每个将他们的专业应用于代码库]

Claude：
## Security Engineer 发现
- [发现...]

## SEO Specialist 发现
- [发现...]

## 综合
两个 Agent 同意：[...]
紧张：安全建议阻止内联样式的 CSP，SEO 需要内联 schema 标记。解决方案：[...]
下一步：[...]
```
