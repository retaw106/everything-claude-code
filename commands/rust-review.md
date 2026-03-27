---
description: 全面性的 Rust 代码审查，涵盖所有权、生命周期、错误处理、unsafe 使用和惯用模式。调用 rust-reviewer Agent。
---

# Rust 代码审查

此命令调用 **rust-reviewer** Agent 进行全面的 Rust 特定代码审查。

## 本命令的作用

1. **验证自动检查**：运行 `cargo check`、`cargo clippy -- -D warnings`、`cargo fmt --check` 和 `cargo test` — 如有失败则停止
2. **识别 Rust 更改**：通过 `git diff HEAD~1`（或 PR 使用 `git diff main...HEAD`）找出修改的 `.rs` 文件
3. **运行安全审计**：如果可用，执行 `cargo audit`
4. **安全扫描**：检查 unsafe 使用、命令注入、硬编码机密
5. **所有权审查**：分析不必要的 clone、生命周期问题、借用模式
6. **生成报告**：按严重性对问题进行分类

## 何时使用

在以下场景使用 `/rust-review`：
- 编写或修改 Rust 代码后
- 提交 Rust 更改前
- 审查包含 Rust 代码的 PR
- 接手新的 Rust 代码库
- 学习惯用的 Rust 模式

## 审查类别

### CRITICAL（必须修复）
- 生产代码路径中未检查的 `unwrap()`/`expect()`
- 没有 `// SAFETY:` 注释记录不变量的 `unsafe`
- 查询中通过字符串插值进行的 SQL 注入
- `std::process::Command` 中通过未验证输入进行的命令注入
- 硬编码凭据
- 通过原始指针的释放后使用

### HIGH（应当修复）
- 为满足借用检查器而进行的不必要 `.clone()`
- 可以使用 `&str` 或 `impl AsRef<str>` 时使用 `String` 参数
- 在 async 上下文中阻塞（`std::thread::sleep`、`std::fs`）
- 共享类型上缺少 `Send`/`Sync` 约束
- 业务关键枚举上的通配符 `_ =>` match
- 大函数（>50 行）

### MEDIUM（需要考虑）
- 热路径中不必要的分配
- 已知大小时缺少 `with_capacity`
- 没有正当理由抑制 clippy 警告
- 公共 API 没有 `///` 文档
- 在忽略值可能是 Bug 的非 `must_use` 返回类型上考虑 `#[must_use]`

## 自动检查运行

```bash
# 构建门槛（审查前必须通过）
cargo check

# Lint 和建议
cargo clippy -- -D warnings

# 格式化
cargo fmt --check

# 测试
cargo test

# 安全审计（如果可用）
if command -v cargo-audit >/dev/null; then cargo audit; else echo "cargo-audit not installed"; fi
```

## 示例用法

```text
用户: /rust-review

Agent:
# Rust 代码审查报告

## 已审查的文件
- src/service/user.rs（已修改）
- src/handler/api.rs（已修改）

## 静态分析结果
- 构建：成功
- Clippy：无警告
- 格式化：通过
- 测试：全部通过

## 发现的问题

[CRITICAL] 生产路径中未检查的 unwrap
文件：src/service/user.rs:28
问题：在数据库查询结果上使用 `.unwrap()`
```rust
let user = db.find_by_id(id).unwrap();  // 缺少用户时会 panic
```
修复：带上下文传播错误
```rust
let user = db.find_by_id(id)
    .context("failed to fetch user")?;
```

[HIGH] 不必要的 Clone
文件：src/handler/api.rs:45
问题：为满足借用检查器而克隆 String
```rust
let name = user.name.clone();
process(&user, &name);
```
修复：重构以避免 clone
```rust
let result = process_name(&user.name);
use_user(&user, result);
```

## 摘要
- CRITICAL: 1
- HIGH: 1
- MEDIUM: 0

建议：在 CRITICAL 问题修复前阻止合并
```

## 审批标准

| 状态 | 条件 |
|------|------|
| 批准 | 无 CRITICAL 或 HIGH 问题 |
| 警告 | 只有 MEDIUM 问题（谨慎合并） |
| 阻止 | 发现 CRITICAL 或 HIGH 问题 |

## 与其他命令的集成

- 先使用 `/rust-test` 确保测试通过
- 如果发生构建错误，使用 `/rust-build`
- 在提交前使用 `/rust-review`
- 对于非 Rust 特定问题，使用 `/code-review`

## 相关

- Agent: `agents/rust-reviewer.md`
- Skills: `skills/rust-patterns/`、`skills/rust-testing/`
