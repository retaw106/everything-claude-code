# Canary Watch — 部署后监控

## 何时使用

- 部署到生产或预发布环境后
- 合并有风险的 PR 后
- 当你想验证修复是否真正解决了问题
- 发布窗口期间的持续监控
- 依赖升级后

## 工作原理

监控已部署的 URL 是否出现回归。循环运行直到停止或监控窗口过期。

### 监控内容

```
1. HTTP 状态 — 页面是否返回 200？
2. 控制台错误 — 是否有新的错误？
3. 网络故障 — API 调用失败、5xx 响应？
4. 性能 — LCP/CLS/INP 相比基线是否回归？
5. 内容 — 关键元素是否消失？（h1、nav、footer、CTA）
6. API 健康 — 关键端点是否在 SLA 内响应？
```

### 监控模式

**快速检查**（默认）：单次检查，报告结果
```
/canary-watch https://myapp.com
```

**持续监控**：每 N 分钟检查一次，持续 M 小时
```
/canary-watch https://myapp.com --interval 5m --duration 2h
```

**差异模式**：比较预发布环境 vs 生产环境
```
/canary-watch --compare https://staging.myapp.com https://myapp.com
```

### 警报阈值

```yaml
critical:  # 立即警报
  - HTTP status != 200
  - Console error count > 5（仅新错误）
  - LCP > 4s
  - API endpoint returns 5xx

warning:   # 在报告中标记
  - LCP increased > 500ms from baseline
  - CLS > 0.1
  - New console warnings
  - Response time > 2x baseline

info:      # 仅记录
  - Minor performance variance
  - New network requests（添加了第三方脚本？）
```

### 通知

当达到严重阈值时：
- 桌面通知（macOS/Linux）
- 可选：Slack/Discord webhook
- 记录到 `~/.claude/canary-watch.log`

## 输出

```markdown
## Canary 报告 — myapp.com — 2026-03-23 03:15 PST

### 状态: 健康 ✓

| 检查 | 结果 | 基线 | 差异 |
|-------|--------|----------|-------|
| HTTP | 200 ✓ | 200 | — |
| Console errors | 0 ✓ | 0 | — |
| LCP | 1.8s ✓ | 1.6s | +200ms |
| CLS | 0.01 ✓ | 0.01 | — |
| API /health | 145ms ✓ | 120ms | +25ms |

### 未检测到回归。部署干净。
```

## 集成

配合使用：
- `/browser-qa` 用于部署前验证
- Hooks: 在 `git push` 上添加为 PostToolUse hook，以便在部署后自动检查
- CI: 在部署步骤后在 GitHub Actions 中运行
