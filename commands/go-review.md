---
description: 全面性的 Go 语言代码审查，聚焦惯用模式、并发安全、错误处理与安全性。调用 go-reviewer Agent。
---

# Go 代码审查

此命令通过 go-reviewer Agent 进行对 Go 语言代码的全面审查。

## 本命令的作用

1. **识别 Go 变更**：通过 `git diff` 找出修改的 `.go` 文件
2. **运行静态分析**：执行 `go vet`、`staticcheck` 以及 `golangci-lint`
3. **安全性扫描**：检查 SQL 注入、命令注入、硬编码凭据
4. **并发性审查**：分析 goroutine 安全、channel 使用、互斥模式
5. **惯用 Go 检查**：验证代码是否遵循 Go 的约定与最佳实践
6. **生成报告**：按严重性对问题进行分类

## 何时使用

在下列场景使用 `/go-review`：
- 编写或修改 Go 代码后
- 提交 Go 代码前
- 审查包含 Go 代码的 PR
- 接手新的 Go 代码库时
- 学习 Go 的惯用模式

## 审查类别

### CRITICAL (Must Fix) / 必须修复
- SQL/命令注入漏洞
- 无同步的竞争条件
- Goroutine 泄漏
- 硬编码凭据
- 不安全的指针使用
- 在关键路径忽略错误

### HIGH (Should Fix) / 应当修复
- 缺少带上下文的错误包装
- 使用 panic 而非错误返回
- 上下文未向下传递
- 未缓冲的 channel 导致死锁
- 接口未实现错误
- 缺少互斥保护

### MEDIUM (Consider) / 需要考虑
- 非惯用的代码模式
- 导出函数缺少 godoc 注释
- 字符串拼接效率低
- 列表未预分配
- 表驱动测试未使用

## 自动检查运行

```bash
# 静态分析
go vet ./...

# 高级检查（若已安装）
staticcheck ./...
golangci-lint run

# 竞态检测
go build -race ./...

# 安全漏洞
govulncheck ./...
```

## 示例用法

```text
用户: /go-review

Agent:
# Go 代码审查报告

## 已审查的文件
- internal/handler/user.go（已修改）
- internal/service/auth.go（已修改）

## 静态分析结果
✓ go vet: 无问题
✓ staticcheck: 无问题

## 发现的问题

[CRITICAL] 竞态条件
文件：internal/service/auth.go:45
问题：共享 map 在没有同步的情况下被访问
```go
var cache = map[string]*Session{}  // 并发访问！

func GetSession(id string) *Session {
    return cache[id]  // 竞态条件
}
```
修复：使用 sync.RWMutex 或 sync.Map
```go
var (
    cache   = map[string]*Session{}
    cacheMu sync.RWMutex
)

func GetSession(id string) *Session {
    cacheMu.RLock()
    defer cacheMu.RUnlock()
    return cache[id]
}
```

[HIGH] 缺少错误上下文
文件：internal/handler/user.go:28
问题：返回的错误没有上下文
```go
return err  // 无上下文
```
修复：用上下文包装
```go
return fmt.Errorf("get user %s: %w", userID, err)
```
```

## 摘要
- CRITICAL: 1
- HIGH: 1
- MEDIUM: 0

建议：❌ 在 CRITICAL 问题修复前阻止合并
```

## 审批标准

| 状态 | 条件 |
|------|------|
| ✅ 批准 | 无 CRITICAL 或 HIGH 问题 |
| ⚠️ 警告 | 只有 MEDIUM 问题（谨慎合并） |
| ❌ 阻止 | 发现 CRITICAL 或 HIGH 问题 |

## 与其他命令的集成

- 先使用 `/go-test` 确保测试通过
- 如果发生构建错误，使用 `/go-build`
- 在提交前使用 `/go-review`
- 对于非 Go 特定问题，使用 `/code-review`

## 相关

- Agent: `agents/go-reviewer.md`
- Skills: `skills/golang-patterns/`、`skills/golang-testing/`
