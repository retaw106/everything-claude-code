# Eval 命令

管理评估驱动的开发工作流。

## 用法

`/eval [define|check|report|list] [feature-name]`

## 定义 Evals

`/eval define feature-name`

创建新的 eval 定义：

1. 使用模板创建 `.claude/evals/feature-name.md`：

```markdown
## EVAL: feature-name
创建时间: $(date)

### 能力 Evals
- [ ] [能力 1 的描述]
- [ ] [能力 2 的描述]

### 回归 Evals
- [ ] [现有行为 1 仍然有效]
- [ ] [现有行为 2 仍然有效]

### 成功标准
- pass@3 > 90% 对于能力 evals
- pass^3 = 100% 对于回归 evals
```

2. 提示用户填写具体标准

## 检查 Evals

`/eval check feature-name`

运行功能的 evals：

1. 从 `.claude/evals/feature-name.md` 读取 eval 定义
2. 对于每个能力 eval：
   - 尝试验证标准
   - 记录 PASS/FAIL
   - 在 `.claude/evals/feature-name.log` 中记录尝试
3. 对于每个回归 eval：
   - 运行相关测试
   - 与基线比较
   - 记录 PASS/FAIL
4. 报告当前状态：

```
EVAL CHECK: feature-name
========================
能力: X/Y 通过
回归: X/Y 通过
状态: 进行中 / 就绪
```

## 报告 Evals

`/eval report feature-name`

生成全面的 eval 报告：

```
EVAL REPORT: feature-name
=========================
生成时间: $(date)

CAPABILITY EVALS
----------------
[eval-1]: PASS (pass@1)
[eval-2]: PASS (pass@2) - 需要重试
[eval-3]: FAIL - 见备注

REGRESSION EVALS
----------------
[test-1]: PASS
[test-2]: PASS
[test-3]: PASS

METRICS
-------
能力 pass@1: 67%
能力 pass@3: 100%
回归 pass^3: 100%

NOTES
-----
[任何问题、边界情况或观察]

RECOMMENDATION
--------------
[可以发布 / 需要工作 / 被阻止]
```

## 列出 Evals

`/eval list`

显示所有 eval 定义：

```
EVAL DEFINITIONS
================
feature-auth      [3/5 通过] 进行中
feature-search    [5/5 通过] 就绪
feature-export    [0/4 通过] 未开始
```

## 参数

$ARGUMENTS:
- `define <name>` - 创建新的 eval 定义
- `check <name>` - 运行并检查 evals
- `report <name>` - 生成完整报告
- `list` - 显示所有 evals
- `clean` - 删除旧的 eval 日志（保留最近 10 次运行）
