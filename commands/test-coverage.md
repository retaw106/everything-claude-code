# Test Coverage

分析测试覆盖率，识别缺口，并生成缺失测试以达到 80%+ 覆盖率。

## 第一步：检测测试框架

| Indicator | Coverage Command |
|-----------|-----------------|
| `jest.config.*` 或 `package.json` jest | `npx jest --coverage --coverageReporters=json-summary` |
| `vitest.config.*` | `npx vitest run --coverage` |
| `pytest.ini` / `pyproject.toml` pytest | `pytest --cov=src --cov-report=json` |
| `Cargo.toml` | `cargo llvm-cov --json` |
| `pom.xml` with JaCoCo | `mvn test jacoco:report` |
| `go.mod` | `go test -coverprofile=coverage.out ./...` |

## 第二步：分析覆盖率报告

1. 运行覆盖率命令
2. 解析输出（JSON 摘要或终端输出）
3. 列出**覆盖率低于 80%** 的文件，按最差的排序
4. 对于每个覆盖率不足的文件，识别：
   - 未测试的函数或方法
   - 缺失的分支覆盖（if/else、switch、错误路径）
   - 增加分母的死代码

## 第三步：生成缺失测试

对于每个覆盖率不足的文件，按照以下优先级生成测试：

1. **正常路径** — 使用有效输入的核心功能
2. **错误处理** — 无效输入、缺失数据、网络故障
3. **边界情况** — 空数组、null/undefined、边界值（0、-1、MAX_INT）
4. **分支覆盖** — 每个 if/else、switch case、三元运算符

### 测试生成规则

- 将测试放置在源文件旁边：`foo.ts` → `foo.test.ts`（或项目约定）
- 使用项目中的现有测试模式（import 风格、断言库、mocking 方法）
- Mock 外部依赖（database、API、file system）
- 每个测试应该是独立的 — 测试之间没有共享的可变状态
- 描述性地命名测试：`test_create_user_with_duplicate_email_returns_409`

## 第四步：验证

1. 运行完整测试套件 — 所有测试必须通过
2. 重新运行覆盖率 — 验证改进
3. 如果仍然低于 80%，对剩余的缺口重复第三步

## 第五步：报告

显示前后对比：

```
Coverage Report
──────────────────────────────
File                   Before  After
src/services/auth.ts   45%     88%
src/utils/validation.ts 32%    82%
──────────────────────────────
Overall:               67%     84%  ✅
```

## 关注领域

- 具有复杂分支的函数（高圈复杂度）
- 错误处理器和 catch 块
- 在代码库中使用的工具函数
- API 端点处理程序（request → response 流）
- 边界情况：null、undefined、空字符串、空数组、零、负数
