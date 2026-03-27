# 钩子（Hooks）

钩子是一组事件驱动的自动化流程，在 Claude Code 工具执行之前或之后触发。它们用于强化代码质量、尽早发现错误，并自动化执行重复性检查。

## 工作原理

```
用户请求 → Claude 选择一个工具 → PreToolUse 钩子运行 → 工具执行 → PostToolUse 钩子运行
```

- PreToolUse 钩子在工具执行之前运行。它们可以 **阻塞**（退出码为 2）或 **警告**（错误输出但不阻塞）。
- PostToolUse 钩子在工具完成后运行。它们可以分析输出，但不能阻塞。 
- Stop 钩子在每次 Claude 响应后运行。 
- SessionStart/SessionEnd 钩子在会话生命周期边界处运行。 
- PreCompact 钩子在上下文压缩之前运行，便于保存状态。

## Hooks in This Plugin

### PreToolUse Hooks

| 钩子 | 匹配器 | 行为 | 退出码 |
|------|---------|----------|-----------|
| **Dev server blocker** | `Bash` | 阻塞 tmux 之外的 `npm run dev` 等命令 — 确保日志可访问 | 2 (阻塞) |
| **Tmux reminder** | `Bash` | 建议对耗时命令使用 tmux（如 npm test、cargo build、docker） | 0 (警告) |
| **Git push reminder** | `Bash` | 在执行 git push 之前提醒审查修改 | 0 (警告) |
| **Doc file warning** | `Write` | 警告非标准的 .md/.txt 文件（允许 README、CLAUDE、CONTRIBUTING、CHANGELOG、LICENSE、SKILL、docs/、skills/），并处理跨平台路径 | 0 (警告) |
| **Strategic compact** | `Edit\|Write` | 在逻辑间隔处建议手动执行 `/compact` | 0 (警告) |
| **InsAIts 安全监控（可选）** | `Bash\|Write\|Edit\|MultiEdit` | 针对高信号工具输入的可选安全扫描。除非 `ECC_ENABLE_INSAITS=1`，否则将禁用。对关键发现阻塞，对非关键发现发出警告，并将审计日志写入 `.insaits_audit_session.jsonl`。需要：`pip install insa-its`。 [Details](../scripts/hooks/insaits-security-monitor.py) | 2（阻塞关键）/ 0（警告） |

### PostToolUse Hooks

| Hook | Matcher | What It Does |
|------|---------|-------------|
| **PR 日志记录器** | `Bash` | 记录 PR URL 并在创建 PR 之后提供审查命令 |
| **Build analysis** | `Bash` | 构建命令完成后的后台分析（异步、非阻塞） |
| **Quality gate** | `Edit\|Write\|MultiEdit` | 编辑后执行快速质量检查 |
| **Prettier format** | `Edit` | 编辑后使用 Prettier 自动格式化 JS/TS 文件 |
| **TypeScript check** | `Edit` | 编辑后运行 `tsc --noEmit` |
| **console.log warning** | `Edit` | 提示已编辑文件中的 `console.log` 语句 |

### Lifecycle Hooks

| Hook | Event | What It Does |
|------|-------|-------------|
| **Session start** | `SessionStart` | 加载先前上下文并检测包管理器 |
| **Pre-compact** | `PreCompact` | 在上下文压缩前保存状态 |
| **Console.log audit** | `Stop` | 检查每次响应后修改的文件中是否存在 `console.log` |
| **Session summary** | `Stop` | 在 transcript 路径可用时持久化会话状态 |
| **Pattern extraction** | `Stop` | 评估会话以提取可提取的模式（持续学习） |
| **Cost tracker** | `Stop` | 发出轻量级运行成本遥测标记 |
| **Desktop notify** | `Stop` | 在 Claude 回应时发送带任务摘要的 macOS 桌面通知（标准+） |
| **Session end marker** | `SessionEnd` | 生命周期标记与清理日志 |

## 自定义钩子

### 禁用一个钩子

Remove or comment out the hook entry in `hooks.json`. If installed as a plugin, override in your `~/.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [],
        "description": "Override: allow all .md file creation"
      }
    ]
  }
}
```

### 运行时钩子控制（推荐）

Use environment variables to control hook behavior without editing `hooks.json`:

```bash
# minimal | standard | strict (default: standard)
export ECC_HOOK_PROFILE=standard

# 禁用特定钩子ID（逗号分隔）
export ECC_DISABLED_HOOKS="pre:bash:tmux-reminder,post:edit:typecheck"
```

Profiles:
- `minimal` — 仅保留核心的生命周期和安全性钩子。
- `standard` — 默认；在质量与安全检查之间取得平衡。
- `strict` — 启用额外的提醒和更严格的防护机制。

### 编写你自己的钩子

Hooks are shell commands that receive tool input as JSON on stdin and must output JSON on stdout.

**Basic structure:**

```javascript
// my-hook.js
let data = '';
process.stdin.on('data', chunk => data += chunk);
process.stdin.on('end', () => {
  const input = JSON.parse(data);

  // Access tool info
  const toolName = input.tool_name;        // "Edit", "Bash", "Write", etc.
  const toolInput = input.tool_input;      // Tool-specific parameters
  const toolOutput = input.tool_output;    // Only available in PostToolUse

  // Warn (non-blocking): write to stderr
  console.error('[Hook] Warning message shown to Claude');

  // Block (PreToolUse only): exit with code 2
  // process.exit(2);

  // Always output the original data to stdout
  console.log(data);
});
```

**Exit codes:**
- `0` — Success (continue execution)
- `2` — Block the tool call (PreToolUse only)
- Other non-zero — Error (logged but does not block)

### Hook Input Schema

```typescript
interface HookInput {
  tool_name: string;          // "Bash", "Edit", "Write", "Read", etc.
  tool_input: {
    command?: string;         // Bash: the command being run
    file_path?: string;       // Edit/Write/Read: target file
    old_string?: string;      // Edit: text being replaced
    new_string?: string;      // Edit: replacement text
    content?: string;         // Write: file content
  };
  tool_output?: {             // PostToolUse only
    output?: string;          // Command/tool output
  };
}
```

### Async Hooks

For hooks that should not block the main flow (e.g., background analysis):

```json
{
  "type": "command",
  "command": "node my-slow-hook.js",
  "async": true,
  "timeout": 30
}
```

Async hooks run in the background. They cannot block tool execution.

## Common Hook Recipes

### Warn about TODO comments

```json
{
  "matcher": "Edit",
  "hooks": [{
    "type": "command",
    "command": "node -e \"let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const ns=i.tool_input?.new_string||'';if(/TODO|FIXME|HACK/.test(ns)){console.error('[Hook] New TODO/FIXME added - consider creating an issue')}console.log(d)})\""
  }],
  "description": "Warn when adding TODO/FIXME comments"
}
```

### Block large file creation

```json
{
  "matcher": "Write",
  "hooks": [{
    "type": "command",
    "command": "node -e \"let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const c=i.tool_input?.content||'';const lines=c.split('\\n').length;if(lines>800){console.error('[Hook] BLOCKED: File exceeds 800 lines ('+lines+' lines)');console.error('[Hook] Split into smaller, focused modules');process.exit(2)}console.log(d)})\""
  }],
  "description": "Block creation of files larger than 800 lines"
}
```

### Auto-format Python files with ruff

```json
{
  "matcher": "Edit",
  "hooks": [{
    "type": "command",
    "command": "node -e \"let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const p=i.tool_input?.file_path||'';if(/\\.py$/.test(p)){const{execFileSync}=require('child_process');try{execFileSync('ruff',['format',p],{stdio:'pipe'})}catch(e){}}console.log(d)})\""
  }],
  "description": "Auto-format Python files with ruff after edits"
}
```

### Require test files alongside new source files

```json
{
  "matcher": "Write",
  "hooks": [{
    "type": "command",
    "command": "node -e \"const fs=require('fs');let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const p=i.tool_input?.file_path||'';if(/src\\/.*\\.(ts|js)$/.test(p)&&!/\\.test\\.|\\.spec\\./.test(p)){const testPath=p.replace(/\\.(ts|js)$/,'.test.$1');if(!fs.existsSync(testPath)){console.error('[Hook] No test file found for: '+p);console.error('[Hook] Expected: '+testPath);console.error('[Hook] Consider writing tests first (/tdd)')}}console.log(d)})\""
  }],
  "description": "Remind to create tests when adding new source files"
}
```

## Cross-Platform Notes

Hook logic is implemented in Node.js scripts for cross-platform behavior on Windows, macOS, and Linux. A small number of shell wrappers are retained for continuous-learning observer hooks; those wrappers are profile-gated and have Windows-safe fallback behavior.

## Related

- [rules/common/hooks.md](../rules/common/hooks.md) — Hook architecture guidelines
- [skills/strategic-compact/](../skills/strategic-compact/) — Strategic compaction skill
- [scripts/hooks/](../scripts/hooks/) — Hook script implementations
