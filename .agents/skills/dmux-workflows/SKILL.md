---
name: dmux-workflows
description: 使用 dmux（AI 代理的 tmux 窗格管理器）进行多代理编排。在 Claude Code、Codex、OpenCode 和其他 harness 之间的并行代理工作流模式。当并行运行多个代理会话或协调多代理开发工作流时使用。
origin: ECC
---

# dmux 工作流

使用 dmux（代理 harness 的 tmux 窗格管理器）编排并行 AI 代理会话。

## 何时启用

- 并行运行多个代理会话
- 跨 Claude Code、Codex 和其他 harness 协调工作
- 受益于分而治之并行性的复杂任务
- 用户说"并行运行"、"拆分这个工作"、"使用 dmux"或"多代理"

## 什么是 dmux

dmux 是一个基于 tmux 的编排工具，用于管理 AI 代理窗格：
- 按 `n` 创建一个带提示的新窗格
- 按 `m` 将窗格输出合并回主会话
- 支持：Claude Code、Codex、OpenCode、Cline、Gemini、Qwen

**安装：** `npm install -g dmux` 或查看 [github.com/standardagents/dmux](https://github.com/standardagents/dmux)

## 快速开始

```bash
# 启动 dmux 会话
dmux

# 创建代理窗格（在 dmux 中按 'n'，然后输入提示）
# 窗格 1："在 src/auth/ 中实现认证中间件"
# 窗格 2："为用户服务编写测试"
# 窗格 3："更新 API 文档"

# 每个窗格运行自己的代理会话
# 按 'm' 将结果合并回来
```

## 工作流模式

### 模式 1：研究 + 实现

将研究和实现拆分为并行轨道：

```
窗格 1（研究）："研究 Node.js 中速率限制的最佳实践。
   检查当前库，比较方法，并将发现写入
   /tmp/rate-limit-research.md"

窗格 2（实现）："为我们的 Express API 实现速率限制中间件。
   从基本的令牌桶开始，研究完成后我们会优化。"

# 窗格 1 完成后，将发现合并到窗格 2 的上下文中
```

### 模式 2：多文件功能

跨独立文件并行化工作：

```
窗格 1："为计费功能创建数据库 schema 和迁移"
窗格 2："在 src/api/billing/ 中构建计费 API 端点"
窗格 3："创建计费仪表板 UI 组件"

# 合并所有内容，然后在主窗格中进行集成
```

### 模式 3：测试 + 修复循环

在一个窗格运行测试，在另一个窗格修复：

```
窗格 1（观察者）："在监视模式下运行测试套件。当测试失败时，
   总结失败原因。"

窗格 2（修复者）："根据窗格 1 的错误输出修复失败的测试"
```

### 模式 4：跨 Harness

为不同任务使用不同的 AI 工具：

```
窗格 1 (Claude Code)："审查认证模块的安全性"
窗格 2 (Codex)："重构工具函数以提高性能"
窗格 3 (Claude Code)："为结账流程编写 E2E 测试"
```

### 模式 5：代码审查流水线

并行审查视角：

```
窗格 1："审查 src/api/ 的安全漏洞"
窗格 2："审查 src/api/ 的性能问题"
窗格 3："审查 src/api/ 的测试覆盖率差距"

# 将所有审查合并到单个报告中
```

## 最佳实践

1. **仅独立任务。** 不要并行化依赖彼此输出的任务。
2. **清晰边界。** 每个窗格应该处理不同的文件或关注点。
3. **战略性合并。** 在合并前审查窗格输出以避免冲突。
4. **使用 git worktrees。** 对于容易产生文件冲突的工作，每个窗格使用独立的 worktree。
5. **资源意识。** 每个窗格使用 API token — 保持总窗格数在 5-6 以下。

## Git Worktree 集成

对于会触及重叠文件的任务：

```bash
# 为隔离创建 worktrees
git worktree add ../feature-auth feat/auth
git worktree add ../feature-billing feat/billing

# 在独立的 worktrees 中运行代理
# 窗格 1: cd ../feature-auth && claude
# 窗格 2: cd ../feature-billing && claude

# 完成后合并分支
git merge feat/auth
git merge feat/billing
```

## 互补工具

| 工具 | 功能 | 何时使用 |
|------|------|----------|
| **dmux** | 代理的 tmux 窗格管理 | 并行代理会话 |
| **Superset** | 10+ 并行代理的终端 IDE | 大规模编排 |
| **Claude Code Task 工具** | 进程内子代理生成 | 会话内的程序化并行 |
| **Codex 多代理** | 内置代理角色 | Codex 特定的并行工作 |

## 故障排除

- **窗格无响应：** 检查代理会话是否在等待输入。使用 `m` 读取输出。
- **合并冲突：** 使用 git worktrees 隔离每个窗格的文件更改。
- **Token 使用量高：** 减少并行窗格数量。每个窗格都是完整的代理会话。
- **tmux 未找到：** 使用 `brew install tmux` (macOS) 或 `apt install tmux` (Linux) 安装。
