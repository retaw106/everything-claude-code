---
description: 通过 Claude DevFleet 编排并行 Claude Code agents — 从自然语言规划项目，在隔离的 worktrees 中调度 agents，监控进度，并阅读结构化报告。
---

# DevFleet — 多 Agent 编排

通过 Claude DevFleet 编排并行 Claude Code agents。每个 agent 在隔离的 git worktree 中运行，拥有完整的工具集。

需要 DevFleet MCP server：`claude mcp add devfleet --transport http http://localhost:18801/mcp`

## 流程

```
用户描述项目
  → plan_project(prompt) → 带依赖关系的任务 DAG
  → 展示计划，获取批准
  → dispatch_mission(M1) → Agent 在 worktree 中启动
  → M1 完成 → 自动合并 → M2 自动调度（依赖 M1）
  → M2 完成 → 自动合并
  → get_report(M2) → files_changed、what_done、errors、next_steps
  → 向用户报告摘要
```

## 工作流

1. **规划项目** 从用户的描述：

```
mcp__devfleet__plan_project(prompt="<用户的描述>")
```

这将返回带有链式任务的项目。向用户展示：
- 项目名称和 ID
- 每个任务：标题、类型、依赖关系
- 依赖关系 DAG（哪些任务阻塞哪些）

2. **等待用户批准** 在调度前。清晰展示计划。

3. **调度第一个任务**（`depends_on` 为空的那个）：

```
mcp__devfleet__dispatch_mission(mission_id="<第一个任务 ID>")
```

剩余任务在其依赖项完成时自动调度（因为 `plan_project` 以 `auto_dispatch=true` 创建它们）。当使用 `create_mission` 手动创建任务时，必须显式设置 `auto_dispatch=true` 才能获得此行为。

4. **监控进度** — 检查运行状态：

```
mcp__devfleet__get_dashboard()
```

或检查特定任务：

```
mcp__devfleet__get_mission_status(mission_id="<id>")
```

对于长时间运行的任务，优先使用 `get_mission_status` 轮询而非 `wait_for_mission`，以便用户看到进度更新。

5. **阅读报告** 为每个完成的任务：

```
mcp__devfleet__get_report(mission_id="<任务 ID>")
```

为每个达到终端状态的任务调用此方法。报告包含：files_changed、what_done、what_open、what_tested、what_untested、next_steps、errors_encountered。

## 所有可用工具

| 工具 | 用途 |
|------|------|
| `plan_project(prompt)` | AI 将描述分解为带 `auto_dispatch=true` 的链式任务 |
| `create_project(name, path?, description?)` | 手动创建项目，返回 `project_id` |
| `create_mission(project_id, title, prompt, depends_on?, auto_dispatch?)` | 添加任务。`depends_on` 是任务 ID 字符串列表。 |
| `dispatch_mission(mission_id, model?, max_turns?)` | 启动 agent |
| `cancel_mission(mission_id)` | 停止运行中的 agent |
| `wait_for_mission(mission_id, timeout_seconds?)` | 阻塞直到完成（长时间任务优先轮询） |
| `get_mission_status(mission_id)` | 检查进度而不阻塞 |
| `get_report(mission_id)` | 阅读结构化报告 |
| `get_dashboard()` | 系统概览 |
| `list_projects()` | 浏览项目 |
| `list_missions(project_id, status?)` | 列出任务 |

## 指南

- 除非用户说"继续"，否则在调度前始终确认计划
- 报告状态时包含任务标题和 ID
- 如果任务失败，在重试前阅读其报告以了解错误
- Agent 并发是可配置的（默认：3）。多余的任务排队，并在槽位空闲时自动调度。检查 `get_dashboard()` 了解槽位可用性。
- 依赖关系形成 DAG — 永远不要创建循环依赖
- 每个 agent 完成后自动合并其 worktree。如果发生合并冲突，更改保留在 worktree 分支中供手动解决。
