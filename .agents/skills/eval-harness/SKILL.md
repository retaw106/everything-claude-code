---
name: eval-harness
description: 实现 eval-driven development (EDD) 原则的 Claude Code 会话正式评估框架
origin: ECC
tools: Read, Write, Edit, Bash, Grep, Glob
---

# Eval Harness 技能

实现 eval-driven development (EDD) 原则的 Claude Code 会话正式评估框架。

## 何时启用

- 为 AI 辅助工作流设置 eval-driven development (EDD)
- 定义 Claude Code 任务完成的通过/失败标准
- 使用 pass@k 指标测量代理可靠性
- 为提示或代理变更创建回归测试套件
- 跨模型版本对代理性能进行基准测试

## 理念

Eval-Driven Development 将 eval 视为"AI 开发的单元测试"：
- 在实现之前定义预期行为
- 在开发过程中持续运行 eval
- 跟踪每次变更的回归
- 使用 pass@k 指标测量可靠性

## Eval 类型

### 能力 Eval
测试 Claude 能否做以前做不到的事情：
```markdown
[CAPABILITY EVAL: feature-name]
Task: Claude 应该完成的任务描述
Success Criteria:
  - [ ] 标准 1
  - [ ] 标准 2
  - [ ] 标准 3
Expected Output: 预期结果的描述
```

### 回归 Eval
确保变更不会破坏现有功能：
```markdown
[REGRESSION EVAL: feature-name]
Baseline: SHA 或检查点名称
Tests:
  - existing-test-1: PASS/FAIL
  - existing-test-2: PASS/FAIL
  - existing-test-3: PASS/FAIL
Result: X/Y passed (previously Y/Y)
```

## 评分器类型

### 1. 基于代码的评分器
使用代码进行确定性检查：
```bash
# 检查文件是否包含预期模式
grep -q "export function handleAuth" src/auth.ts && echo "PASS" || echo "FAIL"

# 检查测试是否通过
npm test -- --testPathPattern="auth" && echo "PASS" || echo "FAIL"

# 检查构建是否成功
npm run build && echo "PASS" || echo "FAIL"
```

### 2. 基于模型的评分器
使用 Claude 评估开放式输出：
```markdown
[MODEL GRADER PROMPT]
评估以下代码变更：
1. 是否解决了所述问题？
2. 结构是否良好？
3. 是否处理了边界情况？
4. 错误处理是否适当？

Score: 1-5 (1=差, 5=优秀)
Reasoning: [解释]
```

### 3. 人工评分器
标记为需要人工审查：
```markdown
[HUMAN REVIEW REQUIRED]
Change: 变更内容描述
Reason: 为什么需要人工审查
Risk Level: LOW/MEDIUM/HIGH
```

## 指标

### pass@k
"k 次尝试中至少成功一次"
- pass@1: 第一次尝试成功率
- pass@3: 3 次尝试内的成功率
- 典型目标: pass@3 > 90%

### pass^k
"所有 k 次试验都成功"
- 更高的可靠性标准
- pass^3: 3 次连续成功
- 用于关键路径

## Eval 工作流

### 1. 定义（编码前）
```markdown
## EVAL DEFINITION: feature-xyz

### Capability Evals
1. Can create new user account
2. Can validate email format
3. Can hash password securely

### Regression Evals
1. Existing login still works
2. Session management unchanged
3. Logout flow intact

### Success Metrics
- pass@3 > 90% for capability evals
- pass^3 = 100% for regression evals
```

### 2. 实现
编写代码以通过定义的 eval。

### 3. 评估
```bash
# 运行能力 eval
[运行每个能力 eval，记录 PASS/FAIL]

# 运行回归 eval
npm test -- --testPathPattern="existing"

# 生成报告
```

### 4. 报告
```markdown
EVAL REPORT: feature-xyz
========================

Capability Evals:
  create-user:     PASS (pass@1)
  validate-email:  PASS (pass@2)
  hash-password:   PASS (pass@1)
  Overall:         3/3 passed

Regression Evals:
  login-flow:      PASS
  session-mgmt:    PASS
  logout-flow:     PASS
  Overall:         3/3 passed

Metrics:
  pass@1: 67% (2/3)
  pass@3: 100% (3/3)

Status: READY FOR REVIEW
```

## 集成模式

### 实现前
```
/eval define feature-name
```
在 `.claude/evals/feature-name.md` 创建 eval 定义文件

### 实现过程中
```
/eval check feature-name
```
运行当前 eval 并报告状态

### 实现后
```
/eval report feature-name
```
生成完整的 eval 报告

## Eval 存储

在项目中存储 eval：
```
.claude/
  evals/
    feature-xyz.md      # Eval 定义
    feature-xyz.log     # Eval 运行历史
    baseline.json       # 回归基线
```

## 最佳实践

1. **在编码前定义 eval** — 强制清晰思考成功标准
2. **频繁运行 eval** — 尽早发现回归
3. **随时间跟踪 pass@k** — 监控可靠性趋势
4. **尽可能使用代码评分器** — 确定性 > 概率性
5. **安全检查需要人工审查** — 永远不要完全自动化安全检查
6. **保持 eval 快速** — 慢的 eval 不会被运行
7. **与代码一起版本化 eval** — eval 是一等公民

## 示例：添加认证

```markdown
## EVAL: add-authentication

### Phase 1: Define (10 min)
Capability Evals:
- [ ] User can register with email/password
- [ ] User can login with valid credentials
- [ ] Invalid credentials rejected with proper error
- [ ] Sessions persist across page reloads
- [ ] Logout clears session

Regression Evals:
- [ ] Public routes still accessible
- [ ] API responses unchanged
- [ ] Database schema compatible

### Phase 2: Implement (varies)
[编写代码]

### Phase 3: Evaluate
Run: /eval check add-authentication

### Phase 4: Report
EVAL REPORT: add-authentication
==============================
Capability: 5/5 passed (pass@3: 100%)
Regression: 3/3 passed (pass^3: 100%)
Status: SHIP IT
```
