---
name: x-api
description: X/Twitter API 集成，用于发布推文、推文串、阅读时间线、搜索和分析。涵盖 OAuth 认证模式、速率限制和原生平台内容发布。当用户想要程序化地与 X 交互时使用。
origin: ECC
---

# X API

程序化地与 X (Twitter) 交互，用于发布、阅读、搜索和分析。

## 何时启用

- 用户想要程序化地发布推文或推文串
- 从 X 读取时间线、提及或用户数据
- 在 X 上搜索内容、趋势或对话
- 构建 X 集成或机器人
- 分析和互动追踪
- 用户说"发布到 X"、"发推"、"X API"或"Twitter API"

## 认证

### OAuth 2.0（仅应用 / 用户上下文）

适用于：读密集型操作、搜索、公共数据。

```bash
# 环境设置
export X_BEARER_TOKEN="your-bearer-token"
```

```python
import os
import requests

bearer = os.environ["X_BEARER_TOKEN"]
headers = {"Authorization": f"Bearer {bearer}"}

# 搜索最近推文
resp = requests.get(
    "https://api.x.com/2/tweets/search/recent",
    headers=headers,
    params={"query": "claude code", "max_results": 10}
)
tweets = resp.json()
```

### OAuth 1.0a（用户上下文）

必需用于：发布推文、管理账户、私信。

```bash
# 环境设置 — 使用前先 source
export X_API_KEY="your-api-key"
export X_API_SECRET="your-api-secret"
export X_ACCESS_TOKEN="your-access-token"
export X_ACCESS_SECRET="your-access-secret"
```

```python
import os
from requests_oauthlib import OAuth1Session

oauth = OAuth1Session(
    os.environ["X_API_KEY"],
    client_secret=os.environ["X_API_SECRET"],
    resource_owner_key=os.environ["X_ACCESS_TOKEN"],
    resource_owner_secret=os.environ["X_ACCESS_SECRET"],
)
```

## 核心操作

### 发布推文

```python
resp = oauth.post(
    "https://api.x.com/2/tweets",
    json={"text": "来自 Claude Code 的问候"}
)
resp.raise_for_status()
tweet_id = resp.json()["data"]["id"]
```

### 发布推文串

```python
def post_thread(oauth, tweets: list[str]) -> list[str]:
    ids = []
    reply_to = None
    for text in tweets:
        payload = {"text": text}
        if reply_to:
            payload["reply"] = {"in_reply_to_tweet_id": reply_to}
        resp = oauth.post("https://api.x.com/2/tweets", json=payload)
        resp.raise_for_status()
        tweet_id = resp.json()["data"]["id"]
        ids.append(tweet_id)
        reply_to = tweet_id
    return ids
```

### 读取用户时间线

```python
resp = requests.get(
    f"https://api.x.com/2/users/{user_id}/tweets",
    headers=headers,
    params={
        "max_results": 10,
        "tweet.fields": "created_at,public_metrics",
    }
)
```

### 搜索推文

```python
resp = requests.get(
    "https://api.x.com/2/tweets/search/recent",
    headers=headers,
    params={
        "query": "from:affaanmustafa -is:retweet",
        "max_results": 10,
        "tweet.fields": "public_metrics,created_at",
    }
)
```

### 通过用户名获取用户

```python
resp = requests.get(
    "https://api.x.com/2/users/by/username/affaanmustafa",
    headers=headers,
    params={"user.fields": "public_metrics,description,created_at"}
)
```

### 上传媒体并发布

```python
# 媒体上传使用 v1.1 端点

# 步骤 1：上传媒体
media_resp = oauth.post(
    "https://upload.twitter.com/1.1/media/upload.json",
    files={"media": open("image.png", "rb")}
)
media_id = media_resp.json()["media_id_string"]

# 步骤 2：带媒体发布
resp = oauth.post(
    "https://api.x.com/2/tweets",
    json={"text": "看看这个", "media": {"media_ids": [media_id]}}
)
```

## 速率限制参考

| 端点 | 限制 | 时间窗口 |
|------|------|----------|
| POST /2/tweets | 200 | 15 分钟 |
| GET /2/tweets/search/recent | 450 | 15 分钟 |
| GET /2/users/:id/tweets | 1500 | 15 分钟 |
| GET /2/users/by/username | 300 | 15 分钟 |
| POST media/upload | 415 | 15 分钟 |

始终检查 `x-rate-limit-remaining` 和 `x-rate-limit-reset` 响应头。

```python
import time

remaining = int(resp.headers.get("x-rate-limit-remaining", 0))
if remaining < 5:
    reset = int(resp.headers.get("x-rate-limit-reset", 0))
    wait = max(0, reset - int(time.time()))
    print(f"接近速率限制。{wait} 秒后重置")
```

## 错误处理

```python
resp = oauth.post("https://api.x.com/2/tweets", json={"text": content})
if resp.status_code == 201:
    return resp.json()["data"]["id"]
elif resp.status_code == 429:
    reset = int(resp.headers["x-rate-limit-reset"])
    raise Exception(f"速率限制。在 {reset} 重置")
elif resp.status_code == 403:
    raise Exception(f"禁止：{resp.json().get('detail', '检查权限')}")
else:
    raise Exception(f"X API 错误 {resp.status_code}: {resp.text}")
```

## 安全

- **绝不要硬编码 token。** 使用环境变量或 `.env` 文件。
- **绝不要提交 `.env` 文件。** 添加到 `.gitignore`。
- **如果暴露则轮换 token。** 在 developer.x.com 重新生成。
- **当不需要写权限时使用只读 token。**
- **安全存储 OAuth 机密** — 不要放在源代码或日志中。

## 与内容引擎集成

使用 `content-engine` 技能生成原生平台内容，然后通过 X API 发布：
1. 用 content-engine 生成内容（X 平台格式）
2. 验证长度（单条推文 280 字符）
3. 使用上面的模式通过 X API 发布
4. 通过 public_metrics 追踪互动

## 相关技能

- `content-engine` — 为 X 生成原生平台内容
- `crosspost` — 跨 X、LinkedIn 和其他平台分发内容
