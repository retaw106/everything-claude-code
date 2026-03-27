---
description: 分析 agents、skills、MCP servers 和 rules 的上下文窗口使用情况，找到优化机会。帮助减少 token 开销并避免性能警告。
---

# 上下文预算优化器

分析你的 Claude Code 配置的上下文窗口消耗，并生成可操作的建议以减少 token 开销。

## 用法

```
/context-budget [--verbose]
```

- 默认：摘要和主要建议
- `--verbose`：每个组件的完整细分

## 参数

$ARGUMENTS

## 做什么

使用以下输入运行 **context-budget** skill (`skills/context-budget/SKILL.md`)：

1. 如果 `$ARGUMENTS` 中存在 `--verbose` 标志，则传递它
2. 假设 200K 上下文窗口（Claude Sonnet 默认），除非用户另有指定
3. 遵循 skill 的四个阶段：清单 → 分类 → 检测问题 → 报告
4. 向用户输出格式化的上下文预算报告

该 skill 处理所有扫描逻辑、token 估算、问题检测和报告格式化。
