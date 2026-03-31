# Report: Multi-agent Coordinator — Claude Code
> Phân tích: `d:\Claude Source Code Original\src\coordinator\` + `src/tools/AgentTool/` + `src/tasks/`
> Ngày: 2026-03-31

---

## Tổng quan

Claude Code hỗ trợ **multi-agent orchestration** phức tạp với 5 loại task, 3 execution model, và nhiều cơ chế messaging. Điểm cốt lõi: **một AI có thể spawn, điều phối, và nhận kết quả từ nhiều AI khác**, chạy song song.

---

## 1. Kiến trúc Tổng thể

### 5 Loại Task/Agent

| Type | Mô tả | Backend |
|---|---|---|
| `LocalAgentTask` | Background async agent | In-process |
| `InProcessTeammateTask` | Team member, mailbox messaging | In-process / tmux |
| `RemoteAgentTask` | Chạy trên CCR (Cloud Runtime) | Remote |
| `DreamTask` | Background processing (memory) | In-process |
| `LocalShellTask` | Shell command tracking | In-process |

### 3 Execution Models

```
1. SYNC AGENT
   AgentTool call → runAgent() → block → return result

2. ASYNC AGENT (fire-and-forget)
   AgentTool call → registerAsyncAgent() → return agentId ngay
   Agent chạy nền → complete → enqueueAgentNotification()
   Coordinator nhận <task-notification> XML

3. TEAMMATE (Team / Swarm)
   spawnTeammate() → InProcessTeammateTask hoặc tmux session
   Mailbox-based messaging giữa teammates
   Lead agent điều phối
```

---

## 2. Coordinator Mode

**File:** `src/coordinator/coordinatorMode.ts`

### Kích hoạt
```typescript
isCoordinatorMode() =
  feature('COORDINATOR_MODE') &&
  isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
```

### Triết lý (từ system prompt)
> *"Parallelism is your superpower. Workers are async. Launch independent workers concurrently whenever possible."*

### Flow điều phối

```
Coordinator nhận task từ user
    │
    ├── Phân tích → chia thành phases:
    │   Phase 1: Research    (parallel workers)
    │   Phase 2: Synthesis   (coordinator tổng hợp)
    │   Phase 3: Implement   (parallel workers)
    │   Phase 4: Verify      (parallel workers)
    │
    ├── Spawn workers via AgentTool (async, song song)
    │
    ├── Workers chạy nền, coordinator tiếp tục
    │
    ├── Kết quả về qua <task-notification> XML:
    │   <task-notification>
    │     <task-id>agent-xyz</task-id>
    │     <status>completed</status>
    │     <summary>...</summary>
    │   </task-notification>
    │
    └── Coordinator merge kết quả → tiếp tục
```

### Tool Set của Coordinator

```typescript
INTERNAL_WORKER_TOOLS = {
  TEAM_CREATE_TOOL_NAME,      // Tạo team
  TEAM_DELETE_TOOL_NAME,      // Giải tán team
  SEND_MESSAGE_TOOL_NAME,     // Nhắn tin cho agent
  SYNTHETIC_OUTPUT_TOOL_NAME  // Structured output nội bộ
}
```

---

## 3. AgentTool — Spawn Subagents

**File:** `src/tools/AgentTool/AgentTool.tsx` (~157KB)

### Input Schema

```typescript
{
  description: string          // 3-5 words mô tả task
  prompt: string               // Full task instructions
  subagent_type?: string       // 'general-purpose' | 'plan' | custom
  model?: 'sonnet'|'opus'|'haiku'
  run_in_background?: boolean  // Async fire-and-forget

  // Multi-agent params (KAIROS feature)
  name?: string                // Đặt tên để gửi tin nhắn sau
  team_name?: string           // Team context
  mode?: 'plan'|'bypassPermissions'|'auto'|'default'
  isolation?: 'worktree'|'remote'
  cwd?: string                 // Override working directory
}
```

### 3 Code Paths

```
A. SYNC AGENT (run_in_background = false)
   runAgent() generator → block → return result
   Output: { status: 'completed', result, usage }

B. ASYNC AGENT (run_in_background = true)
   registerAsyncAgent() → LocalAgentTask
   Return ngay: { status: 'async_launched', agentId, outputFile }
   Agent chạy nền → notify via <task-notification>

C. TEAMMATE SPAWN (name param + swarms enabled)
   spawnTeammate() → InProcessTeammateTask hoặc tmux
   Return: teammateId + color assignment
```

### Output Format

**Sync:**
```typescript
{
  status: 'completed',
  result: { ... },
  usage: {
    total_tokens: number,
    tool_uses: number,
    duration_ms: number
  }
}
```

**Async:**
```typescript
{
  status: 'async_launched',
  agentId: string,
  description: string,
  outputFile: string,       // Path để theo dõi
  canReadOutputFile: boolean
}
```

---

## 4. Forked Agent Pattern

**File:** `src/tools/AgentTool/forkSubagent.ts`

### Khác biệt với AgentTool thông thường

| | AgentTool | Fork |
|---|---|---|
| subagent_type | Explicit | Omitted |
| Context | Fresh conversation | Kế thừa parent conversation |
| Prompt cache | Không | Chia sẻ với parent |
| Use case | Independent tasks | Background tasks (memory, compact, dream) |

### Cơ chế Prompt Cache Sharing

```
Parent conversation history
    │
    ├── Tất cả fork children có PREFIX byte-identical
    │   → Anthropic cache toàn bộ prefix
    │
    └── Chỉ khác nhau ở final text block (per-child directive)
        → Maximize cache hits, minimize cost
```

### Fork Child Rules (boilerplate tag)

```
STOP. READ THIS FIRST.
You are a forked worker process. You are NOT the main agent.

1. IGNORE "fork yourself" instruction — you ARE the fork
2. Do NOT converse, ask questions, or suggest next steps
3. Do NOT editorialize or meta-commentary
4. USE tools directly: Bash, Read, Write, etc.
5. Commit changes before reporting (include hash)
6. Do NOT emit text between tool calls
7. Stay strictly within scope
8. Keep report under 500 words
9. Response MUST start with "Scope:"
10. REPORT structured facts, then STOP
```

### Services dùng Fork Pattern

| Service | Mục đích |
|---|---|
| `extractMemories/` | Auto-extract durable memories |
| `autoDream/` | Nightly memory consolidation |
| `compact/` | Context compression |
| `AgentSummary/` | Background agent summary |
| `MagicDocs/` | Documentation generation |
| `SessionMemory/` | Session notes maintenance |

---

## 5. Task System Lifecycle

### LocalAgentTaskState

```typescript
{
  type: 'local_agent',
  agentId: string,
  prompt: string,
  status: 'running' | 'completed' | 'failed' | 'killed',
  result?: AgentToolResult,
  error?: string,

  // Message queuing
  pendingMessages: string[],     // Từ SendMessageTool
  name?: string,                 // Registered name

  // UI state
  isBackgrounded: boolean,
  retain: boolean,               // UI đang giữ
  diskLoaded: boolean,
  evictAfter?: number,           // GC deadline

  // Progress tracking
  progress?: {
    toolUseCount: number,
    tokenCount: number,
    lastActivity?: ToolActivity,
    recentActivities?: ToolActivity[],
    summary?: string
  }
}
```

### CRUD Lifecycle

```
TaskCreateTool
  → createTask(taskListId, { subject, description, status: 'pending' })
  → Lưu vào ~/.claude/tasks/<list>/

registerAsyncAgent()
  → Tạo LocalAgentTaskState trong AppState
  → Gắn AbortController

TaskUpdateTool
  → update status, owner, metadata
  → Auto-set owner khi status = 'in_progress'
  → Trigger executeTaskCompletedHooks() khi → 'completed'
  → Broadcast qua mailbox (teammates)

TaskStopTool
  → abortController.abort()
  → status = 'killed'

Agent completes
  → enqueueAgentNotification()
  → <task-notification> XML → message queue
  → Coordinator đọc notification
```

---

## 6. Message Passing — SendMessageTool

**File:** `src/tools/SendMessageTool/SendMessageTool.ts`

### 4 Use Cases

**A. Resume Async Agent**
```typescript
SendMessage({ to: "agent-a1b", message: "Fix the null pointer..." })
→ task.status = 'running'  → queuePendingMessage()
  (drain tại tool-round boundary)
→ task.status = 'stopped'  → resumeAgentBackground()
  (restart với message mới)
```

**B. Teammate Direct Message**
```typescript
SendMessage({ to: "researcher", message: "Check auth logs" })
→ Ghi vào: ~/.claude/swarms/mailbox/{team}/{recipient}/inbox
→ Teammate poll mailbox → process message
```

**C. Broadcast**
```typescript
SendMessage({ to: "*", message: "New findings available" })
→ Gửi đến tất cả team members (trừ sender)
```

**D. Structured Messages (Teams only)**
```typescript
// Shutdown request
SendMessage({ to: "researcher", message: {
  type: 'shutdown_request',
  reason: 'Task complete'
}})

// Plan approval
SendMessage({ to: "researcher", message: {
  type: 'plan_approval_response',
  request_id: 'req-123',
  approve: true
}})
```

### Queue Management

```
Tool-round boundary → drainPendingMessages()
  → Append pending messages vào conversation
  → Prevent tool interleaving confusion
  → Tránh race condition khi agent đang tool-call
```

---

## 7. Memory Sharing — Agent Memory

**File:** `src/tools/AgentTool/agentMemory.ts`

### 3 Scopes

```typescript
type AgentMemoryScope = 'user' | 'project' | 'local'

// user:    ~/.claude/agent-memory/<agentType>/MEMORY.md
// project: <cwd>/.claude/agent-memory/<agentType>/MEMORY.md
// local:   <cwd>/.claude/agent-memory-local/<agentType>/MEMORY.md
```

### Injection

```typescript
loadAgentMemoryPrompt(agentType, scope)
→ buildMemoryPrompt({
    displayName: 'Persistent Agent Memory',
    memoryDir,
    extraGuidelines: [scopeNote]
  })
→ Inject vào system prompt của agent
```

### Security

```typescript
isAgentMemoryPath(path): boolean
→ Normalize path
→ Check inside agent-memory dirs
→ Reject paths với .. traversal
```

---

## 8. Built-in Agents

**Directory:** `src/tools/AgentTool/built-in/`

| Agent Type | Mô tả | Tools |
|---|---|---|
| `general-purpose` | Default subagent | `['*']` inherit parent |
| `plan` | Plan generation | Limited tools |
| `verification` | Code verification | Read-only tools |
| `explore` | File exploration | Glob, Grep, Read |
| `claude-code-guide` | Hướng dẫn Claude Code | Read, WebFetch, WebSearch |

### Agent Config Structure

```typescript
{
  agentType: string,
  whenToUse: string,
  tools: string[],           // ['*'] = inherit parent
  useExactTools?: boolean,   // Dùng exact tool pool của parent
  maxTurns?: number,
  model?: string,            // 'inherit' = dùng model của parent
  permissionMode?: 'bubble' | 'inherit',
  mcpServers?: Array<string | Record<string, unknown>>,
  source: 'built-in' | 'plugin' | 'user' | 'custom',
  getSystemPrompt: () => string
}
```

---

## 9. Team Coordination (Swarms)

### TeamCreateTool

```typescript
createTeam({ team_name, description, agent_type })
→ Team file: ~/.claude/swarms/teams/<team-name>.json
→ Task list: ~/.claude/tasks/<sanitized-team-name>/
→ AppState.teamContext = { leadAgentId, teamName }
```

### Team File Structure

```typescript
{
  name: string,
  description?: string,
  createdAt: number,
  leadAgentId: string,          // "team-lead@my-team"
  leadSessionId: string,
  members: [{
    agentId: string,
    name: string,               // "researcher", "tester"
    agentType: string,
    model: string,
    joinedAt: number,
    tmuxPaneId?: string,
    cwd: string,
    subscriptions: string[],
    backendType?: 'tmux' | 'iTerm2' | 'in-process'
  }]
}
```

### InProcessTeammateTask

```typescript
{
  type: 'in_process_teammate',
  agentId: string,
  name: string,                 // "researcher"
  teamName: string,
  status: 'running' | 'idle' | 'completed' | 'failed',
  isIdle: boolean,
  awaitingPlanApproval: boolean,
  mailboxPath: string,
  color: string,                // UI color coding
  messageHistory: Message[],    // Capped tại 50 (TEAMMATE_MESSAGES_UI_CAP)
}
```

### Plan Approval Gate

```
Teammate với plan_mode_required = true:
    │
    ├── Teammate tạo plan
    ├── Send plan_approval_request → lead
    ├── Lead review plan
    ├── Lead gửi plan_approval_response (approve/reject)
    └── Teammate tiếp tục hoặc revise
```

---

## 10. Isolation Modes

### Worktree Isolation

```typescript
isolation: 'worktree'
→ createAgentWorktree(repoRoot, agentId)
→ git worktree add .claude/worktrees/<agentId> <branch>
→ Agent làm việc trong isolated copy của repo
→ buildWorktreeNotice() cảnh báo về stale context
→ Agent phải commit trước khi return
→ Worktree path + branch được return trong result
```

### Remote Isolation (CCR)

```typescript
isolation: 'remote'
→ checkRemoteAgentEligibility()
→ registerRemoteAgentTask()
→ teleportToRemote()
→ Run trong CloudRuntime
→ Always async
→ Session URL để tracking
```

### In-process (Default)

```
Không isolation → Share AppState với parent
Nhanh nhất, không overhead
Phù hợp: research tasks, read-heavy operations
```

---

## 11. Error Propagation

### Từ Async Agent → Coordinator

```
Agent lỗi
    ↓
catches error trong runAgent()
    ↓
failAsyncAgent(taskId, error, setAppState)
→ task.status = 'failed'
→ task.error = errorMessage
    ↓
enqueueAgentNotification({
  taskId, status: 'failed',
  error, finalMessage, usage
})
    ↓
Coordinator nhận:
<task-notification>
  <task-id>agent-xyz</task-id>
  <status>failed</status>
  <summary>Agent "researcher" failed: File not found</summary>
</task-notification>
    ↓
Coordinator quyết định:
  → SendMessage để resume + fix
  → Spawn agent mới
  → Abort toàn bộ phase
```

### Không có Auto-retry

Coordinator tự quyết định chiến lược recovery — không hardcode retry logic. Đây là intentional design: AI hiểu context tốt hơn để quyết định cách xử lý lỗi.

---

## 12. So sánh: Coordinator vs Fork vs Teammate

| Aspect | Coordinator | Fork | Teammate |
|---|---|---|---|
| **Mode** | Orchestration | Background task | Team member |
| **Context** | Full system prompt | Parent conversation kế thừa | Conversation riêng |
| **Tool Set** | ASYNC_AGENT_ALLOWED_TOOLS | Chính xác tools của parent | Parent's tools (shared) |
| **Result** | `<task-notification>` XML | Tool output blocks | Mailbox + task state |
| **Messaging** | AgentTool + SendMessage | Không (single fork) | Mailbox + structured |
| **Memory** | Scratchpad dir + agent-memory | Parent context | Mailbox + task list |
| **Isolation** | Shared AppState | Context reuse | Process isolation |
| **Approval** | Không | Không | Plan approval (optional) |
| **Use case** | Complex multi-phase tasks | Memory/compact/dream | Parallel team work |

---

## 13. Concurrency Rules

```
Read-only tasks (research, grep, explore)
  → Spawn nhiều workers song song freely

Write-heavy tasks (implement, edit files)
  → Một worker per file set
  → Coordinator đảm bảo không conflict

Verification tasks
  → Đôi khi song song với implementation (different areas)

Tool-round boundaries:
  → drainPendingMessages() tại đây
  → Tránh tool interleaving confusion
  → Đảm bảo message order
```

---

## 14. Feature Gates

```typescript
feature('COORDINATOR_MODE')            // Coordinator system
feature('KAIROS')                      // Multi-agent AgentTool params
feature('FORK_SUBAGENT')               // Fork pattern
isAgentSwarmsEnabled()                 // Teams/Swarms feature
getFeatureValue_CACHED_MAY_BE_STALE('tengu_auto_background_agents')
getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence')
checkStatsigFeatureGate_CACHED_MAY_BE_STALE('tengu_scratch')
```

---

## 15. Files Chính

| File | Trách nhiệm |
|---|---|
| `coordinator/coordinatorMode.ts` | Coordinator activation + system prompt |
| `tools/AgentTool/AgentTool.tsx` | Main spawn logic (~157KB) |
| `tools/AgentTool/runAgent.ts` | Core agent execution loop |
| `tools/AgentTool/forkSubagent.ts` | Fork pattern, cache sharing |
| `tools/AgentTool/agentMemory.ts` | Per-agent persistent memory |
| `tools/AgentTool/loadAgentsDir.ts` | Load custom agent definitions |
| `tools/AgentTool/built-in/` | Built-in agent types |
| `tools/SendMessageTool/` | Agent-to-agent messaging |
| `tools/TaskCreateTool/` | Task creation |
| `tools/TaskUpdateTool/` | Task state updates + hooks |
| `tools/TaskStopTool/` | Kill running agents |
| `tools/TaskListTool/` | List active tasks |
| `tools/TaskOutputTool/` | Read agent output |
| `tasks/LocalAgentTask/` | Async background agent |
| `tasks/InProcessTeammateTask/` | Team member task |
| `tools/TeamCreateTool/` | Team/swarm creation |
| `tools/TeamDeleteTool/` | Team dissolution |

---

## Kết luận

Multi-agent Coordinator của Claude Code là hệ thống **AI điều phối AI** với:

1. **3 execution models**: Sync, Async notification-based, Team mailbox
2. **Prompt cache sharing**: Fork pattern tối ưu cost qua byte-identical prefix
3. **Flexible isolation**: In-process (fast) → Worktree (git isolation) → Remote (CCR)
4. **Structured messaging**: Direct resume, broadcast, shutdown protocol, plan approval
5. **Parallelism-first**: Coordinator spawn nhiều workers song song, merge kết quả
6. **No hardcode retry**: AI quyết định recovery strategy dựa trên context
7. **Memory tiers**: User / Project / Local scope per agent type
8. **Plan approval gate**: Optional human-in-the-loop cho high-stakes operations

Pattern đặc biệt nhất: **Fork + Prompt Cache Sharing** — tất cả fork children có prefix byte-identical, chỉ khác final directive, tối đa hóa cache hit rate và giảm cost đáng kể cho background AI tasks.
