# 贡献指南

感谢您想要为 Everything Claude Code 做出贡献！本仓库是 Claude Code 用户的社区资源。

## 目录

- [我们在寻找什么](#我们在寻找什么)
- [快速开始](#快速开始)
- [贡献 Skills](#贡献-skills)
- [贡献 Agents](#贡献-agents)
- [贡献 Hooks](#贡献-hooks)
- [贡献 Commands](#贡献-commands)
- [MCP 和文档（如 Context7）](#mcp-和文档如-context7)
- [跨 Harness 和翻译](#跨-harness-和翻译)
- [Pull Request 流程](#pull-request-流程)

---

## 我们在寻找什么

### Agents
能够很好处理特定任务的新 Agent：
- 语言特定的审查器（Python、Go、Rust）
- 框架专家（Django、Rails、Laravel、Spring）
- DevOps 专家（Kubernetes、Terraform、CI/CD）
- 领域专家（ML 管道、数据工程、移动端）

### Skills
工作流定义和领域知识：
- 语言最佳实践
- 框架模式
- 测试策略
- 架构指南

### Hooks
有用的自动化：
- Linting/格式化 hooks
- 安全检查
- 验证 hooks
- 通知 hooks

### Commands
调用有用工作流的斜杠命令：
- 部署命令
- 测试命令
- 代码生成命令

---

## 快速开始

```bash
# 1. Fork 并克隆
gh repo fork affaan-m/everything-claude-code --clone
cd everything-claude-code

# 2. 创建分支
git checkout -b feat/my-contribution

# 3. 添加您的贡献（见下方各节）

# 4. 本地测试
cp -r skills/my-skill ~/.claude/skills/  # 对于 skills
# 然后用 Claude Code 测试

# 5. 提交 PR
git add . && git commit -m "feat: add my-skill" && git push -u origin feat/my-contribution
```

---

## 贡献 Skills

Skills 是 Claude Code 基于上下文加载的知识模块。

### 目录结构

```
skills/
└── your-skill-name/
    └── SKILL.md
```

### SKILL.md 模板

```markdown
---
name: your-skill-name
description: 技能列表中显示的简短描述
origin: ECC
---

# Your Skill Title

简要概述此技能涵盖的内容。

## 核心概念

解释关键模式和指南。

## 代码示例

\`\`\`typescript
// 包含实用的、经过测试的示例
function example() {
  // 注释良好的代码
}
\`\`\`

## 最佳实践

- 可操作的指南
- 应做和不应做的事
- 需要避免的常见陷阱

## 何时使用

描述此技能适用的场景。
```

### Skill 检查清单

- [ ] 专注于一个领域/技术
- [ ] 包含实用的代码示例
- [ ] 少于 500 行
- [ ] 使用清晰的章节标题
- [ ] 已用 Claude Code 测试

### 示例 Skills

| Skill | 用途 |
|-------|------|
| `coding-standards/` | TypeScript/JavaScript 模式 |
| `frontend-patterns/` | React 和 Next.js 最佳实践 |
| `backend-patterns/` | API 和数据库模式 |
| `security-review/` | 安全检查清单 |

---

## 贡献 Agents

Agents 是通过 Task 工具调用的专业助手。

### 文件位置

```
agents/your-agent-name.md
```

### Agent 模板

```markdown
---
name: your-agent-name
description: 这个 agent 做什么以及 Claude 何时应该调用它。要具体！
tools: ["Read", "Write", "Edit", "Bash", "Grep", "Glob"]
model: sonnet
---

你是一个 [角色] 专家。

## 你的角色

- 主要职责
- 次要职责
- 你不做的事情（边界）

## 工作流程

### 步骤 1: 理解
你如何处理任务。

### 步骤 2: 执行
你如何执行工作。

### 步骤 3: 验证
你如何验证结果。

## 输出格式

你返回给用户的内容。

## 示例

### 示例: [场景]
输入: [用户提供的内容]
操作: [你做什么]
输出: [你返回什么]
```

### Agent 字段

| 字段 | 描述 | 选项 |
|------|------|------|
| `name` | 小写，用连字符连接 | `code-reviewer` |
| `description` | 用于决定何时调用 | 要具体！ |
| `tools` | 只包含需要的工具 | `Read, Write, Edit, Bash, Grep, Glob, WebFetch, Task`，或 MCP 工具名称（如 `mcp__context7__resolve-library-id`, `mcp__context7__query-docs`）当 agent 使用 MCP 时 |
| `model` | 复杂度级别 | `haiku`（简单）、`sonnet`（编码）、`opus`（复杂） |

### 示例 Agents

| Agent | 用途 |
|-------|------|
| `tdd-guide.md` | 测试驱动开发 |
| `code-reviewer.md` | 代码审查 |
| `security-reviewer.md` | 安全扫描 |
| `build-error-resolver.md` | 修复构建错误 |

---

## 贡献 Hooks

Hooks 是由 Claude Code 事件触发的自动行为。

### 文件位置

```
hooks/hooks.json
```

### Hook 类型

| 类型 | 触发器 | 用例 |
|------|--------|------|
| `PreToolUse` | 工具运行前 | 验证、警告、阻止 |
| `PostToolUse` | 工具运行后 | 格式化、检查、通知 |
| `SessionStart` | 会话开始时 | 加载上下文 |
| `Stop` | 会话结束时 | 清理、审计 |

### Hook 格式

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "tool == \"Bash\" && tool_input.command matches \"rm -rf /\"",
        "hooks": [
          {
            "type": "command",
            "command": "echo '[Hook] BLOCKED: Dangerous command' && exit 1"
          }
        ],
        "description": "阻止危险的 rm 命令"
      }
    ]
  }
}
```

### Matcher 语法

```javascript
// 匹配特定工具
tool == "Bash"
tool == "Edit"
tool == "Write"

// 匹配输入模式
tool_input.command matches "npm install"
tool_input.file_path matches "\\.tsx?$"

// 组合条件
tool == "Bash" && tool_input.command matches "git push"
```

### Hook 示例

```json
// 在 tmux 外阻止开发服务器
{
  "matcher": "tool == \"Bash\" && tool_input.command matches \"npm run dev\"",
  "hooks": [{"type": "command", "command": "echo 'Use tmux for dev servers' && exit 1"}],
  "description": "确保开发服务器在 tmux 中运行"
}

// 编辑 TypeScript 后自动格式化
{
  "matcher": "tool == \"Edit\" && tool_input.file_path matches \"\\.tsx?$\"",
  "hooks": [{"type": "command", "command": "npx prettier --write \"$file_path\""}],
  "description": "编辑后格式化 TypeScript 文件"
}

// git push 前警告
{
  "matcher": "tool == \"Bash\" && tool_input.command matches \"git push\"",
  "hooks": [{"type": "command", "command": "echo '[Hook] Review changes before pushing'"}],
  "description": "提醒在推送前审查更改"
}
```

### Hook 检查清单

- [ ] Matcher 是具体的（不要过于宽泛）
- [ ] 包含清晰的错误/信息消息
- [ ] 使用正确的退出代码（`exit 1` 阻止，`exit 0` 允许）
- [ ] 已充分测试
- [ ] 有描述

---

## 贡献 Commands

Commands 是用户通过 `/command-name` 调用的操作。

### 文件位置

```
commands/your-command.md
```

### Command 模板

```markdown
---
description: /help 中显示的简短描述
---

# Command Name

## 用途

此命令的功能。

## 用法

\`\`\`
/your-command [args]
\`\`\`

## 工作流程

1. 第一步
2. 第二步
3. 最后一步

## 输出

用户收到的内容。
```

### 示例 Commands

| Command | 用途 |
|---------|------|
| `commit.md` | 创建 git 提交 |
| `code-review.md` | 审查代码更改 |
| `tdd.md` | TDD 工作流 |
| `e2e.md` | E2E 测试 |

---

## MCP 和文档（如 Context7）

Skills 和 agents 可以使用 **MCP（Model Context Protocol）** 工具来获取最新数据，而不是仅依赖训练数据。这对于文档特别有用。

- **Context7** 是一个 MCP 服务器，暴露了 `resolve-library-id` 和 `query-docs`。当用户询问库、框架或 API 时使用它，这样答案可以反映当前的文档和代码示例。
- 当贡献依赖于实时文档的 **skills**（如设置、API 使用）时，描述如何使用相关的 MCP 工具（如解析库 ID，然后查询文档）并指向 `documentation-lookup` skill 或 Context7 作为模式。
- 当贡献回答文档/API 问题的 **agents** 时，在 agent 的工具中包含 Context7 MCP 工具名称（如 `mcp__context7__resolve-library-id`, `mcp__context7__query-docs`）并记录解析 → 查询的工作流程。
- **mcp-configs/mcp-servers.json** 包含一个 Context7 条目；用户在他们的 harness（如 Claude Code、Cursor）中启用它以使用 documentation-lookup skill（在 `skills/documentation-lookup/` 中）和 `/docs` 命令。

---

## 跨 Harness 和翻译

### Skill 子集（Codex 和 Cursor）

ECC 为其他 harness 提供了 skill 子集：

- **Codex:** `.agents/skills/` — 在 `agents/openai.yaml` 中列出的 skills 会被 Codex 加载。
- **Cursor:** `.cursor/skills/` — 为 Cursor 打包了一部分 skills。

当您 **添加一个新 skill** 且它应该在 Codex 或 Cursor 上可用时：

1. 像往常一样在 `skills/your-skill-name/` 下添加 skill。
2. 如果它应该在 **Codex** 上可用，将其添加到 `.agents/skills/`（复制 skill 目录或添加引用）并确保在 `agents/openai.yaml` 中引用（如果需要）。
3. 如果它应该在 **Cursor** 上可用，按照 Cursor 的布局将其添加到 `.cursor/skills/` 下。

查看这些目录中的现有 skills 以了解预期结构。保持这些子集同步是手动的；如果在 PR 中更新了它们，请提及。

### 翻译

翻译文件位于 `docs/` 下（如 `docs/zh-CN`、`docs/zh-TW`、`docs/ja-JP`）。如果您更改了已翻译的 agents、commands 或 skills，请考虑更新相应的翻译文件或开启一个 issue，以便维护者或翻译人员可以更新它们。

---

## Pull Request 流程

### 1. PR 标题格式

```
feat(skills): add rust-patterns skill
feat(agents): add api-designer agent
feat(hooks): add auto-format hook
fix(skills): update React patterns
docs: improve contributing guide
```

### 2. PR 描述

```markdown
## 摘要
您添加的内容及原因。

## 类型
- [ ] Skill
- [ ] Agent
- [ ] Hook
- [ ] Command

## 测试
您如何测试的。

## 检查清单
- [ ] 遵循格式指南
- [ ] 已用 Claude Code 测试
- [ ] 无敏感信息（API 密钥、路径）
- [ ] 描述清晰
```

### 3. 审查流程

1. 维护者在 48 小时内审查
2. 如有要求，处理反馈
3. 批准后，合并到 main

---

## 指南

### 应该做的
- 保持贡献专注和模块化
- 包含清晰的描述
- 提交前测试
- 遵循现有模式
- 记录依赖关系

### 不应该做的
- 包含敏感数据（API 密钥、令牌、路径）
- 添加过于复杂或小众的配置
- 提交未经测试的贡献
- 创建现有功能的重复项

---

## 文件命名

- 使用小写和连字符：`python-reviewer.md`
- 要有描述性：`tdd-workflow.md` 而不是 `workflow.md`
- 名称与文件名匹配

---

## 有问题？

- **Issues:** [github.com/affaan-m/everything-claude-code/issues](https://github.com/affaan-m/everything-claude-code/issues)
- **X/Twitter:** [@affaanmustafa](https://x.com/affaanmustafa)

---

感谢您的贡献！让我们一起构建一个伟大的资源。
