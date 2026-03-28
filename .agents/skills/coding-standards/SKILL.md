---
name: coding-standards
description: TypeScript、JavaScript、React 和 Node.js 开发的通用编码标准、最佳实践和模式。
origin: ECC
---

# 编码标准与最佳实践

适用于所有项目的通用编码标准。

## 何时启用

- 启动新项目或模块
- 审查代码的质量和可维护性
- 重构现有代码以遵循约定
- 强制执行命名、格式化或结构一致性
- 设置 linting、格式化或类型检查规则
- 帮助新贡献者了解编码约定

## 代码质量原则

### 1. 可读性优先
- 代码被阅读的次数远多于被编写的次数
- 清晰的变量和函数名称
- 自文档化代码优于注释
- 一致的格式化

### 2. KISS（保持简单）
- 能工作的最简单解决方案
- 避免过度工程
- 不做过早优化
- 易于理解 > 炫技代码

### 3. DRY（不重复自己）
- 将通用逻辑提取到函数中
- 创建可重用组件
- 在模块间共享工具
- 避免复制粘贴编程

### 4. YAGNI（你不会需要它）
- 不要提前构建功能
- 避免投机性的通用设计
- 仅在需要时增加复杂性
- 从简单开始，需要时重构

## TypeScript/JavaScript 标准

### 变量命名

```typescript
// ✅ 好：描述性名称
const marketSearchQuery = 'election'
const isUserAuthenticated = true
const totalRevenue = 1000

// ❌ 坏：不清晰的名称
const q = 'election'
const flag = true
const x = 1000
```

### 函数命名

```typescript
// ✅ 好：动词-名词模式
async function fetchMarketData(marketId: string) { }
function calculateSimilarity(a: number[], b: number[]) { }
function isValidEmail(email: string): boolean { }

// ❌ 坏：不清晰或仅名词
async function market(id: string) { }
function similarity(a, b) { }
function email(e) { }
```

### 不可变性模式（关键）

```typescript
// ✅ 始终使用展开运算符
const updatedUser = {
  ...user,
  name: 'New Name'
}

const updatedArray = [...items, newItem]

// ❌ 绝不要直接修改
user.name = 'New Name'  // 坏
items.push(newItem)     // 坏
```

### 错误处理

```typescript
// ✅ 好：全面的错误处理
async function fetchData(url: string) {
  try {
    const response = await fetch(url)

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`)
    }

    return await response.json()
  } catch (error) {
    console.error('获取失败:', error)
    throw new Error('获取数据失败')
  }
}

// ❌ 坏：无错误处理
async function fetchData(url) {
  const response = await fetch(url)
  return response.json()
}
```

### Async/Await 最佳实践

```typescript
// ✅ 好：尽可能并行执行
const [users, markets, stats] = await Promise.all([
  fetchUsers(),
  fetchMarkets(),
  fetchStats()
])

// ❌ 坏：不必要的顺序执行
const users = await fetchUsers()
const markets = await fetchMarkets()
const stats = await fetchStats()
```

### 类型安全

```typescript
// ✅ 好：正确的类型
interface Market {
  id: string
  name: string
  status: 'active' | 'resolved' | 'closed'
  created_at: Date
}

function getMarket(id: string): Promise<Market> {
  // 实现
}

// ❌ 坏：使用 'any'
function getMarket(id: any): Promise<any> {
  // 实现
}
```

## React 最佳实践

### 组件结构

```typescript
// ✅ 好：带类型的功能组件
interface ButtonProps {
  children: React.ReactNode
  onClick: () => void
  disabled?: boolean
  variant?: 'primary' | 'secondary'
}

export function Button({
  children,
  onClick,
  disabled = false,
  variant = 'primary'
}: ButtonProps) {
  return (
    <button
      onClick={onClick}
      disabled={disabled}
      className={`btn btn-${variant}`}
    >
      {children}
    </button>
  )
}

// ❌ 坏：无类型，结构不清晰
export function Button(props) {
  return <button onClick={props.onClick}>{props.children}</button>
}
```

### 自定义 Hooks

```typescript
// ✅ 好：可重用的自定义 hook
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const handler = setTimeout(() => {
      setDebouncedValue(value)
    }, delay)

    return () => clearTimeout(handler)
  }, [value, delay])

  return debouncedValue
}

// 使用方式
const debouncedQuery = useDebounce(searchQuery, 500)
```

### 状态管理

```typescript
// ✅ 好：正确的状态更新
const [count, setCount] = useState(0)

// 基于前一个状态的功能性更新
setCount(prev => prev + 1)

// ❌ 坏：直接引用状态
setCount(count + 1)  // 在异步场景中可能过期
```

### 条件渲染

```typescript
// ✅ 好：清晰的条件渲染
{isLoading && <Spinner />}
{error && <ErrorMessage error={error} />}
{data && <DataDisplay data={data} />}

// ❌ 坏：三元地狱
{isLoading ? <Spinner /> : error ? <ErrorMessage error={error} /> : data ? <DataDisplay data={data} /> : null}
```

## API 设计标准

### REST API 约定

```
GET    /api/markets              # 列出所有市场
GET    /api/markets/:id          # 获取特定市场
POST   /api/markets              # 创建新市场
PUT    /api/markets/:id          # 更新市场（完整）
PATCH  /api/markets/:id          # 更新市场（部分）
DELETE /api/markets/:id          # 删除市场

# 查询参数用于过滤
GET /api/markets?status=active&limit=10&offset=0
```

### 响应格式

```typescript
// ✅ 好：一致的响应结构
interface ApiResponse<T> {
  success: boolean
  data?: T
  error?: string
  meta?: {
    total: number
    page: number
    limit: number
  }
}

// 成功响应
return NextResponse.json({
  success: true,
  data: markets,
  meta: { total: 100, page: 1, limit: 10 }
})

// 错误响应
return NextResponse.json({
  success: false,
  error: '无效请求'
}, { status: 400 })
```

### 输入验证

```typescript
import { z } from 'zod'

// ✅ 好：Schema 验证
const CreateMarketSchema = z.object({
  name: z.string().min(1).max(200),
  description: z.string().min(1).max(2000),
  endDate: z.string().datetime(),
  categories: z.array(z.string()).min(1)
})

export async function POST(request: Request) {
  const body = await request.json()

  try {
    const validated = CreateMarketSchema.parse(body)
    // 使用验证后的数据继续
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json({
        success: false,
        error: '验证失败',
        details: error.errors
      }, { status: 400 })
    }
  }
}
```

## 文件组织

### 项目结构

```
src/
├── app/                    # Next.js App Router
│   ├── api/               # API 路由
│   ├── markets/           # 市场页面
│   └── (auth)/           # 认证页面（路由组）
├── components/            # React 组件
│   ├── ui/               # 通用 UI 组件
│   ├── forms/            # 表单组件
│   └── layouts/          # 布局组件
├── hooks/                # 自定义 React hooks
├── lib/                  # 工具和配置
│   ├── api/             # API 客户端
│   ├── utils/           # 辅助函数
│   └── constants/       # 常量
├── types/                # TypeScript 类型
└── styles/              # 全局样式
```

### 文件命名

```
components/Button.tsx          # 组件使用 PascalCase
hooks/useAuth.ts              # 带 'use' 前缀的 camelCase
lib/formatDate.ts             # 工具使用 camelCase
types/market.types.ts         # 带 .types 后缀的 camelCase
```

## 注释与文档

### 何时添加注释

```typescript
// ✅ 好：解释为什么，而不是做什么
// 使用指数退避以在服务中断时避免压垮 API
const delay = Math.min(1000 * Math.pow(2, retryCount), 30000)

// 为了大数组的性能，此处故意使用修改操作
items.push(newItem)

// ❌ 坏：陈述显而易见的内容
// 将计数器加 1
count++

// 将 name 设置为用户的 name
name = user.name
```

### 公共 API 的 JSDoc

```typescript
/**
 * 使用语义相似度搜索市场。
 *
 * @param query - 自然语言搜索查询
 * @param limit - 最大结果数（默认：10）
 * @returns 按相似度排序的市场数组
 * @throws {Error} 如果 OpenAI API 失败或 Redis 不可用
 *
 * @example
 * ```typescript
 * const results = await searchMarkets('election', 5)
 * console.log(results[0].name) // "Trump vs Biden"
 * ```
 */
export async function searchMarkets(
  query: string,
  limit: number = 10
): Promise<Market[]> {
  // 实现
}
```

## 性能最佳实践

### 记忆化

```typescript
import { useMemo, useCallback } from 'react'

// ✅ 好：记忆化昂贵的计算
const sortedMarkets = useMemo(() => {
  return markets.sort((a, b) => b.volume - a.volume)
}, [markets])

// ✅ 好：记忆化回调
const handleSearch = useCallback((query: string) => {
  setSearchQuery(query)
}, [])
```

### 懒加载

```typescript
import { lazy, Suspense } from 'react'

// ✅ 好：懒加载重型组件
const HeavyChart = lazy(() => import('./HeavyChart'))

export function Dashboard() {
  return (
    <Suspense fallback={<Spinner />}>
      <HeavyChart />
    </Suspense>
  )
}
```

### 数据库查询

```typescript
// ✅ 好：仅选择需要的列
const { data } = await supabase
  .from('markets')
  .select('id, name, status')
  .limit(10)

// ❌ 坏：选择所有
const { data } = await supabase
  .from('markets')
  .select('*')
```

## 测试标准

### 测试结构（AAA 模式）

```typescript
test('正确计算相似度', () => {
  // Arrange（准备）
  const vector1 = [1, 0, 0]
  const vector2 = [0, 1, 0]

  // Act（执行）
  const similarity = calculateCosineSimilarity(vector1, vector2)

  // Assert（断言）
  expect(similarity).toBe(0)
})
```

### 测试命名

```typescript
// ✅ 好：描述性的测试名称
test('当没有市场匹配查询时返回空数组', () => { })
test('当 OpenAI API 密钥缺失时抛出错误', () => { })
test('当 Redis 不可用时回退到子字符串搜索', () => { })

// ❌ 坏：模糊的测试名称
test('works', () => { })
test('test search', () => { })
```

## 代码异味检测

注意这些反模式：

### 1. 过长的函数
```typescript
// ❌ 坏：函数超过 50 行
function processMarketData() {
  // 100 行代码
}

// ✅ 好：拆分成更小的函数
function processMarketData() {
  const validated = validateData()
  const transformed = transformData(validated)
  return saveData(transformed)
}
```

### 2. 深层嵌套
```typescript
// ❌ 坏：5+ 层嵌套
if (user) {
  if (user.isAdmin) {
    if (market) {
      if (market.isActive) {
        if (hasPermission) {
          // 做某事
        }
      }
    }
  }
}

// ✅ 好：提前返回
if (!user) return
if (!user.isAdmin) return
if (!market) return
if (!market.isActive) return
if (!hasPermission) return

// 做某事
```

### 3. 魔法数字
```typescript
// ❌ 坏：未解释的数字
if (retryCount > 3) { }
setTimeout(callback, 500)

// ✅ 好：命名常量
const MAX_RETRIES = 3
const DEBOUNCE_DELAY_MS = 500

if (retryCount > MAX_RETRIES) { }
setTimeout(callback, DEBOUNCE_DELAY_MS)
```

**记住**：代码质量不可妥协。清晰、可维护的代码能实现快速开发和自信重构。
