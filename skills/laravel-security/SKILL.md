---
name: laravel-security
description: Laravel 安全最佳实践，涵盖认证/授权、验证、CSRF、批量赋值、文件上传、密钥、速率限制和安全部署。
origin: ECC
---

# Laravel 安全最佳实践

Laravel 应用的全面安全指南，防范常见漏洞。

## 何时启用

- 添加认证或授权
- 处理用户输入和文件上传
- 构建新的 API 端点
- 管理密钥和环境设置
- 加固生产部署

## 工作原理

- 中间件提供基线保护（通过 `VerifyCsrfToken` 进行 CSRF 保护，通过 `SecurityHeaders` 添加安全头）。
- 守卫和策略强制访问控制（`auth:sanctum`、`$this->authorize`、策略中间件）。
- 表单请求在到达服务之前验证和塑造输入（`UploadInvoiceRequest`）。
- 速率限制在访问控制之外添加滥用保护（`RateLimiter::for('login')`）。
- 数据安全来自加密转换、批量赋值保护和签名路由（`URL::temporarySignedRoute` + `signed` 中间件）。

## 核心安全设置

- 生产环境设置 `APP_DEBUG=false`
- `APP_KEY` 必须设置，泄露时必须轮换
- 设置 `SESSION_SECURE_COOKIE=true` 和 `SESSION_SAME_SITE=lax`（或敏感应用使用 `strict`）
- 配置受信任代理以正确检测 HTTPS

## 会话和 Cookie 加固

- 设置 `SESSION_HTTP_ONLY=true` 以防止 JavaScript 访问
- 高风险流程使用 `SESSION_SAME_SITE=strict`
- 登录和权限变更时重新生成会话

## 认证和令牌

- API 认证使用 Laravel Sanctum 或 Passport
- 敏感数据优先使用短期令牌配合刷新流程
- 登出和账户泄露时撤销令牌

路由保护示例：

```php
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;

Route::middleware('auth:sanctum')->get('/me', function (Request $request) {
    return $request->user();
});
```

## 密码安全

- 使用 `Hash::make()` 哈希密码，永不存储明文
- 使用 Laravel 的密码代理进行重置流程

```php
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\Rules\Password;

$validated = $request->validate([
    'password' => ['required', 'string', Password::min(12)->letters()->mixedCase()->numbers()->symbols()],
]);

$user->update(['password' => Hash::make($validated['password'])]);
```

## 授权：策略和门

- 使用策略进行模型级授权
- 在控制器和服务中强制授权

```php
$this->authorize('update', $project);
```

使用策略中间件进行路由级强制：

```php
use Illuminate\Support\Facades\Route;

Route::put('/projects/{project}', [ProjectController::class, 'update'])
    ->middleware(['auth:sanctum', 'can:update,project']);
```

## 验证和数据清理

- 始终使用表单请求验证输入
- 使用严格的验证规则和类型检查
- 永远不要信任请求负载中的派生字段

## 批量赋值保护

- 使用 `$fillable` 或 `$guarded`，避免 `Model::unguard()`
- 优先使用 DTO 或显式属性映射

## SQL 注入防护

- 使用 Eloquent 或查询构建器的参数绑定
- 除非严格必要，避免原始 SQL

```php
DB::select('select * from users where email = ?', [$email]);
```

## XSS 防护

- Blade 默认转义输出（`{{ }}`）
- 仅对信任的、已清理的 HTML 使用 `{!! !!}`
- 使用专用库清理富文本

## CSRF 保护

- 保持 `VerifyCsrfToken` 中间件启用
- 在表单中包含 `@csrf`，为 SPA 请求发送 XSRF 令牌

对于使用 Sanctum 的 SPA 认证，确保配置了有状态请求：

```php
// config/sanctum.php
'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', 'localhost')),
```

## 文件上传安全

- 验证文件大小、MIME 类型和扩展名
- 尽可能将上传存储在公共路径之外
- 如有需要，扫描文件中的恶意软件

```php
final class UploadInvoiceRequest extends FormRequest
{
    public function authorize(): bool
    {
        return (bool) $this->user()?->can('upload-invoice');
    }

    public function rules(): array
    {
        return [
            'invoice' => ['required', 'file', 'mimes:pdf', 'max:5120'],
        ];
    }
}
```

```php
$path = $request->file('invoice')->store(
    'invoices',
    config('filesystems.private_disk', 'local') // 将此设置为非公共磁盘
);
```

## 速率限制

- 在认证和写入端点应用 `throttle` 中间件
- 对登录、密码重置和 OTP 使用更严格的限制

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

RateLimiter::for('login', function (Request $request) {
    return [
        Limit::perMinute(5)->by($request->ip()),
        Limit::perMinute(5)->by(strtolower((string) $request->input('email'))),
    ];
});
```

## 密钥和凭据

- 永远不要将密钥提交到源代码控制
- 使用环境变量和密钥管理器
- 泄露后轮换密钥并使会话失效

## 加密属性

对静态敏感列使用加密转换。

```php
protected $casts = [
    'api_token' => 'encrypted',
];
```

## 安全头

- 在适当位置添加 CSP、HSTS 和框架保护
- 使用受信任代理配置强制 HTTPS 重定向

设置头的中间件示例：

```php
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

final class SecurityHeaders
{
    public function handle(Request $request, \Closure $next): Response
    {
        $response = $next($request);

        $response->headers->add([
            'Content-Security-Policy' => "default-src 'self'",
            'Strict-Transport-Security' => 'max-age=31536000', // 仅当所有子域名都是 HTTPS 时添加 includeSubDomains/preload
            'X-Frame-Options' => 'DENY',
            'X-Content-Type-Options' => 'nosniff',
            'Referrer-Policy' => 'no-referrer',
        ]);

        return $response;
    }
}
```

## CORS 和 API 暴露

- 在 `config/cors.php` 中限制来源
- 避免对认证路由使用通配符来源

```php
// config/cors.php
return [
    'paths' => ['api/*', 'sanctum/csrf-cookie'],
    'allowed_methods' => ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
    'allowed_origins' => ['https://app.example.com'],
    'allowed_headers' => [
        'Content-Type',
        'Authorization',
        'X-Requested-With',
        'X-XSRF-TOKEN',
        'X-CSRF-TOKEN',
    ],
    'supports_credentials' => true,
];
```

## 日志和 PII

- 永远不要记录密码、令牌或完整卡数据
- 在结构化日志中编辑敏感字段

```php
use Illuminate\Support\Facades\Log;

Log::info('User updated profile', [
    'user_id' => $user->id,
    'email' => '[REDACTED]',
    'token' => '[REDACTED]',
]);
```

## 依赖安全

- 定期运行 `composer audit`
- 谨慎固定依赖，在 CVE 出现时及时更新

## 签名 URL

使用签名路由创建临时、防篡改的链接。

```php
use Illuminate\Support\Facades\URL;

$url = URL::temporarySignedRoute(
    'downloads.invoice',
    now()->addMinutes(15),
    ['invoice' => $invoice->id]
);
```

```php
use Illuminate\Support\Facades\Route;

Route::get('/invoices/{invoice}/download', [InvoiceController::class, 'download'])
    ->name('downloads.invoice')
    ->middleware('signed');
```
