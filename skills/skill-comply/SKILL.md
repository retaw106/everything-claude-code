---
name: skill-comply
description: 可视化技能、规则和 Agent 定义是否真正被遵循 — 自动生成 3 种提示严格性级别的场景，运行 Agent，分类行为序列，并报告合规率及完整工具调用时间线
origin: ECC
tools: Read, Bash
---

# skill-comply：自动化合规性测量

通过以下方式测量编码 Agent 是否真正遵循技能、规则或 Agent 定义：
1. 从任何 .md 文件自动生成期望的行为序列（规格）
2. 自动生成提示严格性递减的场景（支持性 → 中性 → 竞争性）
3. 运行 `claude -p` 并通过 stream-json 捕获工具调用追踪
4. 使用 LLM（而非正则）对规格步骤分类工具调用
5. 确定性地检查时间顺序
6. 生成包含规格、提示和时间线的独立报告

## 支持的目标

- **Skills**（`skills/*/SKILL.md`）：工作流技能如 search-first、TDD 指南
- **Rules**（`rules/common/*.md`）：强制性规则如 testing.md、security.md、git-workflow.md
- **Agent 定义**（`agents/*.md`）：Agent 是否在期望时被调用（内部工作流验证尚不支持）

## 何时启用

- 用户运行 `/skill-comply <path>`
- 用户询问"这条规则真的被遵循吗？"
- 添加新规则/技能后，验证 Agent 合规性
- 作为质量维护的一部分定期运行

## 用法

```bash
# 完整运行
uv run python -m scripts.run ~/.claude/rules/common/testing.md

# 演练运行（无成本，仅规格 + 场景）
uv run python -m scripts.run --dry-run ~/.claude/skills/search-first/SKILL.md

# 自定义模型
uv run python -m scripts.run --gen-model haiku --model sonnet <path>
```

## 关键概念：提示独立性

测量即使提示没有明确支持，技能/规则是否被遵循。

## 报告内容

报告是独立的，包括：
1. 期望的行为序列（自动生成的规格）
2. 场景提示（每种严格性级别问的是什么）
3. 每个场景的合规分数
4. 带 LLM 分类标签的工具调用时间线

### 高级（可选）

对于熟悉 hooks 的用户，报告还包括低合规步骤的 hook 推广建议。这是信息性的 — 主要价值是合规性可见性本身。
