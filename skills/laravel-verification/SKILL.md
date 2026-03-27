---
name: laravel-verification
description: Laravel 项目的验证循环：环境检查、linting、静态分析、带覆盖率的测试、安全扫描和部署就绪检查。
origin: ECC
---

# Laravel 验证循环

在 PR 之前、重大更改之后和部署前运行。

## 何时使用

- 在为 Laravel 项目打开 pull request 之前
- 在重大重构或依赖升级之后
- 预发布或预生产的部署前验证
- 运行完整的 lint -> test -> security -> deploy 就绪流水线

## 工作原理

- 从环境检查到部署就绪顺序运行各阶段，使每一层都建立在上一层之上。
- 环境和 Composer 检查是所有其他步骤的门槛；如果失败则立即停止。
- Linting/静态分析应该在运行完整测试和覆盖率之前保持干净。
- 安全和迁移审查在测试之后进行，以便在数据或发布步骤之前验证行为。
- 构建/部署就绪和队列/调度器检查是最后的关卡；任何失败都会阻止发布。

## Phase 1: 环境检查

```bash
php -v
composer --version
php artisan --version
```

- 验证 `.env` 存在且必需的键存在
- 确认生产环境中 `APP_DEBUG=false`
- 确认 `APP_ENV` 与目标部署匹配（`production`、`staging`）

如果本地使用 Laravel Sail：

```bash
./vendor/bin/sail php -v
./vendor/bin/sail artisan --version
```

## Phase 1.5: Composer 和 Autoload

```bash
composer validate
composer dump-autoload -o
```

## Phase 2: Linting 和静态分析

```bash
vendor/bin/pint --test
vendor/bin/phpstan analyse
```

如果你的项目使用 Psalm 而不是 PHPStan：

```bash
vendor/bin/psalm
```

## Phase 3: 测试和覆盖率

```bash
php artisan test
```

覆盖率（CI）：

```bash
XDEBUG_MODE=coverage php artisan test --coverage
```

CI 示例（格式化 -> 静态分析 -> 测试）：

```bash
vendor/bin/pint --test
vendor/bin/phpstan analyse
XDEBUG_MODE=coverage php artisan test --coverage
```

## Phase 4: 安全和依赖检查

```bash
composer audit
```

## Phase 5: 数据库和迁移

```bash
php artisan migrate --pretend
php artisan migrate:status
```

- 仔细审查破坏性迁移
- 确保迁移文件名遵循 `Y_m_d_His_*`（例如 `2025_03_14_154210_create_orders_table.php`）并清晰描述变更
- 确保可以回滚
- 验证 `down()` 方法，避免在没有明确备份的情况下造成不可逆的数据丢失

## Phase 6: 构建和部署就绪

```bash
php artisan optimize:clear
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

- 确保在生产配置中缓存预热成功
- 验证队列工作进程和调度器已配置
- 确认目标环境中 `storage/` 和 `bootstrap/cache/` 可写

## Phase 7: 队列和调度器检查

```bash
php artisan schedule:list
php artisan queue:failed
```

如果使用 Horizon：

```bash
php artisan horizon:status
```

如果 `queue:monitor` 可用，使用它检查积压而不处理任务：

```bash
php artisan queue:monitor default --max=100
```

主动验证（仅限 staging）：将一个无操作任务分发到专用队列，运行单个工作进程处理它（确保配置了非 `sync` 队列连接）。

```bash
php artisan tinker --execute="dispatch((new App\\Jobs\\QueueHealthcheck())->onQueue('healthcheck'))"
php artisan queue:work --once --queue=healthcheck
```

验证任务产生了预期的副作用（日志条目、健康检查表行或指标）。

仅在处理测试任务安全的非生产环境中运行此操作。

## 示例

最小流程：

```bash
php -v
composer --version
php artisan --version
composer validate
vendor/bin/pint --test
vendor/bin/phpstan analyse
php artisan test
composer audit
php artisan migrate --pretend
php artisan config:cache
php artisan queue:failed
```

CI 风格流水线：

```bash
composer validate
composer dump-autoload -o
vendor/bin/pint --test
vendor/bin/phpstan analyse
XDEBUG_MODE=coverage php artisan test --coverage
composer audit
php artisan migrate --pretend
php artisan optimize:clear
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan schedule:list
```
