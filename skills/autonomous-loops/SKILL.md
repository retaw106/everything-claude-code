---
name: autonomous-loops
description: "自主 Claude Code 循环的模式和架构 — 从简单的顺序流水线到 RFC 驱动的多 Agent DAG 系统。"
origin: ECC
---

# 自主循环技能

> 兼容性说明（v1.8.0）：`autonomous-loops` 保留一个版本。
> 规范的技能名称现在是 `continuous-agent-loop`。新的循环指导
> 应该在那里编写，而此技能仍然可用以避免
> 破坏现有工作流。

用于自主运行 Claude Code 的模式、架构和参考实现。涵盖从简单的 `claude -p` 流水线到完整的 RFC 驱动多 Agent DAG 编排的所有内容。

## 何时使用

- 设置无需人工干预即可运行的自主开发工作流
- 为你的问题选择正确的循环架构（简单 vs 复杂）
- 构建 CI/CD 风格的持续开发流水线
- 运行带有合并协调的并行 Agent
- 在循环迭代中实现上下文持久化
- 向自主工作流添加质量门和清理通过

## 循环模式谱

从最简单到最复杂：

| 模式 | 复杂度 | 最适合 |
|---------|-----------|----------|
| [顺序流水线](#1-sequential-pipeline-claude--p) | 低 | 日常开发步骤、脚本化工作流 |
| [NanoClaw REPL](#2-nanoclaw-repl) | 低 | 交互式持久会话 |
| [无限 Agent 循环](#3-infinite-agentic-loop) | 中 | 并行内容生成、规格驱动工作 |
| [持续 Claude PR 循环](#4-continuous-claude-pr-loop) | 中 | 带 CI 门的多日迭代项目 |
| [De-Sloppify 模式](#5-the-de-sloppify-pattern) | 附加 | 任何实现者步骤后的质量清理 |
| [Ralphinho / RFC 驱动 DAG](#6-ralphinho--rfc-driven-dag-orchestration) | 高 | 大型功能、带合并队列的多单元并行工作 |

---

## 1. 顺序流水线（`claude -p`）

**最简单的循环。** 将日常开发分解为一系列非交互式 `claude -p` 调用。每个调用是一个带有清晰提示的聚焦步骤。

### 核心洞察

> 如果你无法想出这样的循环，这意味着你甚至无法在交互模式下驱动 LLM 修复你的代码。

`claude -p` 标志以非交互方式运行 Claude Code 并带上提示，完成后退出。链接调用以构建流水线：

```bash
#!/bin/bash
# daily-dev.sh — 功能分支的顺序流水线

set -e

# 步骤 1：实现功能
claude -p "阅读 docs/auth-spec.md 中的规格。在 src/auth/ 中实现 OAuth2 登录。先写测试（TDD）。不要创建任何新的文档文件。"

# 步骤 2：De-sloppify（清理通过）
claude -p "审查上次提交更改的所有文件。删除任何不必要的类型测试、过度防御性检查或语言特性测试（例如测试 TypeScript 泛型是否工作）。保留真正的业务逻辑测试。清理后运行测试套件。"

# 步骤 3：验证
claude -p "运行完整的构建、lint、类型检查和测试套件。修复任何失败。不要添加新功能。"

# 步骤 4：提交
claude -p "为所有暂存更改创建约定式提交。使用 'feat: add OAuth2 login flow' 作为消息。"
```

### 关键设计原则

1. **每个步骤都是隔离的** — 每次 `claude -p` 调用一个全新的上下文窗口意味着步骤之间没有上下文泄露。
2. **顺序很重要** — 步骤按顺序执行。每个步骤都建立在前一个步骤留下的文件系统状态上。
3. **否定指令是危险的** — 不要说"不要测试类型系统"。相反，添加一个单独的清理步骤（见 [De-Sloppify 模式](#5-the-de-sloppify-pattern)）。
4. **退出码传播** — `set -e` 在失败时停止流水线。

### 变体

**带模型路由：**
```bash
# 用 Opus 研究（深度推理）
claude -p --model opus "分析代码库架构并编写添加缓存的计划..."

# 用 Sonnet 实现（快速、能干）
claude -p "根据 docs/caching-plan.md 中的计划实现缓存层..."

# 用 Opus 审查（彻底）
claude -p --model opus "审查所有更改的安全问题、竞态条件和边缘情况..."
```

**带环境上下文：**
```bash
# 通过文件传递上下文，而不是提示长度
echo "关注领域：auth 模块、API 速率限制" > .claude-context.md
claude -p "阅读 .claude-context.md 了解优先级。按顺序处理它们。"
rm .claude-context.md
```

**带 `--allowedTools` 限制：**
```bash
# 只读分析通过
claude -p --allowedTools "Read,Grep,Glob" "审计此代码库的安全漏洞..."

# 只写实现通过
claude -p --allowedTools "Read,Write,Edit,Bash" "实现 security-audit.md 中的修复..."
```

---

## 2. NanoClaw REPL

**ECC 内置的持久循环。** 一个会话感知的 REPL，同步调用 `claude -p` 并带有完整的对话历史。

```bash
# 启动默认会话
node scripts/claw.js

# 带技能上下文的命名会话
CLAW_SESSION=my-project CLAW_SKILLS=tdd-workflow,security-review node scripts/claw.js
```

### 工作原理

1. 从 `~/.claude/claw/{session}.md` 加载对话历史
2. 每条用户消息作为上下文发送给 `claude -p` 并带有完整历史
3. 响应追加到会话文件（Markdown 作为数据库）
4. 会话在重启后持久化

### NanoClaw vs 顺序流水线何时使用

| 使用场景 | NanoClaw | 顺序流水线 |
|----------|----------|-------------------|
| 交互式探索 | 是 | 否 |
| 脚本化自动化 | 否 | 是 |
| 会话持久化 | 内置 | 手动 |
| 上下文累积 | 每轮增长 | 每步刷新 |
| CI/CD 集成 | 差 | 优秀 |

有关完整详细信息，请参阅 `/claw` 命令文档。

---

## 3. 无限 Agent 循环

**双提示系统**，编排并行子 Agent 进行规格驱动生成。由 disler 开发（credit: @disler）。

### 架构：双提示系统

```
提示 1（编排者）              提示 2（子 Agent）
┌─────────────────────┐             ┌──────────────────────┐
│ 解析规格文件         │             │ 接收完整上下文        │
│ 扫描输出目录         │  派遣      │ 读取分配的编号        │
│ 规划迭代             │────────────│ 严格遵循规格          │
│ 分配创意方向         │  N 个 Agent │ 生成唯一输出          │
│ 管理波次             │             │ 保存到输出目录        │
└─────────────────────┘             └──────────────────────┘
```

### 模式

1. **规格分析** — 编排者读取定义要生成内容的规格文件（Markdown）
2. **目录侦察** — 扫描现有输出以找到最高迭代编号
3. **并行派遣** — 启动 N 个子 Agent，每个带有：
   - 完整规格
   - 独特的创意方向
   - 特定的迭代编号（无冲突）
   - 现有迭代的快照（用于唯一性）
4. **波次管理** — 对于无限模式，派遣 3-5 个 Agent 的波次直到上下文耗尽

### 通过 Claude Code 命令实现

创建 `.claude/commands/infinite.md`：

```markdown
从 $ARGUMENTS 解析以下参数：
1. spec_file — 规格 Markdown 的路径
2. output_dir — 保存迭代的位置
3. count — 整数 1-N 或 "infinite"

阶段 1：阅读并深入理解规格。
阶段 2：列出 output_dir，找到最高迭代编号。从 N+1 开始。
阶段 3：规划创意方向 — 每个 Agent 获得不同的主题/方法。
阶段 4：并行派遣子 Agent（Task 工具）。每个接收：
  - 完整规格文本
  - 当前目录快照
  - 它们分配的迭代编号
  - 它们独特的创意方向
阶段 5（无限模式）：以 3-5 个为一波循环直到上下文不足。
```

**调用：**
```bash
/project:infinite specs/component-spec.md src/ 5
/project:infinite specs/component-spec.md src/ infinite
```

### 批处理策略

| 数量 | 策略 |
|-------|----------|
| 1-5 | 所有 Agent 同时 |
| 6-20 | 每批 5 个 |
| infinite | 3-5 个一波，递进的复杂性 |

### 关键洞察：通过分配实现唯一性

不要依赖 Agent 自我区分。编排者**分配**每个 Agent 特定的创意方向和迭代编号。这可以防止并行 Agent 之间的重复概念。

---

## 4. 持续 Claude PR 循环

**生产级 shell 脚本**，持续循环运行 Claude Code，创建 PR，等待 CI，并自动合并。由 AnandChowdhary 创建（credit: @AnandChowdhary）。

### 核心循环

```
┌─────────────────────────────────────────────────────┐
│  持续 CLAUDE 迭代                                    │
│                                                     │
│  1. 创建分支 (continuous-claude/iteration-N)        │
│  2. 运行 claude -p 并增强提示                        │
│  3. (可选) 审查者通过 — 单独的 claude -p             │
│  4. 提交更改（claude 生成消息）                       │
│  5. 推送 + 创建 PR (gh pr create)                    │
│  6. 等待 CI 检查 (poll gh pr checks)                 │
│  7. CI 失败？ → 自动修复通过 (claude -p)             │
│  8. 合并 PR (squash/merge/rebase)                    │
│  9. 返回 main → 重复                                 │
│                                                     │
│  限制方式：--max-runs N | --max-cost $X              │
│            --max-duration 2h | 完成信号              │
└─────────────────────────────────────────────────────┘
```

### 安装

> **警告：** 在审查代码后从其仓库安装 continuous-claude。不要将外部脚本直接管道传输到 bash。

### 使用

```bash
# 基础：10 次迭代
continuous-claude --prompt "为所有未测试函数添加单元测试" --max-runs 10

# 成本限制
continuous-claude --prompt "修复所有 linter 错误" --max-cost 5.00

# 时间限制
continuous-claude --prompt "提高测试覆盖率" --max-duration 8h

# 带代码审查通过
continuous-claude \
  --prompt "添加认证功能" \
  --max-runs 10 \
  --review-prompt "运行 npm test && npm run lint，修复任何失败"

# 通过 worktrees 并行
continuous-claude --prompt "添加测试" --max-runs 5 --worktree tests-worker &
continuous-claude --prompt "重构代码" --max-runs 5 --worktree refactor-worker &
wait
```

### 跨迭代上下文：SHARED_TASK_NOTES.md

关键创新：一个 `SHARED_TASK_NOTES.md` 文件在迭代间持久化：

```markdown
## 进度
- [x] 为 auth 模块添加测试（迭代 1）
- [x] 修复 token 刷新的边缘情况（迭代 2）
- [ ] 仍然需要：速率限制测试、错误边界测试

## 下一步
- 下次专注于速率限制模块
- tests/helpers.ts 中的 mock 设置可以重用
```

Claude 在迭代开始时读取此文件，在迭代结束时更新它。这桥接了独立 `claude -p` 调用之间的上下文差距。

### CI 失败恢复

当 PR 检查失败时，持续 Claude 自动：
1. 通过 `gh run list` 获取失败运行 ID
2. 用 CI 修复上下文生成新的 `claude -p`
3. Claude 通过 `gh run view` 检查日志，修复代码，提交，推送
4. 重新等待检查（最多 `--ci-retry-max` 次尝试）

### 完成信号

Claude 可以通过输出魔法短语发出"我完成了"的信号：

```bash
continuous-claude \
  --prompt "修复 issue tracker 中的所有 bug" \
  --completion-signal "CONTINUOUS_CLAUDE_PROJECT_COMPLETE" \
  --completion-threshold 3  # 3 次连续信号后停止
```

三次连续迭代发出完成信号会停止循环，防止在完成的工作上浪费运行。

### 关键配置

| 标志 | 用途 |
|------|---------|
| `--max-runs N` | N 次成功迭代后停止 |
| `--max-cost $X` | 花费 $X 后停止 |
| `--max-duration 2h` | 时间过去后停止 |
| `--merge-strategy squash` | squash、merge 或 rebase |
| `--worktree <name>` | 通过 git worktrees 并行执行 |
| `--disable-commits` | 试运行模式（无 git 操作） |
| `--review-prompt "..."` | 每次迭代添加审查者通过 |
| `--ci-retry-max N` | 自动修复 CI 失败（默认：1） |

---

## 5. De-Sloppify 模式

**任何循环的附加模式。** 在每个实现者步骤后添加专用的清理/重构步骤。

### 问题

当你要求 LLM 用 TDD 实现时，它太按字面意思理解"写测试"：
- 验证 TypeScript 类型系统工作的测试（测试 `typeof x === 'string'`）
- 对类型系统已经保证的事情进行过度防御性运行时检查
- 测试框架行为而不是业务逻辑
- 过度的错误处理，掩盖了实际代码

### 为什么不用否定指令？

向实现者提示添加"不要测试类型系统"或"不要添加不必要的检查"会产生下游影响：
- 模型对所有测试变得犹豫
- 它跳过合法的边缘情况测试
- 质量不可预测地下降

### 解决方案：单独的通过

与其约束实现者，不如让它彻底。然后添加一个聚焦的清理 Agent：

```bash
# 步骤 1：实现（让它彻底）
claude -p "用完整 TDD 实现功能。测试要彻底。"

# 步骤 2：De-sloppify（单独上下文，聚焦清理）
claude -p "审查工作树中的所有更改。删除：
- 测试语言/框架行为而不是业务逻辑的测试
- 类型系统已经强制执行的冗余类型检查
- 对不可能状态的过度防御性错误处理
- Console.log 语句
- 注释掉的代码

保留所有业务逻辑测试。清理后运行测试套件以确保没有任何破坏。"
```

### 在循环上下文中

```bash
for feature in "${features[@]}"; do
  # 实现
  claude -p "用 TDD 实现 $feature。"

  # De-sloppify
  claude -p "清理通过：审查更改，删除测试/代码冗余，运行测试。"

  # 验证
  claude -p "运行构建 + lint + 测试。修复任何失败。"

  # 提交
  claude -p "提交并附带消息：feat: add $feature"
done
```

### 关键洞察

> 与其添加具有下游质量影响的否定指令，不如添加一个单独的 de-sloppify 通过。两个聚焦的 Agent 优于一个受限的 Agent。

---

## 6. Ralphinho / RFC 驱动 DAG 编排

**最复杂的模式。** RFC 驱动的、多 Agent 流水线，将规格分解为依赖 DAG，通过分层质量流水线运行每个单元，并通过 Agent 驱动的合并队列着陆。由 enitrat 创建（credit: @enitrat）。

### 架构概览

```
RFC/PRD 文档
       │
       ▼
  分解（AI）
  将 RFC 分解为带有依赖 DAG 的工作单元
       │
       ▼
┌──────────────────────────────────────────────────────┐
│  RALPH 循环（最多 3 次通过）                           │
│                                                      │
│  对于每个 DAG 层（按依赖顺序）：                       │
│                                                      │
│  ┌── 质量流水线（每个单元并行）────────────────────┐  │
│  │  每个单元在自己的 worktree 中：                  │  │
│  │  研究 → 计划 → 实现 → 测试 → 审查               │  │
│  │  （深度因复杂性层级而异）                        │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  ┌── 合并队列 ─────────────────────────────────────┐  │
│  │  变基到 main → 运行测试 → 着陆或驱逐             │  │
│  │  被驱逐的单元带着冲突上下文重新进入              │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### RFC 分解

AI 读取 RFC 并产生工作单元：

```typescript
interface WorkUnit {
  id: string;              // kebab-case 标识符
  name: string;            // 人类可读名称
  rfcSections: string[];   // 此单元涉及的 RFC 章节
  description: string;     // 详细描述
  deps: string[];          // 依赖（其他单元 ID）
  acceptance: string[];    // 具体验收标准
  tier: "trivial" | "small" | "medium" | "large";
}
```

**分解规则：**
- 偏好更少、内聚的单元（最小化合并风险）
- 最小化跨单元文件重叠（避免冲突）
- 测试与实现保持在一起（永远不要分离"实现 X" + "测试 X"）
- 仅在存在真正代码依赖时才有依赖

依赖 DAG 决定执行顺序：
```
层 0：[unit-a, unit-b]     ← 无依赖，并行运行
层 1：[unit-c]             ← 依赖 unit-a
层 2：[unit-d, unit-e]     ← 依赖 unit-c
```

### 复杂性层级

不同层级获得不同的流水线深度：

| 层级 | 流水线阶段 |
|------|----------------|
| **trivial** | 实现 → 测试 |
| **small** | 实现 → 测试 → code-review |
| **medium** | 研究 → 计划 → 实现 → 测试 → PRD 审查 + code-review → review-fix |
| **large** | 研究 → 计划 → 实现 → 测试 → PRD 审查 + code-review → review-fix → final-review |

这可以防止对简单更改进行昂贵操作，同时确保架构更改得到彻底审查。

### 独立上下文窗口（消除作者偏见）

每个阶段在自己的 Agent 进程中运行，有自己的上下文窗口：

| 阶段 | 模型 | 用途 |
|-------|-------|---------|
| 研究 | Sonnet | 读取代码库 + RFC，产生上下文文档 |
| 计划 | Opus | 设计实现步骤 |
| 实现 | Codex | 按计划编写代码 |
| 测试 | Sonnet | 运行构建 + 测试套件 |
| PRD 审查 | Sonnet | 规格合规检查 |
| 代码审查 | Opus | 质量 + 安全检查 |
| 审查修复 | Codex | 解决审查问题 |
| 最终审查 | Opus | 质量门（仅 large 层级） |

**关键设计：** 审查者从未编写它审查的代码。这消除了作者偏见 — 自我审查中最常见的问题来源。

### 带驱逐的合并队列

质量流水线完成后，单元进入合并队列：

```
单元分支
    │
    ├─ 变基到 main
    │   └─ 冲突？ → 驱逐（捕获冲突上下文）
    │
    ├─ 运行构建 + 测试
    │   └─ 失败？ → 驱逐（捕获测试输出）
    │
    └─ 通过 → 快进 main，推送，删除分支
```

**文件重叠智能：**
- 非重叠单元投机并行着陆
- 重叠单元逐个着陆，每次都变基

**驱逐恢复：**
当被驱逐时，捕获完整上下文（冲突文件、差异、测试输出）并在下一次 Ralph 通过时反馈给实现者：

```markdown
## 合并冲突 — 着陆前解决

你之前的实现与另一个先着陆的单元冲突。
重构你的更改以避免下面的冲突文件/行。

{带有差异的完整驱逐上下文}
```

### 阶段间数据流

```
research.contextFilePath ──────────────────→ plan
plan.implementationSteps ──────────────────→ implement
implement.{filesCreated, whatWasDone} ─────→ test, reviews
test.failingSummary ───────────────────────→ reviews, implement (下次通过)
reviews.{feedback, issues} ────────────────→ review-fix → implement (下次通过)
final-review.reasoning ────────────────────→ implement (下次通过)
evictionContext ───────────────────────────→ implement (合并冲突后)
```

### Worktree 隔离

每个单元在隔离的 worktree 中运行（使用 jj/Jujutsu，不是 git）：
```
/tmp/workflow-wt-{unit-id}/
```

同一单元的流水线阶段**共享**一个 worktree，在研究 → 计划 → 实现 → 测试 → 审查之间保留状态（上下文文件、计划文件、代码更改）。

### 关键设计原则

1. **确定性执行** — 提前分解锁定并行性和顺序
2. **在杠杆点人工审查** — 工作计划是单个最高杠杆干预点
3. **分离关注点** — 每个阶段在单独的上下文窗口中用单独的 Agent
4. **带上下文的冲突恢复** — 完整驱逐上下文支持智能重运行，而非盲目重试
5. **层级驱动深度** — 琐碎更改跳过研究/审查；大型更改获得最大审查
6. **可恢复工作流** — 完整状态持久化到 SQLite；从任何点恢复

### 何时使用 Ralphinho vs 更简单的模式

| 信号 | 使用 Ralphinho | 使用更简单模式 |
|--------|--------------|-------------------|
| 多个相互依赖的工作单元 | 是 | 否 |
| 需要并行实现 | 是 | 否 |
| 可能发生合并冲突 | 是 | 否（顺序即可） |
| 单文件更改 | 否 | 是（顺序流水线） |
| 多日项目 | 是 | 也许（continuous-claude） |
| 规格/RFC 已写好 | 是 | 也许 |
| 快速迭代一件事 | 否 | 是（NanoClaw 或流水线） |

---

## 选择正确的模式

### 决策矩阵

```
任务是单个聚焦更改吗？
├─ 是 → 顺序流水线或 NanoClaw
└─ 否 → 有书面规格/RFC 吗？
         ├─ 是 → 需要并行实现吗？
         │        ├─ 是 → Ralphinho（DAG 编排）
         │        └─ 否 → 持续 Claude（迭代 PR 循环）
         └─ 否 → 需要许多同一事物的变体吗？
                  ├─ 是 → 无限 Agent 循环（规格驱动生成）
                  └─ 否 → 带 de-sloppify 的顺序流水线
```

### 组合模式

这些模式组合得很好：

1. **顺序流水线 + De-Sloppify** — 最常见的组合。每个实现步骤获得一个清理通过。

2. **持续 Claude + De-Sloppify** — 添加带有 de-sloppify 指令的 `--review-prompt` 到每次迭代。

3. **任何循环 + 验证** — 使用 ECC 的 `/verify` 命令或 `verification-loop` 技能作为提交前的门。

4. **Ralphinho 在更简单循环中的分层方法** — 即使在顺序流水线中，你也可以将简单任务路由到 Haiku，复杂任务路由到 Opus：
   ```bash
   # 简单格式修复
   claude -p --model haiku "修复 src/utils.ts 中的导入顺序"

   # 复杂架构更改
   claude -p --model opus "重构 auth 模块以使用策略模式"
   ```

---

## 反模式

### 常见错误

1. **无退出条件的无限循环** — 始终有 max-runs、max-cost、max-duration 或完成信号。

2. **迭代间无上下文桥接** — 每次 `claude -p` 调用都是全新开始。使用 `SHARED_TASK_NOTES.md` 或文件系统状态桥接上下文。

3. **重试相同失败** — 如果迭代失败，不要只是重试。捕获错误上下文并将其反馈给下一次尝试。

4. **否定指令而不是清理通过** — 不要说"不要做 X"。添加一个删除 X 的单独通过。

5. **所有 Agent 在一个上下文窗口中** — 对于复杂工作流，将关注点分离到不同的 Agent 进程中。审查者永远不应该是作者。

6. **在并行工作中忽略文件重叠** — 如果两个并行 Agent 可能编辑同一文件，你需要合并策略（顺序着陆、变基或冲突解决）。

---

## 参考

| 项目 | 作者 | 链接 |
|---------|--------|------|
| Ralphinho | enitrat | credit: @enitrat |
| Infinite Agentic Loop | disler | credit: @disler |
| Continuous Claude | AnandChowdhary | credit: @AnandChowdhary |
| NanoClaw | ECC | 此仓库中的 `/claw` 命令 |
| Verification Loop | ECC | 此仓库中的 `skills/verification-loop/` |
