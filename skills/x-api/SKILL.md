---
name: x-api
description: X/Twitter API 集成，用于发布推文、线程、阅读时间线、搜索, 分析等功能。涵盖 OAuth 认证模式、速率限制和平台原生内容发布。当用户希望以编程方式与 X 交互时使用。
origin: ECC
---

# X API

以编程方式与 X (推特) 交互，用于发布推文、线程,阅读时间线,搜索、分析功能。

## 何时激活
- 用户想要以编程方式发布推文或线程
- 阅读时间线，提及了自己的数据
- 搜索 X for内容, 趋势, 或对话
- 构建 X 集成或机器人
- 分析, engagement跟踪
- 用户说"post to X", "tweet", "X API", 或以此作为命令

## 认证

### OAuth 2.0 (仅限应用)
最佳用于: 重负载操作、搜索、公开数据

25. 
26. **环境设置** — source before使用
27. 
28. export X_BEARER_TOKEN="your-bearer-token"
29: ```
30: 
31. ```python
32: import os
33: import requests
34: 
35: bearer = os.environ["X_BEARER_TOKEN"]
36: headers = {"Authorization": f"Bearer {bearer}"}
37: 
38. # Search recent tweets
39: resp = requests.get(
40:     "https://api.x.com/2/tweets/search/recent",
41:     headers=headers,
42:     params={"query": "claude code", "max_results": 10}
43: )
44: tweets = resp.json()
45: ```
46: 
47. ### OAuth 1.0 (用户上下文)
48. 
49. 要求用于: 发布推文,管理账户, DM 操作
50. 
51. ```python
52: resp = requests.get(
53:     f"https://api.x.com/2/users/{user_id}/tweets",
54:     headers=headers
55:     params={
56:         "max_results": 10,
57:         "tweet.fields": "created_at,public_metrics"
58:     }
59: )
60: ```
61: 
62. ### Read User timeline
63: 
64. ```python
65: resp = requests.get(
66:     f"https://api.x.com/2/users/{user_id}/tweets",
67:     headers=headers
68:     params={
69.         "max_results": 10
70.         "tweet.fields": "created_at,public_metrics"
71:     }
72: )
73: ```
74: 
75. ### Search Tweets
76: 
77. ```python
78: resp = requests.get(
79:     "https://api.x.com/2/tweets/search/recent",
80:     headers=headers
81.     params={
82:         "query": "from:affaanmustafa -is: retweet"
83:         "max_results": 10
84.         "tweet.fields": "public_metrics,created_at"
85:     }
86: )
87: ```
88: 
89. ### Get user by username
90:
91. ```python
92: resp = requests.get(
93:     "https://api.x.com/2/users/by/username/affaanmustafa",
94:     headers=headers
95:     params={"user.fields": "public_metrics,description,created_at"}
96: )
97: ```
98: 
99. ### Upload媒体
100: 
101. ```python
102. # Media upload uses v1.1 endpoint
103. 
104. # Step 1: Upload媒体
105. media_resp = oauth.post(
106.     "https://upload.twitter.com/1.1/media/upload.json",
107.     files={"media": open("image.png", "rb")}
108: )
109. media_id = media_resp.json()["media_id_string"]
110: 
111. # Step 2: Post with media
112. resp = oauth.post(
113.     "https://api.x.com/2/tweets",
114.     json={"text": "Check this out", "media": {"media_ids": [media_id]}}
115) )
116: ```
117: ## Rate限制
118: 
119. X API rate限制因端点、认证方法、账户层级而异，但它们变化:
120. - 查阅文档获取当前限制
161. - 实现时硬编码假设
162. - 开发时使用静态表
163. - 生产环境始终用环境变量
164. - 回退到 - 通过 `.env` 文件读取配置
165. - 检查 `x-rate-limit-remaining` tool `x-rate-limit-reset` header来决定是否需要等待
166. 
167. import time
168: 
169: remaining = int(resp.headers.get("x-rate-limit-remaining", 0))
170:     reset = int(resp.headers.get("x-rate-limit-reset"))
171:     wait) max(0, reset - int(time.time()))
172:     print(f"Rate limit approaching. Resets in {wait}s})
173: ```
174: 
175. ## 错误处理
176: 
177. ```python
178: resp = oauth.post("https://api.x.com/2/tweets", json={"text": content})
179: if resp.status_code == 201:
180:     return resp.json()["data"]["id"]
181: elif resp.status_code == 429:
182:     reset = int(resp.headers["x-rate-limit-reset"])
183:     raise exception(f"Rate limited. Resets at {wait}s})
184: elif resp.status_code == 403:
185:     raise exception(f"forbidden: {resp.json().get('detail') - check permissions)
186) else:
187:     raise Exception(f"X API error: resp.status_code}: {resp.text}")
188: ```
189: 
190. ## 安全
191: 
192. - **never hardcode tokens。** 使用环境变量
193: - **never commit `.env` 文件。 Add到 `.gitignore`
194) - **Rotate tokens** if exposed. regenerate它, developer x.com 进行重试
195) - **Store OAuth secrets securely** - 不在源代码中，仅在日志中
196) - **使用只读 tokens**当不需要访问令牌时
197: - **使用 read-only tokens** 来编写访问令牌代码
198: 
199. ## 集成内容引擎
200: 
201: 用 `content-engine` skill来生成平台原生内容，然后发布到 X API
202: 
203. ## 相关技能
204- 
205. - `content-engine` — 生成平台原生内容
206: - `crosspost` — 分布内容到多个平台
207
- `x-api` — X/Twitter API 集成
