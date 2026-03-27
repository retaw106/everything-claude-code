---
name: iterative-retrieval
description: 渐进式细化上下文检索的模式，用于解决子代理上下文问题
origin: ECC
---

# 迭代检索模式

解决多代理工作流中的"上下文问题"，即子代理在开始工作之前不知道需要什么上下文。

## 何时启用

- 生成需要无法预先预测的代码库上下文的子代理
- 构建上下文渐进细化的多代理工作流
- 在代理任务中遇到"上下文太大"或"缺失上下文"失败
- 为代码探索设计类似 RAG 的检索流水线
- 优化代理编排中的 token 使用

## 问题

子代理在有限上下文中生成。它们不知道：
- 哪些文件包含相关代码
- 代码库中存在什么模式
- 项目使用什么术语

标准方法会失败：
- **发送所有内容**：超出上下文限制
- **什么都不发送**：代理缺少关键信息
- **猜测需要什么**：经常错误

## 解决方案：迭代检索

一个 4 阶段循环，渐进式细化上下文：

```
┌─────────────────────────────────────────────┐
│                                             │
│   ┌──────────┐      ┌──────────┐            │
│   │  调度    │─────▶│  评估    │            │
│   └──────────┘      └──────────┘            │
│        ▲                  │                 │
│        │                  ▼                 │
│   ┌──────────┐      ┌──────────┐            │
│   │  循环    │◀─────│  细化    │            │
│   └──────────┘      └──────────┘            │
│                                             │
│        最多 3 个周期，然后继续               │
└─────────────────────────────────────────────┘
```

### Phase 1: 调度

初始广泛查询以收集候选文件：

```javascript
// 从高层意图开始
const initialQuery = {
  patterns: ['src/**/*.ts', 'lib/**/*.ts'],
  keywords: ['authentication', 'user', 'session'],
  excludes: ['*.test.ts', '*.spec.ts']
};

// 调度到检索代理
const candidates = await retrieveFiles(initialQuery);
```

### Phase 2: 评估

评估检索内容的相关性：

```javascript
function evaluateRelevance(files, task) {
  return files.map(file => ({
    path: file.path,
    relevance: scoreRelevance(file.content, task),
    reason: explainRelevance(file.content, task),
    missingContext: identifyGaps(file.content, task)
  }));
}
```

评分标准：
- **高 (0.8-1.0)**：直接实现目标功能
- **中 (0.5-0.7)**：包含相关模式或类型
- **低 (0.2-0.4)**：间接相关
- **无 (0-0.2)**：不相关，排除

### Phase 3: 细化

根据评估更新搜索条件：

```javascript
function refineQuery(evaluation, previousQuery) {
  return {
    // 添加在高相关性文件中发现的新模式
    patterns: [...previousQuery.patterns, ...extractPatterns(evaluation)],

    // 添加代码库中发现的术语
    keywords: [...previousQuery.keywords, ...extractKeywords(evaluation)],

    // 排除确认不相关的路径
    excludes: [...previousQuery.excludes, ...evaluation
      .filter(e => e.relevance < 0.2)
      .map(e => e.path)
    ],

    // 针对特定缺口
    focusAreas: evaluation
      .flatMap(e => e.missingContext)
      .filter(unique)
  };
}
```

### Phase 4: 循环

用细化的条件重复（最多 3 个周期）：

```javascript
async function iterativeRetrieve(task, maxCycles = 3) {
  let query = createInitialQuery(task);
  let bestContext = [];

  for (let cycle = 0; cycle < maxCycles; cycle++) {
    const candidates = await retrieveFiles(query);
    const evaluation = evaluateRelevance(candidates, task);

    // 检查是否有足够的上下文
    const highRelevance = evaluation.filter(e => e.relevance >= 0.7);
    if (highRelevance.length >= 3 && !hasCriticalGaps(evaluation)) {
      return highRelevance;
    }

    // 细化并继续
    query = refineQuery(evaluation, query);
    bestContext = mergeContext(bestContext, highRelevance);
  }

  return bestContext;
}
```

## 实际示例

### 示例 1: Bug 修复上下文

```
任务: "修复认证令牌过期 bug"

周期 1:
  调度: 在 src/** 中搜索 "token", "auth", "expiry"
  评估: 发现 auth.ts (0.9), tokens.ts (0.8), user.ts (0.3)
  细化: 添加 "refresh", "jwt" 关键词；排除 user.ts

周期 2:
  调度: 搜索细化的术语
  评估: 发现 session-manager.ts (0.95), jwt-utils.ts (0.85)
  细化: 足够的上下文（2 个高相关性文件）

结果: auth.ts, tokens.ts, session-manager.ts, jwt-utils.ts
```

### 示例 2: 功能实现

```
任务: "为 API 端点添加速率限制"

周期 1:
  调度: 在 routes/** 中搜索 "rate", "limit", "api"
  评估: 无匹配 — 代码库使用 "throttle" 术语
  细化: 添加 "throttle", "middleware" 关键词

周期 2:
  调度: 搜索细化的术语
  评估: 发现 throttle.ts (0.9), middleware/index.ts (0.7)
  细化: 需要 router 模式

周期 3:
  调度: 搜索 "router", "express" 模式
  评估: 发现 router-setup.ts (0.8)
  细化: 足够的上下文

结果: throttle.ts, middleware/index.ts, router-setup.ts
```

## 与代理集成

在代理提示中使用：

```markdown
为该任务检索上下文时:
1. 从广泛的关键词搜索开始
2. 评估每个文件的相关性（0-1 评分）
3. 识别还缺少什么上下文
4. 细化搜索条件并重复（最多 3 个周期）
5. 返回相关性 >= 0.7 的文件
```

## 最佳实践

1. **从广泛开始，渐进收窄** — 不要过度指定初始查询
2. **学习代码库术语** — 第一周期通常揭示命名约定
3. **跟踪缺失内容** — 显式缺口识别驱动细化
4. **适可而止** — 3 个高相关性文件胜过 10 个平庸的
5. **自信排除** — 低相关性文件不会变得相关

## 相关

- [详细指南](https://x.com/affaanmustafa/status/2014040193557471352) - 子代理编排部分
- `continuous-learning` 技能 - 用于随时间改进的模式
- ECC 附带的代理定义（手动安装路径: `agents/`）
