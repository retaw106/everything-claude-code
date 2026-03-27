---
name: foundation-models-on-device
description: Apple FoundationModels 框架用于设备端 LLM — 文本生成、配合 @Generable 的引导式生成、工具调用，以及 iOS 26+ 的快照流式传输。
---

# FoundationModels: 设备端 LLM (iOS 26)

使用 FoundationModels 框架将 Apple 设备端语言模型集成到应用中的模式。涵盖文本生成、使用 `@Generable` 的结构化输出、自定义工具调用和快照流式传输 —— 全部在设备端运行，支持隐私和离线功能。

## 何时激活

- 使用 Apple Intelligence 设备端构建 AI 驱动的功能
- 生成或总结文本而无需云依赖
- 从自然语言输入中提取结构化数据
- 为领域特定的 AI 操作实现自定义工具调用
- 为实时 UI 更新流式传输结构化响应
- 需要保护隐私的 AI（数据不离开设备）

## 核心模式 — 可用性检查

始终在创建会话之前检查模型可用性：

```swift
struct GenerativeView: View {
    private var model = SystemLanguageModel.default

    var body: some View {
        switch model.availability {
        case .available:
            ContentView()
        case .unavailable(.deviceNotEligible):
            Text("设备不符合 Apple Intelligence 要求")
        case .unavailable(.appleIntelligenceNotEnabled):
            Text("请在设置中启用 Apple Intelligence")
        case .unavailable(.modelNotReady):
            Text("模型正在下载或未就绪")
        case .unavailable(let other):
            Text("模型不可用: \(other)")
        }
    }
}
```

## 核心模式 — 基本会话

```swift
// 单轮：每次创建新会话
let session = LanguageModelSession()
let response = try await session.respond(to: "哪个月份适合去巴黎？")
print(response.content)

// 多轮：重用会话以保持对话上下文
let session = LanguageModelSession(instructions: """
    你是一个烹饪助手。
    根据食材提供食谱建议。
    保持建议简洁实用。
    """)

let first = try await session.respond(to: "我有鸡肉和米饭")
let followUp = try await session.respond(to: "有什么素食选择？")
```

指令的关键点：
- 定义模型的角色（"你是一个导师"）
- 指定要做什么（"帮助提取日历事件"）
- 设置风格偏好（"尽可能简短回答"）
- 添加安全措施（"对危险请求回复'我无法帮助你'"）

## 核心模式 — 配合 @Generable 的引导式生成

生成结构化的 Swift 类型而非原始字符串：

### 1. 定义 Generable 类型

```swift
@Generable(description: "关于猫咪的基本档案信息")
struct CatProfile {
    var name: String

    @Guide(description: "猫咪的年龄", .range(0...20))
    var age: Int

    @Guide(description: "一句关于猫咪性格的描述")
    var profile: String
}
```

### 2. 请求结构化输出

```swift
let response = try await session.respond(
    to: "生成一只可爱的救援猫",
    generating: CatProfile.self
)

// 直接访问结构化字段
print("名字: \(response.content.name)")
print("年龄: \(response.content.age)")
print("档案: \(response.content.profile)")
```

### 支持的 @Guide 约束

- `.range(0...20)` — 数字范围
- `.count(3)` — 数组元素数量
- `description:` — 生成的语义指导

## 核心模式 — 工具调用

让模型为领域特定任务调用自定义代码：

### 1. 定义工具

```swift
struct RecipeSearchTool: Tool {
    let name = "recipe_search"
    let description = "搜索匹配给定关键词的食谱并返回结果列表。"

    @Generable
    struct Arguments {
        var searchTerm: String
        var numberOfResults: Int
    }

    func call(arguments: Arguments) async throws -> ToolOutput {
        let recipes = await searchRecipes(
            term: arguments.searchTerm,
            limit: arguments.numberOfResults
        )
        return .string(recipes.map { "- \($0.name): \($0.description)" }.joined(separator: "\n"))
    }
}
```

### 2. 创建带工具的会话

```swift
let session = LanguageModelSession(tools: [RecipeSearchTool()])
let response = try await session.respond(to: "给我找一些意大利面食谱")
```

### 3. 处理工具错误

```swift
do {
    let answer = try await session.respond(to: "找一个番茄汤食谱。")
} catch let error as LanguageModelSession.ToolCallError {
    print(error.tool.name)
    if case .databaseIsEmpty = error.underlyingError as? RecipeSearchToolError {
        // 处理特定工具错误
    }
}
```

## 核心模式 — 快照流式传输

使用 `PartiallyGenerated` 类型为实时 UI 流式传输结构化响应：

```swift
@Generable
struct TripIdeas {
    @Guide(description: "即将到来的旅行想法")
    var ideas: [String]
}

let stream = session.streamResponse(
    to: "有什么激动人心的旅行想法？",
    generating: TripIdeas.self
)

for try await partial in stream {
    // partial: TripIdeas.PartiallyGenerated（所有属性为 Optional）
    print(partial)
}
```

### SwiftUI 集成

```swift
@State private var partialResult: TripIdeas.PartiallyGenerated?
@State private var errorMessage: String?

var body: some View {
    List {
        ForEach(partialResult?.ideas ?? [], id: \.self) { idea in
            Text(idea)
        }
    }
    .overlay {
        if let errorMessage { Text(errorMessage).foregroundStyle(.red) }
    }
    .task {
        do {
            let stream = session.streamResponse(to: prompt, generating: TripIdeas.self)
            for try await partial in stream {
                partialResult = partial
            }
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

## 关键设计决策

| 决策 | 理由 |
|----------|-----------|
| 设备端执行 | 隐私 — 数据不离开设备；支持离线 |
| 4,096 token 限制 | 设备端模型约束；将大数据分块到多个会话 |
| 快照流式传输（非增量） | 结构化输出友好；每个快照是完整的部分状态 |
| `@Generable` 宏 | 结构化生成的编译时安全；自动生成 `PartiallyGenerated` 类型 |
| 每个会话单个请求 | `isResponding` 防止并发请求；如需要创建多个会话 |
| `response.content`（非 `.output`） | 正确的 API — 始终通过 `.content` 属性访问结果 |

## 最佳实践

- **始终在创建会话之前检查 `model.availability`** — 处理所有不可用情况
- **使用 `instructions`** 指导模型行为 — 它们优先于提示
- **在发送新请求之前检查 `isResponding`** — 会话一次处理一个请求
- **访问 `response.content`** 获取结果 — 不是 `.output`
- **将大输入分成块** — 4,096 token 限制适用于指令 + 提示 + 输出的总和
- **使用 `@Generable`** 获取结构化输出 — 比解析原始字符串有更强的保证
- **使用 `GenerationOptions(temperature:)`** 调整创造性（更高 = 更有创意）
- **使用 Instruments 监控** — 使用 Xcode Instruments 分析请求性能

## 要避免的反模式

- 创建会话前不先检查 `model.availability`
- 发送超过 4,096 token 上下文窗口的输入
- 尝试在单个会话上并发请求
- 使用 `.output` 而非 `.content` 访问响应数据
- 当 `@Generable` 结构化输出可行时解析原始字符串响应
- 在单个提示中构建复杂的多步逻辑 — 分解为多个专注的提示
- 假设模型始终可用 — 设备资格和设置各不相同

## 何时使用

- 用于隐私敏感应用的设备端文本生成
- 从用户输入中提取结构化数据（表单、自然语言命令）
- 必须离线工作的 AI 辅助功能
- 逐步显示生成内容的流式 UI
- 通过工具调用实现领域特定的 AI 操作（搜索、计算、查找）
