---
description: 使用 Playwright 生成并运行端到端测试。创建测试旅程、运行测试、捕获截图/视频/追踪，并上传工件。
---

# E2E 命令

此命令调用 **e2e-runner** agent 使用 Playwright 生成、维护和执行端到端测试。

## 此命令的功能

1. **生成测试旅程** - 为用户流程创建 Playwright 测试
2. **运行 E2E 测试** - 跨浏览器执行测试
3. **捕获工件** - 失败时的截图、视频、追踪
4. **上传结果** - HTML 报告和 JUnit XML
5. **识别不稳定测试** - 隔离不稳定的测试

## 何时使用

在以下情况下使用 `/e2e`：
- 测试关键用户旅程（登录、交易、支付）
- 验证多步骤流程端到端工作
- 测试 UI 交互和导航
- 验证前端和后端之间的集成
- 准备生产部署

## 工作原理

e2e-runner agent 将：

1. **分析用户流程** 并识别测试场景
2. **使用 Page Object Model 模式生成 Playwright 测试**
3. **跨多个浏览器运行测试**（Chrome、Firefox、Safari）
4. **用截图、视频和追踪捕获失败**
5. **生成包含结果和工件的报告**
6. **识别不稳定测试** 并建议修复

## 使用示例

```
用户: /e2e 测试市场搜索和查看流程

Agent (e2e-runner):
# E2E 测试生成：市场搜索和查看流程

## 识别的测试场景

**用户旅程：** 市场搜索 → 查看结果 → 点击市场 → 查看详情

**测试覆盖：**
1. 导航到市场页面
2. 执行语义搜索
3. 验证搜索结果
4. 点击第一个结果
5. 验证市场详情页面加载
6. 验证图表渲染

## 生成的测试代码

```typescript
// tests/e2e/markets/search-and-view.spec.ts
import { test, expect } from '@playwright/test'
import { MarketsPage } from '../../pages/MarketsPage'
import { MarketDetailsPage } from '../../pages/MarketDetailsPage'

test.describe('市场搜索和查看流程', () => {
  test('用户可以搜索市场并查看详情', async ({ page }) => {
    // 1. 导航到市场页面
    const marketsPage = new MarketsPage(page)
    await marketsPage.goto()

    // 验证页面已加载
    await expect(page).toHaveTitle(/Markets/)
    await expect(page.locator('h1')).toContainText('Markets')

    // 2. 执行语义搜索
    await marketsPage.searchMarkets('election')

    // 等待 API 响应
    await page.waitForResponse(resp =>
      resp.url().includes('/api/markets/search') && resp.status() === 200
    )

    // 3. 验证搜索结果
    const marketCards = marketsPage.marketCards
    await expect(marketCards.first()).toBeVisible()
    const resultCount = await marketCards.count()
    expect(resultCount).toBeGreaterThan(0)

    // 截取搜索结果截图
    await page.screenshot({ path: 'artifacts/search-results.png' })

    // 4. 点击第一个结果
    const firstMarketTitle = await marketCards.first().textContent()
    await marketCards.first().click()

    // 5. 验证市场详情页面加载
    await expect(page).toHaveURL(/\/markets\/[a-z0-9-]+/)

    const detailsPage = new MarketDetailsPage(page)
    await expect(detailsPage.marketName).toBeVisible()
    await expect(detailsPage.marketDescription).toBeVisible()

    // 6. 验证图表渲染
    await expect(detailsPage.priceChart).toBeVisible()

    // 验证市场名称匹配
    const detailsTitle = await detailsPage.marketName.textContent()
    expect(detailsTitle?.toLowerCase()).toContain(
      firstMarketTitle?.toLowerCase().substring(0, 20) || ''
    )

    // 截取市场详情截图
    await page.screenshot({ path: 'artifacts/market-details.png' })
  })

  test('无结果搜索显示空状态', async ({ page }) => {
    const marketsPage = new MarketsPage(page)
    await marketsPage.goto()

    // 搜索不存在的市场
    await marketsPage.searchMarkets('xyznonexistentmarket123456')

    // 验证空状态
    await expect(page.locator('[data-testid="no-results"]')).toBeVisible()
    await expect(page.locator('[data-testid="no-results"]')).toContainText(
      /no.*results|no.*markets/i
    )

    const marketCount = await marketsPage.marketCards.count()
    expect(marketCount).toBe(0)
  })

  test('可以清除搜索并再次查看所有市场', async ({ page }) => {
    const marketsPage = new MarketsPage(page)
    await marketsPage.goto()

    // 初始市场数量
    const initialCount = await marketsPage.marketCards.count()

    // 执行搜索
    await marketsPage.searchMarkets('trump')
    await page.waitForLoadState('networkidle')

    // 验证过滤结果
    const filteredCount = await marketsPage.marketCards.count()
    expect(filteredCount).toBeLessThan(initialCount)

    // 清除搜索
    await marketsPage.searchInput.clear()
    await page.waitForLoadState('networkidle')

    // 验证再次显示所有市场
    const finalCount = await marketsPage.marketCards.count()
    expect(finalCount).toBe(initialCount)
  })
})
```

## 运行测试

```bash
# 运行生成的测试
npx playwright test tests/e2e/markets/search-and-view.spec.ts

Running 3 tests using 3 workers

  ✓  [chromium] › search-and-view.spec.ts:5:3 › 用户可以搜索市场并查看详情 (4.2s)
  ✓  [chromium] › search-and-view.spec.ts:52:3 › 无结果搜索显示空状态 (1.8s)
  ✓  [chromium] › search-and-view.spec.ts:67:3 › 可以清除搜索并再次查看所有市场 (2.9s)

  3 passed (9.1s)

生成的工件：
- artifacts/search-results.png
- artifacts/market-details.png
- playwright-report/index.html
```

## 测试报告

```
╔══════════════════════════════════════════════════════════════╗
║                    E2E 测试结果                              ║
╠══════════════════════════════════════════════════════════════╣
║ 状态:     ✅ 所有测试通过                              ║
║ 总计:     3 个测试                                          ║
║ 通过:     3 (100%)                                         ║
║ 失败:     0                                                ║
║ 不稳定:   0                                                ║
║ 耗时:     9.1s                                             ║
╚══════════════════════════════════════════════════════════════╝

工件：
📸 截图: 2 个文件
📹 视频: 0 个文件（仅在失败时）
🔍 追踪: 0 个文件（仅在失败时）
📊 HTML 报告: playwright-report/index.html

查看报告: npx playwright show-report
```

✅ E2E 测试套件已准备好进行 CI/CD 集成！
```

## 测试工件

测试运行时，会捕获以下工件：

**所有测试：**
- 带有时间线和结果的 HTML 报告
- 用于 CI 集成的 JUnit XML

**仅失败时：**
- 失败状态的截图
- 测试视频录制
- 用于调试的追踪文件（逐步回放）
- 网络日志
- 控制台日志

## 查看工件

```bash
# 在浏览器中查看 HTML 报告
npx playwright show-report

# 查看特定追踪文件
npx playwright show-trace artifacts/trace-abc123.zip

# 截图保存在 artifacts/ 目录中
open artifacts/search-results.png
```

## 不稳定测试检测

如果测试间歇性失败：

```
⚠️  检测到不稳定测试: tests/e2e/markets/trade.spec.ts

测试 10 次运行中通过 7 次（70% 通过率）

常见失败：
"Timeout waiting for element '[data-testid="confirm-btn"]'"

建议修复：
1. 添加显式等待: await page.waitForSelector('[data-testid="confirm-btn"]')
2. 增加超时时间: { timeout: 10000 }
3. 检查组件中的竞态条件
4. 验证元素未被动画隐藏

隔离建议：标记为 test.fixme() 直到修复
```

## 浏览器配置

测试默认在多个浏览器上运行：
- ✅ Chromium（桌面 Chrome）
- ✅ Firefox（桌面）
- ✅ WebKit（桌面 Safari）
- ✅ Mobile Chrome（可选）

在 `playwright.config.ts` 中配置以调整浏览器。

## CI/CD 集成

添加到您的 CI 管道：

```yaml
# .github/workflows/e2e.yml
- name: 安装 Playwright
  run: npx playwright install --with-deps

- name: 运行 E2E 测试
  run: npx playwright test

- name: 上传工件
  if: always()
  uses: actions/upload-artifact@v3
  with:
    name: playwright-report
    path: playwright-report/
```

## 关键流程

优先进行这些 E2E 测试：

**🔴 CRITICAL（必须始终通过）：**
1. 用户可以连接钱包
2. 用户可以浏览市场
3. 用户可以搜索市场（语义搜索）
4. 用户可以查看市场详情
5. 用户可以下订单（使用测试资金）
6. 市场正确结算
7. 用户可以提取资金

**🟡 重要：**
1. 市场创建流程
2. 用户资料更新
3. 实时价格更新
4. 图表渲染
5. 过滤和排序市场
6. 移动端响应式布局

## 最佳实践

**应该做：**
- ✅ 使用 Page Object Model 以提高可维护性
- ✅ 使用 data-testid 属性作为选择器
- ✅ 等待 API 响应，而非任意超时
- ✅ 端到端测试关键用户旅程
- ✅ 合并到 main 前运行测试
- ✅ 测试失败时审查工件

**不应该做：**
- ❌ 使用脆弱的选择器（CSS 类可能会更改）
- ❌ 测试实现细节
- ❌ 对生产环境运行测试
- ❌ 忽略不稳定的测试
- ❌ 失败时跳过工件审查
- ❌ 用 E2E 测试每个边界情况（使用单元测试）

## 重要说明

**关键事项：**
- 涉及真实资金的 E2E 测试必须仅在测试网/暂存环境运行
- 永远不要对生产环境运行交易测试
- 对财务测试设置 `test.skip(process.env.NODE_ENV === 'production')`
- 仅使用带有小额测试资金的测试钱包

## 与其他命令的集成

- 使用 `/plan` 识别要测试的关键旅程
- 使用 `/tdd` 进行单元测试（更快、更细粒度）
- 使用 `/e2e` 进行集成和用户旅程测试
- 使用 `/code-review` 验证测试质量

## 相关 Agent

此命令调用 ECC 提供的 `e2e-runner` agent。

对于手动安装，源文件位于：
`agents/e2e-runner.md`

## 快速命令

```bash
# 运行所有 E2E 测试
npx playwright test

# 运行特定测试文件
npx playwright test tests/e2e/markets/search.spec.ts

# 以有头模式运行（查看浏览器）
npx playwright test --headed

# 调试测试
npx playwright test --debug

# 生成测试代码
npx playwright codegen http://localhost:3000

# 查看报告
npx playwright show-report
```
