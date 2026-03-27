---
name: evolve
description: 分析直觉，建议或生成演进后的结构
command: true
---

# Evolve 命令

## 实现

使用插件根路径运行直觉 CLI：

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" evolve [--generate]
```

如果未设置 `CLAUDE_PLUGIN_ROOT`（手动安装）：

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py evolve [--generate]
```

分析直觉并将相关的聚类成更高级的结构：
- **Commands**：当直觉描述用户调用的操作时
- **Skills**：当直觉描述自动触发的行为时
- **Agents**：当直觉描述复杂、多步骤的过程时

## 用法

```
/evolve                    # 分析所有直觉，建议演进
/evolve --generate         # 同时在 evolved/{skills,commands,agents} 下生成文件
```

## 演进规则

### → Command（用户调用）
当直觉描述用户会显式请求的操作时：
- 多个关于"当用户请求..."的直觉
- 带有类似"当创建新的 X"触发器的直觉
- 遵循可重复序列的直觉

示例：
- `new-table-step1`："当添加数据库表时，创建迁移"
- `new-table-step2`："当添加数据库表时，更新 schema"
- `new-table-step3`："当添加数据库表时，重新生成类型"

→ 创建：**new-table** 命令

### → Skill（自动触发）
当直觉描述应该自动发生的行为时：
- 模式匹配触发器
- 错误处理响应
- 代码风格强制

示例：
- `prefer-functional`："当编写函数时，偏好函数式风格"
- `use-immutable`："当修改状态时，使用不可变模式"
- `avoid-classes`："当设计模块时，避免基于类的设计"

→ 创建：`functional-patterns` skill

### → Agent（需要深度/隔离）
当直觉描述受益于隔离的复杂、多步骤过程时：
- 调试工作流
- 重构序列
- 研究任务

示例：
- `debug-step1`："当调试时，首先检查日志"
- `debug-step2`："当调试时，隔离失败的组件"
- `debug-step3`："当调试时，创建最小复现"
- `debug-step4`："当调试时，用测试验证修复"

→ 创建：**debugger** agent

## 做什么

1. 检测当前项目上下文
2. 读取项目 + 全局直觉（项目在 ID 冲突时优先）
3. 按触发器/域模式分组直觉
4. 识别：
   - Skill 候选（带 2+ 直觉的触发器簇）
   - Command 候选（高置信度工作流直觉）
   - Agent 候选（更大、高置信度的簇）
5. 在适用时展示晋升候选（项目 → 全局）
6. 如果传递了 `--generate`，将文件写入：
   - 项目范围：`~/.claude/homunculus/projects/<project-id>/evolved/`
   - 全局回退：`~/.claude/homunculus/evolved/`

## 输出格式

```
============================================================
  演进分析 - 12 个直觉
  项目：my-app (a1b2c3d4e5f6)
  项目范围：8 | 全局：4
============================================================

高置信度直觉（>=80%）：5 个

## SKILL 候选
1. 簇："添加测试"
   直觉：3 个
   平均置信度：82%
   域：testing
   范围：project

## COMMAND 候选（2）
  /adding-tests
    来自：test-first-workflow [project]
    置信度：84%

## AGENT 候选（1）
  adding-tests-agent
    覆盖 3 个直觉
    平均置信度：82%
```

## 标志

- `--generate`：除了分析输出外，还生成演进后的文件

## 生成的文件格式

### Command
```markdown
---
name: new-table
description: 创建新的数据库表，包含迁移、schema 更新和类型生成
command: /new-table
evolved_from:
  - new-table-migration
  - update-schema
  - regenerate-types
---

# New Table 命令

[基于聚类的直觉生成的内容]

## 步骤
1. ...
2. ...
```

### Skill
```markdown
---
name: functional-patterns
description: 强制执行函数式编程模式
evolved_from:
  - prefer-functional
  - use-immutable
  - avoid-classes
---

# 函数式模式 Skill

[基于聚类的直觉生成的内容]
```

### Agent
```markdown
---
name: debugger
description: 系统化调试 agent
model: sonnet
evolved_from:
  - debug-check-logs
  - debug-isolate
  - debug-reproduce
---

# 调试器 Agent

[基于聚类的直觉生成的内容]
```
