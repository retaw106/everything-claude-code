---
description: "用于审查 Claude 技能和命令的质量。支持快速扫描（仅变更的技能）和全面盘点模式，采用顺序子代理批量评估。"
origin: ECC
---

# skill-stocktake

斜杠命令（`/skill-stocktake`），使用质量检查清单 + AI 整体判断来审查所有 Claude 技能和命令。支持两种模式：用于最近变更技能的快速扫描，以及完整审查的全面盘点。

## 范围

该命令针对**相对于调用目录**的以下路径：

| 路径 | 描述 |
|------|------|
| `~/.claude/skills/` | 全局技能（所有项目） |
| `{cwd}/.claude/skills/` | 项目级技能（如果目录存在） |

**在 Phase 1 开始时，命令会明确列出找到和扫描的路径。**

### 定位特定项目

要包含项目级技能，从项目根目录运行：

```bash
cd ~/path/to/my-project
/skill-stocktake
```

如果项目没有 `.claude/skills/` 目录，则只评估全局技能和命令。

## 模式

| 模式 | 触发条件 | 耗时 |
|------|---------|------|
| 快速扫描 | `results.json` 存在（默认） | 5–10 分钟 |
| 全面盘点 | `results.json` 不存在，或 `/skill-stocktake full` | 20–30 分钟 |

**结果缓存：** `~/.claude/skills/skill-stocktake/results.json`

## 快速扫描流程

仅重新评估自上次运行以来已更改的技能（5–10 分钟）。

1. 读取 `~/.claude/skills/skill-stocktake/results.json`
2. 运行：`bash ~/.claude/skills/skill-stocktake/scripts/quick-diff.sh \
         ~/.claude/skills/skill-stocktake/results.json`
   （项目目录从 `$PWD/.claude/skills` 自动检测；仅在需要时显式传递）
3. 如果输出是 `[]`：报告 "自上次运行以来无变更。" 并停止
4. 仅使用相同的 Phase 2 标准重新评估这些变更文件
5. 从之前的结果中保留未变更的技能
6. 仅输出差异
7. 运行：`bash ~/.claude/skills/skill-stocktake/scripts/save-results.sh \
         ~/.claude/skills/skill-stocktake/results.json <<< "$EVAL_RESULTS"`

## 全面盘点流程

### Phase 1 — 清单

运行：`bash ~/.claude/skills/skill-stocktake/scripts/scan.sh`

该脚本枚举技能文件、提取 frontmatter 并收集 UTC mtime。
项目目录从 `$PWD/.claude/skills` 自动检测；仅在需要时显式传递。
展示脚本输出中的扫描摘要和清单表：

```
Scanning:
  ✓ ~/.claude/skills/         (17 files)
  ✗ {cwd}/.claude/skills/    (未找到 — 仅全局技能)
```

| 技能 | 7天使用 | 30天使用 | 描述 |
|------|---------|----------|------|

### Phase 2 — 质量评估

使用完整清单和检查清单启动 Agent 工具子代理（**通用代理**）：

```text
Agent(
  subagent_type="general-purpose",
  prompt="
根据检查清单评估以下技能清单。

[INVENTORY]

[CHECKLIST]

为每个技能返回 JSON：
{ \"verdict\": \"Keep\"|\"Improve\"|\"Update\"|\"Retire\"|\"Merge into [X]\", \"reason\": \"...\" }
"
)
```

子代理读取每个技能、应用检查清单并返回每个技能的 JSON：

`{ "verdict": "Keep"|"Improve"|"Update"|"Retire"|"Merge into [X]", "reason": "..." }`

**分块指导：** 每次子代理调用处理约 20 个技能以保持上下文可控。每个分块后将中间结果保存到 `results.json`（`status: "in_progress"`）。

所有技能评估完成后：设置 `status: "completed"`，进入 Phase 3。

**恢复检测：** 如果启动时发现 `status: "in_progress"`，从第一个未评估的技能恢复。

每个技能根据此检查清单进行评估：

```
- [ ] 检查与其他技能的内容重叠
- [ ] 检查与 MEMORY.md / CLAUDE.md 的重叠
- [ ] 验证技术参考的新鲜度（如果存在工具名称 / CLI 标志 / API，使用 WebSearch）
- [ ] 考虑使用频率
```

判定标准：

| 判定 | 含义 |
|------|------|
| Keep | 有用且最新 |
| Improve | 值得保留，但需要特定改进 |
| Update | 引用的技术已过时（用 WebSearch 验证） |
| Retire | 质量低、过时或成本不对称 |
| Merge into [X] | 与另一技能有实质性重叠；命名合并目标 |

评估是**整体 AI 判断** — 不是数值评分。指导维度：
- **可操作性**：让你能立即行动的代码示例、命令或步骤
- **范围适配**：名称、触发条件和内容一致；不太宽泛或太窄
- **独特性**：MEMORY.md / CLAUDE.md / 另一技能无法替代的价值
- **时效性**：技术参考在当前环境中有效

**原因质量要求** — `reason` 字段必须自包含且能支持决策：
- 不要只写 "unchanged" — 始终重述核心证据
- 对于 **Retire**：说明 (1) 发现的具体缺陷，(2) 什么替代方案覆盖相同需求
  - 错误：`"Superseded"`
  - 正确：`"disable-model-invocation: true 已设置；被 continuous-learning-v2 取代，后者涵盖所有相同模式外加置信度评分。无独特内容保留。"`
- 对于 **Merge**：命名目标并描述要整合什么内容
  - 错误：`"与 X 重叠"`
  - 正确：`"42 行薄弱内容；chatlog-to-article 的步骤 4 已涵盖相同工作流。将 'article angle' 提示作为注释整合到该技能中。"`
- 对于 **Improve**：描述需要的具体变更（哪个部分、什么操作、目标大小如相关）
  - 错误：`"太长"`
  - 正确：`"276 行；'框架比较' 部分（L80–140）与 ai-era-architecture-principles 重复；删除以达到约 150 行。"`
- 对于 **Keep**（快速扫描中仅 mtime 变更）：重述原始判定理由，不要写 "unchanged"
  - 错误：`"Unchanged"`
  - 正确：`"mtime 更新但内容未变。rules/python/ 显式导入的唯一 Python 参考；未发现重叠。"`

### Phase 3 — 汇总表

| 技能 | 7天使用 | 判定 | 原因 |
|------|---------|------|------|

### Phase 4 — 整合

1. **Retire / Merge**：在确认前向用户展示每个文件的详细理由：
   - 发现的具体问题（重叠、过时、破损引用等）
   - 什么替代方案覆盖相同功能（对于 Retire：哪个现有技能/规则；对于 Merge：目标文件和要整合的内容）
   - 删除的影响（任何依赖的技能、MEMORY.md 引用或受影响的工作流）
2. **Improve**：展示带有理由的具体改进建议：
   - 改什么以及为什么（例如，"将 430→200 行，因为 X/Y 部分与 python-patterns 重复"）
   - 用户决定是否执行
3. **Update**：展示已检查来源的更新内容
4. 检查 MEMORY.md 行数；如果 >100 行则提出压缩建议

## 结果文件模式

`~/.claude/skills/skill-stocktake/results.json`：

**`evaluated_at`**：必须设置为评估完成的实际 UTC 时间。
通过 Bash 获取：`date -u +%Y-%m-%dT%H:%M:%SZ`。永远不要使用仅日期的近似值如 `T00:00:00Z`。

```json
{
  "evaluated_at": "2026-02-21T10:00:00Z",
  "mode": "full",
  "batch_progress": {
    "total": 80,
    "evaluated": 80,
    "status": "completed"
  },
  "skills": {
    "skill-name": {
      "path": "~/.claude/skills/skill-name/SKILL.md",
      "verdict": "Keep",
      "reason": "X 工作流的具体、可操作、独特价值",
      "mtime": "2026-01-15T08:30:00Z"
    }
  }
}
```

## 注意事项

- 评估是盲目的：相同检查清单适用于所有技能，无论来源（ECC、自写、自动提取）
- 归档 / 删除操作始终需要用户明确确认
- 不因技能来源而分支判定
