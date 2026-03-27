---
name: codebase-onboarding
description: 分析陌生代码库并生成结构化的入门指南, 包含架构图、关键入口点、约定和启动器 CLAUDE.md。 在加入新项目或首次在仓库中设置 Claude Code 时使用。
origin: ECC
---

# 代码库入门

系统分析陌生代码库并生成结构化的入门指南。 为首次加入新项目或首次在现有仓库中设置 Claude Code 的开发人员设计。

## 何时使用

- 首次使用 Claude Code 打开项目
- 加入新团队或仓库
- 用户问"帮我理解这个代码库"
- 用户要求为项目生成 CLAUDE.md
- 用户说"带我入门"或"带我过一遍这个仓库"

## 工作原理

### 阶段 1: 侦察

在不阅读每个文件的情况下收集项目的原始信号。 并行运行这些检查

```
1. 包清单检测
   → package.json, go.mod, Cargo.toml, pyproject.toml, pom.xml, build.gradle,
      Gemfile, composer.json, mix.exs, pubspec.yaml

2. 框架指纹识别
   → next.config.*, nuxt.config.*, angular.json, vite.config.*,
      django 设置, flask 应用工厂, fastapi 主, rails 配置

3. 入口点识别
   → main.*, index.*, app.*, server.*, cmd/, src/main/

4. 目录结构快照
   → 顶层 2 级目录树, 忽略 node_modules, vendor,
      .git, dist, build, __pycache__, .next

5. 配置和工具检测
   → .eslintrc*, .prettierrc*, tsconfig.json, Makefile, Dockerfile,
      docker-compose*, .github/workflows/, .env.example, CI 配置

6. 测试结构检测
   → tests/, test/, __tests__/, *_test.go, *.spec.ts, *.test.js,
      pytest.ini, jest.config.*, vitest.config.*
```

### 阶段 2: 架构映射

从侦察数据中识别

**技术栈**
- 语言和版本约束
- 框架和主要库
- 数据库和 ORM
- 构建工具和打包器
- CI/CD 平台

**架构模式**
- 单体、单体仓库、微服务或无服务器
- 前端/后端分离或全栈
- API 风格: REST、GraphQL、gRPC、tRPC

**关键目录**
映射顶级目录及其用途

<!-- React 项目示例 — 替换为检测到的目录 -->
```
src/components/  → React UI 组件
src/api/         → API 路由处理
src/lib/         → 共享工具
src/db/          → 数据库模型和迁移
tests/           → 测试套件
scripts/         → 构建和部署脚本
```

**数据流**
追踪一个请求从入口到响应
- 请求在哪里进入? (路由器、处理器、控制器)
- 如何验证? (中间件、模式、守卫)
- 业务逻辑在哪里? (服务、模型、用例)
- 如何到达数据库? (ORM、原始查询、仓库)

### 阶段 3: 约定检测

识别代码库已经遵循的模式

**命名约定**
- 文件命名: kebab-case、camelCase、PascalCase、snake_case
- 组件/类命名模式
- 测试文件命名: `*.test.ts`、`*.spec.ts`、`*_test.go`

**代码模式**
- 错误处理风格: try/catch、Result 类型、错误码
- 依赖注入风格
- 状态管理方式
- 异步模式: 回调、Promise、async/await、channel

**Git 约定**
- 从最近分支判断分支命名
- 从最近提交判断提交消息风格
- PR 工作流(squash、merge、rebase)
- 如果仓库没有提交历史, 只有一个浅的历史记录(例如 `git clone --depth 1`), 跳过此部分并注明"Git 历史太浅, 无法检测约定"

### 阶段 4: 生成入门制品

产生两个输出

#### 输出 1: 入门指南

```markdown
# 入门指南: [项目名称]

## 概述
[2-3 句话说明这个项目做什么, 服务谁]

## 技术栈
<!-- Next.js 项目示例 — 替换为检测到的技术栈 -->
| 层 | 技术 | 版本 |
|-------|-----------|---------|
| 语言 | TypeScript | 5.x |
| 框架 | Next.js | 14.x |
| 数据库 | PostgreSQL | 16 |
| ORM | Prisma | 5.x |
| 测试 | Jest + Playwright | - |

## 架构
[组件如何连接的图表或描述]

## 关键入口点
<!-- Next.js 项目示例 — 替换为检测到的路径 -->
- **API 路由**: `src/app/api/` — Next.js 路由处理
- **UI 页面**: `src/app/(dashboard)/` — 认证页面
- **数据库**: `prisma/schema.prisma` — 数据模型事实来源
- **配置**: `next.config.ts` — 构建和运行时配置

## 目录映射
[顶级目录 → 用途映射]

## 请求生命周期
[追踪一个 API 请求从入口到响应]

## 约定
- [文件命名模式]
- [错误处理方式]
- [测试模式]
- [Git 工作流]

## 常用任务
<!-- Node.js 项目示例 — 替换为检测到的命令 -->
- **运行开发服务器**: `npm run dev`
- **运行测试**: `npm test`
- **运行 linter**: `npm run lint`
- **数据库迁移**: `npx prisma migrate dev`
- **生产构建**: `npm run build`

## 去哪里找
<!-- Next.js 项目示例 — 替换为检测到的路径 -->
| 我想要... | 去哪里... |
|--------------|-----------|
| 添加 API 端点 | `src/app/api/` |
| 添加 UI 页面 | `src/app/(dashboard)/` |
| 添加数据库表 | `prisma/schema.prisma` |
| 添加测试 | `tests/` 匹配源路径 |
| 修改构建配置 | `next.config.ts` |
```

#### 输出 2: 启动器 CLAUDE.md

基于检测到的约定生成或更新项目特定的 CLAUDE.md。 如果 `CLAUDE.md` 已存在, 先读取然后增强 — 清楚标记新增或更改的内容, 保留现有项目特定的指令。

```markdown
# 项目指令

## 技术栈
[检测到的技术栈摘要]

## 代码风格
- [检测到的命名约定]
- [要遵循的检测模式]

## 测试
- 运行测试: `[检测到的测试命令]`
- 测试模式: [检测到的测试文件约定]
- 覆盖率: [如果配置了覆盖率命令]

## 构建和运行
- 开发: `[检测到的开发命令]`
- 构建: `[检测到的构建命令]`
- Lint: `[检测到的 lint 命令]`

## 项目结构
[关键目录 → 用途映射]

## 约定
- [提交风格, 如果可检测]
- [PR 工作流, 如果可检测]
- [错误处理模式]
```

## 最佳实践

1. **不要阅读所有内容** — 侦察应该使用 Glob 和 Grep, 而不是读取每个文件。 仅对模糊信号进行选择性读取。
2. **验证, 不要猜测** — 如果从配置检测到框架但实际代码使用不同的东西, 信任代码。
3. **尊重现有 CLAUDE.md** — 如果已存在, 增强而不是替换。 清楚标记新增与现有内容。
4. **保持简洁** — 入门指南应该在 2 分钟内可扫描。 细节在代码中, 不在指南中。
5. **标记未知** — 如果无法自信地检测约定, 说明而不是猜测。"无法确定测试运行器" 比错误答案好。

## 要避免的反模式

- 生成超过 100 行的 CLAUDE.md — 保持专注
- 列出每个依赖 — 只突出那些影响你编写代码方式的
- 描述明显的目录名 — `src/` 不需要解释
- 复制 README — 入门指南添加 README 缺少的结构洞察

## 示例

### 示例 1: 首次在新仓库中
**用户**: "带我入门这个代码库"
**操作**: 运行完整的 4 阶段工作流 → 生成入门指南 + 启动器 CLAUDE.md
**输出**: 入门指南直接打印到对话中, 加上写入项目根目录的 `CLAUDE.md`

### 示例 2: 为现有项目生成 CLAUDE.md
**用户**: "为这个项目生成一个 CLAUDE.md"
**操作**: 运行阶段 1-3, 跳过入门指南, 只产生 CLAUDE.md
**输出**: 带有检测约定的项目特定 `CLAUDE.md`

### 示例 3: 增强现有 CLAUDE.md
**用户**: "用当前项目约定更新 CLAUDE.md"
**操作**: 读取现有 CLAUDE.md, 运行阶段 1-3, 合并新发现
**输出**: 清楚标记新增内容的更新 `CLAUDE.md`
