---
name: claude-api
description: Python 和 TypeScript 的 Anthropic Claude API 模式。涵盖 Messages API、流式传输、工具使用、视觉、扩展思考、批处理、提示缓存和 Claude Agent SDK。适用于构建使用 Claude API 或 Anthropic SDK 的应用程序。
origin: ECC
---

# Claude API

使用 Anthropic Claude API 和 SDK 构建应用程序。

## 何时启用

- 构建调用 Claude API 的应用程序
- 代码导入 `anthropic` (Python) 或 `@anthropic-ai/sdk` (TypeScript)
- 用户询问 Claude API 模式、工具使用、流式传输或视觉
- 使用 Claude Agent SDK 实现智能体工作流
- 优化 API 成本、Token 使用或延迟

## 模型选择

| Model | ID | 适用场景 |
|-------|-----|----------|
| Opus 4.6 | `claude-opus-4-6` | 复杂推理、架构、研究 |
| Sonnet 4.6 | `claude-sonnet-4-6` | 平衡编码、大多数开发任务 |
| Haiku 4.5 | `claude-haiku-4-5-20251001` | 快速响应、高吞吐量、成本敏感 |

默认使用 Sonnet 4.6，除非任务需要深度推理（Opus）或速度/成本优化（Haiku）。

## Python SDK

### 安装

```bash
pip install anthropic
```

### 基础消息

```python
import anthropic

client = anthropic.Anthropic()  # 从环境变量读取 ANTHROPIC_API_KEY

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "解释 Python 中的 async/await"}
    ]
)
print(message.content[0].text)
```

### 流式传输

```python
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "写一首关于编程的俳句"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

### 系统提示

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system="你是一位高级 Python 开发者。回答要简洁。",
    messages=[{"role": "user", "content": "审查这个函数"}]
)
```

## TypeScript SDK

### 安装

```bash
npm install @anthropic-ai/sdk
```

### 基础消息

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic(); // 从环境变量读取 ANTHROPIC_API_KEY

const message = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  messages: [
    { role: "user", content: "解释 TypeScript 中的 async/await" }
  ],
});
console.log(message.content[0].text);
```

### 流式传输

```typescript
const stream = client.messages.stream({
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  messages: [{ role: "user", content: "写一首俳句" }],
});

for await (const event of stream) {
  if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
    process.stdout.write(event.delta.text);
  }
}
```

## 工具使用

定义工具并让 Claude 调用它们：

```python
tools = [
    {
        "name": "get_weather",
        "description": "获取指定位置的当前天气",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "城市名称"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
            },
            "required": ["location"]
        }
    }
]

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "旧金山的天气怎么样？"}]
)

# 处理工具使用响应
for block in message.content:
    if block.type == "tool_use":
        # 使用 block.input 执行工具
        result = get_weather(**block.input)
        # 发送结果回去
        follow_up = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            tools=tools,
            messages=[
                {"role": "user", "content": "旧金山的天气怎么样？"},
                {"role": "assistant", "content": message.content},
                {"role": "user", "content": [
                    {"type": "tool_result", "tool_use_id": block.id, "content": str(result)}
                ]}
            ]
        )
```

## 视觉

发送图片进行分析：

```python
import base64

with open("diagram.png", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode("utf-8")

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": image_data}},
            {"type": "text", "text": "描述这个图表"}
        ]
    }]
)
```

## 扩展思考

用于复杂推理任务：

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000
    },
    messages=[{"role": "user", "content": "逐步解决这个数学问题..."}]
)

for block in message.content:
    if block.type == "thinking":
        print(f"思考: {block.thinking}")
    elif block.type == "text":
        print(f"答案: {block.text}")
```

## 提示缓存

缓存大型系统提示或上下文以降低成本：

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {"type": "text", "text": large_system_prompt, "cache_control": {"type": "ephemeral"}}
    ],
    messages=[{"role": "user", "content": "关于缓存上下文的问题"}]
)
# 检查缓存使用情况
print(f"缓存读取: {message.usage.cache_read_input_tokens}")
print(f"缓存创建: {message.usage.cache_creation_input_tokens}")
```

## Batches API

以 50% 成本降低异步处理大批量请求：

```python
import time

batch = client.messages.batches.create(
    requests=[
        {
            "custom_id": f"request-{i}",
            "params": {
                "model": "claude-sonnet-4-6",
                "max_tokens": 1024,
                "messages": [{"role": "user", "content": prompt}]
            }
        }
        for i, prompt in enumerate(prompts)
    ]
)

# 轮询等待完成
while True:
    status = client.messages.batches.retrieve(batch.id)
    if status.processing_status == "ended":
        break
    time.sleep(30)

# 获取结果
for result in client.messages.batches.results(batch.id):
    print(result.result.message.content[0].text)
```

## Claude Agent SDK

构建多步骤智能体：

```python
# 注意：Agent SDK API 可能变化 — 请查看官方文档
import anthropic

# 将工具定义为函数
tools = [{
    "name": "search_codebase",
    "description": "在代码库中搜索相关代码",
    "input_schema": {
        "type": "object",
        "properties": {"query": {"type": "string"}},
        "required": ["query"]
    }
}]

# 使用工具调用运行智能体循环
client = anthropic.Anthropic()
messages = [{"role": "user", "content": "审查认证模块的安全问题"}]

while True:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=4096,
        tools=tools,
        messages=messages,
    )
    if response.stop_reason == "end_turn":
        break
    # 处理工具调用并继续循环
    messages.append({"role": "assistant", "content": response.content})
    # ... 执行工具并追加 tool_result 消息
```

## 成本优化

| 策略 | 节省 | 适用场景 |
|------|------|----------|
| 提示缓存 | 缓存 Token 高达 90% | 重复的系统提示或上下文 |
| Batches API | 50% | 非时间敏感的批量处理 |
| 用 Haiku 代替 Sonnet | ~75% | 简单任务、分类、提取 |
| 减少 max_tokens | 可变 | 确定输出会很短时 |
| 流式传输 | 无（相同成本） | 更好的用户体验，相同价格 |

## 错误处理

```python
import time

from anthropic import APIError, RateLimitError, APIConnectionError

try:
    message = client.messages.create(...)
except RateLimitError:
    # 退避并重试
    time.sleep(60)
except APIConnectionError:
    # 网络问题，带退避重试
    pass
except APIError as e:
    print(f"API 错误 {e.status_code}: {e.message}")
```

## 环境设置

```bash
# 必需
export ANTHROPIC_API_KEY="your-api-key-here"

# 可选：设置默认模型
export ANTHROPIC_MODEL="claude-sonnet-4-6"
```

绝不要硬编码 API 密钥。始终使用环境变量。
