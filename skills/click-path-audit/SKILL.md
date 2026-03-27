---
name: click-path-audit
description: "追踪每个面向用户的按钮/触点通过其完整状态变化序列，以发现函数单独工作但相互抵消、产生错误最终状态或使 UI 处于不一致状态的 bug。使用时机：系统调试未发现 bug 但用户报告按钮损坏，或在任何触及共享状态存储的重大重构之后。"
origin: community
---

# /click-path-audit — 行为流程审计

发现静态代码阅读遗漏的 bug：状态交互副作用、顺序调用之间的竞态条件，以及相互静默撤销的处理程序。

## 解决的问题

传统调试检查：
- 函数是否存在？（缺少连接）
- 是否崩溃？（运行时错误）
- 是否返回正确的类型？（数据流）

但它不检查：
- **最终 UI 状态是否与按钮标签承诺的匹配？**
- **函数 B 是否静默撤销了函数 A 刚刚做的事？**
- **共享状态（Zustand/Redux/context）是否有抵消预期操作的副作用？**

真实案例：一个"新邮件"按钮调用 `setComposeMode(true)` 然后调用 `selectThread(null)`。两者单独都工作。但 `selectThread` 有一个副作用重置 `composeMode: false`。按钮什么也没做。系统调试发现了 54 个 bug — 这个被遗漏了。

---

## 工作原理

对于目标区域中的每个交互触点：

```
1. 识别处理程序（onClick、onSubmit、onChange 等）
2. 按顺序追踪处理程序中的每个函数调用
3. 对于每个函数调用：
   a. 它读取什么状态？
   b. 它写入什么状态？
   c. 它对共享状态有副作用吗？
   d. 它作为副作用重置/清除任何状态吗？
4. 检查：任何后续调用是否撤销了早期调用的状态更改？
5. 检查：最终状态是否是用户从按钮标签期望的？
6. 检查：是否有竞态条件（以错误顺序解析的异步调用）？
```

---

## 执行步骤

### 步骤 1：映射状态存储

在审计任何触点之前，构建每个状态存储操作的副作用映射：

```
对于范围内的每个 Zustand store / React context：
  对于每个 action/setter：
    - 它设置什么字段？
    - 它是否作为副作用重置其他字段？
    - 文档：actionName → {sets: [...], resets: [...]}
```

这是关键的参考。不知道 `selectThread` 重置 `composeMode`，"新邮件" bug 是不可见的。

**输出格式：**
```
STORE: emailStore
  setComposeMode(bool) → sets: {composeMode}
  selectThread(thread|null) → sets: {selectedThread, selectedThreadId, messages, drafts, selectedDraft, summary} RESETS: {composeMode: false, composeData: null, redraftOpen: false}
  setDraftGenerating(bool) → sets: {draftGenerating}
  ...

危险重置（清除它们不拥有的状态的操作）：
  selectThread → 重置 composeMode（由 setComposeMode 拥有）
  reset → 重置一切
```

### 步骤 2：审计每个触点

对于目标区域中的每个按钮/切换/表单提交：

```
TOUCHPOINT: [按钮标签] 在 [Component:line]
  HANDLER: onClick → {
    调用 1: functionA() → 设置 {X: true}
    调用 2: functionB() → 设置 {Y: null} 重置 {X: false}  ← 冲突
  }
  预期：用户看到 [按钮标签承诺的描述]
  实际：X 是 false 因为 functionB 重置了它
  结论：BUG — [描述]
```

**检查这些 bug 模式：**

#### 模式 1：顺序撤销
```
handler() {
  setState_A(true)     // 设置 X = true
  setState_B(null)     // 副作用：重置 X = false
}
// 结果：X 是 false。第一次调用毫无意义。
```

#### 模式 2：异步竞态
```
handler() {
  fetchA().then(() => setState({ loading: false }))
  fetchB().then(() => setState({ loading: true }))
}
// 结果：最终 loading 状态取决于哪个先解析
```

#### 模式 3：过期闭包
```
const [count, setCount] = useState(0)
const handler = useCallback(() => {
  setCount(count + 1)  // 捕获过期的 count
  setCount(count + 1)  // 相同的过期 count — 增加 1，不是 2
}, [count])
```

#### 模式 4：缺少状态转换
```
// 按钮说"保存"但处理程序只验证，从不实际保存
// 按钮说"删除"但处理程序设置标志而不调用 API
// 按钮说"发送"但 API 端点已移除/损坏
```

#### 模式 5：条件死路径
```
handler() {
  if (someState) {        // someState 在此时总是 false
    doTheActualThing()    // 永远不会到达
  }
}
```

#### 模式 6：useEffect 干扰
```
// 按钮设置 stateX = true
// 一个 useEffect 监视 stateX 并将其重置为 false
// 用户看到什么也没发生
```

### 步骤 3：报告

对于发现的每个 bug：

```
CLICK-PATH-NNN: [严重性：CRITICAL/HIGH/MEDIUM/LOW]
  触点：[按钮标签] 在 [file:line]
  模式：[Sequential Undo / Async Race / Stale Closure / Missing Transition / Dead Path / useEffect Interference]
  处理程序：[函数名或内联]
  追踪：
    1. [调用] → 设置 {field: value}
    2. [调用] → 重置 {field: value}  ← 冲突
  预期：[用户期望什么]
  实际：[实际发生什么]
  修复：[具体修复]
```

---

## 范围控制

此审计成本较高。适当限定范围：

- **完整应用审计：** 启动或重大重构后使用。每页启动并行代理。
- **单页审计：** 构建新页面或用户报告按钮损坏后使用。
- **存储聚焦审计：** 修改 Zustand store 后使用 — 审计更改操作的所有消费者。

### 完整应用的推荐代理分割：

```
Agent 1：映射所有状态存储（步骤 1）— 这是所有其他代理的共享上下文
Agent 2：Dashboard（Tasks, Notes, Journal, Ideas）
Agent 3：Chat（DanteChatColumn, JustChatPage）
Agent 4：Emails（ThreadList, DraftArea, EmailsPage）
Agent 5：Projects（ProjectsPage, ProjectOverviewTab, NewProjectWizard）
Agent 6：CRM（所有子标签）
Agent 7：Profile, Settings, Vault, Notifications
Agent 8：Management Suite（所有页面）
```

Agent 1 必须先完成。它的输出是所有其他代理的输入。

---

## 何时使用

- 系统调试发现"没有 bug"但用户报告 UI 损坏后
- 修改任何 Zustand store 操作后（检查所有调用者）
- 任何触及共享状态的重构后
- 发布前，在关键用户流程上
- 当按钮"什么也不做"时 — 这是针对那个的工具

## 何时不使用

- 对于 API 级别的 bug（错误的响应形状、缺少端点）— 使用 systematic-debugging
- 对于样式/布局问题 — 视觉检查
- 对于性能问题 — 性能分析工具

---

## 与其他技能的集成

- 在 `/superpowers:systematic-debugging` 之后运行（它发现其他 54 种 bug 类型）
- 在 `/superpowers:verification-before-completion` 之前运行（它验证修复有效）
- 输入到 `/superpowers:test-driven-development` — 这里发现的每个 bug 都应该有一个测试

---

## 示例：启发此技能的 Bug

**ThreadList.tsx "新邮件"按钮：**
```
onClick={() => {
  useEmailStore.getState().setComposeMode(true)   // ✓ 设置 composeMode = true
  useEmailStore.getState().selectThread(null)      // ✗ 重置 composeMode = false
}}
```

存储定义：
```
selectThread: (thread) => set({
  selectedThread: thread,
  selectedThreadId: thread?.id ?? null,
  messages: [],
  drafts: [],
  selectedDraft: null,
  summary: null,
  composeMode: false,     // ← 这个静默重置杀死了按钮
  composeData: null,
  redraftOpen: false,
})
```

**系统调试错过了它**，因为：
- 按钮有 onClick 处理程序（不是死的）
- 两个函数都存在（没有缺少连接）
- 两个函数都不崩溃（没有运行时错误）
- 数据类型正确（没有类型不匹配）

**点击路径审计捕获它**，因为：
- 步骤 1 映射 `selectThread` 重置 `composeMode`
- 步骤 2 追踪处理程序：调用 1 设置 true，调用 2 重置 false
- 结论：Sequential Undo — 最终状态与按钮意图矛盾
