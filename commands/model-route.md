# Model Route Command

根据复杂度和预算为当前任务推荐最佳 model 层级。

## Usage

`/model-route [task-description] [--budget low|med|high]`

## Routing Heuristic

- `haiku`: 确定性、低风险的机械性更改
- `sonnet`: 实现和重构的默认选择
- `opus`: 架构、深度审查、模糊需求

## Required Output

- 推荐的 model
- 置信度
- 为何此 model 适合
- 如果首次尝试失败的备用 model

## Arguments

$ARGUMENTS:
- `[task-description]` 可选的自由文本
- `--budget low|med|high` 可选
