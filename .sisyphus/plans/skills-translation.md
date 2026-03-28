# Everything Claude Code 中文化工作计划

## TL;DR

> **目标**: 将 rules/、contexts/、.agents/skills/ 目录下的 52 个文件翻译为中文
>
> **交付物**: 
> - rules/common/ (8 个文件) - 通用编码规范
> - rules/python/ (5 个文件) - Python 特定规则
> - rules/cpp/ (5 个文件) - C++ 特定规则
> - contexts/ (3 个文件) - 上下文配置
> - .agents/skills/ (29 个文件) - 技能定义
>
> **预计工作量**: 中等
> **并行执行**: 否，按顺序翻译
> **关键路径**: rules/common → rules/python → rules/cpp → contexts → skills

---

## 上下文

### 原始请求
将 Everything Claude Code 项目中的配置文件改写为中文，使 Claude Code 安装本项目后更适配编写中文项目。

### 已完成的工作
- ✅ AGENTS.md - 已翻译
- ✅ agents/ 目录 - 已翻译
- ✅ commands/ 目录 - 已翻译
- ✅ README.zh-CN.md - 已翻译

### 翻译原则
1. **技术术语**: 混合方式 - 关键术语保留英文（Repository、API、hook、middleware 等）
2. **代码注释**: 翻译为中文
3. **品牌名称**: 保留英文（Claude Code、GitHub、npm 等）
4. **配置参数**: 保留英文（如文件名、命令参数）

---

## 工作目标

### 核心目标
将 52 个英文配置文件翻译为中文，保持技术准确性和可读性。

### 具体交付物
| 目录 | 文件数 | 优先级 |
|------|--------|--------|
| rules/common/ | 8 | 高 |
| rules/python/ | 5 | 中 |
| rules/cpp/ | 5 | 中 |
| contexts/ | 3 | 中 |
| .agents/skills/ | 29 | 高 |

### 完成标准
- [ ] 所有 52 个文件翻译完成
- [ ] 技术术语处理一致
- [ ] 代码注释已翻译
- [ ] 品牌名称保留英文
- [ ] 文件结构保持不变

### 必须完成
- 翻译所有规则、上下文和技能文件

### 禁止事项
- 不要改变文件名
- 不要改变代码示例的结构
- 不要删除英文技术术语

---

## 验证策略

### 验证方法
- 人工审核翻译质量
- 检查技术术语一致性
- 确认代码注释已翻译

### QA 场景
每个文件翻译后验证：
1. 文件可正常读取
2. Markdown 格式正确
3. 技术术语处理一致
4. 代码示例完整

---

## 执行策略

### 按顺序翻译

```
阶段 1: rules/common/ (8 个文件)
├── coding-style.md
├── development-workflow.md
├── git-workflow.md
├── hooks.md
├── patterns.md
├── performance.md
├── security.md
└── testing.md

阶段 2: rules/python/ (5 个文件)
├── coding-style.md
├── hooks.md
├── patterns.md
├── security.md
└── testing.md

阶段 3: rules/cpp/ (5 个文件)
├── coding-style.md
├── hooks.md
├── patterns.md
├── security.md
└── testing.md

阶段 4: contexts/ (3 个文件)
├── dev.md
├── research.md
└── review.md

阶段 5: .agents/skills/ (29 个文件)
├── api-design/SKILL.md
├── article-writing/SKILL.md
├── backend-patterns/SKILL.md
├── bun-runtime/SKILL.md
├── claude-api/SKILL.md
├── coding-standards/SKILL.md
├── content-engine/SKILL.md
├── crosspost/SKILL.md
├── deep-research/SKILL.md
├── dmux-workflows/SKILL.md
├── documentation-lookup/SKILL.md
├── e2e-testing/SKILL.md
├── eval-harness/SKILL.md
├── everything-claude-code/SKILL.md
├── exa-search/SKILL.md
├── fal-ai-media/SKILL.md
├── frontend-patterns/SKILL.md
├── frontend-slides/SKILL.md
├── investor-materials/SKILL.md
├── investor-outreach/SKILL.md
├── market-research/SKILL.md
├── mcp-server-patterns/SKILL.md
├── nextjs-turbopack/SKILL.md
├── security-review/SKILL.md
├── strategic-compact/SKILL.md
├── tdd-workflow/SKILL.md
├── verification-loop/SKILL.md
├── video-editing/SKILL.md
└── x-api/SKILL.md
```

---

## TODOs

### 阶段 1: rules/common/ 目录 (8 个文件)

- [x] 1. 翻译 rules/common/coding-style.md

  **做什么**: 将编码风格规则翻译为中文
  - 标题翻译为 "编码风格"
  - "Immutability" 等技术术语保留，添加中文解释
  - 代码注释翻译为中文
  - 检查清单项翻译

  **不应该做**:
  - 不要改变文件结构
  - 不要翻译代码示例中的变量名

  **推荐 Agent**: `quick`
  **并行化**: 可与任务 2 并行

  **验收标准**:
  - [ ] 文件包含中文标题
  - [ ] 技术术语保留英文
  - [ ] 代码注释为中文

- [x] 2. 翻译 rules/common/development-workflow.md

  **做什么**: 将开发工作流规则翻译为中文
  - "Feature Implementation Workflow" → "功能实现工作流"
  - TDD 相关术语保留 (RED/GREEN/IMPROVE)
  - Agent 名称保留英文 (planner, tdd-guide, code-reviewer)

  **推荐 Agent**: `quick`
  **并行化**: 可与任务 1 并行

- [x] 3. 翻译 rules/common/git-workflow.md

  **做什么**: 将 Git 工作流规则翻译为中文
  - 提交类型保留英文 (feat, fix, refactor 等)
  - PR 流程描述翻译为中文

  **推荐 Agent**: `quick`
  **并行化**: 可与任务 4 并行

- [x] 4. 翻译 rules/common/hooks.md

  **做什么**: 将 Hooks 系统规则翻译为中文
  - Hook 类型名称保留英文 (PreToolUse, PostToolUse, Stop)
  - 描述翻译为中文

  **推荐 Agent**: `quick`
  **并行化**: 可与任务 3 并行

- [x] 5. 翻译 rules/common/patterns.md

  **做什么**: 将设计模式规则翻译为中文
  - "Repository Pattern" 保留，添加 "仓储模式" 解释
  - "API Response Format" → "API 响应格式"

  **推荐 Agent**: `quick`

- [x] 6. 翻译 rules/common/performance.md

  **做什么**: 将性能优化规则翻译为中文
  - 模型名称保留 (Haiku, Sonnet, Opus)
  - 性能策略描述翻译为中文

  **推荐 Agent**: `quick`

- [x] 7. 翻译 rules/common/security.md

  **做什么**: 将安全规则翻译为中文
  - 安全术语保留英文 (XSS, CSRF, SQL Injection)
  - 添加中文解释

  **推荐 Agent**: `quick`

- [x] 8. 翻译 rules/common/testing.md

  **做什么**: 将测试规则翻译为中文
  - TDD 相关术语保留 (RED, GREEN, IMPROVE)
  - 测试类型翻译 (Unit Tests → 单元测试)

  **推荐 Agent**: `quick`

### 阶段 2: rules/python/ 目录 (5 个文件)

- [x] 9. 翻译 rules/python/coding-style.md
  **推荐 Agent**: `quick`

- [x] 10. 翻译 rules/python/hooks.md
  **推荐 Agent**: `quick`

- [x] 11. 翻译 rules/python/patterns.md
  **推荐 Agent**: `quick`

- [x] 12. 翻译 rules/python/security.md
  **推荐 Agent**: `quick`

- [x] 13. 翻译 rules/python/testing.md
  **推荐 Agent**: `quick`

### 阶段 3: rules/cpp/ 目录 (5 个文件)

- [x] 14. 翻译 rules/cpp/coding-style.md
  **推荐 Agent**: `quick`

- [x] 15. 翻译 rules/cpp/hooks.md
  **推荐 Agent**: `quick`

- [x] 16. 翻译 rules/cpp/patterns.md
  **推荐 Agent**: `quick`

- [x] 17. 翻译 rules/cpp/security.md
  **推荐 Agent**: `quick`

- [x] 18. 翻译 rules/cpp/testing.md
  **推荐 Agent**: `quick`

### 阶段 4: contexts/ 目录 (3 个文件)

- [x] 19. 翻译 contexts/dev.md
  **推荐 Agent**: `quick`

- [x] 20. 翻译 contexts/research.md
  **推荐 Agent**: `quick`

- [ ] 21. 翻译 contexts/review.md
  **推荐 Agent**: `quick`

### 阶段 5: .agents/skills/ 目录 (29 个文件) - 第 1 批

- [ ] 22. 翻译 .agents/skills/api-design/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 23. 翻译 .agents/skills/article-writing/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 24. 翻译 .agents/skills/backend-patterns/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 25. 翻译 .agents/skills/bun-runtime/SKILL.md
  **推荐 Agent**: `quick`

- [ ] 26. 翻译 .agents/skills/claude-api/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 27. 翻译 .agents/skills/coding-standards/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 28. 翻译 .agents/skills/content-engine/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 29. 翻译 .agents/skills/crosspost/SKILL.md
  **推荐 Agent**: `quick`

- [ ] 30. 翻译 .agents/skills/deep-research/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 31. 翻译 .agents/skills/dmux-workflows/SKILL.md
  **推荐 Agent**: `unspecified-high`

### 阶段 6: .agents/skills/ 目录 - 第 2 批

- [ ] 32. 翻译 .agents/skills/documentation-lookup/SKILL.md
  **推荐 Agent**: `quick`

- [ ] 33. 翻译 .agents/skills/e2e-testing/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 34. 翻译 .agents/skills/eval-harness/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 35. 翻译 .agents/skills/everything-claude-code/SKILL.md
  **推荐 Agent**: `quick`

- [ ] 36. 翻译 .agents/skills/exa-search/SKILL.md
  **推荐 Agent**: `quick`

- [ ] 37. 翻译 .agents/skills/fal-ai-media/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 38. 翻译 .agents/skills/frontend-patterns/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 39. 翻译 .agents/skills/frontend-slides/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 40. 翻译 .agents/skills/investor-materials/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 41. 翻译 .agents/skills/investor-outreach/SKILL.md
  **推荐 Agent**: `unspecified-high`

### 阶段 7: .agents/skills/ 目录 - 第 3 批

- [ ] 42. 翻译 .agents/skills/market-research/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 43. 翻译 .agents/skills/mcp-server-patterns/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 44. 翻译 .agents/skills/nextjs-turbopack/SKILL.md
  **推荐 Agent**: `quick`

- [ ] 45. 翻译 .agents/skills/security-review/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 46. 翻译 .agents/skills/strategic-compact/SKILL.md
  **推荐 Agent**: `quick`

- [ ] 47. 翻译 .agents/skills/tdd-workflow/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 48. 翻译 .agents/skills/verification-loop/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 49. 翻译 .agents/skills/video-editing/SKILL.md
  **推荐 Agent**: `unspecified-high`

- [ ] 50. 翻译 .agents/skills/x-api/SKILL.md
  **推荐 Agent**: `quick`

---

## 最终验证阶段

- [ ] F1. 检查所有文件翻译完成
- [ ] F2. 验证技术术语一致性
- [ ] F3. 确认代码注释已翻译
- [ ] F4. 用户确认翻译质量

---

## 提交策略

每完成一个阶段提交一次：
- `feat(i18n): translate rules/common to Chinese`
- `feat(i18n): translate rules/python to Chinese`
- `feat(i18n): translate rules/cpp to Chinese`
- `feat(i18n): translate contexts to Chinese`
- `feat(i18n): translate skills to Chinese`

---

## 成功标准

### 验证命令
```bash
# 检查翻译进度
grep -r "何时" rules/ contexts/ .agents/skills/ | wc -l
# 预期: 52+ (每个文件至少有一个中文标题)
```

### 最终检查清单
- [ ] 所有 52 个文件已翻译
- [ ] 技术术语处理一致
- [ ] 代码注释已翻译为中文
- [ ] 品牌名称保留英文
- [ ] 文件结构保持不变
