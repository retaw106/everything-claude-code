---
description: 增量式修复 C++ 构建错误、CMake 问题和链接器问题。调用 cpp-build-resolver Agent 进行最小化、精确的修复。
---

# C++ 构建和修复

此命令调用 **cpp-build-resolver** Agent 以最小化更改增量式修复 C++ 构建错误。

## 本命令的作用

1. **运行诊断**：执行 `cmake --build`、编译器检查
2. **解析错误**：按文件分组，按严重性排序
3. **增量修复**：一次修复一个错误
4. **验证每个修复**：每次更改后重新构建
5. **报告摘要**：显示已修复和剩余内容

## 何时使用

在以下场景使用 `/cpp-build`：
- `cmake --build` 失败并出现错误
- C++ 编译器报告错误
- 链接器报告未定义引用
- 模块依赖问题
- 拉取更改导致构建失败后

## 诊断命令运行

```bash
# 主要构建检查
cmake --build build 2>&1

# 静态分析（如果配置）
clang-tidy -p build src/*.cpp 2>&1 || echo "clang-tidy not configured"

# 其他分析
cppcheck --enable=all --suppress=missingIncludeSystem src/ 2>&1

# 模块问题
cmake --build build --target help 2>&1 | head -50
```

## 示例会话

```text
用户: /cpp-build

Agent:
# C++ 构建解决方案

## 初始诊断

```bash
$ cmake --build build
internal/service/user.cpp:25:15 error: 'UserRepository' was not declared
internal/handler/api.cpp:42:9 error: cannot convert 'std::string' to 'int'
internal/handler/api.cpp:58:2 error: expected ';' before 'return'
```

发现错误：3 个

## 修复 1：未声明的标识符

文件：internal/service/user.cpp:25
错误：'UserRepository' was not declared
原因：缺少 include 或命名空间

```cpp
// 添加 include
#include "repository/UserRepository.hpp"

// 或使用完整命名空间
auto repo = repository::UserRepository();
```

```bash
$ cmake --build build
# 剩余 2 个错误
```

## 修复 2：类型不匹配

文件：internal/handler/api.cpp:42
错误：cannot convert 'std::string' to 'int'

```cpp
// 修改
int count = params.get("count");
// 改为
std::string countStr = params.get("count");
int count = std::stoi(countStr);
```

```bash
$ cmake --build build
# 剩余 1 个错误
```

## 修复 3：缺少分号

文件：internal/handler/api.cpp:58
错误：expected ';' before 'return'

```cpp
// 添加缺少的分号
std::cout << "Debug: " << value << std::endl;
return result;
```

```bash
$ cmake --build build
# 构建成功！
```

## 最终验证

```bash
$ ctest --test-dir build --output-on-failure
100% tests passed
```

## 摘要

| 指标 | 数量 |
|------|------|
| 已修复的构建错误 | 3 |
| 已修复的警告 | 0 |
| 已修改的文件 | 2 |
| 剩余问题 | 0 |

构建状态：✅ 成功
```

## 常见错误修复

| 错误 | 典型修复 |
|------|----------|
| `undeclared identifier` | 添加 `#include` 或修复拼写错误 |
| `no matching function` | 修复参数类型或添加重载 |
| `undefined reference` | 链接库或添加实现 |
| `multiple definition` | 使用 `inline` 或移至 .cpp |
| `incomplete type` | 将前向声明替换为 `#include` |
| `no member named X` | 修复成员名称或 include |
| `cannot convert X to Y` | 添加适当的转换 |
| `CMake Error` | 修复 CMakeLists.txt 配置 |

## 修复策略

1. **先编译错误** - 代码必须能编译
2. **后链接器错误** - 解决未定义引用
3. **最后警告** - 使用 `-Wall -Wextra` 修复
4. **一次修复一个** - 验证每个更改
5. **最小化更改** - 不要重构，只修复

## 停止条件

Agent 将停止并报告如果：
- 同一错误在 3 次尝试后仍然存在
- 修复引入更多错误
- 需要架构更改
- 缺少外部依赖

## 相关命令

- `/cpp-test` - 构建成功后运行测试
- `/cpp-review` - 审查代码质量
- `/verify` - 完整验证循环

## 相关

- Agent: `agents/cpp-build-resolver.md`
- Skill: `skills/cpp-coding-standards/`
