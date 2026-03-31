# Report: Lessons Learned — Sử dụng Claude Code Hiệu quả
> Đúc rút từ phân tích source code: QueryEngine, Memory System, Service Layer, Multi-agent Coordinator
> Ngày: 2026-03-31

---

## Tổng quan

Sau khi phân tích ~512K lines source code của Claude Code, dưới đây là những bài học thực tiễn giúp sử dụng Claude Code hiệu quả hơn — không phải từ documentation, mà từ cách hệ thống thực sự hoạt động bên trong.

---

## Bài học 1: Đầu tư vào Memory Files — ROI cao nhất

### Cơ chế thực tế
- `MEMORY.md` được inject vào **mọi system prompt** — Claude luôn thấy nó dù bạn không nói gì
- Query-time recall dùng Sonnet để chọn **≤5 files** dựa trên `description` trong frontmatter
- `extractMemories` forked agent chạy **sau mỗi turn** và tự quyết định gì cần lưu

### Áp dụng
- Viết `description` thật cụ thể — đây là thứ quyết định file có được đọc không:
  ```markdown
  ---
  name: Testing Rule
  description: Không mock database trong integration tests — gây incident Q3 2025
  type: feedback
  ---
  Lý do: Mock/prod divergence làm test pass nhưng migration lỗi production.
  How to apply: Mọi integration test phải hit real database.
  ```
- Giữ `MEMORY.md` **dưới 200 dòng** — dòng 201+ bị cắt im lặng, không có warning
- Dùng đúng 4 loại type:

  | Type | Khi nào dùng |
  |---|---|
  | `user` | Vai trò, kỹ năng, sở thích của bạn |
  | `feedback` | Quy tắc làm việc, do/don't |
  | `project` | Deadline, quyết định, context dự án |
  | `reference` | Pointer đến Linear, Grafana, Slack... |

### Tại sao quan trọng
`feedback` memories là loại có ROI cao nhất — Claude sẽ không lặp lại sai lầm cũ trong mọi session tương lai mà không cần bạn nhắc lại.

---

## Bài học 2: Nói rõ khi Claude làm Đúng lẫn Sai

### Cơ chế thực tế
`extractMemories` không phân biệt được đúng/sai nếu bạn không nói. Nó extract từ conversation pattern — nếu bạn im lặng sau khi Claude làm sai, nó có thể extract approach sai thành memory.

### Áp dụng
- Khi Claude làm **sai** → nói thẳng, có lý do:
  > *"Đừng dùng cách này vì nó gây N+1 query. Dùng eager loading thay thế."*

- Khi Claude làm **đúng** theo cách không hiển nhiên → confirm:
  > *"Đúng rồi, bundled PR là đúng trong trường hợp này. Tiếp tục theo hướng đó."*

- Cả hai đều được extract thành `feedback` memory cho session sau

---

## Bài học 3: Hiểu Token Budget — Đừng để bị gián đoạn

### Cơ chế thực tế
QueryEngine có **diminishing returns detection**:
```
delta_tokens < 500 VÀ continuationCount >= 3
→ Auto-inject nudge: "Wrap up, you're running low on budget"
```
Khi đạt cost ceiling (`maxBudgetUsd`) → Claude tự inject "wrap up" message.

### Áp dụng
- **Task lớn → chia nhỏ** thành chunks rõ ràng thay vì một prompt khổng lồ
- Nếu Claude bắt đầu "tóm tắt" giữa chừng khi chưa xong → đó là budget signal, không phải lỗi
- Dùng `/compact` **chủ động ở 60-70% context**, không chờ bị ép:
  ```
  /compact Focus on [task hiện tại] — sections 1-3 đã xong, đang làm section 4
  ```
- Dùng `/clear` giữa các task không liên quan để reset context sạch

---

## Bài học 4: Layered Recovery Hoạt động Tự động — Đừng Interrupt

### Cơ chế thực tế
Khi gặp lỗi, QueryEngine thử **4 tầng recovery** trước khi báo lỗi ra ngoài:
```
Tầng 1: Context Collapse drain      → ít destructive nhất
Tầng 2: Reactive Compact (summary)  →
Tầng 3: Output token escalate 8K→64K →
Tầng 4: Multi-turn resume           → nhiều destructive nhất
```
Lỗi chỉ hiện ra khi **cả 4 tầng đều thất bại**.

### Áp dụng
- Nếu Claude "chậm" hoặc "im lặng" một lúc → đang chạy recovery, **không interrupt**
- Lỗi *"prompt too long"* xuất hiện = 4 tầng đều thất bại → lúc đó mới dùng `/compact`
- Lỗi *"max output tokens"* → Claude tự escalate 8K→64K rồi multi-turn resume, thường tự xử lý được

---

## Bài học 5: CLAUDE.md — Viết như Prompt, không như Documentation

### Cơ chế thực tế
`CLAUDE.md` được inject vào **mọi API call** như một phần của user context (không phải system prompt riêng). Nó được đọc mỗi turn, không chỉ đầu session.

### Áp dụng
- Viết **imperative, ngắn gọn**:
  ```markdown
  ✅ "Luôn dùng TypeScript strict mode"
  ❌ "Chúng ta sử dụng TypeScript trong project này với strict mode được bật..."
  ```
- Đặt **rule quan trọng nhất ở đầu** — token budget ảnh hưởng đến phần cuối
- **Không giải thích lý do** trong CLAUDE.md — để lý do trong memory files (tiết kiệm token)
- Dùng `@file-path` để reference file khác thay vì copy nội dung

---

## Bài học 6: Tận dụng Multi-agent cho Task Song song

### Cơ chế thực tế
Coordinator system thiết kế theo triết lý:
> *"Parallelism is your superpower. Workers are async. Launch independent workers concurrently whenever possible."*

Async agents trả về ngay (`agentId`), chạy nền, notify qua `<task-notification>` XML khi xong.

### Áp dụng
- Thay vì tuần tự:
  ```
  Research → Fix → Test → Document
  ```
- Dùng song song:
  ```
  Agent A: Research root cause
  Agent B: Tìm test cases liên quan     ← cùng lúc
  Agent C: Đọc documentation            ← cùng lúc
  → Tổng hợp kết quả → Fix → Done
  ```
- Dùng `run_in_background: true` cho tasks độc lập nhau
- Kết quả về dần — không cần chờ tất cả xong mới tiếp tục

---

## Bài học 7: Worktree Isolation cho Mọi Thứ Rủi ro

### Cơ chế thực tế
```
isolation: 'worktree'
→ git worktree add .claude/worktrees/<agentId>
→ Agent làm việc trong bản copy hoàn toàn riêng biệt
→ Phải commit trước khi return kết quả
→ Worktree path + branch trả về trong result
```

### Áp dụng
- **Refactor lớn** → dùng worktree, xem kết quả, merge nếu ổn
- **Thử nghiệm approach không chắc** → worktree, không sợ ảnh hưởng code chính
- **Tránh situation nguy hiểm**: Claude đang sửa 5 files giữa chừng, bạn interrupt → code broken
- Sau khi worktree xong → review diff → merge thủ công hoặc tự động

---

## Bài học 8: Abort Đúng Lúc — Write/Edit Không Dừng Ngay

### Cơ chế thực tế
```typescript
interruptBehavior() = 'cancel'  → dừng ngay   (Bash, WebFetch, WebSearch)
interruptBehavior() = 'block'   → chờ xong    (Write, Edit — data integrity)
```
Sibling tools: khi 1 tool lỗi → kill sibling tools, nhưng **không kill parent query**.

### Áp dụng
- Nhấn ESC khi Claude đang **write/edit file** → nó hoàn thành file đó trước rồi mới dừng (intentional)
- Đây là tính năng, không phải bug — tránh corrupt file giữa chừng
- Muốn dừng hẳn → đợi write xong → ESC ở bước tiếp theo
- Bash tool có thể bị cancel ngay → an toàn hơn khi interrupt

---

## Bài học 9: MCP Servers — Mở rộng Tool Set đúng cách

### Cơ chế thực tế
MCP client hỗ trợ 4 transport (SSE, stdio, HTTP, WebSocket) và tự động convert MCP tools → Claude tools. Một khi kết nối, Claude dùng như native tools.

### Áp dụng
- Database (PostgreSQL, SQLite), GitHub, Linear, Slack, Grafana → đều có MCP servers
- Khai báo trong settings:
  ```json
  {
    "mcpServers": {
      "github": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"] }
    }
  }
  ```
- Mô tả context trong CLAUDE.md để Claude biết khi nào nên dùng MCP tool nào

---

## Bài học 10: Session Memory ≠ Persistent Memory

### Cơ chế thực tế

| | Session Memory | Persistent Memory |
|---|---|---|
| Lưu ở | Context window (tạm) | `~/.claude/projects/.../memory/*.md` |
| Tồn tại | Trong session hiện tại | Qua mọi session |
| Tạo bởi | Auto (context summary) | extractMemories agent |
| Mục đích | Tiết kiệm token | Kiến thức lâu dài |

### Áp dụng
- *"Claude nhớ trong session"* ≠ *"Claude sẽ nhớ session sau"*
- Muốn Claude nhớ lâu dài → nói rõ:
  > *"Hãy lưu điều này vào memory — chúng ta dùng Railway thay vì Heroku cho project này"*
- Review memory files **định kỳ** — xóa entries lỗi thời, memories stale có freshness warning sau 1 ngày

---

## Bài học 11: Streaming Tool Execution — Tools Chạy Song song với Streaming

### Cơ chế thực tế
Tools bắt đầu execute **ngay khi block stream đến**, không chờ model trả xong message. `StreamingToolExecutor` chạy concurrent-safe tools song song.

### Áp dụng
- Claude trả lời + chạy tools **cùng lúc** → tại sao response cảm giác "nhanh"
- Nếu thấy tool results xuất hiện trong khi Claude vẫn đang stream text → đây là feature
- Đừng tạo dependencies không cần thiết giữa tasks → để tools chạy song song tự nhiên

---

## Bài học 12: Thinking Mode — Khi nào nên bật

### Cơ chế thực tế
```typescript
ThinkingConfig =
  | { type: 'disabled' }
  | { type: 'adaptive' }   // Auto-enable theo complexity
  | { type: 'enabled' }
```
Thinking blocks được **bảo vệ nghiêm ngặt** — không được modify, phải giữ nguyên trajectory.

### Áp dụng
- `adaptive` (default) — Claude tự quyết định khi nào cần think sâu
- Dùng `/effort` để tăng thinking intensity cho tasks phức tạp
- Debug / architecture decisions / complex refactor → thinking mode có lợi rõ ràng
- Simple CRUD, rename, format → thinking mode lãng phí tokens

---

## Priority Order — Làm gì trước

```
🥇 Đầu tư vào Memory files (feedback + project type)
    → Một lần setup, benefit vĩnh viễn

🥈 Viết CLAUDE.md đúng cách (imperative, ngắn)
    → Ảnh hưởng mọi API call

🥉 Chia task lớn thành subtasks song song
    → Tận dụng multi-agent, giảm thời gian

4️⃣  Dùng /compact chủ động ở 60-70%
    → Tránh bị interrupt giữa task quan trọng

5️⃣  Worktree isolation cho mọi thứ rủi ro
    → Không bao giờ corrupt working code
```

---

## Những Điều Hay bị Hiểu Nhầm

| Hiểu nhầm | Thực tế |
|---|---|
| Claude "nhớ" từ session trước | Không — chỉ nhớ nếu có persistent memory files |
| ESC dừng file write ngay | Không — Write/Edit tool finish trước khi abort |
| Lỗi "prompt too long" = cần restart | Không — thử `/compact` trước, Claude có 4-layer recovery |
| Interrupt khi Claude "chậm" | Không — có thể đang chạy recovery cycle |
| CLAUDE.md chỉ đọc đầu session | Không — inject vào mọi API call |
| Multi-agent = phức tạp | Không — `run_in_background: true` là đủ để bắt đầu |

---

## Nguồn

Các bài học trên được đúc rút từ phân tích trực tiếp source code tại `d:\Claude Source Code Original\src\`:

- [REPORT-memory-system.md](REPORT-memory-system.md) — Memory architecture
- [REPORT-service-layer.md](REPORT-service-layer.md) — Service layer patterns
- [REPORT-query-engine.md](REPORT-query-engine.md) — QueryEngine internals
- [REPORT-multi-agent-coordinator.md](REPORT-multi-agent-coordinator.md) — Multi-agent system
