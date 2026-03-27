---
name: prune
description: 删除超过 30 天未被晋升的待定直觉
command: true
---

# 清理待定直觉

删除已自动生成但从未被审查或晋升的过期待定直觉。

## 实现

使用插件根路径运行直觉 CLI：

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" prune
```

如果未设置 `CLAUDE_PLUGIN_ROOT`（手动安装）：

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py prune
```

## 用法

```
/prune                    # 删除超过 30 天的直觉
/prune --max-age 60      # 自定义年龄阈值（天）
/prune --dry-run         # 预览而不删除
```
