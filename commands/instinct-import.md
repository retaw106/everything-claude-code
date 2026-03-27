---
name: instinct-import
description: 从文件或 URL 将直觉导入到项目/全局范围
command: true
---

# 直觉导入命令

## 实现

使用插件根路径运行直觉 CLI：

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" import <file-or-url> [--dry-run] [--force] [--min-confidence 0.7] [--scope project|global]
```

如果未设置 `CLAUDE_PLUGIN_ROOT`（手动安装）：

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py import <file-or-url>
```

从本地文件路径或 HTTP(S) URL 导入直觉。

## 用法

```
/instinct-import team-instincts.yaml
/instinct-import https://github.com/org/repo/instincts.yaml
/instinct-import team-instincts.yaml --dry-run
/instinct-import team-instincts.yaml --scope global --force
```

## 做什么

1. 获取直觉文件（本地路径或 URL）
2. 解析并验证格式
3. 检查与现有直觉的重复
4. 合并或添加新直觉
5. 保存到继承的直觉目录：
   - 项目范围：`~/.claude/homunculus/projects/<project-id>/instincts/inherited/`
   - 全局范围：`~/.claude/homunculus/instincts/inherited/`

## 导入过程

```
📥 从以下位置导入直觉：team-instincts.yaml
================================================

发现 12 个待导入的直觉。

分析冲突...

## 新直觉（8）
这些将被添加：
  ✓ use-zod-validation（置信度：0.7）
  ✓ prefer-named-exports（置信度：0.65）
  ✓ test-async-functions（置信度：0.8）
  ...

## 重复直觉（3）
已有类似直觉：
  ⚠️ prefer-functional-style
     本地：0.8 置信度，12 次观察
     导入：0.7 置信度
     → 保留本地（更高置信度）

  ⚠️ test-first-workflow
     本地：0.75 置信度
     导入：0.9 置信度
     → 更新为导入（更高置信度）

导入 8 个新的，更新 1 个？
```

## 合并行为

当导入具有现有 ID 的直觉时：
- 更高置信度的导入成为更新候选
- 相等/更低置信度的导入被跳过
- 除非使用 `--force`，否则用户确认

## 来源追踪

导入的直觉标记为：
```yaml
source: inherited
scope: project
imported_from: "team-instincts.yaml"
project_id: "a1b2c3d4e5f6"
project_name: "my-project"
```

## 标志

- `--dry-run`：预览而不导入
- `--force`：跳过确认提示
- `--min-confidence <n>`：仅导入高于阈值的直觉
- `--scope <project|global>`：选择目标范围（默认：`project`）

## 输出

导入后：
```
✅ 导入完成！

添加：8 个直觉
更新：1 个直觉
跳过：3 个直觉（已存在相等/更高置信度）

新直觉保存到：~/.claude/homunculus/instincts/inherited/

运行 /instinct-status 查看所有直觉。
```
