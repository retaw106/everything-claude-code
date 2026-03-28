---
name: video-editing
description: AI 辅助的视频编辑工作流，用于剪辑、构建和增强真实素材。涵盖从原始捕获到 FFmpeg、Remotion、ElevenLabs、fal.ai 以及在 Descript 或 CapCut 中最终润色的完整流程。当用户想编辑视频、剪辑素材、创建视频博客或构建视频内容时使用。
origin: ECC
---

# 视频编辑

AI 辅助的真实素材编辑。不是从提示生成。快速编辑现有视频。

## 何时启用

- 用户想编辑、剪辑或构建视频素材
- 将长录像转换为短视频内容
- 从原始素材构建视频博客、教程或演示视频
- 为现有视频添加叠加层、字幕、音乐或旁白
- 为不同平台（YouTube、TikTok、Instagram）重新构图视频
- 用户说"编辑视频"、"剪辑这段素材"、"做个视频博客"或"视频工作流"

## 核心论点

AI 视频编辑在你不再要求它创建整个视频、而是用它来压缩、构建和增强真实素材时才有用。价值不在于生成。价值在于压缩。

## 流水线

```
Screen Studio / 原始素材
  → Claude / Codex
  → FFmpeg
  → Remotion
  → ElevenLabs / fal.ai
  → Descript 或 CapCut
```

每一层都有特定的工作。不要跳过层级。不要试图用一个工具做所有事情。

## 层级 1：捕获（Screen Studio / 原始素材）

收集源素材：
- **Screen Studio**：用于应用演示、编程会话、浏览器工作流的精美屏幕录制
- **原始相机素材**：视频博客素材、访谈、活动录制
- **通过 VideoDB 的桌面捕获**：带实时上下文的会话录制（见 `videodb` 技能）

输出：准备好组织的原始文件。

## 层级 2：组织（Claude / Codex）

使用 Claude Code 或 Codex：
- **转录和标记**：生成转录、识别主题和话题
- **规划结构**：决定保留什么、删掉什么、什么顺序合适
- **识别空白部分**：找到停顿、离题、重复拍摄
- **生成编辑决策列表**：剪辑的时间戳、要保留的段落
- **搭建 FFmpeg 和 Remotion 代码**：生成命令和组合

```
示例提示：
"这是 4 小时录像的转录。为 24 分钟视频博客识别 8 个最强片段。
给我每个片段的 FFmpeg 剪辑命令。"
```

这一层是关于结构，不是最终创意品味。

## 层级 3：确定性剪辑（FFmpeg）

FFmpeg 处理无聊但关键的工作：分割、修剪、拼接和预处理。

### 按时间戳提取片段

```bash
ffmpeg -i raw.mp4 -ss 00:12:30 -to 00:15:45 -c copy segment_01.mp4
```

### 从编辑决策列表批量剪辑

```bash
#!/bin/bash
# cuts.txt: start,end,label
while IFS=, read -r start end label; do
  ffmpeg -i raw.mp4 -ss "$start" -to "$end" -c copy "segments/${label}.mp4"
done < cuts.txt
```

### 拼接片段

```bash
# 创建文件列表
for f in segments/*.mp4; do echo "file '$f'"; done > concat.txt
ffmpeg -f concat -safe 0 -i concat.txt -c copy assembled.mp4
```

### 创建代理文件以加快编辑

```bash
ffmpeg -i raw.mp4 -vf "scale=960:-2" -c:v libx264 -preset ultrafast -crf 28 proxy.mp4
```

### 提取音频用于转录

```bash
ffmpeg -i raw.mp4 -vn -acodec pcm_s16le -ar 16000 audio.wav
```

### 标准化音频电平

```bash
ffmpeg -i segment.mp4 -af loudnorm=I=-16:TP=-1.5:LRA=11 -c:v copy normalized.mp4
```

## 层级 4：可编程组合（Remotion）

Remotion 将编辑问题转化为可组合代码。用于传统编辑器难以处理的事情：

### 何时使用 Remotion

- 叠加层：文本、图像、品牌、字幕条
- 数据可视化：图表、统计、动画数字
- 动态图形：转场、解释动画
- 可组合场景：跨视频可重用的模板
- 产品演示：带注释的截图、UI 高亮

### 基本 Remotion 组合

```tsx
import { AbsoluteFill, Sequence, Video, useCurrentFrame } from "remotion";

export const VlogComposition: React.FC = () => {
  const frame = useCurrentFrame();

  return (
    <AbsoluteFill>
      {/* 主要素材 */}
      <Sequence from={0} durationInFrames={300}>
        <Video src="/segments/intro.mp4" />
      </Sequence>

      {/* 标题叠加 */}
      <Sequence from={30} durationInFrames={90}>
        <AbsoluteFill style={{
          justifyContent: "center",
          alignItems: "center",
        }}>
          <h1 style={{
            fontSize: 72,
            color: "white",
            textShadow: "2px 2px 8px rgba(0,0,0,0.8)",
          }}>
            AI 编辑栈
          </h1>
        </AbsoluteFill>
      </Sequence>

      {/* 下一个片段 */}
      <Sequence from={300} durationInFrames={450}>
        <Video src="/segments/demo.mp4" />
      </Sequence>
    </AbsoluteFill>
  );
};
```

### 渲染输出

```bash
npx remotion render src/index.ts VlogComposition output.mp4
```

查看 [Remotion 文档](https://www.remotion.dev/docs) 获取详细模式和 API 参考。

## 层级 5：生成资产（ElevenLabs / fal.ai）

只生成你需要的东西。不要生成整个视频。

### 用 ElevenLabs 生成旁白

```python
import os
import requests

resp = requests.post(
    f"https://api.elevenlabs.io/v1/text-to-speech/{voice_id}",
    headers={
        "xi-api-key": os.environ["ELEVENLABS_API_KEY"],
        "Content-Type": "application/json"
    },
    json={
        "text": "你的旁白文本",
        "model_id": "eleven_turbo_v2_5",
        "voice_settings": {"stability": 0.5, "similarity_boost": 0.75}
    }
)
with open("voiceover.mp3", "wb") as f:
    f.write(resp.content)
```

### 用 fal.ai 生成音乐和音效

使用 `fal-ai-media` 技能：
- 背景音乐生成
- 音效（ThinkSound 模型用于视频生成音频）
- 转场声音

### 用 fal.ai 生成视觉内容

用于不存在的插入镜头、缩略图或空镜：
```
generate(model_name: "fal-ai/nano-banana-pro", input: {
  "prompt": "技术视频博客的专业缩略图，深色背景，屏幕上有代码",
  "image_size": "landscape_16_9"
})
```

### VideoDB 生成式音频

如果配置了 VideoDB：
```python
voiceover = coll.generate_voice(text="旁白", voice="alloy")
music = coll.generate_music(prompt="编程视频博客的轻背景音乐", duration=120)
sfx = coll.generate_sound_effect(prompt="轻微的嗖嗖转场声")
```

## 层级 6：最终润色（Descript / CapCut）

最后一层是人工的。使用传统编辑器：
- **节奏**：调整感觉太快或太慢的剪辑
- **字幕**：自动生成，然后手动清理
- **调色**：基础校正和氛围
- **最终音频混音**：平衡人声、音乐和音效电平
- **导出**：特定平台的格式和质量设置

这是品味所在。AI 清除重复性工作。你做最终决定。

## 社交媒体重新构图

不同平台需要不同的宽高比：

| 平台 | 宽高比 | 分辨率 |
|------|--------|--------|
| YouTube | 16:9 | 1920x1080 |
| TikTok / Reels | 9:16 | 1080x1920 |
| Instagram Feed | 1:1 | 1080x1080 |
| X / Twitter | 16:9 或 1:1 | 1280x720 或 720x720 |

### 用 FFmpeg 重新构图

```bash
# 16:9 转 9:16（中心裁剪）
ffmpeg -i input.mp4 -vf "crop=ih*9/16:ih,scale=1080:1920" vertical.mp4

# 16:9 转 1:1（中心裁剪）
ffmpeg -i input.mp4 -vf "crop=ih:ih,scale=1080:1080" square.mp4
```

### 用 VideoDB 重新构图

```python
# 智能重新构图（AI 引导的主体追踪）
reframed = video.reframe(start=0, end=60, target="vertical", mode=ReframeMode.smart)
```

## 场景检测和自动剪辑

### FFmpeg 场景检测

```bash
# 检测场景变化（阈值 0.3 = 中等灵敏度）
ffmpeg -i input.mp4 -vf "select='gt(scene,0.3)',showinfo" -vsync vfr -f null - 2>&1 | grep showinfo
```

### 静音检测用于自动剪辑

```bash
# 查找静音片段（用于剪掉空白）
ffmpeg -i input.mp4 -af silencedetect=noise=-30dB:d=2 -f null - 2>&1 | grep silence
```

### 高光提取

使用 Claude 分析转录 + 场景时间戳：
```
"给定这个带时间戳的转录和这些场景变化点，
识别 5 个最吸引人的 30 秒片段用于社交媒体。"
```

## 各工具最擅长的

| 工具 | 优势 | 劣势 |
|------|------|------|
| Claude / Codex | 组织、规划、代码生成 | 不是创意品味层 |
| FFmpeg | 确定性剪辑、批量处理、格式转换 | 无视觉编辑 UI |
| Remotion | 可编程叠加、可组合场景、可重用模板 | 非开发者的学习曲线 |
| Screen Studio | 立即获得精美的屏幕录制 | 仅限屏幕捕获 |
| ElevenLabs | 语音、旁白、音乐、音效 | 不是工作流中心 |
| Descript / CapCut | 最终节奏、字幕、润色 | 手动，不可自动化 |

## 核心原则

1. **编辑，不要生成。** 此工作流用于剪辑真实素材，而非从提示创建。
2. **结构先于风格。** 在层级 2 把故事理顺后再碰任何视觉内容。
3. **FFmpeg 是骨干。** 无聊但关键。长素材在此变得可管理。
4. **Remotion 用于可重复性。** 如果你不止做一次，就做成 Remotion 组件。
5. **选择性生成。** 只为不存在的资产使用 AI 生成，不要为所有东西。
6. **品味是最后一层。** AI 清除重复性工作。你做最终创意决定。

## 相关技能

- `fal-ai-media` — AI 图像、视频和音频生成
- `videodb` — 服务端视频处理、索引和流媒体
- `content-engine` — 原生平台内容分发
