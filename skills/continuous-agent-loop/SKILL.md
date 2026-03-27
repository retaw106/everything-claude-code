---
name: continuous-agent-loop
description: 具有质量门控、评估和恢复控制的持续自主 Agent 循环模式。
origin: ECC
---

# 持续 Agent 循环

这是 v1.8+ 规范的循环技能名称。 它取代了 `autonomous-loops`,同时保持一个版本的兼容性。

## 循环选择流程

```text
开始
   |
   +-- 需要严格的 CI/PR 控制? -- 是 --> continuous-pr
   |                                    
   +-- 需要 RFC 分解? -- 是 --> rfc-dag
   |
   +-- 需要探索性并行生成? -- 是 --> infinite
   |
   +-- 默认 --> sequential
```

## 组合模式

推荐的生产技术栈:
1. RFC 分解 (`ralphinho-rfc-pipeline`)
2. 质量门控 (`plankton-code-quality` + `/quality-gate`)
3. 评估循环 (`eval-harness`)
4. 会话持久化 (`nanoclaw-repl`)

## 失败模式

- 循环震荡但没有可衡量的进展
- 相同根因的重复重试
- 合并队列停滞
- 无界升级导致的成本漂移

## 恢复

- 冻结循环
- 运行 `/harness-audit`
- 将范围缩小到失败单元
- 使用明确的验收标准重放
