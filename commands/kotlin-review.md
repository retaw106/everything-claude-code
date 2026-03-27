---
description: 全面性的 Kotlin 代码审查，涵盖惯用模式、空安全、协程安全和安全性。调用 kotlin-reviewer Agent。
---

# Kotlin 代码审查

此命令调用 **kotlin-reviewer** Agent 进行全面的 Kotlin 特定代码审查。

## 本命令的作用

1. **识别 Kotlin 更改**：通过 `git diff` 找出修改的 `.kt` 和 `.kts` 文件
2. **运行构建和静态分析**：执行 `./gradlew build`、`detekt`、`ktlintCheck`
3. **安全性扫描**：检查 SQL 注入、命令注入、硬编码密钥
4. **空安全审查**：分析 `!!` 使用、平台类型处理、不安全转换
5. **协程审查**：检查结构化并发、dispatcher 使用、取消
6. **生成报告**：按严重性对问题进行分类

## 何时使用

在以下场景使用 `/kotlin-review`：
- 编写或修改 Kotlin 代码后
- 提交 Kotlin 更改前
- 审查包含 Kotlin 代码的 PR
- 接手新的 Kotlin 代码库
- 学习惯用的 Kotlin 模式

## 审查类别

### CRITICAL（必须修复）
- SQL/命令注入漏洞
- 无正当理由的强制解包 `!!`
- 平台类型空安全违规
- GlobalScope 使用（结构化并发违规）
- 硬编码凭据
- 不安全的反序列化

### HIGH（应当修复）
- 可用不可变的地方使用了可变状态
- 协程上下文内的阻塞调用
- 长循环中缺少取消检查
- Sealed 类型上的非穷尽 `when`
- 大函数（>50 行）
- 深层嵌套（>4 层）

### MEDIUM（需要考虑）
- 非惯用的 Kotlin（Java 风格模式）
- 缺少尾随逗号
- 作用域函数误用或嵌套
- 大型集合链缺少 sequence
- 冗余的显式类型

## 自动检查运行

```bash
# 构建检查
./gradlew build

# 静态分析
./gradlew detekt

# 格式检查
./gradlew ktlintCheck

# 测试
./gradlew test
```

## 示例用法

```text
用户: /kotlin-review

Agent:
# Kotlin 代码审查报告

## 已审查的文件
- src/main/kotlin/com/example/service/UserService.kt（已修改）
- src/main/kotlin/com/example/routes/UserRoutes.kt（已修改）

## 静态分析结果
✓ 构建：成功
✓ detekt：无问题
⚠ ktlint：2 个格式警告

## 发现的问题

[CRITICAL] 强制解包空安全
文件：src/main/kotlin/com/example/service/UserService.kt:28
问题：在可空的仓库结果上使用 !!
```kotlin
val user = repository.findById(id)!!  // NPE 风险
```
修复：使用安全调用配合错误处理
```kotlin
val user = repository.findById(id)
    ?: throw UserNotFoundException("User $id not found")
```

[HIGH] GlobalScope 使用
文件：src/main/kotlin/com/example/routes/UserRoutes.kt:45
问题：使用 GlobalScope 破坏了结构化并发
```kotlin
GlobalScope.launch {
    notificationService.sendWelcome(user)
}
```
修复：使用调用的协程作用域
```kotlin
launch {
    notificationService.sendWelcome(user)
}
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

- 先使用 `/kotlin-test` 确保测试通过
- 如果发生构建错误，使用 `/kotlin-build`
- 在提交前使用 `/kotlin-review`
- 对于非 Kotlin 特定问题，使用 `/code-review`

## 相关

- Agent: `agents/kotlin-reviewer.md`
- Skills: `skills/kotlin-patterns/`、`skills/kotlin-testing/`
