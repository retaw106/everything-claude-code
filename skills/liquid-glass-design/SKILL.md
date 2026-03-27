---
name: liquid-glass-design
description: iOS 26 Liquid Glass 设计系统 — 动态玻璃材质，具有模糊、反射和交互变形效果，适用于 SwiftUI、UIKit 和 WidgetKit。
---

# Liquid Glass 设计系统（iOS 26）

实现 Apple Liquid Glass 的模式 — 一种动态材质，可模糊后方内容，反射周围内容的颜色和光线，并对触摸和指针交互做出反应。涵盖 SwiftUI、UIKit 和 WidgetKit 集成。

## 何时启用

- 使用新设计语言构建或更新 iOS 26+ 应用
- 实现玻璃风格按钮、卡片、工具栏或容器
- 创建玻璃元素之间的变形过渡
- 为小组件应用 Liquid Glass 效果
- 将现有模糊/材质效果迁移到新的 Liquid Glass API

## 核心模式 — SwiftUI

### 基本玻璃效果

为任何视图添加 Liquid Glass 的最简单方法：

```swift
Text("Hello, World!")
    .font(.title)
    .padding()
    .glassEffect()  // 默认：regular 变体，capsule 形状
```

### 自定义形状和色调

```swift
Text("Hello, World!")
    .font(.title)
    .padding()
    .glassEffect(.regular.tint(.orange).interactive(), in: .rect(cornerRadius: 16.0))
```

关键自定义选项：
- `.regular` — 标准玻璃效果
- `.tint(Color)` — 添加颜色色调以突出显示
- `.interactive()` — 对触摸和指针交互做出反应
- 形状：`.capsule`（默认）、`.rect(cornerRadius:)`、`.circle`

### 玻璃按钮样式

```swift
Button("Click Me") { /* action */ }
    .buttonStyle(.glass)

Button("Important") { /* action */ }
    .buttonStyle(.glassProminent)
```

### 多个元素的 GlassEffectContainer

始终将多个玻璃视图包装在容器中以获得性能和变形效果：

```swift
GlassEffectContainer(spacing: 40.0) {
    HStack(spacing: 40.0) {
        Image(systemName: "scribble.variable")
            .frame(width: 80.0, height: 80.0)
            .font(.system(size: 36))
            .glassEffect()

        Image(systemName: "eraser.fill")
            .frame(width: 80.0, height: 80.0)
            .font(.system(size: 36))
            .glassEffect()
    }
}
```

`spacing` 参数控制合并距离 — 更近的元素会融合它们的玻璃形状。

### 联合玻璃效果

使用 `glassEffectUnion` 将多个视图合并为单个玻璃形状：

```swift
@Namespace private var namespace

GlassEffectContainer(spacing: 20.0) {
    HStack(spacing: 20.0) {
        ForEach(symbolSet.indices, id: \.self) { item in
            Image(systemName: symbolSet[item])
                .frame(width: 80.0, height: 80.0)
                .glassEffect()
                .glassEffectUnion(id: item < 2 ? "group1" : "group2", namespace: namespace)
        }
    }
}
```

### 变形过渡

当玻璃元素出现/消失时创建平滑变形：

```swift
@State private var isExpanded = false
@Namespace private var namespace

GlassEffectContainer(spacing: 40.0) {
    HStack(spacing: 40.0) {
        Image(systemName: "scribble.variable")
            .frame(width: 80.0, height: 80.0)
            .glassEffect()
            .glassEffectID("pencil", in: namespace)

        if isExpanded {
            Image(systemName: "eraser.fill")
                .frame(width: 80.0, height: 80.0)
                .glassEffect()
                .glassEffectID("eraser", in: namespace)
        }
    }
}

Button("Toggle") {
    withAnimation { isExpanded.toggle() }
}
.buttonStyle(.glass)
```

### 在侧边栏下扩展水平滚动

要允许水平滚动内容延伸到侧边栏或检查器下方，确保 `ScrollView` 内容到达容器的前导/尾随边缘。当布局延伸到边缘时，系统会自动处理侧边栏下方滚动行为 — 不需要额外的修饰符。

## 核心模式 — UIKit

### 基本 UIGlassEffect

```swift
let glassEffect = UIGlassEffect()
glassEffect.tintColor = UIColor.systemBlue.withAlphaComponent(0.3)
glassEffect.isInteractive = true

let visualEffectView = UIVisualEffectView(effect: glassEffect)
visualEffectView.translatesAutoresizingMaskIntoConstraints = false
visualEffectView.layer.cornerRadius = 20
visualEffectView.clipsToBounds = true

view.addSubview(visualEffectView)
NSLayoutConstraint.activate([
    visualEffectView.centerXAnchor.constraint(equalTo: view.centerXAnchor),
    visualEffectView.centerYAnchor.constraint(equalTo: view.centerYAnchor),
    visualEffectView.widthAnchor.constraint(equalToConstant: 200),
    visualEffectView.heightAnchor.constraint(equalToConstant: 120)
])

// 向 contentView 添加内容
let label = UILabel()
label.text = "Liquid Glass"
label.translatesAutoresizingMaskIntoConstraints = false
visualEffectView.contentView.addSubview(label)
NSLayoutConstraint.activate([
    label.centerXAnchor.constraint(equalTo: visualEffectView.contentView.centerXAnchor),
    label.centerYAnchor.constraint(equalTo: visualEffectView.contentView.centerYAnchor)
])
```

### 多个元素的 UIGlassContainerEffect

```swift
let containerEffect = UIGlassContainerEffect()
containerEffect.spacing = 40.0

let containerView = UIVisualEffectView(effect: containerEffect)

let firstGlass = UIVisualEffectView(effect: UIGlassEffect())
let secondGlass = UIVisualEffectView(effect: UIGlassEffect())

containerView.contentView.addSubview(firstGlass)
containerView.contentView.addSubview(secondGlass)
```

### 滚动边缘效果

```swift
scrollView.topEdgeEffect.style = .automatic
scrollView.bottomEdgeEffect.style = .hard
scrollView.leftEdgeEffect.isHidden = true
```

### 工具栏玻璃集成

```swift
let favoriteButton = UIBarButtonItem(image: UIImage(systemName: "heart"), style: .plain, target: self, action: #selector(favoriteAction))
favoriteButton.hidesSharedBackground = true  // 选择不使用共享玻璃背景
```

## 核心模式 — WidgetKit

### 渲染模式检测

```swift
struct MyWidgetView: View {
    @Environment(\.widgetRenderingMode) var renderingMode

    var body: some View {
        if renderingMode == .accented {
            // 着色模式：白色着色、主题化玻璃背景
        } else {
            // 全彩模式：标准外观
        }
    }
}
```

### 视觉层次的重音组

```swift
HStack {
    VStack(alignment: .leading) {
        Text("Title")
            .widgetAccentable()  // 重音组
        Text("Subtitle")
            // 主要组（默认）
    }
    Image(systemName: "star.fill")
        .widgetAccentable()  // 重音组
}
```

### 着色模式下的图片渲染

```swift
Image("myImage")
    .widgetAccentedRenderingMode(.monochrome)
```

### 容器背景

```swift
VStack { /* content */ }
    .containerBackground(for: .widget) {
        Color.blue.opacity(0.2)
    }
```

## 关键设计决策

| 决策 | 理由 |
|----------|-----------|
| GlassEffectContainer 包装 | 性能优化，启用玻璃元素之间的变形 |
| `spacing` 参数 | 控制合并距离 — 微调元素必须多近才能融合 |
| `@Namespace` + `glassEffectID` | 在视图层次变化时启用平滑变形过渡 |
| `interactive()` 修饰符 | 明确选择触摸/指针反应 — 不是所有玻璃都应该响应 |
| UIKit 中的 UIGlassContainerEffect | 与 SwiftUI 相同的容器模式以保持一致性 |
| 小组件中的着色渲染模式 | 当用户选择着色的主屏幕时，系统应用着色玻璃 |

## 最佳实践

- **始终使用 GlassEffectContainer** 将玻璃应用于多个同级视图 — 它启用变形并改善渲染性能
- **在其他外观修饰符之后应用 `.glassEffect()`**（frame、font、padding）
- **仅对响应用户交互的元素使用 `.interactive()`**（按钮、可切换项目）
- **仔细选择容器中的 spacing** 以控制玻璃效果何时合并
- **在更改视图层次结构时使用 `withAnimation`** 以启用平滑变形过渡
- **跨外观测试** — 浅色模式、深色模式和着色/色调模式
- **确保可访问性对比度** — 玻璃上的文字必须保持可读

## 要避免的反模式

- 在没有 GlassEffectContainer 的情况下使用多个独立的 `.glassEffect()` 视图
- 嵌套太多玻璃效果 — 降低性能和视觉清晰度
- 对每个视图应用玻璃 — 保留给交互元素、工具栏和卡片
- 在 UIKit 中使用圆角半径时忘记 `clipsToBounds = true`
- 在小组件中忽略着色渲染模式 — 破坏着色主屏幕外观
- 在玻璃后面使用不透明背景 — 破坏半透明效果

## 何时使用

- 使用新 iOS 26 设计的导航栏、工具栏和标签栏
- 浮动操作按钮和卡片样式容器
- 需要视觉深度和触摸反馈的交互控件
- 应与系统 Liquid Glass 外观集成的小组件
- 相关 UI 状态之间的变形过渡
