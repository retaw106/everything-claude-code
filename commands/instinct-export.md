---
name: instinct-export
description: 将直觉从项目/全局范围导出到文件
command: /instinct-export
---

# 直觉导出命令

将直觉导出为可共享的格式。适用于：
- 与队友分享
- 迁移到新机器
- 贡献项目约定

## 用法

```
/instinct-export                           # 导出所有个人直觉
/instinct-export --domain testing          # 仅导出测试直觉
/instinct-export --min-confidence 0.7      # 仅导出高置信度直觉
/instinct-export --output team-instincts.yaml
/instinct-export --scope project --output project-instincts.yaml
```

## 做什么

1. 检测当前项目上下文
2. 按选定的范围加载直觉：
   - `project`：仅当前项目
   - `global`：仅全局
   - `all`：项目 + 全局合并（默认）
3. 应用过滤器（`--domain`、`--min-confidence`）
4. 将 YAML 风格的导出写入文件（如果未提供输出路径则打印到 stdout）

## 输出格式

创建 YAML 文件：

```yaml
# 直觉导出
# 生成时间：2025-01-22
# 来源：personal
# 数量：12 个直觉

---
id: prefer-functional-style
trigger: "when writing new functions"
confidence: 0.8
domain: code-style
source: session-observation
scope: project
project_id: a1b2c3d4e5f6
project_name: my-app
---

# 偏好函数式风格

## 操作
使用函数式模式而非类。
```

## 标志

- `--domain <name>`：仅导出指定的域
- `--min-confidence <n>`：最小置信度阈值
- `--output <file>`：输出文件路径（省略时打印到 stdout）
- `--scope <project|global|all>`：导出范围（默认：`all`）
