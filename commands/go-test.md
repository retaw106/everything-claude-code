---
description: 强制执行 Go 的 TDD 工作流。先编写表驱动测试，然后实现。使用 go test -cover 验证 80%+ 的覆盖率。
---

# Go TDD 命令

此命令使用惯用的 Go 测试模式强制执行 Go 代码的测试驱动开发方法论。

## 本命令的作用

1. **定义类型/接口**：先搭建函数签名
2. **编写表驱动测试**：创建全面的测试用例（RED）
3. **运行测试**：验证测试因正确原因失败
4. **实现代码**：编写最小代码以通过（GREEN）
5. **重构**：在测试保持绿色的同时改进代码
6. **检查覆盖率**：确保 80%+ 覆盖率

## 何时使用

在以下场景使用 `/go-test`：
- 实现新的 Go 函数
- 为现有代码添加测试覆盖
- 修复 Bug（先编写失败的测试）
- 构建关键业务逻辑
- 学习 Go 中的 TDD 工作流

## TDD 循环

```
RED     → 编写失败的表驱动测试
GREEN   → 实现最小代码以通过
REFACTOR → 改进代码，测试保持绿色
REPEAT  → 下一个测试用例
```

## 示例会话

````
用户: /go-test 我需要一个验证电子邮件地址的函数

Agent:
# TDD 会话：电子邮件验证器

## 步骤 1：定义接口

```go
// validator/email.go
package validator

// ValidateEmail 检查给定字符串是否为有效的电子邮件地址。
// 如果有效则返回 nil，否则返回描述问题的错误。
func ValidateEmail(email string) error {
    panic("not implemented")
}
```

## 步骤 2：编写表驱动测试（RED）

```go
// validator/email_test.go
package validator

import (
    "testing"
)

func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name    string
        email   string
        wantErr bool
    }{
        // 有效的电子邮件
        {"simple email", "user@example.com", false},
        {"with subdomain", "user@mail.example.com", false},
        {"with plus", "user+tag@example.com", false},
        {"with dots", "first.last@example.com", false},

        // 无效的电子邮件
        {"empty string", "", true},
        {"no at sign", "userexample.com", true},
        {"no domain", "user@", true},
        {"no local part", "@example.com", true},
        {"double at", "user@@example.com", true},
        {"spaces", "user @example.com", true},
        {"no tld", "user@example", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ValidateEmail(tt.email)
            if tt.wantErr && err == nil {
                t.Errorf("ValidateEmail(%q) = nil; want error", tt.email)
            }
            if !tt.wantErr && err != nil {
                t.Errorf("ValidateEmail(%q) = %v; want nil", tt.email, err)
            }
        })
    }
}
```

## 步骤 3：运行测试 - 验证失败

```bash
$ go test ./validator/...

--- FAIL: TestValidateEmail (0.00s)
    --- FAIL: TestValidateEmail/simple_email (0.00s)
        panic: not implemented

FAIL
```

✓ 测试按预期失败（panic）。

## 步骤 4：实现最小代码（GREEN）

```go
// validator/email.go
package validator

import (
    "errors"
    "regexp"
)

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

var (
    ErrEmailEmpty   = errors.New("email cannot be empty")
    ErrEmailInvalid = errors.New("email format is invalid")
)

func ValidateEmail(email string) error {
    if email == "" {
        return ErrEmailEmpty
    }
    if !emailRegex.MatchString(email) {
        return ErrEmailInvalid
    }
    return nil
}
```

## 步骤 5：运行测试 - 验证通过

```bash
$ go test ./validator/...

PASS
ok      project/validator    0.003s
```

✓ 所有测试通过！

## 步骤 6：检查覆盖率

```bash
$ go test -cover ./validator/...

PASS
coverage: 100.0% of statements
ok      project/validator    0.003s
```

✓ 覆盖率: 100%

## TDD 完成！
````

## 测试模式

### 表驱动测试
```go
tests := []struct {
    name     string
    input    InputType
    want     OutputType
    wantErr  bool
}{
    {"case 1", input1, want1, false},
    {"case 2", input2, want2, true},
}

for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got, err := Function(tt.input)
        // 断言
    })
}
```

### 并行测试
```go
for _, tt := range tests {
    tt := tt // 捕获
    t.Run(tt.name, func(t *testing.T) {
        t.Parallel()
        // 测试体
    })
}
```

### 测试辅助函数
```go
func setupTestDB(t *testing.T) *sql.DB {
    t.Helper()
    db := createDB()
    t.Cleanup(func() { db.Close() })
    return db
}
```

## 覆盖率命令

```bash
# 基本覆盖率
go test -cover ./...

# 覆盖率配置文件
go test -coverprofile=coverage.out ./...

# 在浏览器中查看
go tool cover -html=coverage.out

# 按函数的覆盖率
go tool cover -func=coverage.out

# 带竞态检测
go test -race -cover ./...
```

## 覆盖率目标

| 代码类型 | 目标 |
|----------|------|
| 关键业务逻辑 | 100% |
| 公共 API | 90%+ |
| 常规代码 | 80%+ |
| 生成的代码 | 排除 |

## TDD 最佳实践

**要做的：**
- 在任何实现之前**先编写测试**
- 每次更改后运行测试
- 使用表驱动测试进行全面的覆盖
- 测试行为，而非实现细节
- 包含边界情况（空值、nil、最大值）

**不要做的：**
- 在测试之前编写实现
- 跳过 RED 阶段
- 直接测试私有函数
- 在测试中使用 `time.Sleep`
- 忽略不稳定的测试

## 相关命令

- `/go-build` - 修复构建错误
- `/go-review` - 实现后审查代码
- `/verify` - 运行完整验证循环

## 相关

- Skill: `skills/golang-testing/`
- Skill: `skills/tdd-workflow/`
