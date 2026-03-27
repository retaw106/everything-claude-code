---
description: 强制执行 Kotlin 的 TDD 工作流。先编写 Kotest 测试，然后实现。使用 Kover 验证 80%+ 的覆盖率。
---

# Kotlin TDD 命令

此命令使用 Kotest、MockK 和 Kover 强制执行 Kotlin 代码的测试驱动开发方法论。

## 本命令的作用

1. **定义类型/接口**：先搭建函数签名
2. **编写 Kotest 测试**：创建全面的测试规格（RED）
3. **运行测试**：验证测试因正确原因失败
4. **实现代码**：编写最小代码以通过（GREEN）
5. **重构**：在测试保持绿色的同时改进代码
6. **检查覆盖率**：使用 Kover 确保 80%+ 覆盖率

## 何时使用

在以下场景使用 `/kotlin-test`：
- 实现新的 Kotlin 函数或类
- 为现有 Kotlin 代码添加测试覆盖
- 修复 Bug（先编写失败的测试）
- 构建关键业务逻辑
- 学习 Kotlin 中的 TDD 工作流

## TDD 循环

```
RED     → 编写失败的 Kotest 测试
GREEN   → 实现最小代码以通过
REFACTOR → 改进代码，测试保持绿色
REPEAT  → 下一个测试用例
```

## 示例会话

````
用户: /kotlin-test 我需要一个验证用户注册请求的函数

Agent:
# TDD 会话：注册验证器

## 步骤 1：定义接口

```kotlin
// validator/RegistrationValidator.kt
package com.example.validator

data class RegistrationRequest(
    val name: String,
    val email: String,
    val password: String,
)

sealed class ValidationResult {
    data object Valid : ValidationResult()
    data class Invalid(val errors: List<String>) : ValidationResult()
}

fun validateRegistration(request: RegistrationRequest): ValidationResult {
    TODO("not implemented")
}
```

## 步骤 2：编写 Kotest 测试（RED）

```kotlin
// validator/RegistrationValidatorTest.kt
package com.example.validator

import io.kotest.core.spec.style.FunSpec
import io.kotest.matchers.shouldBe
import io.kotest.matchers.types.shouldBeInstanceOf

class RegistrationValidatorTest : FunSpec({
    test("valid registration returns Valid") {
        val request = RegistrationRequest(
            name = "Alice",
            email = "alice@example.com",
            password = "SecureP@ss1",
        )

        val result = validateRegistration(request)

        result.shouldBeInstanceOf<ValidationResult.Valid>()
    }

    test("blank name returns Invalid") {
        val request = RegistrationRequest(
            name = "",
            email = "alice@example.com",
            password = "SecureP@ss1",
        )

        val result = validateRegistration(request)

        val invalid = result.shouldBeInstanceOf<ValidationResult.Invalid>()
        invalid.errors shouldBe listOf("Name is required")
    }

    test("invalid email returns Invalid") {
        val request = RegistrationRequest(
            name = "Alice",
            email = "not-an-email",
            password = "SecureP@ss1",
        )

        val result = validateRegistration(request)

        val invalid = result.shouldBeInstanceOf<ValidationResult.Invalid>()
        invalid.errors shouldBe listOf("Invalid email format")
    }

    test("short password returns Invalid") {
        val request = RegistrationRequest(
            name = "Alice",
            email = "alice@example.com",
            password = "short",
        )

        val result = validateRegistration(request)

        val invalid = result.shouldBeInstanceOf<ValidationResult.Invalid>()
        invalid.errors shouldBe listOf("Password must be at least 8 characters")
    }

    test("multiple errors returns all errors") {
        val request = RegistrationRequest(
            name = "",
            email = "bad",
            password = "short",
        )

        val result = validateRegistration(request)

        val invalid = result.shouldBeInstanceOf<ValidationResult.Invalid>()
        invalid.errors.size shouldBe 3
    }
})
```

## 步骤 3：运行测试 - 验证失败

```bash
$ ./gradlew test

RegistrationValidatorTest > valid registration returns Valid 失败
  kotlin.NotImplementedError: An operation is not implemented

失败（5 个测试，0 个通过，5 个失败）
```

✓ 测试按预期失败（NotImplementedError）。

## 步骤 4：实现最小代码（GREEN）

```kotlin
// validator/RegistrationValidator.kt
package com.example.validator

private val EMAIL_REGEX = Regex("^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$")
private const val MIN_PASSWORD_LENGTH = 8

fun validateRegistration(request: RegistrationRequest): ValidationResult {
    val errors = buildList {
        if (request.name.isBlank()) add("Name is required")
        if (!EMAIL_REGEX.matches(request.email)) add("Invalid email format")
        if (request.password.length < MIN_PASSWORD_LENGTH) add("Password must be at least $MIN_PASSWORD_LENGTH characters")
    }

    return if (errors.isEmpty()) ValidationResult.Valid
    else ValidationResult.Invalid(errors)
}
```

## 步骤 5：运行测试 - 验证通过

```bash
$ ./gradlew test

RegistrationValidatorTest > valid registration returns Valid 通过
RegistrationValidatorTest > blank name returns Invalid 通过
RegistrationValidatorTest > invalid email returns Invalid 通过
RegistrationValidatorTest > short password returns Invalid 通过
RegistrationValidatorTest > multiple errors returns all errors 通过

通过（5 个测试，5 个通过，0 个失败）
```

✓ 所有测试通过！

## 步骤 6：检查覆盖率

```bash
$ ./gradlew koverHtmlReport

Coverage: 100.0% of statements
```

✓ 覆盖率: 100%

## TDD 完成！
````

## 测试模式

### StringSpec（最简单）

```kotlin
class CalculatorTest : StringSpec({
    "add two positive numbers" {
        Calculator.add(2, 3) shouldBe 5
    }
})
```

### BehaviorSpec（BDD）

```kotlin
class OrderServiceTest : BehaviorSpec({
    Given("a valid order") {
        When("placed") {
            Then("should be confirmed") { /* ... */ }
        }
    }
})
```

### 数据驱动测试

```kotlin
class ParserTest : FunSpec({
    context("valid inputs") {
        withData("2026-01-15", "2026-12-31", "2000-01-01") { input ->
            parseDate(input).shouldNotBeNull()
        }
    }
})
```

### 协程测试

```kotlin
class AsyncServiceTest : FunSpec({
    test("concurrent fetch completes") {
        runTest {
            val result = service.fetchAll()
            result.shouldNotBeEmpty()
        }
    }
})
```

## 覆盖率命令

```bash
# 带覆盖率运行测试
./gradlew koverHtmlReport

# 验证覆盖率阈值
./gradlew koverVerify

# 用于 CI 的 XML 报告
./gradlew koverXmlReport

# 打开 HTML 报告
open build/reports/kover/html/index.html

# 运行特定测试类
./gradlew test --tests "com.example.UserServiceTest"

# 带详细输出运行
./gradlew test --info
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
- 使用 Kotest 匹配器进行表达性断言
- 对 suspend 函数使用 MockK 的 `coEvery`/`coVerify`
- 测试行为，而非实现细节
- 包含边界情况（空值、null、最大值）

**不要做的：**
- 在测试之前编写实现
- 跳过 RED 阶段
- 直接测试私有函数
- 在协程测试中使用 `Thread.sleep()`
- 忽略不稳定的测试

## 相关命令

- `/kotlin-build` - 修复构建错误
- `/kotlin-review` - 实现后审查代码
- `/verify` - 运行完整验证循环

## 相关

- Skill: `skills/kotlin-testing/`
- Skill: `skills/tdd-workflow/`
