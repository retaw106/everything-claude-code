# Harness Audit Command

运行确定性的 repository harness 审计并返回优先级评分卡。

## Usage

`/harness-audit [scope] [--format text|json]`

- `scope` (可选): `repo` (默认), `hooks`, `skills`, `commands`, `agents`
- `--format`: 输出样式 (`text` 默认, `json` 用于自动化)

## Deterministic Engine

始终运行:

```bash
node scripts/harness-audit.js <scope> --format <text|json>
```

此脚本是评分和检查的权威来源。不要创造额外的维度或临时的评分点。

评分标准版本: `2026-03-16`。

脚本计算 7 个固定类别（每个类别 `0-10` 归一化）:

1. Tool Coverage (工具覆盖)
2. Context Efficiency (上下文效率)
3. Quality Gates (质量门)
4. Memory Persistence (内存持久化)
5. Eval Coverage (评估覆盖)
6. Security Guardrails (安全防护)
7. Cost Efficiency (成本效率)

评分来源于明确的文件/规则检查，对于相同的提交是可复现的。

## Output Contract

返回:

1. `overall_score` 占 `max_score` 的分数（`repo` 为 70；范围审计的分数更小）
2. 类别评分和具体发现
3. 失败检查及确切文件路径
4. 来源于确定性输出的前 3 个操作（`top_actions`）
5. 建议下一步应用的 ECC 技能

## Checklist

- 直接使用脚本输出；不要手动重新评分。
- 如果请求 `--format json`，直接返回脚本的 JSON。
- 如果请求文本格式，总结失败的检查和顶级操作。
- 包含来自 `checks[]` 和 `top_actions[]` 的确切文件路径。

## Example Result

```text
Harness Audit (repo): 66/70
- Tool Coverage: 10/10 (10/10 pts)
- Context Efficiency: 9/10 (9/10 pts)
- Quality Gates: 10/10 (10/10 pts)

Top 3 Actions:
1) [Security Guardrails] 在 hooks/hooks.json 中添加 prompt/tool 预检安全防护。 (hooks/hooks.json)
2) [Tool Coverage] 同步 commands/harness-audit.md 和 .opencode/commands/harness-audit.md。 (.opencode/commands/harness-audit.md)
3) [Eval Coverage] 增加跨 scripts/hooks/lib 的自动化测试覆盖。 (tests/)
```

## Arguments

$ARGUMENTS:
- `repo|hooks|skills|commands|agents` (可选范围)
- `--format text|json` (可选输出格式)
