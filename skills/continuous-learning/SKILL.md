---
name: continuous-learning
description: 自动从 Claude Code 会话中提取可重用模式，并将其保存为学习技能供未来使用。
origin: ECC
---

# 持续学习技能

在会话结束时自动评估 Claude Code 会话，提取可保存为学习技能的可重用模式。

## 何时使用

- 设置从 Claude Code 会话自动提取模式
- 配置 Stop hook 进行会话评估
- 审查或管理 `~/.claude/skills/learned/` 中的学习技能
- 调整提取阈值或模式类别
- 比较 v1（本版本）与 v2（基于直觉）方法

## 工作原理

此技能在每个会话结束时作为 **Stop hook** 运行：

1. **会话评估**：检查会话是否有足够的消息（默认：10+ 条）
2. **模式检测**：从会话中识别可提取的模式
3. **技能提取**：将有用模式保存到 `~/.claude/skills/learned/`

## 配置

编辑 `config.json` 进行自定义：

```json
{
  "min_session_length": 10,
  "extraction_threshold": "medium",
  "auto_approve": false,
  "learned_skills_path": "~/.claude/skills/learned/",
  "patterns_to_detect": [
    "error_resolution",
    "user_corrections",
    "workarounds",
    "debugging_techniques",
    "project_specific"
  ],
  "ignore_patterns": [
    "simple_typos",
    "one_time_fixes",
    "external_api_issues"
  ]
}
```

## 模式类型

| 模式 | 描述 |
|---------|-------------|
| `error_resolution` | 如何解决特定错误 |
| `user_corrections` | 来自用户更正的模式 |
| `workarounds` | 框架/库特殊行为的解决方案 |
| `debugging_techniques` | 有效的调试方法 |
| `project_specific` | 项目特定约定 |

## Hook 设置

添加到你的 `~/.claude/settings.json`：

```json
{
  "hooks": {
    "Stop": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/continuous-learning/evaluate-session.sh"
      }]
    }]
  }
}
```

## 为什么用 Stop Hook？

- **轻量级**：在会话结束时运行一次
- **非阻塞**：不给每条消息增加延迟
- **完整上下文**：可以访问完整的会话记录

## 相关

- [详细指南](https://x.com/affaanmustafa/status/2014040193557471352) - 关于持续学习的章节
- `/learn` 命令 - 会话中手动提取模式

---

## 比较说明（研究：2025 年 1 月）

### vs Homunculus

Homunculus v2 采用了更复杂的方法：

| 特性 | 我们的方法 | Homunculus v2 |
|---------|--------------|---------------|
| 观察 | Stop hook（会话结束时） | PreToolUse/PostToolUse hooks（100% 可靠） |
| 分析 | 主上下文 | 后台 Agent（Haiku） |
| 粒度 | 完整技能 | 原子"直觉" |
| 置信度 | 无 | 0.3-0.9 加权 |
| 演进 | 直接到技能 | 直觉 → 聚类 → skill/command/agent |
| 分享 | 无 | 导出/导入直觉 |

**来自 homunculus 的关键洞察：**
> "v1 依赖技能来观察。技能是概率性的 — 它们大约 50-80% 的时间触发。v2 使用 hooks 进行观察（100% 可靠），直觉作为学习行为的原子单位。"

### 潜在的 v2 增强

1. **基于直觉的学习** - 更小的、带有置信度评分的原子行为
2. **后台观察者** - Haiku Agent 并行分析
3. **置信度衰减** - 直觉如果被反驳则失去置信度
4. **领域标签** - code-style、testing、git、debugging 等
5. **演进路径** - 将相关直觉聚类成 skills/commands

参见：`docs/continuous-learning-v2-spec.md` 获取完整规格。
