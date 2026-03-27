---
name: instinct-status
description: 显示已学习的直觉（项目 + 全局）及其置信度
command: true
---

# 直觉状态命令

显示当前项目的已学习直觉以及全局直觉，按域分组。

## 实现

使用插件根路径运行直觉 CLI：

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" status
```

如果未设置 `CLAUDE_PLUGIN_ROOT`（手动安装），使用：

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py status
```

## 用法

```
/instinct-status
```

## 做什么

1. 检测当前项目上下文（git remote/path hash）
2. 从 `~/.claude/homunculus/projects/<project-id>/instincts/` 读取项目直觉
3. 从 `~/.claude/homunculus/instincts/` 读取全局直觉
4. 使用优先规则合并（当 ID 冲突时项目覆盖全局）
5. 按域分组显示，带置信度条和观察统计

## 输出格式

```
============================================================
  直觉状态 - 共 12 个
============================================================

  项目：my-app (a1b2c3d4e5f6)
  项目直觉：8 个
  全局直觉：4 个

## 项目范围（my-app）
  ### 工作流（3）
    ███████░░░  70%  grep-before-edit [project]
                触发器：当修改代码时

## 全局（适用于所有项目）
  ### 安全（2）
    █████████░  85%  validate-user-input [global]
                触发器：当处理用户输入时
```
