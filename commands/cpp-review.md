---
description: 全面性的 C++ 代码审查，涵盖内存安全、现代 C++ 惯用模式、并发和安全性。调用 cpp-reviewer Agent。
---

# C++ 代码审查

此命令调用 **cpp-reviewer** Agent 进行全面的 C++ 特定代码审查。

## 本命令的作用

1. **识别 C++ 更改**：通过 `git diff` 找出修改的 `.cpp`、`.hpp`、`.cc`、`.h` 文件
2. **运行静态分析**：执行 `clang-tidy` 和 `cppcheck`
3. **内存安全扫描**：检查原始 new/delete、缓冲区溢出、释放后使用
4. **并发性审查**：分析线程安全、互斥锁使用、数据竞争
5. **现代 C++ 检查**：验证代码是否遵循 C++17/20 约定和最佳实践
6. **生成报告**：按严重性对问题进行分类

## 何时使用

在以下场景使用 `/cpp-review`：
- 编写或修改 C++ 代码后
- 提交 C++ 更改前
- 审查包含 C++ 代码的 PR
- 接手新的 C++ 代码库
- 检查内存安全问题

## 审查类别

### CRITICAL（必须修复）
- 没有 RAII 的原始 `new`/`delete`
- 缓冲区溢出和释放后使用
- 没有同步的数据竞争
- 通过 `system()` 进行命令注入
- 未初始化的变量读取
- 空指针解引用

### HIGH（应当修复）
- 违反三/五/零规则
- 缺少 `std::lock_guard` / `std::scoped_lock`
- 没有适当生命周期管理的分离线程
- 使用 C 风格转换而非 `static_cast`/`dynamic_cast`
- 缺少 `const` 正确性

- 过度复杂的模板元编程
- 缺少 `[[nodiscard]]`（在重要返回值上）

### MEDIUM（需要考虑）
- 不必要的拷贝（传值而非 `const&`）
- 已知大小容器缺少 `reserve()`
- 头文件中的 `using namespace std;`
- 过于复杂的模板元编程

## 自动检查运行

```bash
# 静态分析
clang-tidy --checks='*,-llvmlibc-*' src/*.cpp -- -std=c++17

# 额外分析
cppcheck --enable=all --suppress=missingIncludeSystem src/

# 带警告构建
cmake --build build -- -Wall -Wextra -Wpedantic
```

## 示例用法

```text
用户: /cpp-review

Agent:
# C++ 代码审查报告

## 已审查的文件
- src/handler/user.cpp（已修改）
- src/service/auth.cpp（已修改）

## 静态分析结果
✓ clang-tidy: 2 个警告
✓ cppcheck: 无问题

## 发现的问题

[CRITICAL] 内存泄漏
文件: src/service/auth.cpp:45
问题: 没有 `delete` 匹配的原始 `new`
```cpp
auto* session = new Session(userId);  // 内存泄漏！
cache[userId] = session;
```
修复: 使用 `std::unique_ptr`
```cpp
auto session = std::make_unique<Session>(userId);
cache[userId] = std::move(session);
```

[HIGH] 缺少 const 引用
文件: src/handler/user.cpp:28
问题: 大对象按值传递
```cpp
void processUser(User user) {  // 不必要的拷贝
```
修复: 按 const 引用传递
```cpp
void processUser(const User& user) {
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

- 先使用 `/cpp-test` 确保测试通过
- 如果发生构建错误，使用 `/cpp-build`
- 在提交前使用 `/cpp-review`
- 对于非 C++ 特定问题，使用 `/code-review`

## 相关

- Agent: `agents/cpp-reviewer.md`
- Skills: `skills/cpp-coding-standards/`、`skills/cpp-testing/`
