# Redmine Local AI Workspace — Product & Technical Plan v2.1

**Phiên bản:** v2.1  
**Ngôn ngữ:** Tiếng Việt  
**Định hướng mới:** Local web app + deterministic tool gateway + Codex CLI orchestration + specialized skills/sub-agents + optional MarkItDown document conversion  
**Mục tiêu:** Xây dựng một app local giúp theo dõi Redmine ticket bằng browser automation, lưu snapshot Markdown/JSON, phát hiện thay đổi thật, dịch nội dung Nhật → Việt, gom context ngoài Redmine, và tận dụng Codex CLI cho các tác vụ AI thông qua skills/sub-agents có kiểm soát.

---

## 1. Tóm tắt ý tưởng

Redmine Local AI Workspace là một **personal local workbench** cho developer/maintainer làm việc với Redmine.

App không thay thế Redmine. App đóng vai trò là lớp workspace cá nhân:

- Theo dõi danh sách ticket được user add thủ công.
- Tự mở Redmine bằng browser debug mode.
- Login bằng user/pass từ `.env`.
- Vào ticket bằng `baseUrl + ticketId`.
- Lấy HTML ticket.
- Convert HTML thành Markdown/JSON ổn định.
- Lưu snapshot khi nội dung thật sự thay đổi.
- Hiển thị raw diff giữa các snapshot.
- Phát hiện comment mới/sửa/xóa ở mức best-effort.
- Dịch nội dung tiếng Nhật sang tiếng Việt theo block/hash.
- Cho phép paste manual external context vào ticket.
- Chạy watcher trong giờ làm việc.
- Retry, rate-limit, notification batch.
- Dùng Codex CLI để chạy AI skills/sub-agents cho summary, translation, classification, action, risk, digest, Q&A.
- Tùy chọn dùng Microsoft MarkItDown cho attachment/document conversion sang Markdown, nhưng không dùng làm Redmine ticket normalizer chính.

Giá trị chính:

> Mở app lên là biết ticket nào vừa thay đổi, thay đổi cụ thể là gì, phần nào cần đọc/dịch/xử lý, context ngoài Redmine nằm đâu, và AI chỉ đóng vai trò phân tích phụ trợ có evidence.

---

## 2. Nguyên tắc thiết kế cốt lõi

## 2.1. Code/tool làm việc cơ học

Các việc cơ học, có rule rõ, cần ổn định, cần test được thì phải viết thành deterministic tools.

Agent không được tự làm những việc này nếu app đã có tool.

| Nhóm việc | Xử lý bằng deterministic tool |
|---|---|
| Mở browser Redmine | Playwright/browser automation |
| Login Redmine | Code đọc `.env`, quản lý session |
| Fetch HTML ticket | Tool có contract rõ |
| Extract vùng nội dung chính | Parser deterministic |
| Convert HTML → Markdown | Normalizer deterministic |
| Convert HTML → blocks JSON | Normalizer deterministic |
| Tính hash | Code |
| Lưu snapshot | Code |
| So sánh snapshot | Code |
| Raw diff Markdown | Code |
| Detect comment added/edited/deleted | Code best-effort |
| Scheduler | Code |
| Retry/rate-limit | Code |
| Context CRUD | Code |
| Translation cache/stale detection | Code |
| Notification batch | Code |
| SQLite/file system writes | Code |
| Git scan sau này | Code |

## 2.2. Codex CLI làm AI orchestration

Codex CLI được dùng như **AI orchestration layer**, không phải deterministic backend.

Codex CLI chạy:

- skill nhỏ: translate, classify, summarize, extract action, detect risk.
- sub-agent nhiều bước: ticket analyst, daily digest, ticket Q&A, commit explainer.

Codex CLI chỉ được:

- đọc context package đã export.
- gọi tool được whitelist.
- tạo AI artifact JSON/Markdown.
- trả suggestion có evidence.

Codex CLI không được:

- tự crawl Redmine bằng browser/shell tùy ý.
- tự sửa SQLite trực tiếp.
- tự ghi đè snapshot.
- tự xóa/sửa raw data.
- tự post comment lên Redmine.
- tự sửa source code production.
- tự chạy command ngoài whitelist.
- tự quyết định Done/Resolved thay user.

## 2.3. AI chỉ tạo derived artifacts

Dữ liệu gốc/source of truth gồm:

- raw HTML từ Redmine.
- normalized Markdown.
- normalized blocks JSON.
- snapshots.
- raw diff.
- external context user paste.
- fetch logs.
- tool run logs.

AI chỉ tạo artifact phụ trợ:

- summary.
- translation.
- classification.
- action suggestions.
- risk flags.
- open questions.
- daily digest.
- suggested reply.
- Q&A answer.

Mọi AI artifact phải có:

- `artifact_type`
- `input_refs`
- `input_hash`
- `evidence_refs`
- `model_name`
- `created_at`
- `review_status`

---

## 3. Phạm vi sản phẩm

## 3.1. In scope

App cần hỗ trợ:

1. UI local web.
2. Add ticket ID thủ công.
3. Cấu hình Redmine base URL cố định.
4. Login Redmine qua browser bằng credential trong `.env`.
5. Fetch ticket bằng browser debug mode.
6. Extract HTML vùng ticket chính.
7. Convert HTML → Markdown.
8. Convert HTML → structured blocks JSON.
9. Tính `content_hash`.
10. Chỉ tạo snapshot mới khi `content_hash` đổi.
11. Lưu fetch run log dù không có thay đổi.
12. Hiển thị raw Markdown diff.
13. Detect comment added/edited/deleted ở mức best-effort.
14. Watcher trong giờ làm việc.
15. Daily scan 08:00.
16. Chạy bù daily scan nếu app/máy ngủ qua giờ.
17. Interval scan theo profile, chỉnh được số giờ.
18. Retry fetch 3 lần.
19. Rate-limit giữa các ticket.
20. Notification gom batch.
21. Manual external context theo từng ticket.
22. Source URL optional.
23. Translation Nhật → Việt mặc định, nhưng đổi được config.
24. Tận dụng Codex CLI cho AI skills/sub-agents.
25. Tùy chọn dùng MarkItDown để convert attachment/document sang Markdown sau MVP core.
26. Lưu AI artifact vào DB/file system.
27. Dashboard Today Focus và Ticket Detail.

## 3.2. Out of scope giai đoạn đầu

Không làm trong MVP đầu:

- Redmine API.
- Tự update Redmine.
- Tự post comment Redmine.
- Auto-read Slack/Teams/Gmail.
- Một context gắn nhiều ticket.
- Context image/screenshot.
- Context sync status “đã đưa lên Redmine”.
- Phân biệt customer/team/internal/QA trong context.
- Việt → Nhật suggested reply.
- Glossary nội bộ.
- User edit translation.
- Reviewed translation workflow.
- Dịch attachment.
- Attachment parsing trong MVP core.
- Dùng MarkItDown thay thế custom Redmine normalizer.
- Git/commit center.
- Multi-user/team collaboration.
- Cloud sync.
- Mobile app.
- Agent tự thao tác cơ học ngoài tool gateway.

---

## 4. Kiến trúc tổng quan

```text
UI / Scheduler / User Action
        ↓
Backend API
        ↓
Redmine Local Tool Gateway (`rlaw`)
        ↓
Deterministic Tools
- browser fetch
- login/session
- HTML extraction
- HTML → Markdown
- HTML → blocks JSON
- hash/snapshot/diff
- comment event detection
- context CRUD
- scheduler/retry/rate-limit
- notification batch
        ↓
Local Store
- SQLite
- file system snapshots
- raw HTML
- Markdown
- blocks JSON
- AI artifacts
        ↓
Codex CLI Orchestrator
        ↓
Specialized Skills / Sub-agents
- summarize-ticket
- translate-ja-vi
- classify-change
- extract-actions
- detect-risks
- daily-digest
- ticket-qa
```

## 4.1. Thành phần chính

| Thành phần | Vai trò |
|---|---|
| `apps/web` | React/Vite UI |
| `apps/worker` | Scheduler/background worker |
| `packages/core` | Business logic deterministic |
| `packages/cli` | `rlaw` CLI/tool gateway |
| `packages/redmine-browser` | Playwright/browser automation |
| `packages/normalizer` | HTML → Markdown/blocks |
| `packages/diff-engine` | Hash/diff/comment event detection |
| `packages/db` | SQLite schema/migrations |
| `packages/notification` | Batch desktop notification |
| `agents/skills` | AI skills nhỏ |
| `agents/subagents` | AI workflow nhiều bước |
| `.codex/agents` | Codex CLI agent definitions |
| `tool-contracts` | JSON schema cho tool input/output |

---

## 5. Redmine data collection

## 5.1. Browser-first strategy

MVP không dùng Redmine API.

Primary:

```text
Browser automation in debug mode
```

Input:

```text
REDMINE_BASE_URL + ticketId
```

Ví dụ:

```text
https://redmine.example.com/issues/12345
```

## 5.2. Credential

Credential nằm trong `.env`.

Ví dụ:

```env
REDMINE_BASE_URL=https://redmine.example.com
REDMINE_USERNAME=locdt
REDMINE_PASSWORD=*****
```

MVP có thể đọc trực tiếp `.env`. Sau này có thể đổi sang encrypted local store.

## 5.3. Browser session

Mục tiêu:

- mở browser debug/headful để dễ quan sát.
- reuse session nếu đã login.
- nếu chưa login thì auto login bằng `.env`.
- không lưu password vào DB.
- không log password.

Config:

```yaml
redmine:
  baseUrl: https://redmine.example.com
  auth:
    mode: browser
    usernameEnv: REDMINE_USERNAME
    passwordEnv: REDMINE_PASSWORD
  browser:
    debugMode: true
    headless: false
    reuseSession: true
    userDataDir: ~/.redmine-ai-workspace/browser-profile
```

## 5.4. Fetch flow

```text
rlaw redmine fetch --ticket 12345
  ↓
Open/reuse browser session
  ↓
Login if needed
  ↓
Open ticket URL
  ↓
Wait for main ticket content
  ↓
Extract ticket HTML
  ↓
Save raw HTML
  ↓
Normalize HTML
  ↓
Convert to Markdown
  ↓
Convert to blocks JSON
  ↓
Compute content_hash
  ↓
Return JSON result
```

## 5.5. HTML extraction rule

Không lấy full page. Chỉ lấy vùng ticket chính.

Cần loại bỏ:

- header.
- footer.
- menu.
- sidebar không liên quan.
- CSRF token.
- session-specific URL.
- random element id.
- tracking/layout noise.
- whitespace thừa.
- timestamp chỉ gây noise nếu không phải nội dung nghiệp vụ.

---

## 6. Normalized Markdown & Blocks JSON

## 6.1. Vì sao cần cả Markdown và JSON

Markdown dùng để:

- user đọc.
- raw diff.
- export.
- đưa cho AI đọc.

Blocks JSON dùng để:

- hash ổn định.
- dịch theo block.
- detect comment added/edited/deleted.
- evidence reference.
- stale detection.
- render UI chi tiết.

## 6.2. Markdown format

```md
# Ticket #12345 - Title

## Metadata
- Project: XXX
- Status: In Progress
- Priority: Normal
- Assignee: LocDT
- Author: Tanaka
- Created: 2026-07-01 10:00
- Updated: 2026-07-02 08:00

## Description

...

## Comments

### 2026-07-02 08:15 - Tanaka

...

### 2026-07-02 09:30 - Sato

...

## Attachments

- api_spec_v2.xlsx
- screenshot.png

## Related Issues

- #12346
```

## 6.3. Blocks JSON format

```json
{
  "ticket": {
    "redmine_id": "12345",
    "title": "Title",
    "project": "XXX",
    "status": "In Progress",
    "priority": "Normal",
    "assignee": "LocDT",
    "created_at": "2026-07-01T10:00:00+07:00",
    "updated_at": "2026-07-02T08:00:00+07:00"
  },
  "blocks": [
    {
      "block_id": "metadata",
      "type": "metadata",
      "hash": "..."
    },
    {
      "block_id": "description",
      "type": "description",
      "text": "...",
      "hash": "..."
    },
    {
      "block_id": "comment:tanaka:2026-07-02T08:15:00:0",
      "type": "comment",
      "author": "Tanaka",
      "created_at": "2026-07-02T08:15:00+07:00",
      "text": "...",
      "hash": "..."
    }
  ]
}
```

## 6.4. Block ID strategy

Nếu HTML Redmine có comment/journal ID ổn định, dùng ID đó:

```text
comment:journal-98765
```

Nếu không có ID ổn định, dùng best-effort:

```text
comment:{author}:{created_at}:{index}
```

Ví dụ:

```text
comment:tanaka:2026-07-02T08:15:00:0
```

Cách này đủ cho MVP nhưng không tuyệt đối nếu Redmine reorder comment hoặc timestamp không ổn định.

---

## 7. Hash, Snapshot & Diff

## 7.1. Content hash

`content_hash` quyết định có tạo snapshot mới hay không.

Nên tính vào `content_hash`:

- title.
- status.
- priority.
- assignee.
- description.
- comments.
- attachments list.
- related issues.
- custom fields nếu parser lấy được.

Không nên tính vào `content_hash`:

- updated timestamp nếu chỉ là metadata noise.
- CSRF token.
- session URL.
- random DOM id.
- header/footer/menu.
- whitespace noise.
- layout-only HTML.

## 7.2. Metadata hash

Vẫn lưu `metadata_hash` riêng để debug.

Ví dụ:

```json
{
  "content_hash": "hash-for-real-content",
  "metadata_hash": "hash-for-debug-metadata",
  "redmine_updated_at": "2026-07-02T08:00:00+07:00"
}
```

## 7.3. Snapshot rule

Không tạo snapshot ở mọi lần fetch.

Rule mới:

```text
Fetch ticket
  ↓
Normalize
  ↓
Compute content_hash
  ↓
Compare latest snapshot content_hash
  ↓
If different:
    create new snapshot
    create diff/change records
  Else:
    do not create snapshot
    create fetch_run log only
```

## 7.4. Snapshot content

Mỗi snapshot gồm:

- raw HTML path.
- normalized Markdown path.
- blocks JSON path.
- metadata JSON path.
- content hash.
- metadata hash.
- fetch time.
- source type: browser.

## 7.5. Raw diff

MVP dùng raw Markdown diff.

Ví dụ:

```diff
- PRE2だけ修正してください。
+ PRE2と本番を修正してください。
```

## 7.6. Comment added/edited/deleted detection

App cần detect best-effort:

| Event | Điều kiện |
|---|---|
| `comment_added` | block comment mới xuất hiện |
| `comment_edited` | block ID cũ còn nhưng hash đổi |
| `comment_deleted` | block ID cũ biến mất |
| `description_edited` | description hash đổi |
| `metadata_changed` | status/assignee/priority đổi |
| `attachment_added` | attachment mới xuất hiện |
| `attachment_deleted` | attachment cũ biến mất |

Nếu không có comment ID ổn định, kết quả `comment_edited/deleted` chỉ là best-effort.

---

## 8. Fetch run log

Vì snapshot chỉ tạo khi có diff, cần bảng log riêng để biết app đã check ticket lúc nào.

```sql
CREATE TABLE fetch_runs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id TEXT NOT NULL,
  ticket_id INTEGER NOT NULL,
  started_at TEXT NOT NULL,
  finished_at TEXT,
  status TEXT NOT NULL,
  attempt_count INTEGER DEFAULT 1,
  error_code TEXT,
  error_message TEXT,
  content_changed INTEGER DEFAULT 0,
  snapshot_id INTEGER,
  content_hash TEXT,
  metadata_hash TEXT,
  created_at TEXT NOT NULL,
  FOREIGN KEY(ticket_id) REFERENCES tickets(id)
);
```

Status:

| Status | Ý nghĩa |
|---|---|
| `success_changed` | Fetch thành công và có snapshot mới |
| `success_no_change` | Fetch thành công nhưng nội dung không đổi |
| `failed_retryable` | Lỗi có thể retry |
| `failed_final` | Lỗi sau retry |
| `login_failed` | Login thất bại |
| `parse_failed` | Không extract/parse được ticket |
| `rate_limited` | Bị delay/rate-limit |

---

## 9. Watcher & Scheduler

## 9.1. Yêu cầu đã chốt

- Có UI.
- Có chạy bù.
- Interval scan chỉ trong giờ làm việc.
- User chỉnh được số giờ interval.
- Retry 3 lần.
- Có rate limit.
- Notification gom batch.

## 9.2. Watch profiles

| Profile | Ý nghĩa | Default interval |
|---|---|---|
| `active` | Ticket đang làm trực tiếp | 2h |
| `waiting` | Đang chờ khách/team | 3h |
| `managing` | Đang quản lý/theo dõi | 4h |
| `daily_only` | Chỉ check đầu ngày | 08:00 |
| `paused` | Tạm ngừng | Không check |
| `archived` | Lưu trữ | Không check |

## 9.3. Working hours

Config:

```yaml
watcher:
  timezone: Asia/Saigon
  dailyScanTime: "08:00"
  workingHours:
    enabled: true
    start: "08:00"
    end: "18:00"
```

Interval scan chỉ chạy trong giờ làm việc.

Daily scan chạy lúc `08:00`.

## 9.4. Missed run catch-up

Nếu app/máy sleep qua giờ daily scan:

```text
App start/resume
  ↓
Check hôm nay đã chạy daily scan chưa
  ↓
Nếu chưa:
    run one catch-up daily scan
  ↓
Không chạy bù nhiều interval đã miss
```

Không nên chạy bù toàn bộ các mốc interval đã miss, vì dễ tạo burst request.

## 9.5. Retry

```yaml
watcher:
  retry:
    maxAttempts: 3
    delaySeconds: 30
```

Retry áp dụng cho lỗi tạm thời:

- timeout.
- page load failed.
- selector chưa sẵn sàng.
- browser disconnected.
- network tạm lỗi.

Không retry hoặc retry hạn chế cho:

- wrong username/password.
- permission denied.
- ticket not found.
- parse rule broken.

## 9.6. Rate limit

```yaml
watcher:
  rateLimit:
    minDelayBetweenTicketsMs: 3000
    maxConcurrentFetches: 1
```

MVP nên để `maxConcurrentFetches = 1` cho an toàn với Redmine/browser.

## 9.7. Batch notification

Không notify từng ticket. Mỗi batch scan xong mới notify.

Ví dụ:

```text
Redmine Watcher
Đã check 8 ticket.
2 ticket có thay đổi.
1 ticket cần đọc.
1 ticket cần dịch.
```

---

## 10. External Context

## 10.1. Yêu cầu đã chốt

- Paste manual.
- Một context chỉ gắn một ticket.
- Không cần trạng thái “đã đưa lên Redmine”.
- Không cần phân biệt customer/team/internal/QA.
- Không cần lưu ảnh/screenshot trong context.
- Source URL optional.

## 10.2. Context fields

```text
Ticket
Type
Source type
Source person optional
Source time optional
Source URL optional
Original content
Language
Importance
Status
Created at
Updated at
```

## 10.3. Context types

| Type | Ý nghĩa |
|---|---|
| `requirement` | Yêu cầu mới |
| `clarification` | Làm rõ yêu cầu |
| `decision` | Quyết định đã chốt |
| `question` | Câu hỏi chưa trả lời |
| `todo` | Việc cần làm |
| `internal_note` | Ghi chú cá nhân |
| `risk` | Rủi ro |
| `test_note` | Ghi chú test |
| `release_note` | Ghi chú release |

## 10.4. Source types

| Source type | Ý nghĩa |
|---|---|
| `chat` | Chat copy/paste |
| `email` | Email copy/paste |
| `meeting` | Ghi chú meeting |
| `manual` | Ghi chú tự nhập |
| `redmine` | Trích lại từ Redmine nếu cần |

## 10.5. Context flow

```text
User mở ticket
  ↓
Add Context
  ↓
Paste content
  ↓
Chọn type/source hoặc để AI suggest
  ↓
Save
  ↓
Context hiển thị trong ticket detail
```

AI có thể gợi ý type/language/importance, nhưng user vẫn duyệt trước khi lưu nếu bật review mode.

---

## 11. Translation

## 11.1. Yêu cầu đã chốt

- Ngôn ngữ chính: Nhật → Việt.
- Có thể đổi bằng config.
- Không làm Việt → Nhật reply.
- Không làm glossary nội bộ.
- Không cho user sửa bản dịch trong MVP.
- Không có reviewed translation workflow.
- Không dịch attachment.

## 11.2. Config

```yaml
translation:
  enabled: true
  defaultSourceLanguage: ja
  targetLanguage: vi
  configurable: true
  translateAttachments: false
  allowUserEdit: false
  reviewedStatus: false
```

## 11.3. Translation status

MVP chỉ cần:

| Status | Ý nghĩa |
|---|---|
| `not_translated` | Chưa dịch |
| `translated` | Đã dịch |
| `stale` | Source hash đổi sau khi dịch |
| `failed` | Dịch lỗi |

Không cần:

- `needs_review`
- `reviewed`

## 11.4. Translation cache

Dịch theo block/hash:

```text
block source_hash không đổi
  → dùng lại translation

block source_hash đổi
  → mark translation stale
  → tạo pending translation
```

## 11.5. AI translator skill

AI skill chỉ nhận block cần dịch và trả JSON. Việc chọn block nào cần dịch, lưu kết quả, mark stale là deterministic tool.

---

## 12. Optional Document Conversion via MarkItDown

## 12.1. Mục tiêu

MarkItDown có thể được dùng như một **deterministic document conversion adapter** cho attachment/document pipeline.

Mục tiêu:

- Convert Redmine attachment/local document sang Markdown.
- Tạo text input dễ đọc cho Codex skills/sub-agents.
- Hỗ trợ Document/Attachment Center sau khi core watcher ổn định.
- Tránh tự viết parser cho nhiều định dạng file như PDF/DOCX/XLSX/PPTX/CSV/HTML.

Không dùng MarkItDown để thay thế custom Redmine ticket normalizer.

## 12.2. Vì sao không dùng cho Redmine ticket HTML core

Redmine ticket snapshot cần:

- stable block ID.
- tách metadata/description/comment/attachment.
- detect comment added/edited/deleted.
- kiểm soát `content_hash`.
- evidence refs theo block.
- translation cache theo block/hash.
- raw diff ít noise.

MarkItDown chỉ nên convert document sang Markdown tổng quát. Nó không đủ ngữ nghĩa Redmine-specific cho snapshot/diff engine.

Decision:

```text
Redmine ticket HTML → custom normalizer
Attachment/document → optional MarkItDown adapter
```

## 12.3. Vị trí trong architecture

Thêm package:

```text
packages/
  document-converter/
    markitdown-adapter.ts
    document-converter-service.ts
```

Thêm tool gateway commands:

```bash
rlaw document convert --ticket 12345 --file api_spec.xlsx --json
rlaw document list --ticket 12345 --json
rlaw document get-markdown DOCUMENT_ID --json
```

## 12.4. Flow

```text
Redmine attachment / local file
  ↓
rlaw document convert
  ↓
DocumentConverterService
  ↓
MarkItDownAdapter
  ↓
output.md
  ↓
document_conversions DB record
  ↓
context package for Codex if user enables
```

## 12.5. Security rule

MarkItDown chỉ được chạy qua deterministic tool.

Codex agent không được gọi `markitdown` trực tiếp.

Input file phải nằm trong workspace được phép:

```text
~/.redmine-ai-workspace/tickets/<ticketId>/attachments/
~/.redmine-ai-workspace/tickets/<ticketId>/documents/
```

Không cho convert mặc định:

```text
~/.ssh/*
/etc/*
file ngoài workspace
remote URL
network resource
```

Config mặc định:

```yaml
documents:
  enabled: false
  converter: markitdown
  mode: cli
  allowRemoteUrls: false
  enablePlugins: false
  useCloudOcr: false
  allowedInputRoots:
    - ~/.redmine-ai-workspace/tickets
  outputFormat: markdown
```

## 12.6. Integration style

Vì MarkItDown là Python package/CLI, MVP TypeScript nên tích hợp bằng CLI adapter:

```text
Node.js tool
  ↓ spawn
markitdown input-file -o output.md
  ↓
read output.md
  ↓
save metadata
```

Không reimplement converter bằng TypeScript.

## 12.7. Khi nào làm

Không đưa vào v0.1–v0.4.

Nên đưa vào sau khi core đã ổn:

```text
v0.6 — Document Conversion via MarkItDown
```

Git/commit center chuyển sang v0.7 nếu cần giữ scope sạch.


---

## 13. Tool Gateway `rlaw`

## 13.1. Mục tiêu

`rlaw` là CLI/tool gateway để UI, worker và Codex CLI gọi.

Mọi thao tác cơ học đi qua `rlaw`.

Agent không được bypass `rlaw` để tự sửa DB/file/Redmine.

## 13.2. JSON response contract

Mọi command trả JSON cùng format:

```json
{
  "ok": true,
  "tool": "redmine.fetch",
  "run_id": "run_20260703_080000",
  "data": {},
  "warnings": [],
  "errors": []
}
```

Lỗi:

```json
{
  "ok": false,
  "tool": "redmine.fetch",
  "run_id": "run_20260703_080000",
  "error_code": "LOGIN_FAILED",
  "message": "Cannot login Redmine with provided credential",
  "retryable": false,
  "warnings": [],
  "errors": [
    {
      "code": "LOGIN_FAILED",
      "message": "Cannot login Redmine with provided credential"
    }
  ]
}
```

## 13.3. Ticket commands

```bash
rlaw ticket add 12345 --json
rlaw ticket list --json
rlaw ticket get 12345 --json
rlaw ticket update 12345 --watch-profile active --json
rlaw ticket archive 12345 --json
```

## 13.4. Redmine/browser commands

```bash
rlaw redmine login --json
rlaw redmine fetch --ticket 12345 --json
rlaw redmine open --ticket 12345 --json
```

## 13.5. Snapshot/diff commands

```bash
rlaw snapshot create-if-changed --ticket 12345 --json
rlaw snapshot latest --ticket 12345 --json
rlaw snapshot list --ticket 12345 --json
rlaw diff latest --ticket 12345 --json
rlaw diff raw --from SNAPSHOT_ID --to SNAPSHOT_ID --json
```

## 13.6. Context commands

```bash
rlaw context add --ticket 12345 --type clarification --source chat --file input.txt --json
rlaw context list --ticket 12345 --json
rlaw context update CONTEXT_ID --json
rlaw context delete CONTEXT_ID --json
```

## 13.7. Translation commands

```bash
rlaw translation pending --ticket 12345 --json
rlaw translation save --ticket 12345 --block BLOCK_ID --file output.json --json
rlaw translation list --ticket 12345 --json
rlaw translation mark-stale --ticket 12345 --json
```

## 13.8. Scheduler commands

```bash
rlaw scheduler due --json
rlaw scheduler run-batch --json
rlaw scheduler run-ticket 12345 --json
rlaw scheduler mark-run --json
```

## 13.9. Context package commands

```bash
rlaw context-package build --ticket 12345 --json
rlaw context-package build-batch --date 2026-07-03 --json
```

## 13.10. AI artifact commands

```bash
rlaw ai-artifact save --ticket 12345 --type summary --file output.json --json
rlaw ai-artifact list --ticket 12345 --json
rlaw ai-artifact get ARTIFACT_ID --json
rlaw ai-artifact mark-stale --ticket 12345 --json
```

## 13.11. Document commands

```bash
rlaw document convert --ticket 12345 --file api_spec.xlsx --json
rlaw document list --ticket 12345 --json
rlaw document get-markdown DOCUMENT_ID --json
```

---

## 14. Codex CLI Integration

## 14.1. Vai trò

Codex CLI chạy các AI job theo input package do app export.

App gọi Codex qua wrapper command hoặc qua `rlaw agent run`.

Ví dụ:

```bash
rlaw agent run ticket-summarizer --ticket 12345 --json
rlaw agent run translate-pending --ticket 12345 --json
rlaw agent run daily-digest --date 2026-07-03 --json
```

Bên dưới có thể gọi:

```bash
codex run --agent ticket-summarizer --input context-package.json --output artifact.json
```

## 14.2. Input package

Agent không tự đọc lung tung filesystem. App build input package rõ ràng.

Ví dụ:

```json
{
  "package_type": "ticket_context_package",
  "ticket_id": "12345",
  "snapshot_ref": "snapshot:100",
  "markdown_path": ".../ticket.md",
  "blocks_path": ".../blocks.json",
  "latest_diff_path": ".../diff.md",
  "contexts": [
    {
      "ref": "context:10",
      "type": "clarification",
      "source_type": "chat",
      "content": "..."
    }
  ],
  "allowed_tools": [
    "rlaw ai-artifact save",
    "rlaw translation save"
  ]
}
```

## 14.3. Output artifact

Mọi agent output phải là JSON hợp lệ.

Ví dụ summary:

```json
{
  "artifact_type": "ticket_summary",
  "ticket_id": "12345",
  "summary": "Ticket yêu cầu sửa URL cancel ở PRE2.",
  "important_points": [
    {
      "text": "Scope hiện chỉ nhắc PRE2.",
      "evidence_refs": ["context:10", "snapshot:100:block:comment:tanaka"]
    }
  ],
  "open_questions": [],
  "confidence": "high"
}
```

---

## 15. Skills

Skill là tác vụ AI nhỏ, input/output rõ, không workflow dài.

## 15.1. `translate-ja-vi`

Input:

- text block.
- source language.
- target language.
- block metadata.

Output:

```json
{
  "source_language": "ja",
  "target_language": "vi",
  "translated_text": "...",
  "warnings": [],
  "evidence_refs": ["block:comment:..."]
}
```

## 15.2. `summarize-ticket`

Input:

- ticket Markdown.
- external contexts.
- latest diff.

Output:

```json
{
  "summary": "...",
  "current_state": "...",
  "important_points": [],
  "evidence_refs": []
}
```

## 15.3. `classify-change`

Input:

- raw diff.
- structural change events.
- ticket metadata.

Output:

```json
{
  "severity": "normal",
  "change_meaning": "...",
  "requires_attention": true,
  "suggested_local_status": "need_read",
  "evidence_refs": []
}
```

## 15.4. `classify-context`

Input:

- pasted context content.
- selected source if any.

Output:

```json
{
  "type": "clarification",
  "source_type": "chat",
  "language": "ja",
  "importance": "high",
  "suggested_local_status": "need_read",
  "translation": "..."
}
```

## 15.5. `extract-actions`

Input:

- ticket summary.
- latest diff.
- contexts.

Output:

```json
{
  "actions": [
    {
      "title": "...",
      "priority": "high",
      "status": "open",
      "evidence_refs": ["snapshot:100", "context:10"]
    }
  ],
  "open_questions": []
}
```

## 15.6. `detect-risks`

Input:

- ticket Markdown.
- contexts.
- latest changes.

Output:

```json
{
  "risks": [
    {
      "title": "Scope PRE2/production chưa rõ",
      "severity": "high",
      "why": "...",
      "evidence_refs": []
    }
  ]
}
```

---

## 16. Sub-agents

Sub-agent là workflow AI nhiều bước, có thể gọi một số tool được whitelist.

## 16.1. `ticket-analyst`

Mục tiêu:

- đọc ticket context package.
- tạo summary.
- highlight important points.
- gợi ý local status.
- tạo action/open questions nếu cần.
- phát hiện risk cơ bản.

Allowed tools:

```json
[
  "rlaw context-package build",
  "rlaw diff latest",
  "rlaw ai-artifact save"
]
```

Không được:

- fetch Redmine.
- sửa snapshot.
- sửa context.
- update Redmine.

## 16.2. `translation-agent`

Mục tiêu:

- lấy pending translation blocks.
- dịch theo config.
- lưu translation result.

Allowed tools:

```json
[
  "rlaw translation pending",
  "rlaw translation save"
]
```

Không được:

- sửa source block.
- sửa snapshot.
- dịch attachment.

## 16.3. `daily-digest-agent`

Mục tiêu:

- nhận batch result trong ngày.
- tạo daily digest.
- nhóm Need Action / Need Read / Need Translation / No Change.

Allowed tools:

```json
[
  "rlaw context-package build-batch",
  "rlaw ai-artifact save"
]
```

## 16.4. `ticket-qa`

Mục tiêu:

- user hỏi trong ticket.
- trả lời dựa trên local context package.
- luôn có evidence refs.

Allowed tools:

```json
[
  "rlaw context-package build",
  "rlaw ai-artifact save"
]
```

Manual only trong MVP nâng cao.

---

## 17. Guardrails cho agent

## 17.1. Whitelist theo agent

Mỗi agent có `allowed-tools.json`.

Ví dụ translator:

```json
{
  "allowed_tools": [
    "rlaw translation pending",
    "rlaw translation save"
  ],
  "forbidden": [
    "rlaw redmine fetch",
    "rlaw snapshot delete",
    "sqlite3",
    "git",
    "curl",
    "rm"
  ]
}
```

## 17.2. Không direct DB write

Agent không được gọi:

```bash
sqlite3 app.db ...
```

Mọi write vào DB phải qua `rlaw`.

## 17.3. Không raw shell nguy hiểm

Agent không được gọi command ngoài whitelist trong workflow app.

## 17.4. Artifact validation

Trước khi lưu artifact:

- validate JSON schema.
- kiểm tra `artifact_type`.
- kiểm tra `ticket_id`.
- kiểm tra `input_hash`.
- kiểm tra `evidence_refs` nếu artifact cần evidence.

Nếu không hợp lệ, lưu vào `agent_runs` trạng thái failed, không hiển thị như artifact hợp lệ.

---

## 18. UI/UX Plan

## 18.1. Layout

```text
┌───────────────────────────────────────────┐
│ Redmine Local AI Workspace                │
├───────────────┬───────────────────────────┤
│ Today Focus   │ Ticket Detail             │
│ Tickets       │ - Overview                │
│ Need Read     │ - Diff                    │
│ Need Action   │ - Translation             │
│ Need Translate│ - Context                 │
│ Settings      │ - AI Artifacts            │
└───────────────┴───────────────────────────┘
```

## 18.2. Today Focus

Hiển thị:

- số ticket changed today.
- số ticket need read.
- số ticket need action.
- số ticket need translation.
- số ticket fetch failed.
- daily scan status.
- next scheduled scan.

Table:

| Ticket | Title | Redmine Status | Local Status | Watch Profile | Last Checked | Last Changed |
|---|---|---|---|---|---|---|

## 18.3. Ticket Detail

Tabs MVP:

```text
Overview | Diff | Translation | Context | History | AI
```

### Overview

- title.
- project.
- Redmine status.
- local status.
- priority.
- assignee.
- watch profile.
- last checked.
- last changed.
- latest AI summary.

### Diff

- raw Markdown diff.
- structural events:
  - comment added.
  - comment edited.
  - comment deleted.
  - description edited.
  - status changed.
- read/action status.

### Translation

- pending blocks.
- translated blocks.
- stale translations.
- failed translations.
- rerun translation button.

### Context

- manual context list.
- add context form.
- source URL optional.
- type/source/language/importance.

### History

- snapshots.
- fetch runs.
- tool runs.
- agent runs.

### AI

- summary.
- action suggestions.
- risk flags.
- open questions.
- daily digest refs.

---

## 19. Local status

## 19.1. Status list

| Local status | Ý nghĩa |
|---|---|
| `watching` | Đang theo dõi |
| `need_read` | Có thay đổi cần đọc |
| `need_translate` | Có block chưa dịch |
| `need_action` | Cần xử lý |
| `need_clarification` | Cần hỏi/xác nhận |
| `waiting` | Đang chờ người khác |
| `in_progress` | Đang làm |
| `need_review` | Cần review lại requirement/code |
| `need_verify` | Cần test/xác minh |
| `done` | Xong phía cá nhân |
| `archived` | Lưu trữ |

Không dùng `need_attention` trong MVP để tránh trùng nghĩa.

## 19.2. Mapping

| Điều kiện | Suggested local status |
|---|---|
| Comment mới | `need_read` |
| Comment bị sửa | `need_read` hoặc `need_review` |
| Comment bị xóa | `need_review` |
| Status đổi sang Feedback | `need_action` |
| Có block Nhật chưa dịch | `need_translate` |
| Context type question | `need_clarification` |
| Context type todo | `need_action` |
| Redmine closed | `done` hoặc `archived` |

AI có thể suggest, nhưng user quyết định.

---

## 20. Data Model

## 20.1. `tickets`

```sql
CREATE TABLE tickets (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  redmine_id TEXT NOT NULL UNIQUE,
  url TEXT NOT NULL,
  title TEXT,
  project TEXT,
  redmine_status TEXT,
  local_status TEXT DEFAULT 'watching',
  role TEXT,
  priority_local TEXT,
  watch_enabled INTEGER DEFAULT 1,
  watch_profile TEXT DEFAULT 'daily_only',
  translate_enabled INTEGER DEFAULT 1,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL
);
```

## 20.2. `ticket_snapshots`

```sql
CREATE TABLE ticket_snapshots (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ticket_id INTEGER NOT NULL,
  fetched_at TEXT NOT NULL,
  source_type TEXT NOT NULL DEFAULT 'browser',
  raw_html_path TEXT,
  markdown_path TEXT,
  blocks_json_path TEXT,
  metadata_json_path TEXT,
  content_hash TEXT NOT NULL,
  metadata_hash TEXT,
  FOREIGN KEY(ticket_id) REFERENCES tickets(id)
);
```

## 20.3. `fetch_runs`

```sql
CREATE TABLE fetch_runs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id TEXT NOT NULL,
  ticket_id INTEGER NOT NULL,
  started_at TEXT NOT NULL,
  finished_at TEXT,
  status TEXT NOT NULL,
  attempt_count INTEGER DEFAULT 1,
  error_code TEXT,
  error_message TEXT,
  content_changed INTEGER DEFAULT 0,
  snapshot_id INTEGER,
  content_hash TEXT,
  metadata_hash TEXT,
  created_at TEXT NOT NULL,
  FOREIGN KEY(ticket_id) REFERENCES tickets(id),
  FOREIGN KEY(snapshot_id) REFERENCES ticket_snapshots(id)
);
```

## 20.4. `ticket_changes`

```sql
CREATE TABLE ticket_changes (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ticket_id INTEGER NOT NULL,
  from_snapshot_id INTEGER,
  to_snapshot_id INTEGER NOT NULL,
  change_type TEXT NOT NULL,
  severity TEXT DEFAULT 'normal',
  summary TEXT,
  details_json TEXT,
  read_status TEXT DEFAULT 'new',
  created_at TEXT NOT NULL,
  FOREIGN KEY(ticket_id) REFERENCES tickets(id),
  FOREIGN KEY(from_snapshot_id) REFERENCES ticket_snapshots(id),
  FOREIGN KEY(to_snapshot_id) REFERENCES ticket_snapshots(id)
);
```

## 20.5. `ticket_contexts`

```sql
CREATE TABLE ticket_contexts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ticket_id INTEGER NOT NULL,
  type TEXT NOT NULL,
  source_type TEXT NOT NULL,
  source_person TEXT,
  source_time TEXT,
  source_url TEXT,
  original_content TEXT NOT NULL,
  language TEXT,
  importance TEXT DEFAULT 'normal',
  status TEXT DEFAULT 'open',
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  FOREIGN KEY(ticket_id) REFERENCES tickets(id)
);
```

## 20.6. `translations`

```sql
CREATE TABLE translations (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ticket_id INTEGER NOT NULL,
  source_type TEXT NOT NULL,
  source_ref TEXT NOT NULL,
  source_hash TEXT NOT NULL,
  source_text TEXT NOT NULL,
  translated_text TEXT NOT NULL,
  language_from TEXT DEFAULT 'ja',
  language_to TEXT DEFAULT 'vi',
  status TEXT DEFAULT 'translated',
  model_name TEXT,
  translated_at TEXT NOT NULL,
  FOREIGN KEY(ticket_id) REFERENCES tickets(id)
);
```

## 20.7. `document_conversions`

```sql
CREATE TABLE document_conversions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ticket_id INTEGER NOT NULL,
  source_type TEXT NOT NULL DEFAULT 'attachment',
  filename TEXT NOT NULL,
  original_path TEXT NOT NULL,
  markdown_path TEXT,
  converter TEXT DEFAULT 'markitdown',
  source_hash TEXT,
  status TEXT DEFAULT 'pending',
  error_message TEXT,
  converted_at TEXT,
  created_at TEXT NOT NULL,
  FOREIGN KEY(ticket_id) REFERENCES tickets(id)
);
```

## 20.8. `ai_artifacts`

```sql
CREATE TABLE ai_artifacts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ticket_id INTEGER,
  artifact_type TEXT NOT NULL,
  input_refs TEXT,
  input_hash TEXT NOT NULL,
  evidence_refs TEXT,
  output_markdown TEXT,
  output_json TEXT,
  model_name TEXT,
  confidence TEXT,
  review_status TEXT DEFAULT 'unreviewed',
  created_at TEXT NOT NULL,
  reviewed_at TEXT,
  FOREIGN KEY(ticket_id) REFERENCES tickets(id)
);
```

## 20.9. `tool_runs`

```sql
CREATE TABLE tool_runs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id TEXT NOT NULL,
  tool_name TEXT NOT NULL,
  input_json TEXT,
  output_json TEXT,
  status TEXT NOT NULL,
  started_at TEXT NOT NULL,
  finished_at TEXT,
  error_message TEXT
);
```

## 20.10. `agent_runs`

```sql
CREATE TABLE agent_runs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  run_id TEXT NOT NULL,
  agent_name TEXT NOT NULL,
  ticket_id INTEGER,
  input_refs TEXT,
  input_hash TEXT,
  output_artifact_id INTEGER,
  status TEXT NOT NULL,
  started_at TEXT NOT NULL,
  finished_at TEXT,
  error_message TEXT,
  FOREIGN KEY(ticket_id) REFERENCES tickets(id),
  FOREIGN KEY(output_artifact_id) REFERENCES ai_artifacts(id)
);
```

## 20.11. `daily_digests`

```sql
CREATE TABLE daily_digests (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  digest_date TEXT NOT NULL,
  output_markdown TEXT NOT NULL,
  input_hash TEXT NOT NULL,
  created_at TEXT NOT NULL
);
```

---

## 21. File System Layout

```text
~/.redmine-ai-workspace/
  app.db
  config.yaml
  .env
  browser-profile/
  tickets/
    12345/
      snapshots/
        2026-07-03T080000/
          raw.html
          ticket.md
          blocks.json
          metadata.json
        2026-07-03T100000/
          raw.html
          ticket.md
          blocks.json
          metadata.json
      diffs/
        2026-07-03T100000.diff.md
      ai/
        summary.json
        actions.json
        risks.json
      documents/
        api_spec.xlsx
        api_spec.md
      notes/
        context.md
  prompts/
  exports/
    daily-digest-2026-07-03.md
```

---

## 22. Config

```yaml
app:
  timezone: Asia/Saigon
  dataDir: ~/.redmine-ai-workspace
  ui:
    port: 3417

redmine:
  baseUrl: https://redmine.example.com
  auth:
    mode: browser
    usernameEnv: REDMINE_USERNAME
    passwordEnv: REDMINE_PASSWORD
  browser:
    debugMode: true
    headless: false
    reuseSession: true
    userDataDir: ~/.redmine-ai-workspace/browser-profile
    timeoutMs: 60000

watcher:
  enabled: true
  dailyScanTime: "08:00"
  missedRun:
    runOnAppStart: true
    runOnlyDailyScan: true
  workingHours:
    enabled: true
    start: "08:00"
    end: "18:00"
  retry:
    maxAttempts: 3
    delaySeconds: 30
  rateLimit:
    minDelayBetweenTicketsMs: 3000
    maxConcurrentFetches: 1
  profiles:
    active:
      intervalHours: 2
      notify: true
    waiting:
      intervalHours: 3
      notify: true
    managing:
      intervalHours: 4
      notify: true
    dailyOnly:
      dailyOnly: true
      notify: true
    paused:
      enabled: false
    archived:
      enabled: false

snapshot:
  createOnlyWhenChanged: true
  retention:
    mode: unlimited
  hash:
    excludeUpdatedTimestamp: true

diff:
  mode: raw_markdown
  detectCommentAdded: true
  detectCommentEdited: true
  detectCommentDeleted: true
  commentDetectionMode: best_effort

context:
  manualOnly: true
  allowMultiTicket: false
  sourceUrlRequired: false
  allowImage: false

translation:
  enabled: true
  defaultSourceLanguage: ja
  targetLanguage: vi
  configurable: true
  translateAttachments: false
  allowUserEdit: false
  reviewedStatus: false

documents:
  enabled: false
  converter: markitdown
  mode: cli
  allowRemoteUrls: false
  enablePlugins: false
  useCloudOcr: false
  allowedInputRoots:
    - ~/.redmine-ai-workspace/tickets
  outputFormat: markdown

codex:
  enabled: true
  mode: cli
  command: codex
  agentsDir: .codex/agents
  skillsDir: agents/skills
  defaultModel: gpt-5.5-medium
  allowedToolGateway: rlaw

ai:
  artifactValidation: true
  requireEvidenceRefs: true
  provider:
    type: codex-cli

notifications:
  desktop: true
  mode: batch
  minSeverity: normal
```

---

## 23. Repository structure

```text
redmine-local-ai-workspace/
  apps/
    web/
      src/
      package.json
    worker/
      src/
      package.json

  packages/
    core/
    cli/
    db/
    redmine-browser/
    normalizer/
    diff-engine/
    notification/
    document-converter/
    config/

  agents/
    skills/
      translate-ja-vi/
        SKILL.md
        prompt.md
        schema.json
        examples/
      summarize-ticket/
        SKILL.md
        prompt.md
        schema.json
      classify-change/
      classify-context/
      extract-actions/
      detect-risks/

    subagents/
      ticket-analyst/
        AGENT.md
        allowed-tools.json
        output-schema.json
      translation-agent/
      daily-digest-agent/
      ticket-qa/

  .codex/
    agents/
      ticket-analyst.md
      translation-agent.md
      daily-digest-agent.md
      ticket-qa.md

  tool-contracts/
    redmine.fetch.schema.json
    snapshot.create-if-changed.schema.json
    diff.latest.schema.json
    context.add.schema.json
    ai-artifact.save.schema.json

  migrations/
  config.example.yaml
  .env.example
  README.md
```

---

## 24. MVP Roadmap

## 24.1. MVP v0.1 — Browser Snapshot & Raw Diff

Mục tiêu: chứng minh core fetch/snapshot/diff chạy với Redmine thật.

Scope:

- Local web UI basic.
- SQLite schema.
- `rlaw` CLI skeleton.
- Add ticket ID thủ công.
- Base URL cố định.
- `.env` credential.
- Browser debug/headful mode.
- Login Redmine.
- Fetch ticket HTML.
- Extract main content.
- Save raw HTML.
- Convert HTML → Markdown.
- Convert HTML → blocks JSON.
- Compute content hash bỏ updated timestamp.
- Chỉ tạo snapshot khi content hash đổi.
- Fetch run log.
- Raw Markdown diff.
- Best-effort comment added/edited/deleted.
- Ticket detail basic.

Acceptance criteria:

- Add được ticket #12345.
- Bấm fetch mở browser và vào đúng ticket.
- Login bằng `.env` nếu chưa login.
- Tạo được Markdown snapshot.
- Fetch lần sau không đổi thì không tạo snapshot mới.
- Fetch lần sau có đổi thì tạo snapshot mới.
- UI hiển thị raw diff.
- Fetch run log cho biết lần check success/no-change/fail.

## 24.2. MVP v0.2 — Scheduler & Notification

Mục tiêu: tự động theo dõi trong giờ làm việc.

Scope:

- Daily scan 08:00.
- Missed daily scan catch-up khi app start/resume.
- Interval scan theo profile.
- Chỉnh interval hours trong UI.
- Working hours 08:00–18:00.
- Retry 3 lần.
- Rate-limit giữa ticket.
- Batch notification.
- Today Focus dashboard.

Acceptance criteria:

- Active check mỗi 2h trong giờ làm việc.
- Daily Only chỉ check daily scan.
- App sleep qua 08:00 thì mở lên chạy bù một lần.
- Không tạo burst request cho các interval đã miss.
- Fetch fail retry tối đa 3 lần.
- Notification gom batch.

## 24.3. MVP v0.3 — Manual External Context

Mục tiêu: lưu context ngoài Redmine.

Scope:

- Add context trong ticket.
- Type/source/person/time/url optional.
- Source URL optional.
- Một context chỉ thuộc một ticket.
- Context tab.
- Context package builder.
- AI classify-context optional/review mode.

Acceptance criteria:

- Paste được nội dung chat/email/manual vào ticket.
- Context lưu đúng ticket.
- Source URL không bắt buộc.
- Context xuất hiện trong context package cho AI.

## 24.4. MVP v0.4 — Codex Skills Basic

Mục tiêu: thêm AI có kiểm soát qua Codex CLI.

Scope:

- `translate-ja-vi` skill.
- `summarize-ticket` skill.
- `classify-change` skill.
- `extract-actions` skill.
- `detect-risks` skill.
- `rlaw agent run`.
- `rlaw ai-artifact save`.
- JSON schema validation.
- Evidence refs.

Acceptance criteria:

- Codex CLI chạy được skill từ context package.
- Translation lưu theo block hash.
- Summary lưu như AI artifact.
- Action/risk có evidence refs.
- Agent không tự fetch/sửa DB ngoài `rlaw`.

## 24.5. MVP v0.5 — Sub-agents

Mục tiêu: workflow AI nhiều bước.

Scope:

- `ticket-analyst`.
- `translation-agent`.
- `daily-digest-agent`.
- `ticket-qa`.
- Agent run logs.
- Whitelist tools per agent.

Acceptance criteria:

- Ticket analyst tạo summary/action/risk từ context package.
- Daily digest gom batch scan result.
- Ticket Q&A trả lời có evidence refs.
- Agent run được log đầy đủ.

## 24.6. MVP v0.6 — Document Conversion via MarkItDown

Mục tiêu: convert attachment/document sang Markdown để Codex có thể đọc như context phụ trợ.

Scope:

- `packages/document-converter`.
- MarkItDown CLI adapter.
- `rlaw document convert`.
- `rlaw document list`.
- `rlaw document get-markdown`.
- `document_conversions` table.
- Chỉ cho convert file trong workspace được whitelist.
- Không cho agent gọi MarkItDown trực tiếp.
- Không bật remote URL/cloud OCR/plugin mặc định.

Acceptance criteria:

- Convert được file document phổ biến nếu MarkItDown hỗ trợ và dependency đã cài.
- Output Markdown được lưu dưới ticket documents folder.
- UI hiển thị converted document.
- Context package có thể include document Markdown khi user bật.
- File ngoài workspace bị từ chối.

## 24.7. MVP v0.7 — Git/Commit Tools

Mục tiêu: gom code context sau khi core ổn.

Scope:

- Repo registry.
- Link repo vào ticket.
- Git scan theo branch/commit message.
- Detect unpushed commits.
- Commit explainer agent.

Acceptance criteria:

- Một ticket link được nhiều repo.
- App scan được commit message chứa ticket ID.
- Commit explainer chỉ đọc git scan result, không tự chạy git ngoài tool.

---

## 25. Test Plan

## 25.1. Unit tests

- Config loader.
- `.env` loader.
- URL builder.
- HTML cleaner.
- HTML → Markdown converter.
- HTML → blocks JSON converter.
- Content hash calculation.
- Exclude updated timestamp from hash.
- Snapshot create-if-changed.
- Raw diff generation.
- Comment added detection.
- Comment edited detection.
- Comment deleted detection.
- Watch profile schedule calculation.
- Working hours filtering.
- Retry logic.
- Rate-limit logic.
- Context CRUD.
- Translation stale detection.
- Tool JSON contract validation.
- AI artifact schema validation.
- Document converter path allowlist.
- MarkItDown adapter command builder.

## 25.2. Integration tests

- Mock Redmine HTML → Markdown snapshot.
- Snapshot A/B → raw diff.
- Snapshot A/B → comment events.
- Fetch no-change → no snapshot + fetch run log.
- Fetch changed → new snapshot + change records.
- Scheduler due tickets → run batch.
- Batch result → notification summary.
- Context add → context package build.
- Translation pending → Codex skill → save translation.
- Ticket package → Codex summary → AI artifact.
- Document conversion outside workspace denied.
- Document conversion output Markdown saved correctly.

## 25.3. Manual tests

- Login Redmine bằng browser.
- Add ticket thật.
- Fetch ticket thật.
- Đổi content trên Redmine hoặc dùng ticket có update.
- Verify snapshot only when changed.
- Verify raw diff.
- Verify comment edited/deleted best-effort.
- Sleep/miss daily scan giả lập.
- Paste context tiếng Nhật.
- Run translation skill.
- Run summary/action/risk skill.

---

## 26. Security & Privacy

## 26.1. Credential

- `.env` không commit.
- `.env.example` chỉ chứa key mẫu.
- Không log password.
- Không lưu password vào DB.
- Browser session lưu trong local profile.

## 26.2. Local data

- Mặc định lưu tại `~/.redmine-ai-workspace`.
- Không cloud sync.
- Không gửi attachment/code lên AI trong MVP.
- Không post Redmine tự động.
- Không cho agent direct DB write.

## 26.3. AI/Codex

- AI chỉ đọc context package.
- AI output là derived artifact.
- AI artifact phải lưu model/input/evidence.
- Per-ticket có thể tắt AI sau này.
- Cloud model nếu dùng phải do user cấu hình chủ động.

---

## 27. Rủi ro & cách giảm

| Rủi ro | Cách giảm |
|---|---|
| Redmine HTML thay đổi layout | Tách extractor config, test bằng fixture HTML |
| Login fail | Rõ error code, không retry vô hạn |
| Browser automation không ổn định | Headful debug mode, reuse session, retry 3 lần |
| Diff bị noise | Normalize kỹ, bỏ updated timestamp khỏi content hash |
| Comment ID không ổn định | Best-effort ID, ghi rõ confidence |
| Snapshot phình to | MVP giữ vô hạn, sau thêm retention |
| MarkItDown đọc file ngoài ý muốn | Chỉ cho input trong allowed workspace roots, tắt remote URL/plugin/cloud OCR mặc định |
| Agent gọi MarkItDown trực tiếp | Chỉ expose qua `rlaw document convert`, forbidden trong allowed-tools nếu không cần |
| Agent làm việc cơ học sai | Chỉ cho gọi `rlaw`, whitelist tools |
| AI hallucination | Bắt evidence refs, artifact validation |
| Dữ liệu nội bộ lộ ra ngoài | Local-first, user cấu hình provider rõ ràng |
| Scheduler miss | Run catch-up daily scan on app start |
| Redmine bị spam request | maxConcurrentFetches=1, delay giữa ticket |

---

## 28. Acceptance Criteria tổng thể

Giai đoạn đầu đạt khi:

1. User add được ticket thủ công.
2. App mở browser và vào đúng Redmine ticket.
3. App login được bằng `.env`.
4. App lấy được HTML ticket.
5. App convert được Markdown ổn định.
6. App lưu snapshot đầu tiên.
7. App không tạo snapshot mới nếu nội dung không đổi.
8. App tạo snapshot mới nếu nội dung đổi.
9. App hiển thị raw diff.
10. App phát hiện best-effort comment mới/sửa/xóa.
11. App có fetch run log.
12. App scheduler chạy trong giờ làm việc.
13. App chạy bù daily scan khi miss.
14. App retry 3 lần khi lỗi tạm thời.
15. App notification theo batch.
16. User paste được external context.
17. Context chỉ thuộc một ticket.
18. Translation Nhật → Việt chạy qua Codex skill.
19. Summary/action/risk chạy qua Codex skill/sub-agent.
20. Agent không bypass tool gateway.
21. AI artifact có input refs/evidence refs.
22. UI giúp user biết ticket nào changed/need read/need translate/need action.

---

## 29. Backlog sau MVP

## 29.1. Attachment Center

- List attachment.
- Download local.
- Open file.
- Parse PDF/Excel sau.

## 29.2. Git/Commit Center

- Repo registry.
- Branch/commit scan.
- Unpushed commit detection.
- Commit relevance AI artifact.

## 29.3. Suggested Redmine Reply

- AI tạo draft reply.
- User copy/paste thủ công.
- Không auto-post trong MVP.

## 29.4. Requirement consistency check

AI kiểm tra lệch giữa:

- Redmine.
- external context.
- commit/code changes.

## 29.5. Agent context package for coding agents

Export package:

```text
Ticket summary
Latest diff
External context
Action checklist
Risks
Related files
Related commits
```

Dùng để giao task cho Codex/Claude/Gemini rõ hơn.

## 29.6. Retention & cleanup

- Giới hạn snapshot theo ngày/số lượng.
- Archive ticket.
- Export/delete data theo ticket.

---

## 30. References

- Microsoft MarkItDown GitHub: https://github.com/microsoft/markitdown
- PyPI MarkItDown: https://pypi.org/project/markitdown/

---

## 31. Kết luận

Kiến trúc v2.0 nên chốt như sau:

```text
Code/tool = làm thật, ổn định, test được
Codex CLI = điều phối AI
Skill = tác vụ AI nhỏ
Sub-agent = workflow AI nhiều bước
UI = nơi user đọc và quyết định
SQLite/snapshot = source of truth
AI artifact = derived artifact
```

Nguyên tắc quan trọng nhất:

> Không để agent tự làm việc cơ học nếu có thể viết tool. Agent chỉ nên đọc context chuẩn, gọi tool được whitelist, suy luận phần mềm dẻo, và trả artifact có evidence.

Thứ tự build đúng:

```text
rlaw CLI skeleton
→ SQLite schema
→ Browser fetch
→ HTML normalize
→ Markdown/blocks snapshot
→ Raw diff
→ UI basic
→ Scheduler/batch notification
→ Manual context
→ Codex skills
→ Codex sub-agents
→ MarkItDown document conversion
→ Git/commit tools
```
