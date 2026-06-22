# Claude Code 源码逆向分析报告

> 版本：v2.1.162  
> 分析时间：2025-06  
> 来源：npm 包 `@anthropic-ai/claude-code` + `@anthropic-ai/claude-code-darwin-arm64`

---

## 一、包结构

Claude Code 以 npm 包形式发布，采用"wrapper + native binary"架构：

```
@anthropic-ai/claude-code          # 主包（wrapper）
├── bin/claude.exe                 # 占位符，postinstall 后被替换为真实二进制
├── cli-wrapper.cjs                # Node.js fallback 启动器（--ignore-scripts 时用）
├── install.cjs                    # postinstall 脚本，按平台下载对应 native binary
├── sdk-tools.d.ts                 # 工具 JSON Schema 的 TypeScript 类型定义（126KB，明文！）
├── package.json
└── LICENSE.md

@anthropic-ai/claude-code-darwin-arm64   # 平台包（208MB）
└── claude                               # Mach-O arm64 可执行文件（Bun v1.3.14 打包）
```

**支持平台：** darwin-arm64, darwin-x64, linux-x64, linux-arm64, linux-x64-musl, linux-arm64-musl, win32-x64, win32-arm64

**打包工具：** Bun v1.3.14 (521eedd6)，JS 源码编译为字节码嵌入二进制，字符串常量明文可读。

---

## 二、工具系统（Tool System）

### 2.1 完整工具列表

从 `sdk-tools.d.ts` 提取的所有工具：

**核心工具（Input/Output 均有定义）：**

| 工具名 | 功能 |
|--------|------|
| `Bash` | 执行 shell 命令，支持超时、后台运行、沙箱禁用 |
| `FileRead` | 读取文件（支持分页、图片、PDF、Notebook） |
| `FileWrite` | 写入文件 |
| `FileEdit` | 字符串替换编辑（支持 replace_all） |
| `Glob` | 文件 glob 搜索 |
| `Grep` | 正则搜索（基于 ripgrep，支持多种输出模式） |
| `NotebookEdit` | 编辑 Jupyter Notebook 单元格 |
| `WebFetch` | 抓取 URL 内容并用 prompt 处理 |
| `WebSearch` | 网络搜索（支持域名白/黑名单） |
| `TodoWrite` | 写入 Todo 列表 |
| `Agent` | 派生子 Agent（支持后台运行、worktree 隔离） |
| `TaskStop` | 停止后台任务 |
| `TaskCreate/Get/Update/List` | 任务管理系统 |
| `ListMcpResources` | 列出 MCP 资源 |
| `ReadMcpResource` | 读取 MCP 资源 |
| `Mcp` | 执行 MCP 工具 |
| `AskUserQuestion` | 向用户提问（结构化选项，1-4个问题，每题2-4个选项） |
| `ExitPlanMode` | 退出 Plan 模式（携带权限声明） |
| `EnterPlanMode` | 进入 Plan 模式 |
| `EnterWorktree` | 进入 git worktree 隔离环境 |
| `ExitWorktree` | 退出 worktree（keep/remove） |
| `REPL` | JavaScript REPL 执行环境 |
| `Workflow` | 工作流执行（支持本地/远程） |
| `CronCreate/Delete/List` | 定时任务管理 |
| `ScheduleWakeup` | 定时唤醒（60-3600秒） |
| `Monitor` | 后台监控任务 |
| `RemoteTrigger` | 远程触发 |
| `PushNotification` | 推送通知 |
| `TaskOutput` | 获取后台任务输出 |

### 2.2 关键工具参数细节

**BashInput：**
```typescript
{
  command: string;
  timeout?: number;          // 最大 600000ms
  description?: string;      // 命令描述（用于权限展示）
  run_in_background?: boolean;
  dangerouslyDisableSandbox?: boolean;
}
```

**AgentInput（子 Agent）：**
```typescript
{
  description: string;       // 3-5词任务描述
  prompt: string;
  subagent_type?: string;
  model?: "sonnet" | "opus" | "haiku";
  run_in_background?: boolean;
  name?: string;             // 可寻址名称（SendMessage 用）
  team_name?: string;
  mode?: "acceptEdits" | "auto" | "bypassPermissions" | "default" | "dontAsk" | "plan";
  isolation?: "worktree";    // git worktree 隔离
}
```

**FileReadOutput** 支持多种类型：`text` | `image` | `notebook` | `pdf` | `parts` | `file_unchanged`

**GrepInput** 完整支持 ripgrep 参数：`-B`, `-A`, `-C`, `-n`, `-i`, `-o`, `type`, `head_limit`, `offset`, `multiline`

---

## 三、Agent 角色系统

从二进制字符串中提取的所有 Agent 系统提示词：

```
You are Claude Code, Anthropic's official CLI for Claude.
You are Claude Code, Anthropic's official CLI for Claude, running within the Claude Agent SDK.
You are a Claude agent, built on Anthropic's Claude Agent SDK.
You are a file search specialist for Claude Code. You excel at thoroughly navigating and exploring codebases.
  → 角色限制：只有 Grep/Read 工具，无文件编辑权限
You are a software architect and planning specialist for Claude Code.
  → 角色限制：只探索代码库和设计实现方案，无文件编辑权限
You are the Claude guide agent. Your primary responsibility is helping users understand and use Claude Code, the Claude Agent SDK, and the Claude API effectively.
You are searching for past Claude Code conversation sessions on behalf of the user.
  → 使用 Grep + Read 工具扫描 transcript 文件
You are a helpful AI assistant tasked with summarizing conversations.
You are running in non-interactive mode and cannot return a response to the user until your team is shut down.
You are a teammate in team "{teamName}".
You are returning to plan mode after having previously exited it. A plan file exists at {planFilePath}.
You are running in an isolated git worktree at {worktreePath}.
  → 变更不影响主工作目录，worktree 自动清理
You are a helpful code reviewer who...
You are coming up with a succinct title and git branch name for a coding session.
You have access to an `advisor` tool backed by a stronger reviewer model.
  → advisor 工具：无参数，自动转发完整对话历史
You have access to browser automation tools (mcp__claude-in-chrome__*) for interacting with web pages in Chrome.
You have a persistent, file-based memory system at `...`
```

---

## 四、环境变量（完整列表，共 200+ 个）

### 认证相关
```
CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR
CLAUDE_CODE_API_KEY_HELPER_TTL_MS
CLAUDE_CODE_OAUTH_TOKEN
CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR
CLAUDE_CODE_OAUTH_REFRESH_TOKEN
CLAUDE_CODE_OAUTH_CLIENT_ID
CLAUDE_CODE_OAUTH_SCOPES
CLAUDE_CODE_SESSION_ACCESS_TOKEN
CLAUDE_CODE_HFI_BEARER_TOKEN
CLAUDE_CODE_WEBSOCKET_AUTH_FILE_DESCRIPTOR
CLAUDE_TRUSTED_DEVICE_TOKEN
CLAUDE_SESSION_INGRESS_TOKEN_FILE
```

### 模型/API 配置
```
CLAUDE_CODE_API_BASE_URL
CLAUDE_CODE_AUTO_MODE_MODEL
CLAUDE_CODE_BG_CLASSIFIER_MODEL
CLAUDE_CODE_SUBAGENT_MODEL
CLAUDE_CODE_MAX_CONTEXT_TOKENS
CLAUDE_CODE_MAX_OUTPUT_TOKENS
CLAUDE_CODE_MAX_TURNS
CLAUDE_CODE_MAX_RETRIES
CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY
CLAUDE_CODE_EFFORT_LEVEL
CLAUDE_CODE_DISABLE_THINKING
CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING
CLAUDE_CODE_DISABLE_1M_CONTEXT
CLAUDE_CODE_EXTRA_BODY                    # JSON 对象，附加到 API 请求 body
CLAUDE_CODE_EXTRA_METADATA               # JSON 对象
```

### 云服务提供商
```
CLAUDE_CODE_USE_BEDROCK
CLAUDE_CODE_USE_VERTEX
CLAUDE_CODE_USE_FOUNDRY
CLAUDE_CODE_USE_MANTLE
CLAUDE_CODE_USE_GATEWAY
CLAUDE_CODE_USE_ANTHROPIC_AWS
CLAUDE_CODE_SKIP_BEDROCK_AUTH
CLAUDE_CODE_SKIP_VERTEX_AUTH
CLAUDE_CODE_SKIP_FOUNDRY_AUTH
CLAUDE_CODE_SKIP_MANTLE_AUTH
CLAUDE_CODE_SKIP_ANTHROPIC_AWS_AUTH
CLAUDE_CODE_BYOC_ENABLE_DATADOG
```

### 代理/网络
```
CLAUDE_CODE_HTTP_PROXY
CLAUDE_CODE_HTTPS_PROXY
CLAUDE_CODE_PROXY_URL
CLAUDE_CODE_PROXY_HOST
CLAUDE_CODE_PROXY_AUTH_HELPER_TTL_MS
CLAUDE_CODE_ENABLE_PROXY_AUTH_HELPER
CLAUDE_CODE_PROXY_AUTHENTICATE
CLAUDE_CODE_PROXY_RESOLVES_HOSTS
CLAUDE_CODE_HOST_HTTP_PROXY_PORT
CLAUDE_CODE_HOST_SOCKS_PROXY_PORT
CLAUDE_CODE_CERT_STORE
CLAUDE_CODE_CLIENT_CERT
CLAUDE_CODE_CLIENT_KEY
CLAUDE_CODE_CLIENT_KEY_PASSPHRASE
```

### 功能开关（DISABLE_*）
```
CLAUDE_CODE_DISABLE_AUTO_MEMORY
CLAUDE_CODE_DISABLE_BACKGROUND_TASKS
CLAUDE_CODE_DISABLE_CLAUDE_API_SKILL
CLAUDE_CODE_DISABLE_CLAUDE_CODE_SKILL
CLAUDE_CODE_DISABLE_CLAUDE_MDS
CLAUDE_CODE_DISABLE_CRON
CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS
CLAUDE_CODE_DISABLE_FAST_MODE
CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY
CLAUDE_CODE_DISABLE_FILE_CHECKPOINTING
CLAUDE_CODE_DISABLE_GIT_INSTRUCTIONS
CLAUDE_CODE_DISABLE_MEMORY_BULK_INFLATE
CLAUDE_CODE_DISABLE_MOUSE
CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC
CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK
CLAUDE_CODE_DISABLE_OFFICIAL_MARKETPLACE_AUTOINSTALL
CLAUDE_CODE_DISABLE_POLICY_SKILLS
CLAUDE_CODE_DISABLE_PRECOMPACT_SKIP
CLAUDE_CODE_DISABLE_REFUSAL_FALLBACK
CLAUDE_CODE_DISABLE_TERMINAL_TITLE
CLAUDE_CODE_DISABLE_THINKING
CLAUDE_CODE_DISABLE_VIRTUAL_SCROLL
CLAUDE_CODE_DISABLE_WORKFLOWS
CLAUDE_CODE_DISABLE_ADVISOR_TOOL
CLAUDE_CODE_DISABLE_AGENT_VIEW
CLAUDE_CODE_DISABLE_ATTACHMENTS
CLAUDE_CODE_DISABLE_ALTERNATE_SCREEN
```

### 功能开关（ENABLE_*）
```
CLAUDE_CODE_ENABLE_AUTO_MODE
CLAUDE_CODE_ENABLE_AWAY_SUMMARY
CLAUDE_CODE_ENABLE_BACKGROUND_PLUGIN_REFRESH
CLAUDE_CODE_ENABLE_CFC
CLAUDE_CODE_ENABLE_EXPERIMENTAL_ADVISOR_TOOL
CLAUDE_CODE_ENABLE_FINE_GRAINED_TOOL_STREAMING
CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY
CLAUDE_CODE_ENABLE_OPUS_4_7_FAST_MODE
CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION
CLAUDE_CODE_ENABLE_PROXY_AUTH_HELPER
CLAUDE_CODE_ENABLE_SDK_FILE_CHECKPOINTING
CLAUDE_CODE_ENABLE_TASKS
CLAUDE_CODE_ENABLE_TELEMETRY
CLAUDE_CODE_ENABLE_TOKEN_USAGE_ATTACHMENT
CLAUDE_CODE_ENABLE_XAA
CLAUDE_CODE_ENHANCED_TELEMETRY_BETA
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS
```

### 调试/开发
```
CLAUDE_CODE_SESSION_LOG
CLAUDE_CODE_JSONL_TRANSCRIPT
CLAUDE_CODE_TERMINAL_RECORDING
CLAUDE_PTY_RECORD
CLAUDE_DEBUG
CLAUDE_CODE_DEBUG_LOGS_DIR
CLAUDE_CODE_DEBUG_LOG_LEVEL
CLAUDE_CODE_PROFILE_STARTUP
CLAUDE_CODE_PROFILE_QUERY
CLAUDE_CODE_PERFETTO_TRACE
CLAUDE_CODE_PERFETTO_WRITE_INTERVAL_S
CLAUDE_CODE_FRAME_TIMING_LOG
CLAUDE_CODE_TEE_SDK_STDOUT
```

### 沙箱/安全
```
CLAUDE_CODE_SANDBOXED
CLAUDE_CODE_FORCE_SANDBOX
CLAUDE_CODE_BUBBLEWRAP
CLAUDE_CODE_BASH_SANDBOX_SHOW_INDICATOR
CLAUDE_CODE_SUBPROCESS_ENV_SCRUB
CLAUDE_CODE_DONT_INHERIT_ENV
CLAUDE_CODE_SCRIPT_CAPS
CLAUDE_CODE_ADDITIONAL_PROTECTION
```

### 插件/Skill 系统
```
CLAUDE_CODE_INVOKED_SKILLS
CLAUDE_CODE_SKILL_NAME
CLAUDE_CODE_SKILL_DESCRIPTION
CLAUDE_CODE_SYNC_SKILLS
CLAUDE_CODE_SYNC_SKILLS_WAIT_TIMEOUT_MS
CLAUDE_CODE_SYNC_PLUGINS
CLAUDE_CODE_SYNC_PLUGINS_INSTALL_TIMEOUT_MS
CLAUDE_CODE_SYNC_PLUGINS_MCP_TIMEOUT_MS
CLAUDE_CODE_SYNC_PLUGIN_INSTALL
CLAUDE_CODE_PLUGIN_CACHE_DIR
CLAUDE_CODE_PLUGIN_GIT_TIMEOUT_MS
CLAUDE_CODE_PLUGIN_KEEP_MARKETPLACE_ON_FAILURE
CLAUDE_CODE_PLUGIN_PREFER_HTTPS
CLAUDE_CODE_PLUGIN_SEED_DIR
CLAUDE_CODE_PLUGIN_USE_ZIP_CACHE
CLAUDE_CODE_DISABLE_OFFICIAL_MARKETPLACE_AUTOINSTALL
CLAUDE_CODE_DISABLE_POLICY_SKILLS
```

### 会话/上下文
```
CLAUDE_CODE_SESSION_ID
CLAUDE_CODE_SESSION_NAME
CLAUDE_CODE_SESSION_KIND
CLAUDE_CODE_REMOTE_SESSION_ID
CLAUDE_CODE_RESUME_FROM_SESSION
CLAUDE_CODE_RESUME_INTERRUPTED_TURN
CLAUDE_CODE_RESUME_PROMPT
CLAUDE_CODE_RESUME_THRESHOLD_MINUTES
CLAUDE_CODE_RESUME_TOKEN_THRESHOLD
CLAUDE_CODE_AUTO_COMPACT_WINDOW
CLAUDE_CODE_COLD_COMPACT
CLAUDE_CODE_IDLE_THRESHOLD_MINUTES
CLAUDE_CODE_IDLE_TOKEN_THRESHOLD
CLAUDE_CODE_LOOP_KEEPALIVE
CLAUDE_CODE_LOOP_PERSISTENT
```

### 其他有趣的
```
CLAUDE_CODE_PLAN_MODE_REQUIRED          # 强制 Plan 模式
CLAUDE_CODE_PLAN_MODE_INTERVIEW_PHASE
CLAUDE_CODE_PLAN_V2_AGENT_COUNT
CLAUDE_CODE_PLAN_V2_EXPLORE_AGENT_COUNT
CLAUDE_CODE_COORDINATOR_MODE
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1  # 实验性多 Agent 团队
CLAUDE_CODE_FORK_SUBAGENT
CLAUDE_CODE_FORK_SUBAGENT_DEFAULT_ON
CLAUDE_CODE_PERFORCE_MODE               # Perforce 版本控制支持
CLAUDE_CODE_PEWTER_OWL                  # 内部代号？
CLAUDE_CODE_PROACTIVE
CLAUDE_CODE_INVESTIGATE_FIRST
CLAUDE_CODE_SIMPLE_SYSTEM_PROMPT
CLAUDE_CODE_VERIFY_PROMPT
CLAUDE_CODE_BRIEF
CLAUDE_CODE_BRIEF_UPLOAD
CLAUDE_CODE_VOICE_FORWARD_INTERIMS_TYPED  # 语音输入支持
CLAUDE_CODE_ACCESSIBILITY
CLAUDE_CODE_NATIVE_CURSOR
CLAUDE_CODE_SCROLL_SPEED
CLAUDE_CODE_TMUX_PREFIX
CLAUDE_CODE_TMUX_SESSION
CLAUDE_CODE_TMUX_TRUECOLOR
```

---

## 五、内部架构发现

### 5.1 Agent 团队系统
- 支持多 Agent 协作（`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`）
- Agent 可命名并通过 `SendMessage({to: name})` 互相通信
- 支持 `worktree` 隔离模式：每个 Agent 在独立 git worktree 工作
- Plan 模式：Agent 先规划再执行，支持用户审批

### 5.2 Workflow 系统
- 支持本地和远程（CCR）工作流
- 工作流脚本可持久化（`scriptPath`），支持重用
- 子 Agent transcript 写入独立目录

### 5.3 Cron/定时任务
- 内置 cron 系统（`CronCreate/Delete/List`）
- `ScheduleWakeup`：60-3600秒定时唤醒
- `Monitor`：持久后台监控任务

### 5.4 内存系统
- 文件持久化内存（`CLAUDE.md` 机制）
- 支持私有目录 + 共享目录两种内存
- `CLAUDE_CODE_DISABLE_AUTO_MEMORY` 可关闭自动内存

### 5.5 MCP 集成
- 完整 MCP 资源读取（`ListMcpResources`, `ReadMcpResource`）
- MCP 工具执行（`Mcp`）
- `CLAUDE_CODE_MCP_ALLOWLIST_ENV`, `CLAUDE_CODE_MCP_SERVER_NAME/URL`

### 5.6 Advisor 工具
- 特殊工具：调用更强的 reviewer 模型
- 无参数，自动转发完整对话历史
- 可通过 `CLAUDE_CODE_DISABLE_ADVISOR_TOOL` 关闭

### 5.7 REPL 工具
- 内置 JavaScript REPL 执行环境
- 支持注册自定义工具（`registeredTools`）
- 可返回图片和 PDF 内容块

### 5.8 AskUserQuestion 工具
- 结构化问答：1-4个问题，每题2-4个选项
- 支持多选（`multiSelect`）
- 支持选项预览（`preview`，用于代码片段/mockup 对比）
- 支持自由文本输入（`response`）

---

## 六、API 集成细节

从二进制字符串中发现的 API 相关信息：

```javascript
betas: ["files-api-2025-04-14"]          // Files API beta
betas: ["managed-agents-2026-04-01"]     // Managed Agents beta
cache_control: { type: "ephemeral" }     // 提示词缓存
cache_control: { type: "ephemeral" }     // auto-caches the last cacheable block
```

**Token 使用统计结构：**
```typescript
usage: {
  input_tokens: number;
  output_tokens: number;
  cache_creation_input_tokens: number | null;
  cache_read_input_tokens: number | null;
  server_tool_use: {
    web_search_requests: number;
    web_fetch_requests: number;
  } | null;
  service_tier: "standard" | "priority" | "batch";
  cache_creation: {
    ephemeral_1h_input_tokens: number;
    ephemeral_5m_input_tokens: number;
  } | null;
}
```

---

## 七、CLAUDE.md 相关

从二进制中提取的 CLAUDE.md 初始化提示词：

```
Please analyze this codebase and create a CLAUDE.md file, which will be given to future 
instances of Claude Code to operate in this repository.
- If there's already a CLAUDE.md, suggest improvements to it.
- When you make the initial CLAUDE.md, do not repeat yourself and do not include obvious 
  instructions like "Provide helpful error messages to users", "Write unit tests for all 
  new utilities", "Never include sensitive information (API keys, tokens) in code or commits".
```

文档地址：`https://code.claude.com/docs/en/memory.md`

---

## 八、文件结构说明

| 文件 | 大小 | 说明 |
|------|------|------|
| `claude`（darwin-arm64） | 208MB | Bun v1.3.14 打包的 native binary，含完整 JS 运行时 |
| `sdk-tools.d.ts` | 126KB | 所有工具的 TypeScript 类型定义，**完全明文** |
| `cli-wrapper.cjs` | 4.8KB | Node.js fallback 启动器，**完全明文** |
| `install.cjs` | 7.0KB | postinstall 脚本，**完全明文** |

> **注意：** 核心业务逻辑（JS 源码）以 Bun 字节码形式编译进二进制，无法直接反编译。但字符串常量（提示词、错误信息、配置键名）均以明文存储，可通过 `strings` 命令提取。
