---
description: 强制执行 C++ 的 TDD 工作流。首先编写 GoogleTest 测试，然后实现。使用 gcov/lcov 验证覆盖率。
---

# C++ TDD 命令

此命令使用 GoogleTest/GoogleMock 和 CMake/CTest 强制执行 C++ 代码的测试驱动开发方法论。

## 本命令的作用

1. **定义接口**：首先搭建类/函数签名
2. **编写测试**：创建全面的 GoogleTest 测试用例（RED）
3. **运行测试**：验证测试因正确原因失败
4. **实现代码**：编写最小代码以通过（GREEN）
5. **重构**：在测试保持绿色的同时改进代码
6. **检查覆盖率**：确保 80%+ 覆盖率

## 何时使用

在以下场景使用 `/cpp-test`：
- 实现新的 C++ 函数或类
- 为现有代码添加测试覆盖
- 修复 Bug（首先编写失败的测试）
- 构建关键业务逻辑
- 学习 C++ 中的 TDD 工作流

## TDD 循环

```
RED     → 编写失败的 GoogleTest 测试
GREEN   → 实现最小代码以通过
REFACTOR → 改进代码，测试保持绿色
REPEAT  → 下一个测试用例
```

## 示例会话

````
用户: /cpp-test 我需要一个验证电子邮件地址的函数

Agent:
# TDD 会话：电子邮件验证器

## 步骤 1：定义接口

```cpp
// validator/email.hpp
#pragma once
#include <string>
#include <expected>

enum class EmailError {
    Empty,
    InvalidFormat
};

std::expected<void, EmailError> validate_email(const std::string& email);
```

## 步骤 2：编写测试（RED）

```cpp
// validator/email_test.cpp
#include <gtest/gtest.h>
#include "email.hpp"

TEST(ValidateEmail, AcceptsSimpleEmail) {
    auto result = validate_email("user@example.com");
    EXPECT_TRUE(result.has_value());
}

TEST(ValidateEmail, AcceptsSubdomain) {
    EXPECT_TRUE(validate_email("user@mail.example.com").has_value());
}

TEST(ValidateEmail, AcceptsPlus) {
    EXPECT_TRUE(validate_email("user+tag@example.com").has_value());
}

TEST(ValidateEmail, RejectsEmpty) {
    auto result = validate_email("");
    ASSERT_FALSE(result.has_value());
    EXPECT_EQ(result.error(), EmailError::Empty);
}

TEST(ValidateEmail, RejectsNoAtSign) {
    EXPECT_FALSE(validate_email("userexample.com").has_value());
}

TEST(ValidateEmail, RejectsNoDomain) {
    EXPECT_FALSE(validate_email("user@").has_value());
}

TEST(ValidateEmail, RejectsNoLocalPart) {
    EXPECT_FALSE(validate_email("@example.com").has_value());
}
```

## 步骤 3：运行测试 - 验证失败

```bash
$ cmake --build build && ctest --test-dir build --output-on-failure

1/1 Test #1: email_validator_test .....***失败
    --- undefined reference to `validate_email`

失败
```

✓ 测试按预期失败（未实现）。

## 步骤 4：实现最小代码（GREEN）

```cpp
// validator/email.cpp
#include "email.hpp"
#include <regex>

std::expected<void, EmailError> validate_email(const std::string& email) {
    if (email.empty()) {
        return std::unexpected(EmailError::Empty);
    }
    static const std::regex pattern(R"([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})");
    if (!std::regex_match(email, pattern)) {
        return std::unexpected(EmailError::InvalidFormat);
    }
    return {};
}
```

## 步骤 5：运行测试 - 验证通过

```bash
$ cmake --build build && ctest --test-dir build --output-on-failure

1/1 Test #1: email_validator_test .....   通过    0.01 秒

100% 测试通过。
```

✓ 所有测试通过！

## 步骤 6：检查覆盖率

```bash
$ cmake -DCMAKE_CXX_FLAGS="--coverage" -B build && cmake --build build
$ ctest --test-dir build
$ lcov --capture --directory build --output-file coverage.info
$ lcov --list coverage.info

validator/email.cpp     | 100%
```

✓ 覆盖率：100%

## TDD 完成！````

## 测试模式

### 基本测试
```cpp
TEST(SuiteName, TestName) {
    EXPECT_EQ(add(2, 3), 5);
    EXPECT_NE(result, nullptr);
    EXPECT_TRUE(is_valid);
    EXPECT_THROW(func(), std::invalid_argument);
}
```

### Fixtures
```cpp
class DatabaseTest : public ::testing::Test {
protected:
    void SetUp() override { db_ = create_test_db(); }
    void TearDown() override { db_.reset(); }
    std::unique_ptr<Database> db_;
};

TEST_F(DatabaseTest, InsertsRecord) {
    db_->insert("key", "value");
    EXPECT_EQ(db_->get("key"), "value");
}
```

### 参数化测试
```cpp
class PrimeTest : public ::testing::TestWithParam<std::pair<int, bool>> {};

TEST_P(PrimeTest, ChecksPrimality) {
    auto [input, expected] = GetParam();
    EXPECT_EQ(is_prime(input), expected);
}

INSTANTIATE_TEST_SUITE_P(Primes, PrimeTest, ::testing::Values(
    std::make_pair(2, true),
    std::make_pair(4, false),
    std::make_pair(7, true)
));
```

## 覆盖率命令

```bash
# 带覆盖率构建
cmake -DCMAKE_CXX_FLAGS="--coverage" -DCMAKE_EXE_LINKER_FLAGS="--coverage" -B build

# 运行测试
cmake --build build && ctest --test-dir build

# 生成覆盖率报告
lcov --capture --directory build --output-file coverage.info
lcov --remove coverage.info '/usr/*' --output-file coverage.info
genhtml coverage.info --output-directory coverage_html
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
- 在适当情况下使用 `EXPECT_*`（继续）而非 `ASSERT_*`（停止）
- 测试行为，而非实现细节
- 包含边界情况（空值、null、最大值、边界条件）

**不要做的：**
- 在测试之前编写实现
- 跳过 RED 阶段
- 直接测试私有方法（通过公共 API 测试）
- 在测试中使用 `sleep`
- 忽略不稳定的测试

## 相关命令

- `/cpp-build` - 修复构建错误
- `/cpp-review` - 实现后审查代码
- `/verify` - 运行完整验证循环

## 相关

- Skill: `skills/cpp-testing/`
- Skill: `skills/tdd-workflow/`
