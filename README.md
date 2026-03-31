# Bản Sao Mã Nguồn Claude Code phục vụ Nghiên cứu Bảo mật

> Repository này lưu trữ **bản sao mã nguồn Claude Code bị lộ công khai**, có thể truy cập được vào ngày **31 tháng 3 năm 2026** thông qua việc lộ file source map trên hệ thống phân phối npm. Repository được duy trì phục vụ **nghiên cứu giáo dục, nghiên cứu bảo mật phòng thủ, và phân tích rủi ro chuỗi cung ứng phần mềm**.

---

## Ngữ cảnh Nghiên cứu

Repository này được duy trì bởi một **sinh viên đại học** chuyên nghiên cứu về:

- Lỗ hổng chuỗi cung ứng phần mềm và rò rỉ artifact khi build
- Quy trình kỹ thuật phần mềm an toàn
- Kiến trúc hệ thống công cụ phát triển Agentic (Agentic developer tooling)
- Phân tích phòng thủ các hệ thống CLI thực tế

Kho lưu trữ này nhằm mục đích hỗ trợ:

- Học tập và giáo dục
- Thực hành nghiên cứu bảo mật an toàn thông tin
- Đánh giá kiến trúc hệ thống
- Thảo luận về các lỗi trong quá trình đóng gói và phát hành phần mềm

Repository này **không** nhận quyền sở hữu đối với mã nguồn gốc và không nên được xem là repository chính thức của Anthropic.

---

## Ảnh chụp mã nguồn công khai (Snapshot) bị lọt bằng cách nào

[Chaofan Shou (@Fried_rice)](https://x.com/Fried_rice) đã lên tiếng xác nhận mã nguồn của Claude Code có thể lấy được qua tệp định dạng `.map` bị lộ bên trong các package npm:

> **"Mã nguồn của Claude code đã bị rò rỉ thông qua một file map nằm trong kho registry npm của họ!"**
>
> — [@Fried_rice, Ngày 31 tháng 3, 2026](https://x.com/Fried_rice/status/2038894956459290963)

File source map bị đăng tải đã trỏ tới mã nguồn gốc TypeScript hoàn toàn không bị làm rối (unobfuscated) lưu trữ trên R2 bucket của Anthropic. Điều này dẫn đến việc thư mục `src/` có thể bị tải xuống công khai.

---

## Phạm vi Repository

Claude Code là công cụ CLI của Anthropic dùng để tương tác với Claude trực tiếp từ terminal, hỗ trợ thực hiện các tác vụ lập trình như chỉnh sửa file, chạy các dòng lệnh, tìm kiếm trong codebase, và điều phối quy trình làm việc (workflow).

Repository này chứa bản sao/mirror thư mục `src/` nhằm nghiên cứu và phân tích.

- **Ngày phát hiện rò rỉ công khai**: 31-03-2026
- **Ngôn ngữ**: TypeScript
- **Môi trường chạy (Runtime)**: Bun
- **Giao diện Terminal**: React + [Ink](https://github.com/vadimdemedes/ink)
- **Quy mô**: ~1.900 files, 512.000+ dòng code

---

## Cấu trúc Thư mục

```text
src/
├── main.tsx                 # Entrypoint điều phối luồng (Dùng Commander.js cho luồng CLI)
├── commands.ts              # Nơi đăng ký các câu lệnh Command
├── tools.ts                 # Nơi đăng ký các Tools
├── Tool.ts                  # Định nghĩa các type (kiểu dữ liệu) của Tool
├── QueryEngine.ts           # LLM query engine (Lõi điều phối truy vấn AI)
├── context.ts               # Thu thập ngữ cảnh System/Use
├── cost-tracker.ts          # Theo dõi chi phí Token
│
├── commands/                # Nơi triển khai Slash command (~50)
├── tools/                   # Nơi triển khai Agent tool (~40)
├── components/              # Các UI Component của thư viện Ink (~140)
├── hooks/                   # Các React hooks
├── services/                # Tích hợp dịch vụ bên ngoài (External service)
├── screens/                 # Các giao diện Full-screen (Doctor, REPL, Resume)
├── types/                   # Định nghĩa các kiểu TypeScript
├── utils/                   # Các hàm tiện ích
│
├── bridge/                  # Cầu nối kết nối với các IDE và điểu khiển Remote
├── coordinator/             # Bộ điều phối Đa tác nhân (Multi-agent coordinator)
├── plugins/                 # Hệ thống Plugin
├── skills/                  # Hệ thống Kỹ năng (Skill)
├── keybindings/             # Cấu hình gán phím (Keybinding)
├── vim/                     # Chế độ Vim
├── voice/                   # Chế độ nhập giọng nói (Voice input)
├── remote/                  # Các session từ xa (Remote sessions)
├── server/                  # Chế độ máy chủ (Server mode)
├── memdir/                  # Thư mục bộ nhớ lưu trữ bền vững (Persistent memory)
├── tasks/                   # Quản lý tác vụ (Task management)
├── state/                   # Quản lý trạng thái (State management)
├── migrations/              # Di chuyển/Nâng cấp cấu hình (Migrations)
├── schemas/                 # Lược đồ cấu hình dữ liệu (Zod)
├── entrypoints/             # Logic khởi tạo (Initialization)
├── ink/                     # Wrapper xử lý hiển thị Ink
├── buddy/                   # Hình hoạt họa đồng hành (Companion sprite)
├── native-ts/               # Các utility TypeScript thuần túy (Native TypeScript)
├── outputStyles/            # Xử lý định dạng giao diện output
├── query/                   # Pipeline xử lý Truy vấn
└── upstreamproxy/           # Cấu hình máy chủ Proxy (Proxy configuration)
```

---

## Tóm tắt Kiến trúc

### 1. Hệ thống Tool (`src/tools/`)

Mỗi công cụ (tool) mà Claude Code có thể gọi đều được thiết kế dưới dạng một module độc lập. Mỗi tool tự định nghĩa lược đồ (schema) đầu vào, mô hình quản lý quyền (permissions), và luồng xử lý thực thi.

| Công cụ | Mô tả |
|---|---|
| `BashTool` | Thực thi các đoạn lệnh trong Shell |
| `FileReadTool` | Đọc tệp tin (ảnh, PDF, notebook) |
| `FileWriteTool` | Tạo tệp tin mới / ghi đè (Overwrite) |
| `FileEditTool` | Chỉnh sửa một phần của tệp (Tìm và thay thế - string replacement) |
| `GlobTool` | Tìm kiếm dựa trên so khớp mẫu pattern tệp (Glob) |
| `GrepTool` | Tìm kiếm nội dung dựa trên bộ máy ripgrep |
| `WebFetchTool` | Tải dữ liệu từ URL |
| `WebSearchTool` | Tìm kiếm trang web |
| `AgentTool` | Triệu gọi (Spawning) các sub-agent (tác nhân con) |
| `SkillTool` | Thực thi các Kỹ năng (Skill) |
| `MCPTool` | Gọi trực tiếp tool từ các máy chủ MCP (Model Context Protocol) |
| `LSPTool` | Tích hợp cùng Giao thức máy chủ ngôn ngữ (Language Server Protocol) |
| `NotebookEditTool` | Trình sửa lỗi Jupyter notebook |
| `TaskCreateTool` / `TaskUpdateTool` | Quản lý và tạo mới tác vụ (Task) |
| `SendMessageTool` | Chức năng nhắn tin liên tác nhân (Inter-agent messaging) |
| `TeamCreateTool` / `TeamDeleteTool` | Quản lý team gồm các Agent |
| `EnterPlanModeTool` / `ExitPlanModeTool` | Bật/tắt chế độ lên kế hoạch (Plan mode) |
| `EnterWorktreeTool` / `ExitWorktreeTool` | Cách ly dự án thông qua Git worktree |
| `ToolSearchTool` | Cơ chế khám phá công cụ trả sau (Deferred tool discovery) |
| `CronCreateTool` | Lên lịch kích hoạt định kỳ (Cron Trigger) |
| `RemoteTriggerTool` | Kích hoạt từ xa (Remote Trigger) |
| `SleepTool` | Đợi chủ động (trong Proactive mode) |
| `SyntheticOutputTool` | Sinh ra các luồng phản hồi có cấu trúc (Structured output) |

### 2. Hệ thống Command (`src/commands/`)

Hệ thống cung cấp các lệnh "slash" cho người dùng, sử dụng tiền tố `/`.

| Lệnh | Mô tả |
|---|---|
| `/commit` | Tạo mới một Git commit |
| `/review` | Rà soát mã nguồn (Code review) |
| `/compact` | Nén các Context lại (Compression) |
| `/mcp` | Quản lý kết nối các máy chủ MCP |
| `/config` | Quản lý các thiết lập chung |
| `/doctor` | Chuẩn đoán và kiểm tra môi trường hệ thống |
| `/login` / `/logout` | Chứng thực người dùng |
| `/memory` | Quản lý bộ nhớ dài hạn |
| `/skills` | Quản lý bộ kỹ năng |
| `/tasks` | Quản lý danh sách tác vụ |
| `/vim` | Kích hoạt chế độ Vim |
| `/diff` | Xem các thay đổi |
| `/cost` | Phân tích chi phí (Token usage) |
| `/theme` | Thay đổi hiển thị màu sắc |
| `/context` | Diễn họa trực quan các context hiện tại |
| `/pr_comments` | Xem danh sách các Comment trên PR |
| `/resume` | Khôi phục lại session trước đó |
| `/share` | Chia sẻ session hiện hành |
| `/desktop` | Chuyển luồng sang ứng dụng máy tính (Desktop App) |
| `/mobile` | Chuyển luồng sang điện thoại (Mobile app) |

### 3. Tầng Dịch vụ (Service Layer) (`src/services/`)

| Dịch vụ | Mô tả |
|---|---|
| `api/` | API client cho Anthropic, file API, bootstrap |
| `mcp/` | Kết nối và quản trị liên thông MCP Server |
| `oauth/` | Quy trình luồng OAuth 2.0 |
| `lsp/` | Công cụ quản lý LSP (Language Server Protocol) |
| `analytics/` | Quản lý thống kê & công tắc tắt/mở năng tính (GrowthBook Feature flags) |
| `plugins/` | Bộ loader dành cho plugins |
| `compact/` | Cơ chế nén context cho cuộc trò chuyện |
| `policyLimits/` | Các chính sách và giới hạn cho cá nhân/doanh nghiệp |
| `remoteManagedSettings/` | Các tùy chọn hệ thống điều khiển từ xa |
| `extractMemories/` | Chiết xuất tự động ký ức vào bộ nhớ dài hạn |
| `tokenEstimation.ts` | Ước tính số token tiêu thụ |
| `teamMemorySync/` | Đồng bộ bộ nhớ nội bộ (Sync Memory) xuyên nhóm Agent |

### 4. Hệ thống Cầu nối (Bridge System) (`src/bridge/`)

Tầng giao tiếp hai chiều liên kết giữa các extension/plugin IDE (VS Code, JetBrains) với Claude Code CLI.

- `bridgeMain.ts` — Vòng lặp chính của bridge (Main loop)
- `bridgeMessaging.ts` — Giao thức truyền tin
- `bridgePermissionCallbacks.ts` — Gọi hàm xin lại quyền truy cập (Permission callbacks)
- `replBridge.ts` — REPL session bridge
- `jwtUtils.ts` — Xác thực thông qua JWT
- `sessionRunner.ts` — Trình thực thi session management

### 5. Hệ thống Quản quyền (Permission System) (`src/hooks/toolPermission/`)

Kiểm tra gắt gao trên **mọi quá trình Tool được kích hoạt**. Hệ thống sẽ đưa ra prompt hiển thị trên màn hình xác nhận yêu cầu Đồng ý/Từ chối tùy theo ngữ cảnh của mã nguồn hoặc tự động chấp thuận dựa theo chế độ Permission đã trọn (`default`, `plan`, `bypassPermissions`, `auto`,...).

### 6. Đóng cắt chức năng (Feature Flags)

Cơ chế chặn/xóa các dòng code thừa hoặc đóng gói không sử dụng (Dead code elimination) của tính năng `bun:bundle`:

```typescript
import { feature } from 'bun:bundle'

// Các đoạn mã chưa/phải loại bỏ sẽ tự động bị stripped khi Bun compile
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

Các tính năng nổi bật có Flag chèn ẩn: `PROACTIVE`, `KAIROS`, `BRIDGE_MODE`, `DAEMON`, `VOICE_MODE`, `AGENT_TRIGGERS`, `MONITOR_TOOL`.

---

## Chi tiết các Tệp lõi (Key Files in Detail)

### `QueryEngine.ts` (~46K dòng)

Cỗ máy lõi phân tích yêu cầu LLM API. Công cụ xử lý luồng stream AI response, vòng lặp Tool gọi đệ quy (Tool-call loops), chế độ suy nghĩ đa tuyến (thinking mode), thử lại khi gặp lỗi (retry logic), và bộ cộng tích tokens.

### `Tool.ts` (~29K dòng)

Khung (Base) chuyên môn chỉ định tính năng cấu tạo nên Tool — Input Validation lược đồ, quản lý quyền hạn/Model Permissions, xử lý lộ trình tiến độ.

### `commands.ts` (~25K dòng)

Hỗ trợ đăng ký và thực thi hàng loạt các "slash command". Tự động import và dùng điều kiện môi trường tương thích.

### `main.tsx`

Tệp gốc dựa trên Commander.js parser và khởi chạy khung nhìn hiển thị Terminal Ink. Lúc bắt đầu boot, nó tối ưu hóa cực đỉnh bằng sự giao thoa chạy Fetch keychain từ trước lên RAM, cùng GrowthBook, MDM cho phép khởi chạy gần như tức thời.

---

## Tech Stack (Hệ Sinh Thái)

| Danh mục | Công nghệ sử dụng |
|---|---|
| Môi trường Runtime | [Bun](https://bun.sh) |
| Ngôn ngữ phát triển | TypeScript (chế độ gắt gao - strict mode) |
| Lớp Giao diện Terminal | [React](https://react.dev) + [Ink](https://github.com/vadimdemedes/ink) |
| CLI Parsing | [Commander.js](https://github.com/tj/commander.js) (Kèm extra-typings) |
| Schema Data Validation | [Zod v4](https://zod.dev) |
| Tính năng Search Code | [ripgrep](https://github.com/BurntSushi/ripgrep) |
| Giao thức chuẩn hóa | [MCP SDK](https://modelcontextprotocol.io), LSP |
| Endpoint API | [Anthropic SDK](https://docs.anthropic.com) |
| Thu thập Telemetry | OpenTelemetry + gRPC |
| Đóng/Tắt Feature Flags| GrowthBook |
| Quy trình Auth| OAuth 2.0, JWT, macOS Keychain |

---

## Các Pattern Đáng Chú Ý (Notable Design Patterns)

### Mồi Song Song (Parallel Prefetch)

Độ trễ khởi động được ưu hóa tối đa nhằm giúp các Settings (MDM), chuẩn bị chuỗi khóa bí mật (Keychain), và bật các Session liên kết ngay cả trước khi Module JS được Evaluation xử lý.

```typescript
// main.tsx — kích hoạt từ ban đầu dưới dạng side-effects 
startMdmRawRead()
startKeychainPrefetch()
```

### Lazy Loading

Các logic cồng kềnh (Analytics, gRPC, OpenTelemetry, và một vài Sub hệ thống khoá tính năng) chỉ bị nhét vào qua dynamic class `import()` khi và chỉ khi nó thực sự cần được dùng.

### Cụm Đại Cử (Agent Swarms)

Khả năng đẻ con Agent vô hạn dưới tay `AgentTool`, giao phó quyền quản trị vào `coordinator/` đóng vai Multi-agent. Tính năng `TeamCreateTool` giúp tạo cả một group riêng lẻ gọi nhau làm việc song song.

### Hệ thống Kỹ Năng Kèm Theo (Skill System)

Hệ thống cho phép ném vào thư mục `skills/` một loạt các luồng logic định sẵn, qua mặt Tool gốc tự custom quy trình làm việc theo ý Client.

### Cấp Bậc Kiến trúc Plugin (Plugin Architecture)

Hệ thống mở giúp nạp thêm bên thứ 3 và các chức năng ẩn dựa tên thư mục `plugins/`.

---

## Tuyên Bố Về Bản Quyền / Nghiên Cứu (Research / Ownership Disclaimer)

- Đây là một **Kho nghiên cứu tự vệ giáo dục (Educational and Defensive Security)** của sinh viên tạo ra nhằm rà soát bảo mật.
- Dự án nhắm tới mổ xẻ lỗi bị tuồn API, cách các nền tảng phát tán bị sập (packaging failures), và lối thiết kế "Đại Tác nhân CLI mới".
- Bản quyền thực thụ và tài sản thuộc về đội ngũ sản xuất và quản trị tại **Anthropic**.
- Dự án này **không trực thuộc hợp quy, tiếp sức, hay công bố đại diện bảo trì với Anthropic**.
