---
name: videodb
description: 查看、理解、操作视频和音频。查看 — 从本地文件、URL、RTSP/直播流或实时录制桌面摄取内容；返回实时上下文和可播放流链接。理解 — 提取帧、构建视觉/语义/时间索引，并搜索带时间戳的时刻和自动剪辑。操作 — 转码和规范化（编解码器、fps、分辨率、宽高比），执行时间线编辑（字幕、文本/图像叠加、品牌、音频叠加、配音、翻译），生成媒体资产（图像、音频、视频），并为直播流或桌面捕获中的事件创建实时警报。
origin: ECC
allowed-tools: Read Grep Glob Bash(python:*)
argument-hint: "[任务描述]"
---

# VideoDB 技能

**视频、直播流和桌面会话的感知 + 记忆 + 操作。**

## 何时使用

### 桌面感知
- 启动/停止**桌面会话**，捕获**屏幕、麦克风和系统音频**
- 流式传输**实时上下文**并存储**情节会话记忆**
- 对屏幕上说的内容和发生的事情运行**实时警报/触发器**
- 生成**会话摘要**、可搜索的时间线和**可播放的证据链接**

### 视频摄取 + 流
- 摄取**文件或 URL** 并返回**可播放的 Web 流链接**
- 转码/规范化：**编解码器、比特率、fps、分辨率、宽高比**

### 索引 + 搜索（时间戳 + 证据）
- 构建**视觉**、**语音**和**关键词**索引
- 搜索并返回精确时刻及其**时间戳**和**可播放证据**
- 从搜索结果自动创建**剪辑**

### 时间线编辑 + 生成
- 字幕：**生成**、**翻译**、**内嵌烧录**
- 叠加：**文本/图像/品牌**、动态字幕
- 音频：**背景音乐**、**配音**、**译制**
- 通过**时间线操作**进行程序化组合和导出

### 直播流（RTSP）+ 监控
- 连接**RTSP/直播源**
- 运行**实时视觉和语音理解**并为监控工作流发出**事件/警报**

## 工作原理

### 常见输入
- 本地**文件路径**、公开**URL** 或 **RTSP URL**
- 桌面捕获请求：**开始 / 停止 / 摘要会话**
- 期望的操作：获取理解上下文、转码规格、索引规格、搜索查询、剪辑范围、时间线编辑、警报规则

### 常见输出
- **流 URL**
- 带**时间戳**和**证据链接**的搜索结果
- 生成的资产：字幕、音频、图像、剪辑
- 直播流的**事件/警报负载**
- 桌面**会话摘要**和记忆条目

### 运行 Python 代码

在运行任何 VideoDB 代码之前，切换到项目目录并加载环境变量：

```python
from dotenv import load_dotenv
load_dotenv(".env")

import videodb
conn = videodb.connect()
```

这从以下位置读取 `VIDEO_DB_API_KEY`：
1. 环境（如果已导出）
2. 当前目录中项目的 `.env` 文件

如果密钥缺失，`videodb.connect()` 会自动引发 `AuthenticationError`。

当简短的内联命令可行时，不要编写脚本文件。

在编写内联 Python（`python -c "..."`）时，始终使用正确格式的代码 — 使用分号分隔语句并保持可读性。对于超过约 3 个语句的任何内容，改用 heredoc：

```bash
python << 'EOF'
from dotenv import load_dotenv
load_dotenv(".env")

import videodb
conn = videodb.connect()
coll = conn.get_collection()
print(f"Videos: {len(coll.get_videos())}")
EOF
```

### 设置

当用户要求"设置 videodb"或类似内容时：

### 1. 安装 SDK

```bash
pip install "videodb[capture]" python-dotenv
```

如果 `videodb[capture]` 在 Linux 上失败，安装不带 capture 扩展的版本：

```bash
pip install videodb python-dotenv
```

### 2. 配置 API 密钥

用户必须使用**任一**方法设置 `VIDEO_DB_API_KEY`：

- **在终端中导出**（启动 Claude 之前）：`export VIDEO_DB_API_KEY=your-key`
- **项目 `.env` 文件**：在项目的 `.env` 文件中保存 `VIDEO_DB_API_KEY=your-key`

在 [console.videodb.io](https://console.videodb.io) 获取免费 API 密钥（50 次免费上传，无需信用卡）。

**不要**自己读取、写入或处理 API 密钥。始终让用户设置它。

### 快速参考

### 上传媒体

```python
# URL
video = coll.upload(url="https://example.com/video.mp4")

# YouTube
video = coll.upload(url="https://www.youtube.com/watch?v=VIDEO_ID")

# 本地文件
video = coll.upload(file_path="/path/to/video.mp4")
```

### 转录稿 + 字幕

```python
# force=True 跳过视频已索引时的错误
video.index_spoken_words(force=True)
text = video.get_transcript_text()
stream_url = video.add_subtitle()
```

### 在视频内搜索

```python
from videodb.exceptions import InvalidRequestError

video.index_spoken_words(force=True)

# search() 在未找到结果时引发 InvalidRequestError。
# 始终用 try/except 包装并将"No results found"视为空。
try:
    results = video.search("product demo")
    shots = results.get_shots()
    stream_url = results.compile()
except InvalidRequestError as e:
    if "No results found" in str(e):
        shots = []
    else:
        raise
```

### 场景搜索

```python
import re
from videodb import SearchType, IndexType, SceneExtractionType
from videodb.exceptions import InvalidRequestError

# index_scenes() 没有 force 参数 — 如果场景
# 索引已存在，它会引发错误。从错误中提取现有索引 ID。
try:
    scene_index_id = video.index_scenes(
        extraction_type=SceneExtractionType.shot_based,
        prompt="Describe the visual content in this scene.",
    )
except Exception as e:
    match = re.search(r"id\s+([a-f0-9]+)", str(e))
    if match:
        scene_index_id = match.group(1)
    else:
        raise

# 使用 score_threshold 过滤低相关性噪声（推荐：0.3+）
try:
    results = video.search(
        query="person writing on a whiteboard",
        search_type=SearchType.semantic,
        index_type=IndexType.scene,
        scene_index_id=scene_index_id,
        score_threshold=0.3,
    )
    shots = results.get_shots()
    stream_url = results.compile()
except InvalidRequestError as e:
    if "No results found" in str(e):
        shots = []
    else:
        raise
```

### 时间线编辑

**重要：** 在构建时间线之前始终验证时间戳：
- `start` 必须 >= 0（负值会被静默接受但产生损坏的输出）
- `start` 必须 < `end`
- `end` 必须 <= `video.length`

```python
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, TextAsset, TextStyle

timeline = Timeline(conn)
timeline.add_inline(VideoAsset(asset_id=video.id, start=10, end=30))
timeline.add_overlay(0, TextAsset(text="The End", duration=3, style=TextStyle(fontsize=36)))
stream_url = timeline.generate_stream()
```

### 转码视频（分辨率 / 质量更改）

```python
from videodb import TranscodeMode, VideoConfig, AudioConfig

# 在服务器端更改分辨率、质量或宽高比
job_id = conn.transcode(
    source="https://example.com/video.mp4",
    callback_url="https://example.com/webhook",
    mode=TranscodeMode.economy,
    video_config=VideoConfig(resolution=720, quality=23, aspect_ratio="16:9"),
    audio_config=AudioConfig(mute=False),
)
```

### 重构宽高比（用于社交平台）

**警告：** `reframe()` 是一个慢速服务器端操作。对于长视频，可能需要几分钟并可能超时。最佳实践：
- 尽可能使用 `start`/`end` 限制为短片段
- 对于全长视频，使用 `callback_url` 进行异步处理
- 首先在 `Timeline` 上修剪视频，然后重构较短的结果

```python
from videodb import ReframeMode

# 始终优先重构短片段：
reframed = video.reframe(start=0, end=60, target="vertical", mode=ReframeMode.smart)

# 全长视频的异步重构（返回 None，结果通过 webhook）：
video.reframe(target="vertical", callback_url="https://example.com/webhook")

# 预设："vertical" (9:16)、"square" (1:1)、"landscape" (16:9)
reframed = video.reframe(start=0, end=60, target="square")

# 自定义尺寸
reframed = video.reframe(start=0, end=60, target={"width": 1280, "height": 720})
```

### 生成式媒体

```python
image = coll.generate_image(
    prompt="a sunset over mountains",
    aspect_ratio="16:9",
)
```

## 错误处理

```python
from videodb.exceptions import AuthenticationError, InvalidRequestError

try:
    conn = videodb.connect()
except AuthenticationError:
    print("Check your VIDEO_DB_API_KEY")

try:
    video = coll.upload(url="https://example.com/video.mp4")
except InvalidRequestError as e:
    print(f"Upload failed: {e}")
```

### 常见陷阱

| 场景 | 错误消息 | 解决方案 |
|------|----------|----------|
| 索引已索引的视频 | `Spoken word index for video already exists` | 使用 `video.index_spoken_words(force=True)` 跳过已索引的情况 |
| 场景索引已存在 | `Scene index with id XXXX already exists` | 用 `re.search(r"id\s+([a-f0-9]+)", str(e))` 从错误中提取现有的 `scene_index_id` |
| 搜索未找到匹配 | `InvalidRequestError: No results found` | 捕获异常并视为空结果（`shots = []`） |
| 重构超时 | 在长视频上无限期阻塞 | 使用 `start`/`end` 限制片段，或传递 `callback_url` 进行异步处理 |
| 时间线上的负时间戳 | 静默产生损坏的流 | 在创建 `VideoAsset` 之前始终验证 `start >= 0` |
| `generate_video()` / `create_collection()` 失败 | `Operation not allowed` 或 `maximum limit` | 计划门控功能 — 告知用户计划限制 |

## 示例

### 典型提示词
- "启动桌面捕获并在密码字段出现时发出警报。"
- "录制我的会话并在结束时生成可操作的摘要。"
- "摄取此文件并返回可播放的流链接。"
- "索引此文件夹并找到每个有人的场景，返回时间戳。"
- "生成字幕，内嵌烧录，并添加轻背景音乐。"
- "连接此 RTSP URL 并在有人进入区域时发出警报。"

### 屏幕录制（桌面捕获）

在录制会话期间使用 `ws_listener.py` 捕获 WebSocket 事件。桌面捕获仅支持 **macOS**。

#### 快速开始

1. **选择状态目录**：`STATE_DIR="${VIDEODB_EVENTS_DIR:-$HOME/.local/state/videodb}"`
2. **启动监听器**：`VIDEODB_EVENTS_DIR="$STATE_DIR" python scripts/ws_listener.py --clear "$STATE_DIR" &`
3. **获取 WebSocket ID**：`cat "$STATE_DIR/videodb_ws_id"`
4. **运行捕获代码**（参见 reference/capture.md 获取完整工作流）
5. **事件写入**：`$STATE_DIR/videodb_events.jsonl`

每当开始新的捕获运行时使用 `--clear`，以便陈旧的转录稿和视觉事件不会泄漏到新会话中。

#### 查询事件

```python
import json
import os
import time
from pathlib import Path

events_dir = Path(os.environ.get("VIDEODB_EVENTS_DIR", Path.home() / ".local" / "state" / "videodb"))
events_file = events_dir / "videodb_events.jsonl"
events = []

if events_file.exists():
    with events_file.open(encoding="utf-8") as handle:
        for line in handle:
            try:
                events.append(json.loads(line))
            except json.JSONDecodeError:
                continue

transcripts = [e["data"]["text"] for e in events if e.get("channel") == "transcript"]
cutoff = time.time() - 300
recent_visual = [
    e for e in events
    if e.get("channel") == "visual_index" and e["unix_ts"] > cutoff
]
```

## 附加文档

参考文档位于此 SKILL.md 文件相邻的 `reference/` 目录中。如果需要，使用 Glob 工具定位它。

- [reference/api-reference.md](reference/api-reference.md) - 完整的 VideoDB Python SDK API 参考
- [reference/search.md](reference/search.md) - 视频搜索深入指南（语音和基于场景）
- [reference/editor.md](reference/editor.md) - 时间线编辑、资产和组合
- [reference/streaming.md](reference/streaming.md) - HLS 流媒体和即时播放
- [reference/generative.md](reference/generative.md) - AI 驱动的媒体生成（图像、视频、音频）
- [reference/rtstream.md](reference/rtstream.md) - 直播流摄取工作流（RTSP/RTMP）
- [reference/rtstream-reference.md](reference/rtstream-reference.md) - RTStream SDK 方法和 AI 流水线
- [reference/capture.md](reference/capture.md) - 桌面捕获工作流
- [reference/capture-reference.md](reference/capture-reference.md) - 捕获 SDK 和 WebSocket 事件
- [reference/use-cases.md](reference/use-cases.md) - 常见视频处理模式和示例

**当 VideoDB 支持该操作时，不要使用 ffmpeg、moviepy 或本地编码工具。** 以下全部由 VideoDB 在服务器端处理 — 修剪、组合剪辑、叠加音频或音乐、添加字幕、文本/图像叠加、转码、分辨率更改、宽高比转换、为平台要求调整大小、转录和媒体生成。只有对于 reference/editor.md 中 Limitations 下列出的操作（转场、速度更改、裁剪/缩放、颜色分级、音量混合），才回退到本地工具。

### 何时使用什么

| 问题 | VideoDB 解决方案 |
|------|-----------------|
| 平台拒绝视频宽高比或分辨率 | `video.reframe()` 或带 `VideoConfig` 的 `conn.transcode()` |
| 需要为 Twitter/Instagram/TikTok 调整视频大小 | `video.reframe(target="vertical")` 或 `target="square"` |
| 需要更改分辨率（例如 1080p → 720p） | 带 `VideoConfig(resolution=720)` 的 `conn.transcode()` |
| 需要在视频上叠加音频/音乐 | `Timeline` 上的 `AudioAsset` |
| 需要添加字幕 | `video.add_subtitle()` 或 `CaptionAsset` |
| 需要组合/修剪剪辑 | `Timeline` 上的 `VideoAsset` |
| 需要生成配音、音乐或音效 | `coll.generate_voice()`、`generate_music()`、`generate_sound_effect()` |

## 来源

此技能的参考材料在 `skills/videodb/reference/` 下本地归档。
在运行时使用上面的本地副本，而不是跟踪外部仓库链接。
