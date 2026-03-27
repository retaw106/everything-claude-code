---
name: springboot-patterns
description: Spring Boot 架构模式、REST API 设计、分层服务、数据访问、缓存、异步处理和日志记录。用于 Java Spring Boot 后端工作。
origin: ECC
---

# Spring Boot 开发模式

适用于可扩展、生产级服务的 Spring Boot 架构和 API 模式。

## 何时启用

- 使用 Spring MVC 或 WebFlux 构建 REST API
- 构建控制器 → 服务 → 仓储层
- 配置 Spring Data JPA、缓存或异步处理
- 添加验证、异常处理或分页
- 为 dev/staging/production 环境设置 profile
- 使用 Spring Events 或 Kafka 实现事件驱动模式

## REST API 结构

```java
@RestController
@RequestMapping("/api/markets")
@Validated
class MarketController {
  private final MarketService marketService;

  MarketController(MarketService marketService) {
    this.marketService = marketService;
  }

  @GetMapping
  ResponseEntity<Page<MarketResponse>> list(
      @RequestParam(defaultValue = "0") int page,
      @RequestParam(defaultValue = "20") int size) {
    Page<Market> markets = marketService.list(PageRequest.of(page, size));
    return ResponseEntity.ok(markets.map(MarketResponse::from));
  }

  @PostMapping
  ResponseEntity<MarketResponse> create(@Valid @RequestBody CreateMarketRequest request) {
    Market market = marketService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(MarketResponse.from(market));
  }
}
```

## 仓储模式（Spring Data JPA）

```java
public interface MarketRepository extends JpaRepository<MarketEntity, Long> {
  @Query("select m from MarketEntity m where m.status = :status order by m.volume desc")
  List<MarketEntity> findActive(@Param("status") MarketStatus status, Pageable pageable);
}
```

## 带事务的服务层

```java
@Service
public class MarketService {
  private final MarketRepository repo;

  public MarketService(MarketRepository repo) {
    this.repo = repo;
  }

  @Transactional
  public Market create(CreateMarketRequest request) {
    MarketEntity entity = MarketEntity.from(request);
    MarketEntity saved = repo.save(entity);
    return Market.from(saved);
  }
}
```

## DTO 和验证

```java
public record CreateMarketRequest(
    @NotBlank @Size(max = 200) String name,
    @NotBlank @Size(max = 2000) String description,
    @NotNull @FutureOrPresent Instant endDate,
    @NotEmpty List<@NotBlank String> categories) {}

public record MarketResponse(Long id, String name, MarketStatus status) {
  static MarketResponse from(Market market) {
    return new MarketResponse(market.id(), market.name(), market.status());
  }
}
```

## 异常处理

```java
@ControllerAdvice
class GlobalExceptionHandler {
  @ExceptionHandler(MethodArgumentNotValidException.class)
  ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException ex) {
    String message = ex.getBindingResult().getFieldErrors().stream()
        .map(e -> e.getField() + ": " + e.getDefaultMessage())
        .collect(Collectors.joining(", "));
    return ResponseEntity.badRequest().body(ApiError.validation(message));
  }

  @ExceptionHandler(AccessDeniedException.class)
  ResponseEntity<ApiError> handleAccessDenied() {
    return ResponseEntity.status(HttpStatus.FORBIDDEN).body(ApiError.of("Forbidden"));
  }

  @ExceptionHandler(Exception.class)
  ResponseEntity<ApiError> handleGeneric(Exception ex) {
    // 记录带有堆栈跟踪的意外错误
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
        .body(ApiError.of("Internal server error"));
  }
}
```

## 缓存

需要在配置类上使用 `@EnableCaching`。

```java
@Service
public class MarketCacheService {
  private final MarketRepository repo;

  public MarketCacheService(MarketRepository repo) {
    this.repo = repo;
  }

  @Cacheable(value = "market", key = "#id")
  public Market getById(Long id) {
    return repo.findById(id)
        .map(Market::from)
        .orElseThrow(() -> new EntityNotFoundException("Market not found"));
  }

  @CacheEvict(value = "market", key = "#id")
  public void evict(Long id) {}
}
```

## 异步处理

需要在配置类上使用 `@EnableAsync`。

```java
@Service
public class NotificationService {
  @Async
  public CompletableFuture<Void> sendAsync(Notification notification) {
    // 发送邮件/SMS
    return CompletableFuture.completedFuture(null);
  }
}
```

## 日志记录（SLF4J）

```java
@Service
public class ReportService {
  private static final Logger log = LoggerFactory.getLogger(ReportService.class);

  public Report generate(Long marketId) {
    log.info("generate_report marketId={}", marketId);
    try {
      // 逻辑
    } catch (Exception ex) {
      log.error("generate_report_failed marketId={}", marketId, ex);
      throw ex;
    }
    return new Report();
  }
}
```

## 中间件 / 过滤器

```java
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {
  private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain filterChain) throws ServletException, IOException {
    long start = System.currentTimeMillis();
    try {
      filterChain.doFilter(request, response);
    } finally {
      long duration = System.currentTimeMillis() - start;
      log.info("req method={} uri={} status={} durationMs={}",
          request.getMethod(), request.getRequestURI(), response.getStatus(), duration);
    }
  }
}
```

## 分页和排序

```java
PageRequest page = PageRequest.of(pageNumber, pageSize, Sort.by("createdAt").descending());
Page<Market> results = marketService.list(page);
```

## 错误弹性的外部调用

```java
public <T> T withRetry(Supplier<T> supplier, int maxRetries) {
  int attempts = 0;
  while (true) {
    try {
      return supplier.get();
    } catch (Exception ex) {
      attempts++;
      if (attempts >= maxRetries) {
        throw ex;
      }
      try {
        Thread.sleep((long) Math.pow(2, attempts) * 100L);
      } catch (InterruptedException ie) {
        Thread.currentThread().interrupt();
        throw ex;
      }
    }
  }
}
```

## 速率限制（Filter + Bucket4j）

**安全注意**：`X-Forwarded-For` 头默认是不可信的，因为客户端可以伪造它。
只有在以下情况下才使用转发头：
1. 你的应用位于受信任的反向代理（nginx、AWS ALB 等）后面
2. 你已将 `ForwardedHeaderFilter` 注册为 bean
3. 你在 application properties 中配置了 `server.forward-headers-strategy=NATIVE` 或 `FRAMEWORK`
4. 你的代理配置为覆盖（而非追加）`X-Forwarded-For` 头

当 `ForwardedHeaderFilter` 正确配置后，`request.getRemoteAddr()` 将自动从转发头返回正确的客户端 IP。没有此配置时，直接使用 `request.getRemoteAddr()` — 它返回直接连接 IP，这是唯一可信的值。

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {
  private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

  /*
   * 安全：此过滤器使用 request.getRemoteAddr() 来识别客户端进行速率限制。
   *
   * 如果你的应用位于反向代理（nginx、AWS ALB 等）后面，必须配置
   * Spring 以正确处理转发头以进行准确的客户端 IP 检测：
   *
   * 1. 在 application.properties/yaml 中设置
   *    server.forward-headers-strategy=NATIVE（用于云平台）或 FRAMEWORK
   *
   * 2. 如果使用 FRAMEWORK 策略，注册 ForwardedHeaderFilter：
   *
   *    @Bean
   *    ForwardedHeaderFilter forwardedHeaderFilter() {
   *        return new ForwardedHeaderFilter();
   *    }
   *
   * 3. 确保你的代理覆盖（而非追加）X-Forwarded-For 头以防止欺骗
   * 4. 为你的容器配置 server.tomcat.remoteip.trusted-proxies 或等效配置
   *
   * 没有此配置，request.getRemoteAddr() 返回代理 IP，而非客户端 IP。
   * 不要直接读取 X-Forwarded-For —— 没有正确的代理处理，它很容易被伪造。
   */
  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain filterChain) throws ServletException, IOException {
    // 使用 getRemoteAddr()，当 ForwardedHeaderFilter 配置后
    // 它返回正确的客户端 IP，否则返回直接连接 IP。永远不要直接信任
    // X-Forwarded-For 头，除非有正确的代理配置。
    String clientIp = request.getRemoteAddr();

    Bucket bucket = buckets.computeIfAbsent(clientIp,
        k -> Bucket.builder()
            .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
            .build());

    if (bucket.tryConsume(1)) {
      filterChain.doFilter(request, response);
    } else {
      response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
    }
  }
}
```

## 后台任务

使用 Spring 的 `@Scheduled` 或与队列集成（如 Kafka、SQS、RabbitMQ）。保持处理程序幂等且可观察。

## 可观察性

- 结构化日志（JSON）通过 Logback encoder
- 指标：Micrometer + Prometheus/OTel
- 追踪：Micrometer Tracing with OpenTelemetry or Brave backend

## 生产默认值

- 优先使用构造函数注入，避免字段注入
- 启用 `spring.mvc.problemdetails.enabled=true` 以获得 RFC 7807 错误（Spring Boot 3+）
- 为工作负载配置 HikariCP 池大小，设置超时
- 对查询使用 `@Transactional(readOnly = true)`
- 通过 `@NonNull` 和 `Optional` 在适当位置强制空安全

**记住**：保持控制器轻量、服务专注、仓储简单、错误集中处理。优先考虑可维护性和可测试性。
