# Safety Guard — 防止破坏性操作

## 何时使用

- 在生产系统上工作时
- 当代理自主运行时（全自动模式）
- 当你想限制编辑到特定目录时
- 在敏感操作期间（迁移、部署、数据变更）

## 工作原理

三种保护模式：

### 模式 1：谨慎模式

在执行前拦截破坏性命令并发出警告：

```
监视模式：
- rm -rf（特别是 /、~ 或项目根目录）
- git push --force
- git reset --hard
- git checkout .（丢弃所有更改）
- DROP TABLE / DROP DATABASE
- docker system prune
- kubectl delete
- chmod 777
- sudo rm
- npm publish（意外发布）
- 任何带有 --no-verify 的命令
```

检测到时：显示命令的作用、请求确认、建议更安全的替代方案。

### 模式 2：冻结模式

将文件编辑锁定到特定目录树：

```
/safety-guard freeze src/components/
```

任何在 `src/components/` 之外的 Write/Edit 都会被阻止并给出解释。当你希望代理专注于一个区域而不触及无关代码时很有用。

### 模式 3：守护模式（谨慎 + 冻结合并）

两种保护同时激活。自主代理的最大安全保障。

```
/safety-guard guard --dir src/api/ --allow-read-all
```

代理可以读取任何内容，但只能写入 `src/api/`。破坏性命令在任何地方都被阻止。

### 解锁

```
/safety-guard off
```

## 实现

使用 PreToolUse 钩子拦截 Bash、Write、Edit 和 MultiEdit 工具调用。在允许执行前检查命令/路径是否符合活动规则。

## 集成

- 默认为 `codex -a never` 会话启用
- 与 ECC 2.0 中的可观察性风险评分配对
- 将所有阻止的操作记录到 `~/.claude/safety-guard.log`
