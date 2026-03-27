---
name: agent-eval
description: 编码 Agent（Claude Code、Aider、Codex 等）在自定义任务上的头对头比较，包含通过率、成本、时间和一致性指标
origin: ECC
tools: Read, Write, Edit, Bash, Grep, Glob
---

# Agent Eval Skill

一个轻量级 CLI 工具，用于在可复现任务上头对头比较编码 Agent。每个"哪个编码 Agent 最好？"的比较都是凭感觉运行的 — 这个工具使其系统化。

## 何时启用

- 在你自己的代码库上比较编码 Agent（Claude Code、Aider、Codex 等）
- 在采用新工具或模型之前测量 Agent 性能
- 在 Agent 更新其模型或工具时运行回归检查
- 为团队生成数据支持的 Agent 选择决策

## 安装

> **注意：** 审查源代码后从其仓库安装 agent-eval。

## 核心概念

### YAML 任务定义

声明式定义任务。每个任务指定要做什么、涉及哪些文件以及如何判断成功：

```yaml
name: add-retry-logic
description: 为 HTTP 客户端添加指数退避重试
repo: ./my-project
files:
  - src/http_client.py
prompt: |
  为所有 HTTP 请求添加指数退避重试逻辑。
  最多 3 次重试。初始延迟 1s，最大延迟 30s。
judge:
  - type: pytest
    command: pytest tests/test_http_client.py -v
  - type: grep
    pattern: "exponential_backoff|retry"
    files: src/http_client.py
commit: "abc1234"  # 固定到特定提交以保证可复现性
```

### Git Worktree 隔离

每次 Agent 运行获得自己的 git worktree — 不需要 Docker。这提供可复现性隔离，使 Agent 不能相互干扰或破坏基础仓库。

### 收集的指标

| 指标 | 测量内容 |
|--------|-----------------|
| Pass rate | Agent 是否生成了通过判断器的代码？ |
| Cost | 每任务的 API 花费（当可用时） |
| Time | 完成的实际秒数 |
| Consistency | 重复运行中的通过率（如 3/3 = 100%） |

## 工作流程

### 1. 定义任务

创建一个 `tasks/` 目录，每个任务一个 YAML 文件：

```bash
mkdir tasks
# 编写任务定义（见上方模板）
```

### 2. 运行 Agent

针对你的任务执行 Agent：

```bash
agent-eval run --task tasks/add-retry-logic.yaml --agent claude-code --agent aider --runs 3
```

每次运行：
1. 从指定提交创建新的 git worktree
2. 将提示传递给 Agent
3. 运行判断器标准
4. 记录通过/失败、成本和时间

### 3. 比较结果

生成比较报告：

```bash
agent-eval report --format table
```

```
Task: add-retry-logic (各 3 次运行)
┌──────────────┬───────────┬────────┬────────┬─────────────┐
│ Agent        │ Pass Rate │ Cost   │ Time   │ Consistency │
├──────────────┼───────────┼────────┼────────┼─────────────┤
│ claude-code  │ 3/3       │ $0.12  │ 45s    │ 100%        │
│ aider        │ 2/3       │ $0.08  │ 38s    │  67%        │
└──────────────┴───────────┴────────┴────────┴─────────────┘
```

## 判断器类型

### 基于代码（确定性）

```yaml
judge:
  - type: pytest
    command: pytest tests/ -v
  - type: command
    command: npm run build
```

### 基于模式

```yaml
judge:
  - type: grep
    pattern: "class.*Retry"
    files: src/**/*.py
```

### 基于模型（LLM-as-judge）

```yaml
judge:
  - type: llm
    prompt: |
      此实现是否正确处理了指数退避？
      检查：最大重试次数、递增延迟、抖动。
```

## 最佳实践

- **从 3-5 个代表你真实工作负载的任务开始**，而不是玩具示例
- **每个 Agent 至少运行 3 次试验**以捕获方差 — Agent 是非确定性的
- **在任务 YAML 中固定提交**，以便结果在数天/数周内可复现
- **每个任务至少包含一个确定性判断器**（测试、构建）— LLM 判断器会增加噪音
- **在通过率旁边跟踪成本** — 10 倍成本的 95% Agent 可能不是正确选择
- **版本化你的任务定义** — 它们是测试装置，像代码一样对待它们

## 链接

- 仓库：[github.com/joaquinhuigomez/agent-eval](https://github.com/joaquinhuigomez/agent-eval)
