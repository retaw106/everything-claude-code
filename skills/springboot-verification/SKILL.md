---
name: springboot-verification
description: Spring Boot 项目验证循环：在发布或 PR 之前进行构建、静态分析、带覆盖率的测试、安全扫描和差异审查。
origin: ECC
---

# Spring Boot 验证循环

在 PR 之前、重大更改之后和部署前运行。

## 何时激活

- 在为 Spring Boot 服务开启拉取请求之前
- 重大重构或依赖升级之后
- 预部署验证（staging 或 production）
- 运行完整构建 → lint → 测试 → 安全扫描流水线
- 验证测试覆盖率满足阈值

## 阶段 1：构建

```bash
mvn -T 4 clean verify -DskipTests
# 或
./gradlew clean assemble -x test
```

如果构建失败，停止并修复。

## 阶段 2：静态分析

Maven（常用插件）:
```bash
mvn -T 4 spotbugs:check pmd:check checkstyle:check
```

Gradle（如已配置）:
```bash
./gradlew checkstyleMain pmdMain spotbugsMain
```

## 阶段 3：测试 + 覆盖率

```bash
mvn -T 4 test
mvn jacoco:report   # 验证 80%+ 覆盖率
# 或
./gradlew test jacocoTestReport
```

报告:
- 总测试数，通过/失败
- 覆盖率 %（行/分支）

### 单元测试

使用 mock 依赖隔离测试服务逻辑:

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

  @Mock private UserRepository userRepository;
  @InjectMocks private UserService userService;

  @Test
  void createUser_validInput_returnsUser() {
    var dto = new CreateUserDto("Alice", "alice@example.com");
    var expected = new User(1L, "Alice", "alice@example.com");
    when(userRepository.save(any(User.class))).thenReturn(expected);

    var result = userService.create(dto);

    assertThat(result.name()).isEqualTo("Alice");
    verify(userRepository).save(any(User.class));
  }

  @Test
  void createUser_duplicateEmail_throwsException() {
    var dto = new CreateUserDto("Alice", "existing@example.com");
    when(userRepository.existsByEmail(dto.email())).thenReturn(true);

    assertThatThrownBy(() -> userService.create(dto))
        .isInstanceOf(DuplicateEmailException.class);
  }
}
```

### 使用 Testcontainers 进行集成测试

针对真实数据库而非 H2 进行测试:

```java
@SpringBootTest
@Testcontainers
class UserRepositoryIntegrationTest {

  @Container
  static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
      .withDatabaseName("testdb");

  @DynamicPropertySource
  static void configureProperties(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", postgres::getJdbcUrl);
    registry.add("spring.datasource.username", postgres::getUsername);
    registry.add("spring.datasource.password", postgres::getPassword);
  }

  @Autowired private UserRepository userRepository;

  @Test
  void findByEmail_existingUser_returnsUser() {
    userRepository.save(new User("Alice", "alice@example.com"));

    var found = userRepository.findByEmail("alice@example.com");

    assertThat(found).isPresent();
    assertThat(found.get().getName()).isEqualTo("Alice");
  }
}
```

### 使用 MockMvc 进行 API 测试

使用完整 Spring 上下文测试控制器层:

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

  @Autowired private MockMvc mockMvc;
  @MockBean private UserService userService;

  @Test
  void createUser_validInput_returns201() throws Exception {
    var user = new UserDto(1L, "Alice", "alice@example.com");
    when(userService.create(any())).thenReturn(user);

    mockMvc.perform(post("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"name": "Alice", "email": "alice@example.com"}
                """))
        .andExpect(status().isCreated())
        .andExpect(jsonPath("$.name").value("Alice"));
  }

  @Test
  void createUser_invalidEmail_returns400() throws Exception {
    mockMvc.perform(post("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"name": "Alice", "email": "not-an-email"}
                """))
        .andExpect(status().isBadRequest());
  }
}
```

## 阶段 4：安全扫描

```bash
# 依赖 CVE
mvn org.owasp:dependency-check-maven:check
# 或
./gradlew dependencyCheckAnalyze

# 源代码中的密钥
grep -rn "password\s*=\s*\"" src/ --include="*.java" --include="*.yml" --include="*.properties"
grep -rn "sk-\|api_key\|secret" src/ --include="*.java" --include="*.yml"

# 密钥（git 历史）
git secrets --scan  # 如已配置
```

### 常见安全发现

```
# 检查 System.out.println（改用 logger）
grep -rn "System\.out\.print" src/main/ --include="*.java"

# 检查响应中的原始异常消息
grep -rn "e\.getMessage()" src/main/ --include="*.java"

# 检查通配符 CORS
grep -rn "allowedOrigins.*\*" src/main/ --include="*.java"
```

## 阶段 5：Lint/格式（可选门禁）

```bash
mvn spotless:apply   # 如使用 Spotless 插件
./gradlew spotlessApply
```

## 阶段 6：差异审查

```bash
git diff --stat
git diff
```

检查清单:
- 无调试日志残留（`System.out`、无防护的 `log.debug`）
- 有意义的错误和 HTTP 状态
- 需要的地方有事务和验证
- 配置变更已文档化

## 输出模板

```
验证报告
==================
构建：     [通过/失败]
静态分析： [通过/失败]（spotbugs/pmd/checkstyle）
测试：     [通过/失败]（X/Y 通过，Z% 覆盖率）
安全：     [通过/失败]（CVE 发现：N）
差异：     [X 个文件已更改]

总体：     [就绪 / 未就绪]

待修复问题：
1. ...
2. ...
```

## 持续模式

- 在重大更改时或长会话中每 30-60 分钟重新运行各阶段
- 保持简短循环：`mvn -T 4 test` + spotbugs 用于快速反馈

**记住**：快速反馈胜过迟来的意外。保持门禁严格——在生产系统中将警告视为缺陷。
