---
description: 顺序和 tmux/worktree 编排指导，用于多 agent 工作流。
---

# 编排命令

用于复杂任务的顺序 agent 工作流。

## 用法

`/orchestrate [工作流类型] [任务描述]`

## 工作流类型

### feature（功能）
完整的功能实现工作流：
```
planner -> tdd-guide -> code-reviewer -> security-reviewer
```

### bugfix（Bug 修复）
Bug 调查和修复工作流：
```
planner -> tdd-guide -> code-reviewer
```

### refactor（重构）
安全的重构工作流：
```
architect -> code-reviewer -> tdd-guide
```

### security（安全）
安全审查焦点：
```
security-reviewer -> code-reviewer -> architect
```

## 执行模式

对于工作流中的每个 agent：

1. **调用 agent**，携带上一个 agent 的上下文
2. **收集输出**作为结构化的交接文档
3. **传递给下一个**链中的 agent
4. **聚合结果**到最终报告

## 交接文档格式

在 agent 之间，创建交接文档：

```markdown
## HANDOFF: [上一个-agent] -> [下一个-agent]

### 上下文
[已完成的摘要]

### 发现
[关键发现或决策]

### 已修改的文件
[触及的文件列表]

### 未解决的问题
[下一个 agent 的未解决项]

### 建议
[建议的后续步骤]
```

## 示例：功能工作流

```
/orchestrate feature "添加用户认证"
```

执行：

1. **Planner Agent**
   - 分析需求
   - 创建实现计划
   - 识别依赖
   - 输出：`HANDOFF: planner -> tdd-guide`

2. **TDD Guide Agent**
   - 读取 planner 交接
   - 先编写测试
   - 实现以通过测试
   - 输出：`HANDOFF: tdd-guide -> code-reviewer`

3. **Code Reviewer Agent**
   - 审查实现
   - 检查问题
   - 建议改进
   - 输出：`HANDOFF: code-reviewer -> security-reviewer`

4. **Security Reviewer Agent**
   - 安全审计
   - 漏洞检查
   - 最终批准
   - 输出：最终报告

## 最终报告格式

```
编排报告
===================
工作流：feature
任务：添加用户认证
Agents：planner -> tdd-guide -> code-reviewer -> security-reviewer

摘要
-------
[一段摘要]

AGENT 输出
-------------
Planner: [摘要]
TDD Guide: [摘要]
Code Reviewer: [摘要]
Security Reviewer: [摘要]

已更改的文件
-------------
[列出所有修改的文件]

测试结果
------------
[测试通过/失败摘要]

安全状态
---------------
[安全发现]

建议
--------------
[可发布 / 需要工作 / 被阻止]
```

## 并行执行

对于独立检查，并行运行 agents：

```markdown
### 并行阶段
同时运行：
- code-reviewer（质量）
- security-reviewer（安全）
- architect（设计）

### 合并结果
将输出合并为单个报告
```

对于使用独立 git worktrees 的外部 tmux-pane workers，使用 `node scripts/orchestrate-worktrees.js plan.json --execute`。内置编排模式保持进程内；helper 用于长时间运行或跨 harness 会话。

当 workers 需要从主 checkout 看到脏或未跟踪的本地文件时，在计划文件中添加 `seedPaths`。ECC 仅将那些选定的路径覆盖到每个 worker worktree 中，在保持分支隔离的同时仍然暴露进行中的本地脚本、计划或文档。

```json
{
  "sessionName": "workflow-e2e",
  "seedPaths": [
    "scripts/orchestrate-worktrees.js",
    "scripts/lib/tmux-worktree-orchestrator.js",
    ".claude/plan/workflow-e2e-test.json"
  ],
  "workers": [
    { "name": "docs", "task": "更新编排文档。" }
  ]
}
```

要为实时 tmux/worktree 会话导出控制平面快照，运行：

```bash
node scripts/orchestration-status.js .claude/plan/workflow-visual-proof.json
```

快照以 JSON 形式包含会话活动、tmux pane 元数据、worker 状态、目标、种子覆盖和最近的交接摘要。

## Operator Command-Center 交接

当工作流跨越多个会话、worktrees 或 tmux panes 时，在最终交接中附加控制平面块：

```markdown
控制平面
-------------
会话：
- 活动会话 ID 或别名
- 每个活动 worker 的分支 + worktree 路径
- 适用的 tmux pane 或分离会话名称

差异：
- git 状态摘要
- 触及文件的 git diff --stat
- 合并/冲突风险说明

审批：
- 待处理的用户审批
- 等待确认的阻塞步骤

遥测：
- 最后活动时间戳或空闲信号
- 估算的 token 或成本漂移
- hooks 或 reviewers 提出的策略事件
```

这使得 planner、implementer、reviewer 和 loop workers 在 operator 表面上可见。

## 参数

$ARGUMENTS:
- `feature <描述>` - 完整功能工作流
- `bugfix <描述>` - Bug 修复工作流
- `refactor <描述>` - 重构工作流
- `security <描述>` - 安全审查工作流
- `custom <agents> <描述>` - 自定义 agent 序列

## 自定义工作流示例

```
/orchestrate custom "architect,tdd-guide,code-reviewer" "重新设计缓存层"
```

## 提示

1. 对于复杂功能，**从 planner 开始**
2. 合并前**始终包含 code-reviewer**
3. 对于 auth/payment/PII，**使用 security-reviewer**
4. **保持交接简洁** - 关注下一个 agent 需要什么
5. 如有需要，在 agents 之间**运行验证**
