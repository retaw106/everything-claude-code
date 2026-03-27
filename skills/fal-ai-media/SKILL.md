---
name: fal-ai-media
description: 通过 fal.ai MCP 的统一媒体生成 — 图片、视频和音频。涵盖文本生成图片(Nano Banana)、文本/图片生成视频(Seedance, Kling, Veo 3)、文本生成语音(CSM-1B)和视频生成音频(ThinkSound)。当用户想要使用 AI 生成图片、视频或音频时使用。
origin: ECC
---

# fal.ai 媒体生成

通过 MCP 使用 fal.ai 模型生成图片、视频和音频。

## 何时激活

- 用户想要从文本提示生成图片
- 从文本或图片创建视频
- 生成语音、音乐或音效
- 任何媒体生成任务
- 用户说"生成图片"、"创建视频"、"文本转语音"、"制作缩略图"或类似表述

## MCP 要求

需要配置 fal.ai MCP 服务器。添加到 `~/.claude.json`:

```json
"fal-ai": {
  "command": "npx",
  "args": ["-y", "fal-ai-mcp-server"],
  "env": { "FAL_KEY": "YOUR_FAL_KEY_HERE" }
}
```

在 [fal.ai](https://fal.ai) 获取 API 密钥。

## MCP 工具

fal.ai MCP 提供以下工具:
- `search` — 按关键词查找可用模型
- `find` — 获取模型详情和参数
- `generate` — 使用参数运行模型
- `result` — 检查异步生成状态
- `status` — 检查任务状态
- `cancel` — 取消正在运行的任务
- `estimate_cost` — 估算生成成本
- `models` — 列出热门模型
- `upload` — 上传文件作为输入

---

## 图片生成

### Nano Banana 2 (快速)
最适用于: 快速迭代、草稿、文本生成图片、图片编辑。

```
generate(
  app_id: "fal-ai/nano-banana-2",
  input_data: {
    "prompt": "未来城市景观在黄昏, 赛博朋克风格",
    "image_size": "landscape_16_9",
    "num_images": 1,
    "seed": 42
  }
)
```

### Nano Banana Pro (高保真)
最适用于: 产品图片、真实感、排版、详细提示词。

```
generate(
  app_id: "fal-ai/nano-banana-pro",
  input_data: {
    "prompt": "大理石表面上的无线耳机专业产品照片, 演播室灯光",
    "image_size": "square",
    "num_images": 1,
    "guidance_scale": 7.5
  }
)
```

### 常用图片参数

| 参数 | 类型 | 选项 | 说明 |
|-------|------|---------|-------|
| `prompt` | string | 必需 | 描述你想要的内容 |
| `image_size` | string | `square`, `portrait_4_3`, `landscape_16_9`, `portrait_16_9`, `landscape_4_3` | 宽高比 |
| `num_images` | number | 1-4 | 生成数量 |
| `seed` | number | 任意整数 | 可重现性 |
| `guidance_scale` | number | 1-20 | 遵循提示词的紧密程度(越高 = 越字面) |

### 图片编辑
使用 Nano Banana 2 配合输入图片进行重绘、扩展绘制或风格迁移:

```
# 首先上传源图片
upload(file_path: "/path/to/image.png")

# 然后使用图片输入生成
generate(
  app_id: "fal-ai/nano-banana-2",
  input_data: {
    "prompt": "相同场景但用水彩风格",
    "image_url": "<uploaded_url>",
    "image_size": "landscape_16_9"
  }
)
```

---
## 视频生成

### Seedance 1.0 Pro (ByteDance)
最适用于: 文本生成视频、图片生成视频, 高动态质量。

```
generate(
  app_id: "fal-ai/seedance-1-0-pro",
  input_data: {
    "prompt": "黄金时刻无人机飞越山间湖泊, 电影感",
    "duration": "5s",
    "aspect_ratio": "16:9",
    "seed": 42
  }
)
```

### Kling Video v3 Pro
最适用于: 文本/图片生成视频, 原生音频生成。

```
generate(
  app_id: "fal-ai/kling-video/v3/pro",
  input_data: {
    "prompt": "海浪拍打岩石海岸, 戏剧性云层",
    "duration": "5s",
    "aspect_ratio": "16:9"
  }
)
```

### Veo 3 (Google DeepMind)
最适用于: 带生成声音的视频, 高视觉质量。

```
generate(
  app_id: "fal-ai/veo-3",
  input_data: {
    "prompt": "夜晚繁忙的东京街市, 霓虹灯牌, 人群噪音",
    "aspect_ratio": "16:9"
  }
)
```

### 图片生成视频
从现有图片开始:

```
generate(
  app_id: "fal-ai/seedance-1-0-pro",
  input_data: {
    "prompt": "镜头缓慢拉远, 微风吹动树木",
    "image_url": "<uploaded_image_url>",
    "duration": "5s"
  }
)
```

### 视频参数

| 参数 | 类型 | 选项 | 说明 |
|-------|------|---------|-------|
| `prompt` | string | 必需 | 描述视频内容 |
| `duration` | string | `"5s"`, `"10s"` | 视频时长 |
| `aspect_ratio` | string | `"16:9"`, `"9:16"`, `"1:1"` | 画面比例 |
| `seed` | number | 任意整数 | 可重现性 |
| `image_url` | string | URL | 图片生成视频的源图片 |

---
## 音频生成

### CSM-1B (对话式语音)
具有自然、对话质量的文本转语音。

```
generate(
  app_id: "fal-ai/csm-1b",
  input_data: {
    "text": "你好, 欢迎来到演示。让我展示这是如何工作的。",
    "speaker_id": 0
  }
)
```

### ThinkSound (视频生成音频)
根据视频内容生成匹配的音频。

```
generate(
  app_id: "fal-ai/thinksound",
  input_data: {
    "video_url": "<video_url>",
    "prompt": "森林环境音, 鸟鸣声"
  }
)
```

### ElevenLabs (通过 API, 无 MCP)
对于专业语音合成, 直接使用 ElevenLabs

```python
import os
import requests

resp = requests.post(
    "https://api.elevenlabs.io/v1/text-to-speech/<voice_id>",
    headers={
        "xi-api-key": os.environ["ELEVENLABS_API_KEY"],
        "Content-Type": "application/json"
    },
    json={
        "text": "你的文本在这里",
        "model_id": "eleven_turbo_v2_5",
        "voice_settings": {"stability": 1.5, "similarity_boost": 1.75}
    }
)
with open("output.mp3", "wb") as f:
    f.write(resp.content)
```

### VideoDB 生成式音频
如果配置了 VideoDB, 使用其生成式音频

```python
# 语音生成
audio = coll.generate_voice(text="你的解说在这里", voice="alloy")

# 音乐生成
music = coll.generate_music(prompt="欢快的电子背景音乐", duration=30)

# 音效生成
sfx = coll.generate_sound_effect(prompt="雷声后跟随雨声")
```

---
## 成本估算

生成前, 检查预估成本

```
estimate_cost(
  estimate_type: "unit_price",
  endpoints: {
    "fal-ai/nano-banana-pro": {
      "unit_quantity": 1
    }
  }
)
```

## 模型发现

为特定任务查找模型

```
search(query: "text to video")
find(endpoint_ids: ["fal-ai/seedance-1-0-pro"])
models()
```

## 技巧

- 迭代提示词时使用 `seed` 获得可重现的结果
- 提示词迭代从低成本模型(Nano Banana 2)开始, 最终版本再切换到 Pro
- 视频提示词要描述性强但简洁 — 专注于动作和场景
- 图片生成视频比纯文本生成视频产生更可控的结果
- 运行昂贵的视频生成前检查 `estimate_cost`

## 相关技能

- `videodb` — 视频处理、编辑和流式传输
- `video-editing` — AI 辅助视频编辑工作流
- `content-engine` — 社交平台内容创建
