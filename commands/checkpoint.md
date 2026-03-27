# Checkpoint 命令

在您的工作流程中创建或验证检查点。

## 用法

`/checkpoint [create|verify|list] [name]`

## 创建检查点

创建检查点时：

1. 运行 `/verify quick` 确保当前状态干净
2. 使用检查点名称创建 git stash 或 commit
3. 将检查点记录到 `.claude/checkpoints.log`：

```bash
echo "$(date +%Y-%m-%d-%H:%M) | $CHECKPOINT_NAME | $(git rev-parse --short HEAD)" >> .claude/checkpoints.log
```

4. 报告检查点已创建

## 验证检查点

根据检查点验证时：

1. 从日志读取检查点
2. 比较当前状态与检查点：
   - 自检查点以来添加的文件
   - 自检查点以来修改的文件
   - 现在与当时的测试通过率
   - 现在与当时的覆盖率

3. 报告：
```
CHECKPOINT COMPARISON: $NAME
============================
Files changed: X
Tests: +Y passed / -Z failed
Coverage: +X% / -Y%
Build: [PASS/FAIL]
```

## 列出检查点

显示所有检查点，包括：
- 名称
- 时间戳
- Git SHA
- 状态（current、behind、ahead）

## 工作流程

典型检查点流程：

```
[开始] --> /checkpoint create "feature-start"
   |
[实现] --> /checkpoint create "core-done"
   |
[测试] --> /checkpoint verify "core-done"
   |
[重构] --> /checkpoint create "refactor-done"
   |
[PR] --> /checkpoint verify "feature-start"
```

## 参数

$ARGUMENTS：
- `create <name>` - 创建命名检查点
- `verify <name>` - 根据命名检查点验证
- `list` - 显示所有检查点
- `clear` - 删除旧检查点（保留最近 5 个）
