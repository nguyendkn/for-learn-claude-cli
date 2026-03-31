# Codemap — Claude Source Code Original
> Phân tích cấu trúc: `d:\Claude Source Code Original\src`
> Ngày tạo: 2026-03-31

---

## Tổng quan

Claude Code là một **CLI + REPL agentic tool** của Anthropic.
- **Runtime**: Bun (TypeScript)
- **UI**: React + Ink (terminal UI)
- **Quy mô**: ~1,884 files · ~512K lines · 34MB source

---

## Cấu trúc theo Layer

```
src/
│
├── ENTRY POINTS
│   ├── main.tsx                    # React entry point chính (Commander.js CLI)
│   ├── replLauncher.tsx            # REPL launcher
│   ├── setup.ts                    # Setup logic khởi động
│   ├── bootstrap/state.ts          # Khởi động state ban đầu
│   └── entrypoints/
│       ├── init.ts                 # Khởi tạo ứng dụng
│       ├── mcp.ts                  # MCP integration entry
│       ├── agentSdkTypes.ts        # Kiểu dữ liệu SDK Agent
│       ├── sandboxTypes.ts         # Kiểu dữ liệu sandbox
│       └── sdk/                    # SDK schemas & types
│           ├── controlSchemas.ts
│           ├── coreSchemas.ts
│           └── coreTypes.ts
│
├── CORE ENGINE
│   ├── QueryEngine.ts              # Lõi xử lý LLM (~46K lines): tool-call loops, thinking mode
│   ├── Tool.ts                     # Base type definitions cho tools (~29K lines)
│   ├── Task.ts                     # Task management types & utilities
│   ├── commands.ts                 # Central registry tất cả commands (~25K lines)
│   ├── query.ts                    # Query execution logic
│   ├── tools.ts                    # Tool registry & loading
│   ├── tasks.ts                    # Task execution
│   └── context.ts                  # Context management
│
├── COMMANDS (~80 slash commands)
│   └── commands/
│       ├── add-dir/                # Thêm working directory
│       ├── agents/                 # Quản lý agents
│       ├── branch/                 # Git branch management
│       ├── bridge/                 # Remote control bridge toggle
│       ├── btw/                    # Quick note taking
│       ├── chrome/                 # Chrome DevTools integration
│       ├── clear/                  # Xóa transcript/cache
│       ├── color/                  # Thay đổi agent color
│       ├── commit/                 # Git commit automation
│       ├── compact/                # Nén context
│       ├── config/                 # Settings configuration
│       ├── context/                # File context management
│       ├── cost/                   # Hiển thị chi phí API
│       ├── desktop/                # Desktop app control
│       ├── diff/                   # Git diff viewer
│       ├── doctor/                 # Diagnostic checks
│       ├── effort/                 # Effort estimation
│       ├── exit/                   # Thoát ứng dụng
│       ├── export/                 # Xuất conversation
│       ├── fast/                   # Fast mode toggle
│       ├── feedback/               # Gửi feedback
│       ├── files/                  # Liệt kê files đang theo dõi
│       ├── heapdump/               # Memory debugging
│       ├── help/                   # Trợ giúp commands
│       ├── hooks/                  # Quản lý hooks
│       ├── ide/                    # IDE integration status
│       ├── init/                   # Khởi tạo project
│       ├── keybindings/            # Quản lý keybindings
│       ├── login/                  # Login Anthropic account
│       ├── logout/                 # Logout
│       ├── mcp/                    # MCP servers management
│       ├── memory/                 # Memory management
│       ├── mobile/                 # Mobile QR code
│       ├── model/                  # Chọn AI model
│       ├── output-style/           # Output styling
│       ├── permissions/            # Tool permissions management
│       ├── plan/                   # Plan mode toggle
│       ├── plugin/                 # Plugin management
│       ├── pr_comments/            # PR comments
│       ├── release-notes/          # Changelog viewer
│       ├── rename/                 # Rename agent
│       ├── review/                 # Code review
│       ├── rewind/                 # Undo last message
│       ├── session/                # Session management
│       ├── skills/                 # Skills listing
│       ├── status/                 # Status info
│       ├── tag/                    # Tag management
│       ├── tasks/                  # Task management
│       ├── theme/                  # Theme management
│       ├── usage/                  # Usage statistics
│       ├── vim/                    # Vim mode toggle
│       └── voice/                  # Voice mode
│
├── TOOLS (40+ model-invocable tools)
│   └── tools/
│       ├── AgentTool/              # Spawn subagents
│       │   ├── AgentTool.tsx
│       │   ├── runAgent.ts
│       │   ├── agentMemory.ts
│       │   └── built-in/           # Built-in agents (general, plan, ...)
│       ├── BashTool/               # Thực thi bash commands
│       │   ├── BashTool.tsx
│       │   ├── bashSecurity.ts
│       │   └── commandSemantics.ts
│       ├── FileEditTool/           # Partial file modification (string replace)
│       ├── FileReadTool/           # Đọc files (images, PDFs, notebooks)
│       ├── FileWriteTool/          # Tạo/ghi đè files
│       ├── GlobTool/               # File pattern matching
│       ├── GrepTool/               # ripgrep-based content search
│       ├── WebSearchTool/          # Web search integration
│       ├── WebFetchTool/           # Fetch web content
│       ├── MCPTool/                # Model Context Protocol tool
│       ├── ListMcpResourcesTool/   # Liệt kê MCP resources
│       ├── ReadMcpResourceTool/    # Đọc MCP resources
│       ├── REPLTool/               # REPL execution
│       ├── PowerShellTool/         # PowerShell execution
│       ├── LSPTool/                # Language Server Protocol
│       ├── SkillTool/              # Gọi skill commands
│       ├── AskUserQuestionTool/    # Hỏi user interactively
│       ├── TaskCreateTool/         # Tạo tasks
│       ├── TaskUpdateTool/         # Update tasks
│       ├── TaskStopTool/           # Stop tasks
│       ├── TaskListTool/           # Liệt kê tasks
│       ├── TaskOutputTool/         # Task output handling
│       ├── RemoteTriggerTool/      # Trigger remote agents
│       ├── ScheduleCronTool/       # Schedule cron jobs
│       ├── TodoWriteTool/          # Todo list management
│       ├── SendMessageTool/        # Send messages giữa agents
│       ├── NotebookEditTool/       # Edit Jupyter notebooks
│       ├── EnterPlanModeTool/      # Enter plan mode
│       ├── ExitPlanModeTool/       # Exit plan mode
│       ├── EnterWorktreeTool/      # Enter git worktree isolation
│       ├── ExitWorktreeTool/       # Exit git worktree
│       ├── ConfigTool/             # Config management
│       ├── SyntheticOutputTool/    # Internal output formatting
│       ├── shared/                 # Shared utilities giữa tools
│       └── testing/                # Tool testing utilities
│
├── SERVICES (20 modules)
│   └── services/
│       ├── api/                    # Anthropic API client (Bedrock, Azure, Vertex, Direct)
│       │   ├── client.ts           # Factory function đa provider
│       │   ├── claude.ts           # Main API calls
│       │   ├── errors.ts           # Error types
│       │   ├── usage.ts            # Usage tracking
│       │   └── logging.ts          # Logging utilities
│       ├── mcp/                    # Model Context Protocol (122KB client!)
│       │   ├── client.ts           # Multi-transport: SSE, stdio, HTTP, WS, in-process
│       │   ├── mcpManager.ts
│       │   └── mcpLoader.ts
│       ├── analytics/              # Event logging (zero deps — base layer)
│       │   ├── index.ts            # logEvent(), attachAnalyticsSink()
│       │   ├── firstPartyEventLogger.ts
│       │   ├── datadog.ts
│       │   └── growthbook.ts       # Feature flags
│       ├── oauth/                  # OAuth 2.0 + PKCE flow
│       ├── lsp/                    # Language Server Protocol manager
│       ├── compact/                # Context compression service
│       │   ├── compact.ts
│       │   ├── autoCompact.ts
│       │   └── microCompact.ts
│       ├── extractMemories/        # Auto-extract durable memories (forked agent)
│       ├── autoDream/              # Background memory consolidation (forked agent)
│       ├── SessionMemory/          # Auto-maintain session notes (forked agent)
│       ├── AgentSummary/           # Background summary cho sub-agents
│       ├── plugins/                # Plugin installation & marketplace
│       │   └── PluginInstallationManager.ts
│       ├── remoteManagedSettings/  # Enterprise settings từ remote (polling)
│       ├── settingsSync/           # Sync settings giữa máy
│       ├── teamMemorySync/         # Team memory + secret scanning
│       ├── policyLimits/           # API policy limit enforcement
│       ├── MagicDocs/              # Magic documentation generation
│       ├── PromptSuggestion/       # Prompt suggestion engine
│       ├── claudeAiLimits.ts       # Rate limit tracking (5h, 7d, opus)
│       ├── notifier.ts             # Notification service
│       ├── voice.ts                # Audio recording + push-to-talk
│       ├── vcr.ts                  # VCR recording/playback for testing
│       ├── preventSleep.ts         # Prevent system sleep
│       └── tokenEstimation.ts      # Token count utilities
│
├── COMPONENTS (140+ React/Ink components)
│   ├── components/
│   │   ├── agents/                 # Agent management UI
│   │   │   ├── AgentsList.tsx
│   │   │   ├── AgentDetail.tsx
│   │   │   ├── ModelSelector.tsx
│   │   │   └── new-agent-creation/ # Wizard tạo agent mới
│   │   ├── CustomSelect/           # Custom select component
│   │   ├── Spinner/                # Spinner
│   │   ├── mcp/                    # MCP UI components
│   │   └── wizard/                 # Wizard UI
│   └── screens/                    # Full-screen UIs
│       ├── Doctor.tsx              # Environment diagnostics
│       ├── REPL.tsx                # Interactive shell
│       └── ResumeConversation.tsx  # Session restoration
│
├── HOOKS (100+ React hooks)
│   └── hooks/
│       ├── useCanUseTool.ts        # Kiểm tra tool permissions
│       ├── fileSuggestions.ts      # File suggestion hook
│       ├── notifs/                 # Notification hooks
│       │   ├── useAutoModeUnavailableNotification.ts
│       │   ├── useFastModeNotification.tsx
│       │   ├── useIDEStatusIndicator.tsx
│       │   └── useMcpConnectivityStatus.tsx
│       └── toolPermission/         # Tool permission hooks
│
├── TYPES
│   └── types/
│       ├── command.ts              # Command type definitions
│       ├── message.ts              # Message types
│       ├── permissions.ts          # Permission types
│       ├── plugin.ts               # Plugin types
│       ├── hooks.ts                # Hook types
│       ├── tools.ts                # Tool progress types
│       └── generated/              # Generated protobuf types
│
├── STATE
│   └── state/AppState.ts           # Global application state (React context)
│
├── UTILS (80+ utility modules)
│   └── utils/
│       ├── config.ts               # Configuration loading
│       ├── cwd.ts                  # Current working directory
│       ├── Shell.ts                # Shell execution wrapper
│       ├── log.ts                  # Logging utilities
│       ├── errors.ts               # Error handling
│       ├── theme.ts                # Theme management
│       ├── analyzeContext.ts       # Context analysis
│       ├── thinking.ts             # Extended thinking support
│       ├── fileStateCache.ts       # File state caching
│       ├── fileHistory.ts          # File history tracking
│       ├── attachments.ts          # File attachment handling
│       ├── fastMode.ts             # Fast mode utilities
│       ├── activityManager.ts      # Activity tracking
│       ├── abortController.ts      # Abort control
│       ├── agentContext.ts         # Agent context management
│       ├── advisor.ts              # Advisor/recommendation logic
│       ├── model/                  # Model selection & management
│       ├── permissions/            # Permission system & denial tracking
│       ├── settings/               # Settings management
│       ├── session/                # Session utilities
│       ├── messages/               # Message processing & mappers
│       ├── task/                   # Task utilities
│       ├── skill-search/           # Skill searching & indexing
│       ├── secureStorage/          # Secure credential storage
│       ├── plugins/                # Plugin utilities
│       └── processUserInput/       # User input processing
│
├── REMOTE & BRIDGE
│   ├── bridge/                     # IDE integration (VS Code, JetBrains)
│   │   ├── bridgeMain.ts
│   │   ├── replBridge.ts
│   │   └── jwtUtils.ts
│   ├── remote/                     # Remote sessions (CCR)
│   ├── server/                     # Server mode
│   └── cli/                        # CLI transport layer
│       └── transports/
│           ├── WebSocketTransport.ts
│           ├── SSETransport.ts
│           └── HybridTransport.ts
│
├── SPECIAL SUBSYSTEMS
│   ├── coordinator/                # Multi-agent orchestration
│   ├── skills/                     # Skills system
│   │   ├── bundledSkills.ts        # Built-in skills
│   │   └── loadSkillsDir.ts        # Dynamic skill loading
│   ├── plugins/                    # Plugin architecture
│   │   └── builtinPlugins.ts
│   ├── memdir/                     # Memory directory management
│   │   ├── memdir.ts
│   │   └── paths.ts
│   ├── migrations/                 # 10 config migration files
│   ├── vim/                        # Vim mode implementation
│   ├── voice/                      # Voice mode support
│   ├── keybindings/                # Keybinding configuration
│   ├── assistant/                  # KAIROS assistant mode (feature-flagged)
│   ├── buddy/                      # Companion sprite
│   │   ├── companion.ts
│   │   └── sprites.ts
│   ├── native-ts/                  # Native TypeScript bindings
│   │   ├── color-diff/             # Native color diff
│   │   ├── file-index/             # Native file indexing
│   │   └── yoga-layout/            # Yoga layout engine
│   └── upstreamproxy/              # Proxy configuration
│
└── CONSTANTS & CONFIG
    ├── constants/
    │   ├── xml.ts                  # XML tag constants
    │   └── querySource.ts
    └── schemas/                    # Zod validation schemas
```

---

## Dependency Graph (Services)

```
analytics/ (zero deps — base layer)
    ↑ mọi service đều log vào đây

api/client.ts ←─── oauth/ (token exchange)
    │
    └── forkedAgent ──→ autoDream/
                   ├──→ extractMemories/
                   ├──→ SessionMemory/
                   ├──→ compact/
                   ├──→ AgentSummary/
                   └──→ MagicDocs/

mcp/        ── độc lập (multi-transport: SSE, stdio, HTTP, WS)
lsp/        ── độc lập (closure factory)
plugins/    ── độc lập
voice.ts    ── độc lập (lazy-loaded native module)
```

---

## Architectural Patterns

| Pattern | Ví dụ |
|---|---|
| **Queue Buffer** | analytics — events queued, drained async |
| **Closure Factory** | createLSPServerManager() — no class, no singleton |
| **Forked Subagent** | autoDream, extractMemories, compact — AI gọi AI |
| **Feature Gating** | sessionMemory, remoteManagedSettings — fail-open |
| **Lazy Loading** | voice/audio module — load khi lần đầu dùng |
| **Memoization** | getKubernetesNamespace(), probeArecord() |
| **Marker Types** | AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS |
| **Multi-transport** | MCP client — SSE / stdio / HTTP / WebSocket |
| **Distributed Lock** | autoDream — file mtime lock, prevent concurrent runs |

---

## Public APIs Chính

```typescript
// commands.ts
export async function getCommands(cwd): Promise<Command[]>
export function findCommand(name, commands): Command | undefined
export const REMOTE_SAFE_COMMANDS: string[]
export const BRIDGE_SAFE_COMMANDS: string[]

// services/analytics/index.ts
export function logEvent(eventName, metadata): void
export function logEventAsync(eventName, metadata): Promise<void>
export function attachAnalyticsSink(sink: AnalyticsSink): void

// services/api/client.ts
export async function getAPIClient(config): AnthropicClient
// Supports: Direct API / AWS Bedrock / Azure Foundry / Vertex AI
```

---

## Notes quan trọng

1. **Commands vs Tools**: Commands = slash commands cho user · Tools = functions Claude gọi
2. **Feature Flags**: `feature('FLAG_NAME')` từ `bun:bundle` cho dead code elimination
3. **Forked Agent Pattern**: AI background tasks spawn sub-agent riêng (autoDream, compact, extractMemories)
4. **Permission System**: Mỗi tool có permission check riêng trên từng invocation
5. **Analytics isolation**: `analytics/` không import bất kỳ internal service nào — tránh circular deps
6. **MCP client complexity**: `mcp/client.ts` là file lớn nhất (122KB), 40+ transport/auth combinations
