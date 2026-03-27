---
description: 修复 Android 和 KMP 项目的 Gradle 构建错误
---

# Gradle 构建修复

增量式修复 Android 和 Kotlin Multiplatform 项目的 Gradle 构建和编译错误。

## 步骤 1：检测构建配置

识别项目类型并运行适当的构建：

| 标识 | 构建命令 |
|------|----------|
| `build.gradle.kts` + `composeApp/`（KMP） | `./gradlew composeApp:compileKotlinMetadata 2>&1` |
| `build.gradle.kts` + `app/`（Android） | `./gradlew app:compileDebugKotlin 2>&1` |
| 带模块的 `settings.gradle.kts` | `./gradlew assemble 2>&1` |
| 配置了 Detekt | `./gradlew detekt 2>&1` |

同时检查 `gradle.properties` 和 `local.properties` 的配置。

## 步骤 2：解析和分组错误

1. 运行构建命令并捕获输出
2. 将 Kotlin 编译错误与 Gradle 配置错误分开
3. 按模块和文件路径分组
4. 排序：先配置错误，然后按依赖顺序的编译错误

## 步骤 3：修复循环

对于每个错误：

1. **读取文件** — 错误行周围的完整上下文
2. **诊断** — 常见类别：
   - 缺少 import 或未解析的引用
   - 类型不匹配或不兼容的类型
   - `build.gradle.kts` 中缺少依赖
   - Expect/actual 不匹配（KMP）
   - Compose 编译器错误
3. **最小化修复** — 解决错误的最小更改
4. **重新构建** — 验证修复并检查新错误
5. **继续** — 移至下一个错误

## 步骤 4：防护措施

如果出现以下情况，停止并询问用户：
- 修复引入的错误比解决的多
- 同一错误在 3 次尝试后仍然存在
- 错误需要添加新依赖或更改模块结构
- Gradle sync 本身失败（配置阶段错误）
- 错误在生成的代码中（Room、SQLDelight、KMP）

## 步骤 5：摘要

报告：
- 已修复的错误（模块、文件、描述）
- 剩余错误
- 引入的新错误（应为零）
- 建议的后续步骤

## 常见 Gradle/KMP 修复

| 错误 | 修复 |
|------|------|
| `commonMain` 中未解析的引用 | 检查依赖是否在 `commonMain.dependencies {}` 中 |
| Expect 声明没有 actual | 在每个平台源集添加 `actual` 实现 |
| Compose 编译器版本不匹配 | 在 `libs.versions.toml` 中对齐 Kotlin 和 Compose 编译器版本 |
| 重复的类 | 使用 `./gradlew dependencies` 检查冲突依赖 |
| KSP 错误 | 运行 `./gradlew kspCommonMainKotlinMetadata` 重新生成 |
| 配置缓存问题 | 检查不可序列化的任务输入 |
