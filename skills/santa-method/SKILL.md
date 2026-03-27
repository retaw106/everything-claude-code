---
name: santa-method
description: "多代理对抗性验证与收敛循环。两个独立审查代理必须在输出发布前都通过。"
origin: "Ronald Skelton - Founder, RapportScore.ai"
---

# Santa 方法

多代理对抗性验证框架。列个清单，检查两次。如果是淘气的，修复到变乖为止。

核心洞察：单个代理审查自己的输出共享产生该输出的相同偏见、知识差距和系统性错误。两个没有共享上下文的独立审查者打破这种失败模式。

## 何时启用

在以下情况下调用此技能：
- 输出将被发布、部署或被最终用户消费
- 必须强制执行合规、监管或品牌约束
- 代码在无人审查的情况下发布到生产
- 内容准确性很重要（技术文档、教育材料、面向客户的文案）
- 大规模批量生成，抽查会错过系统性模式
- 幻觉风险升高（声明、统计数据、API 引用、法律语言）

不要用于内部草稿、探索性研究或具有确定性验证的任务（对这些使用构建/测试/lint 流水线）。

## 架构

```
┌─────────────┐
│  生成器      │  阶段 1：列个清单
│  (代理 A)    │  产生可交付物
└──────┬───────┘
       │ output
       ▼
┌──────────────────────────────┐
│     双重独立审查               │  阶段 2：检查两次
│                                │
│  ┌───────────┐ ┌───────────┐  │  两个代理，相同评分标准，
│  │ 审查者 B  │ │ 审查者 C  │  │  无共享上下文
│  └─────┬─────┘ └─────┬─────┘  │
│        │              │        │
└────────┼──────────────┼────────┘
         │              │
         ▼              ▼
┌──────────────────────────────┐
│        裁决门                  │  阶段 3：淘气还是乖
│                                │
│  B 通过 且 C 通过 → 乖          │  两者都必须通过。
│  否则 → 淘气                   │  无例外。
└──────┬──────────────┬─────────┘
       │              │
    乖              淘气
       │              │
       ▼              ▼
   [ 发布 ]    ┌─────────────┐
               │  修复循环    │  阶段 4：修复到变乖
               │              │
               │ iteration++  │  收集所有标记。
               │ if i > MAX:  │  修复所有问题。
               │   escalate   │  重新运行两个审查者。
               │ else:        │  循环直到收敛。
               │   goto Ph.2  │
               └──────────────┘
```

## 阶段详情

### 阶段 1：列个清单（生成）

执行主要任务。不对你的正常生成工作流做任何更改。Santa 方法是生成后验证层，不是生成策略。

```python
# 生成器正常运行
output = generate(task_spec)
```

### 阶段 2：检查两次（独立双重审查）

并行生成两个审查代理。关键不变量：

1. **上下文隔离** — 两个审查者都不看对方的评估
2. **相同评分标准** — 两者接收相同的评估标准
3. **相同输入** — 两者接收原始规格和生成的输出
4. **结构化输出** — 每个返回类型化裁决，不是散文

```python
REVIEWER_PROMPT = """
你是一个独立的质量审查者。你还没有看到此输出的任何其他审查。

## 任务规格
{task_spec}

## 待审查输出
{output}

## 评分标准
{rubric}

## 指示
根据每个评分标准评估输出。对于每个：
- PASS：标准完全满足，无问题
- FAIL：发现具体问题（引用确切问题）

以结构化 JSON 返回你的评估：
{
  "verdict": "PASS" | "FAIL",
  "checks": [
    {"criterion": "...", "result": "PASS|FAIL", "detail": "..."}
  ],
  "critical_issues": ["..."],   // 必须修复的阻塞项
  "suggestions": ["..."]         // 非阻塞改进
}

要严格。你的工作是发现问题，不是批准。
"""
```

```python
# 并行生成审查者（Claude Code 子代理）
review_b = Agent(prompt=REVIEWER_PROMPT.format(...), description="Santa 审查者 B")
review_c = Agent(prompt=REVIEWER_PROMPT.format(...), description="Santa 审查者 C")

# 两者并发运行 — 彼此看不到对方
```

### 评分标准设计

评分标准是最重要的输入。模糊的标准产生模糊的审查。每个标准必须有客观的通过/失败条件。

| 标准 | 通过条件 | 失败信号 |
|------|----------|----------|
| 事实准确性 | 所有声明可对照源材料或常识验证 | 编造的统计数据、错误的版本号、不存在的 API |
| 无幻觉 | 无虚构的实体、引言、URL 或引用 | 指向不存在页面的链接、无来源的归因引言 |
| 完整性 | 规格中的每个要求都被处理 | 缺失章节、跳过的边缘情况、不完整的覆盖 |
| 合规性 | 通过所有项目特定约束 | 使用了禁用术语、语气违规、监管不合规 |
| 内部一致性 | 输出内无矛盾 | A 节说 X，B 节说非 X |
| 技术正确性 | 代码编译/运行，算法合理 | 语法错误、逻辑 bug、错误的复杂度声明 |

#### 领域特定评分标准扩展

**内容/营销：**
- 品牌语气符合
- SEO 要求满足（关键词密度、meta 标签、结构）
- 无竞争对手商标滥用
- CTA 存在且正确链接

**代码：**
- 类型安全（无 `any` 泄漏，正确的 null 处理）
- 错误处理覆盖
- 安全性（代码中无机密、输入验证、注入防护）
- 新路径的测试覆盖

**合规敏感（受监管、法律、金融）：**
- 无结果保证或无根据的声明
- 存在必需的免责声明
- 仅使用批准的术语
- 适当的管辖区域语言

### 阶段 3：淘气还是乖（裁决门）

```python
def santa_verdict(review_b, review_c):
    """两个审查者都必须通过。无部分学分。"""
    if review_b.verdict == "PASS" and review_c.verdict == "PASS":
        return "NICE"  # 发布它

    # 合并两个审查者的标记，去重
    all_issues = dedupe(review_b.critical_issues + review_c.critical_issues)
    all_suggestions = dedupe(review_b.suggestions + review_c.suggestions)

    return "NAUGHTY", all_issues, all_suggestions
```

为什么两者都必须通过：如果只有一个审查者发现问题，那个问题是真实的。另一个审查者的盲点正是 Santa 方法存在要消除的失败模式。

### 阶段 4：修复到变乖（收敛循环）

```python
MAX_ITERATIONS = 3

for iteration in range(MAX_ITERATIONS):
    verdict, issues, suggestions = santa_verdict(review_b, review_c)

    if verdict == "NICE":
        log_santa_result(output, iteration, "passed")
        return ship(output)

    # 修复所有关键问题（建议是可选的）
    output = fix_agent.execute(
        output=output,
        issues=issues,
        instruction="仅修复标记的问题。不要重构或添加未请求的更改。"
    )

    # 在修复的输出上重新运行两个审查者（新鲜代理，无前几轮记忆）
    review_b = Agent(prompt=REVIEWER_PROMPT.format(output=output, ...))
    review_c = Agent(prompt=REVIEWER_PROMPT.format(output=output, ...))

# 耗尽迭代 — 升级
log_santa_result(output, MAX_ITERATIONS, "escalated")
escalate_to_human(output, issues)
```

关键：每轮审查使用**新鲜代理**。审查者不能携带前几轮的记忆，因为先前的上下文会产生锚定偏见。

## 实现模式

### 模式 A：Claude Code 子代理（推荐）

子代理提供真正的上下文隔离。每个审查者是具有无共享状态的独立进程。

```bash
# 在 Claude Code 会话中，使用 Agent 工具生成审查者
# 两个代理并行运行以提高速度
```

```python
# Agent 工具调用的伪代码
reviewer_b = Agent(
    description="Santa 审查 B",
    prompt=f"审查此输出的质量...\n\n评分标准：\n{rubric}\n\n输出：\n{output}"
)
reviewer_c = Agent(
    description="Santa 审查 C",
    prompt=f"审查此输出的质量...\n\n评分标准：\n{rubric}\n\n输出：\n{output}"
)
```

### 模式 B：顺序内联（后备）

当子代理不可用时，用显式上下文重置模拟隔离：

1. 生成输出
2. 新上下文："你是审查者 1。仅针对此评分标准评估。发现问题。"
3. 逐字记录发现
4. 完全清除上下文
5. 新上下文："你是审查者 2。仅针对此评分标准评估。发现问题。"
6. 比较两个审查，修复，重复

子代理模式严格优越 — 内联模拟有审查者之间上下文泄漏的风险。

### 模式 C：批量抽样

对于大批量（100+ 项），对每项完整 Santa 成本过高。使用分层抽样：

1. 在随机样本上运行 Santa（批量的 10-15%，最少 5 项）
2. 按类型分类失败（幻觉、合规、完整性等）
3. 如果出现系统性模式，对整个批量应用针对性修复
4. 重新抽样并重新验证修复的批量
5. 继续直到干净样本通过

```python
import random

def santa_batch(items, rubric, sample_rate=0.15):
    sample = random.sample(items, max(5, int(len(items) * sample_rate)))

    for item in sample:
        result = santa_full(item, rubric)
        if result.verdict == "NAUGHTY":
            pattern = classify_failure(result.issues)
            items = batch_fix(items, pattern)  # 修复所有匹配模式的项
            return santa_batch(items, rubric)   # 重新抽样

    return items  # 干净样本 → 发布批量
```

## 失败模式与缓解

| 失败模式 | 症状 | 缓解 |
|---------|------|------|
| 无限循环 | 修复后审查者不断发现新问题 | 最大迭代上限（3）。升级。 |
| 橡皮图章 | 两个审查者通过一切 | 对抗性提示："你的工作是发现问题，不是批准。" |
| 主观漂移 | 审查者标记风格偏好，而非错误 | 紧凑评分标准，仅客观通过/失败条件 |
| 修复回归 | 修复问题 A 引入问题 B | 每轮新鲜审查者捕获回归 |
| 审查者一致性偏见 | 两个审查者错过相同的东西 | 通过独立性缓解，未消除。对于关键输出，添加第三个审查者或人工抽查。 |
| 成本爆炸 | 大输出上太多迭代 | 批量抽样模式。每个验证周期的预算上限。 |

## 与其他技能的集成

| 技能 | 关系 |
|------|------|
| 验证循环 | 用于确定性检查（构建、lint、测试）。Santa 用于语义检查（准确性、幻觉）。先运行 verification-loop，再运行 Santa。 |
| 评估框架 | Santa 方法结果输入评估指标。跟踪 Santa 运行的 pass@k 以随时间测量生成器质量。 |
| 持续学习 v2 | Santa 发现成为直觉。同一标准的重复失败 → 学习行为以避免该模式。 |
| 战略压缩 | 在压缩前运行 Santa。不要在验证中途丢失审查上下文。 |

## 指标

跟踪这些以测量 Santa 方法有效性：

- **首次通过率**：第 1 轮通过 Santa 的输出百分比（目标：>70%）
- **收敛平均迭代**：到乖的平均轮数（目标：<1.5）
- **问题分类**：失败类型分布（幻觉 vs 完整性 vs 合规）
- **审查者一致性**：两个审查者都标记的问题 vs 只有一个标记的百分比（低一致性 = 评分标准需要收紧）
- **逃逸率**：Santa 应该捕获但发布后发现的问题（目标：0）

## 成本分析

Santa 方法每验证周期成本约为单独生成令牌成本的 2-3 倍。对于大多数高风险输出，这是划算的：

```
Santa 成本 = (生成令牌) + 2×(每轮审查令牌) × (平均轮数)
不 Santa 的成本 = (声誉损害) + (纠正努力) + (信任侵蚀)
```

对于批量操作，抽样模式将成本降低到完整验证的约 15-20%，同时捕获 >90% 的系统性问题。
