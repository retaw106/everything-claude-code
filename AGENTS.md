# Everything Claude Code (ECC) — Agent 指令

这是一个**生产级 AI 编码插件**，提供 28 个专业化 Agent、125 个技能、60 个命令和自动化 Hook 工作流，用于软件开发。

**版本：** 1.9.0

## 核心原则

1. **Agent 优先** — 将领域任务委托给专业化 Agent
2. **测试驱动** — 在实现之前编写测试，要求 80%+ 覆盖率
3. **安全优先** — 绝不在安全上妥协；验证所有输入
4. **不可变性** — 始终创建新对象，绝不修改现有对象
5. **先计划后执行** — 在编写代码之前规划复杂功能

## 可用 Agent

| Agent | 用途 | 使用场景 |
|-------|---------|-------------|
| planner | 实现规划 | 复杂功能、重构 |
| architect | 系统设计和可扩展性 | 架构决策 |
| tdd-guide | 测试驱动开发 | 新功能、Bug 修复 |
| code-reviewer | 代码质量和可维护性 | 编写/修改代码后 |
| security-reviewer | 漏洞检测 | 提交前、敏感代码 |
| build-error-resolver | 修复构建/类型错误 | 构建失败时 |
| e2e-runner | 端到端 Playwright 测试 | 关键用户流程 |
| refactor-cleaner | 清理死代码 | 代码维护 |
| doc-updater | 文档和代码地图 | 更新文档 |
| docs-lookup | 文档和 API 参考研究 | 库/API 文档问题 |
| cpp-reviewer | C++ 代码审查 | C++ 项目 |
| cpp-build-resolver | C++ 构建错误 | C++ 构建失败 |
| go-reviewer | Go 代码审查 | Go 项目 |
| go-build-resolver | Go 构建错误 | Go 构建失败 |
| kotlin-reviewer | Kotlin 代码审查 | Kotlin/Android/KMP 项目 |
| kotlin-build-resolver | Kotlin/Gradle 构建错误 | Kotlin 构建失败 |
| database-reviewer | PostgreSQL/Supabase 专家 | 模式设计、查询优化 |
| python-reviewer | Python 代码审查 | Python 项目 |
| java-reviewer | Java 和 Spring Boot 代码审查 | Java/Spring Boot 项目 |
| java-build-resolver | Java/Maven/Gradle 构建错误 | Java 构建失败 |
| chief-of-staff | 沟通分诊和草稿 | 多渠道邮件、Slack、LINE、Messenger |
| loop-operator | 自主循环执行 | 安全运行循环、监控停滞、干预 |
| harness-optimizer | Harness 配置调优 | 可靠性、成本、吞吐量 |
| rust-reviewer | Rust 代码审查 | Rust 项目 |
| rust-build-resolver | Rust 构建错误 | Rust 构建失败 |
| pytorch-build-resolver | PyTorch 运行时/CUDA/训练错误 | PyTorch 构建/训练失败 |
| typescript-reviewer | TypeScript/JavaScript 代码审查 | TypeScript/JavaScript 项目 |

## Agent 编排

主动使用 Agent，无需用户提示：
- 复杂功能请求 → **planner**
- 刚编写/修改的代码 → **code-reviewer**
- Bug 修复或新功能 → **tdd-guide**
- 架构决策 → **architect**
- 安全敏感代码 → **security-reviewer**
- 多渠道沟通分诊 → **chief-of-staff**
- 自主循环 / 循环监控 → **loop-operator**
- Harness 配置可靠性和成本 → **harness-optimizer**

对独立操作使用并行执行 — 同时启动多个 Agent。

## 安全指南

**在 ANY 提交之前：**
- 无硬编码机密（API 密钥、密码、令牌）
- 所有用户输入已验证
- SQL 注入预防（参数化查询）
- XSS 预防（清理 HTML）
- CSRF 保护已启用
- 身份验证/授权已验证
- 所有端点都有速率限制
- 错误消息不泄露敏感数据

**密钥管理：** 绝不硬编码机密。使用环境变量或密钥管理器。在启动时验证所需的机密。立即轮换任何暴露的机密。

**如果发现安全问题：** 停止 → 使用 security-reviewer agent → 修复 CRITICAL 问题 → 轮换暴露的机密 → 审查代码库中的类似问题。

## 编码风格

**不可变性（关键）：** 始终创建新对象，绝不修改。返回应用了更改的新副本。

**文件组织：** 许多小文件优于少数大文件。典型 200-400 行，最多 800 行。按功能/领域组织，而不是按类型。高内聚，低耦合。

**错误处理：** 在每个级别处理错误。在 UI 代码中提供用户友好的消息。在服务器端记录详细上下文。绝不静默吞掉错误。

**输入验证：** 在系统边界验证所有用户输入。使用基于架构的验证。快速失败并提供清晰消息。绝不信任外部数据。

**代码质量检查清单：**
- 函数小（<50 行），文件专注（<800 行）
- 无深层嵌套（>4 层）
- 正确的错误处理，无硬编码值
- 可读的、命名良好的标识符

## 测试要求

**最低覆盖率：80%**

测试类型（全部必需）：
1. **单元测试** — 单个函数、工具、组件
2. **集成测试** — API 端点、数据库操作
3. **E2E 测试** — 关键用户流程

**TDD 工作流（强制性）：**
1. 先编写测试（RED）— 测试应该失败
2. 编写最小实现（GREEN）— 测试应该通过
3. 重构（IMPROVE）— 验证覆盖率 80%+

故障排除：检查测试隔离 → 验证模拟 → 修复实现（而不是测试，除非测试错误）。

## 开发工作流

1. **计划** — 使用 planner agent，识别依赖项和风险，分解为阶段
2. **TDD** — 使用 tdd-guide agent，先编写测试，实现，重构
3. **审查** — 立即使用 code-reviewer agent，解决 CRITICAL/HIGH 问题
4. **在正确的位置捕获知识**
   - 个人调试笔记、偏好和临时上下文 → 自动记忆
   - 团队/项目知识（架构决策、API 更改、运行手册）→ 项目现有文档结构
   - 如果当前任务已经产生相关文档或代码注释，不要在其他地方重复相同信息
   - 如果没有明显的项目文档位置，在创建新的顶级文件之前询问
5. **提交** — 约定提交格式，全面的 PR 摘要

## Git 工作流

**提交格式：** `<type>: <description>` — 类型：feat、fix、refactor、docs、test、chore、perf、ci

**PR 工作流：** 分析完整的提交历史 → 起草全面的摘要 → 包括测试计划 → 使用 `-u` 标志推送。

## 架构模式

**API 响应格式：** 一致的外壳，包含成功指示器、数据负载、错误消息和分页元数据。

**Repository 模式：** 将数据访问封装在标准接口后面（findAll、findById、create、update、delete）。业务逻辑依赖于抽象接口，而不是存储机制。

**骨架项目：** 搜索经过战斗测试的模板，使用并行 agent 评估（安全性、可扩展性、相关性），克隆最佳匹配，在已验证的结构内迭代。

## 性能

**上下文管理：** 对于大型重构和多文件功能，避免上下文窗口的最后 20%。较低敏感度的任务（单个编辑、文档、简单修复）容忍更高的利用率。

**构建故障排除：** 使用 build-error-resolver agent → 分析错误 → 增量修复 → 每次修复后验证。

## 项目结构

```
agents/          — 28 个专业化子 Agent
skills/          — 125 个工作流技能和领域知识
commands/        — 60 个斜杠命令
hooks/           — 基于触发器的自动化
rules/           — 始终遵循的指南（通用 + 每种语言）
scripts/         — 跨平台 Node.js 工具
mcp-configs/     — 14 个 MCP 服务器配置
tests/           — 测试套件
```

## 成功指标

- 所有测试通过，覆盖率为 80%+
- 无安全漏洞
- 代码可读且可维护
- 性能可接受
- 满足用户要求
