---
name: nanoclaw-repl
description: 操作和扩展 NanoClaw v2，ECC 基于 claude -p 构建的无依赖会话感知 REPL。
origin: ECC
---

# NanoClaw REPL

在运行或扩展 `scripts/claw.js` 时使用此技能。

## 功能

- 持久化的 Markdown 后端会话
- 使用 `/model` 切换模型
- 使用 `/load` 动态加载技能
- 使用 `/branch` 分支会话
- 使用 `/search` 跨会话搜索
- 使用 `/compact` 压缩历史
- 使用 `/export` 导出为 md/json/txt
- 使用 `/metrics` 查看会话指标

## 操作指南

1. 保持会话任务聚焦。
2. 在高风险更改前分支。
3. 在主要里程碑后压缩。
4. 在分享或归档前导出。

## 扩展规则

- 保持零外部运行时依赖
- 保持 Markdown 作为数据库的兼容性
- 保持命令处理程序确定性和本地化
