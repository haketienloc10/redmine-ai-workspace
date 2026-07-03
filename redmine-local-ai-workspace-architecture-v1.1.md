# Redmine Local AI Workspace — Architecture v1.1

**Dựa trên:** Product & Technical Plan v2.1  
**Kiểu kiến trúc:** Local-first web app + deterministic tool gateway + Codex CLI AI orchestration + optional MarkItDown document conversion  
**Ngôn ngữ triển khai khuyến nghị:** TypeScript/Node.js cho MVP  
**Mục tiêu kiến trúc:** Tách rõ phần cơ học ổn định khỏi phần AI mềm dẻo. Mọi thao tác stateful/security-sensitive phải đi qua tool có contract rõ, không để agent tự thao tác trực tiếp.

---

## 1. Architectural Principles

## 1.1. Local-first

Toàn bộ dữ liệu mặc định nằm trên máy local:

```text
~/.redmine-ai-workspace/
  app.db
  config.yaml
  .env
  browser-profile/
  tickets/
  exports/
```

Không cloud sync trong MVP.

## 1.2. Tool-first, agent-second

Agent không tự làm việc cơ học.

```text
Deterministic tool = làm thật
Codex agent/skill = phân tích/suy luận/dịch/tóm tắt
```

Các thao tác sau luôn thuộc về deterministic tools:

- Login Redmine.
- Fetch HTML.
- Convert HTML → Markdown/blocks.
- Hash/diff/snapshot.
- SQLite writes.
- File writes.
- Scheduler/retry/rate-limit.
- Notification.
- Context CRUD.
- Translation cache/stale.
- Git scan sau này.

## 1.3. AI output là derived artifact

AI không là source of truth.

AI chỉ tạo:

- summary.
- translation.
- action suggestions.
- risk flags.
- classification.
- daily digest.
- Q&A answer.

Mọi artifact phải có:

```text
input_refs
input_hash
evidence_refs
model_name
review_status
```

## 1.4. Custom Redmine normalizer, optional MarkItDown for documents

Redmine ticket HTML must use a custom normalizer because the app needs stable block IDs, comment-level diff, content hash control, and evidence refs.

MarkItDown can be used later as an optional deterministic adapter for attachment/document conversion, but it must not replace the Redmine ticket normalizer.

## 1.5. Browser-first Redmine integration

MVP không phụ thuộc Redmine API.

```text
baseUrl + ticketId
  ↓
browser debug/headful mode
  ↓
login bằng .env
  ↓
extract ticket HTML
```

---

## 2. High-level Architecture

```text
┌──────────────────────────────────────────────────────────────┐
│                          User                                │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                      Local Web UI                            │
│                    React + Vite                              │
│  - Today Focus                                                │
│  - Ticket Detail                                              │
│  - Diff / Translation / Context / AI                          │
│  - Settings                                                   │
└──────────────────────────────┬───────────────────────────────┘
                               │ HTTP/IPC
                               ▼
┌──────────────────────────────────────────────────────────────┐
│                     Backend API Server                       │
│                  Node.js / Fastify                           │
│  - Ticket API                                                 │
│  - Snapshot API                                               │
│  - Context API                                                │
│  - Scheduler API                                              │
│  - Agent API                                                  │
│  - Settings API                                               │
└───────────────┬──────────────────────────────┬───────────────┘
                │                              │
                ▼                              ▼
┌──────────────────────────────┐   ┌───────────────────────────┐
│      Worker / Scheduler      │   │        rlaw CLI            │
│  - daily scan                │   │  Deterministic Tool        │
│  - interval scan             │   │  Gateway                   │
│  - retry                     │   │                           │
│  - rate-limit                │   │  rlaw redmine fetch        │
│  - batch notification        │   │  rlaw snapshot ...         │
└───────────────┬──────────────┘   │  rlaw diff ...             │
                │                  │  rlaw context ...          │
                │                  │  rlaw agent run ...        │
                │                  └────────────┬──────────────┘
                │                               │
                ▼                               ▼
┌──────────────────────────────────────────────────────────────┐
│                    Core Packages                             │
│  - redmine-browser                                            │
│  - normalizer                                                 │
│  - diff-engine                                                │
│  - db                                                         │
│  - config                                                     │
│  - notification                                               │
│  - document-converter / MarkItDown adapter                     │
│  - agent-runner                                               │
└───────────────┬──────────────────────────────┬───────────────┘
                │                              │
                ▼                              ▼
┌──────────────────────────────┐   ┌───────────────────────────┐
│       Local Store            │   │      Codex CLI             │
│  SQLite + File System        │   │  AI Orchestration          │
│                              │   │                           │
│  - tickets                   │   │  Skills:                  │
│  - snapshots                 │   │  - translate-ja-vi         │
│  - fetch_runs                │   │  - summarize-ticket        │
│  - changes                   │   │  - classify-change         │
│  - contexts                  │   │  - extract-actions         │
│  - translations              │   │  - detect-risks            │
│  - ai_artifacts              │   │                           │
│  - tool_runs                 │   │  Sub-agents:              │
│  - agent_runs                │   │  - ticket-analyst          │
└──────────────────────────────┘   │  - translation-agent       │
                                   │  - daily-digest-agent      │
                                   │  - ticket-qa               │
                                   └───────────────────────────┘
```

---

## 3. Runtime Containers

## 3.1. Web UI

**Tech:** React + Vite + TypeScript.

Vai trò:

- Hiển thị dashboard.
- Add/edit ticket.
- Manual fetch button.
- Ticket detail.
- Raw diff viewer.
- Translation viewer.
- Context form.
- AI artifact viewer.
- Settings.

Không làm:

- Direct Redmine fetch.
- Direct DB write.
- Direct Codex call.

Web UI chỉ gọi Backend API.

## 3.2. Backend API Server

**Tech khuyến nghị:** Node.js + Fastify + TypeScript.

Lý do chọn Node.js:

- Playwright tích hợp tốt.
- CLI/tooling tiện.
- React/Vite cùng TypeScript.
- Dễ spawn Codex CLI.
- Dễ build MVP local web app.

Vai trò:

- Expose REST API cho UI.
- Gọi core services.
- Gọi `rlaw` hoặc dùng chung core packages.
- Quản lý config/settings.
- Trả dữ liệu dashboard/ticket detail.

Không làm:

- Không chứa prompt logic phức tạp.
- Không để AI trực tiếp mutate source data.

## 3.3. Worker / Scheduler

Có thể là process riêng hoặc module trong backend ở MVP.

Vai trò:

- Daily scan 08:00.
- Missed daily scan catch-up.
- Interval scan theo watch profile.
- Working hours filter.
- Retry 3 lần.
- Rate limit.
- Batch notification.

Trong MVP có thể chạy cùng backend:

```text
apps/server
  api + scheduler in same process
```

Sau này tách:

```text
apps/api
apps/worker
```

## 3.4. `rlaw` CLI / Tool Gateway

Đây là lớp quan trọng nhất để kiểm soát agent.

Vai trò:

- Cung cấp command deterministic.
- Trả JSON contract ổn định.
- Log tool runs.
- Chặn thao tác nguy hiểm.
- Là entrypoint cho Codex agents gọi tool.

Ví dụ:

```bash
rlaw redmine fetch --ticket 12345 --json
rlaw snapshot create-if-changed --ticket 12345 --json
rlaw diff latest --ticket 12345 --json
rlaw context add --ticket 12345 --type clarification --source chat --file input.txt --json
rlaw agent run ticket-analyst --ticket 12345 --json
```

## 3.5. Codex CLI

Codex CLI không phải backend chính.

Vai trò:

- Chạy AI skills.
- Chạy AI sub-agents.
- Nhận context package.
- Trả artifact JSON.
- Không tự thao tác Redmine/DB/file nếu không qua `rlaw`.

---

## 4. Package Architecture

Đề xuất monorepo TypeScript:

```text
redmine-local-ai-workspace/
  apps/
    web/
    server/
    worker/

  packages/
    cli/
    core/
    config/
    db/
    redmine-browser/
    normalizer/
    diff-engine/
    notification/
    document-converter/
    agent-runner/
    shared-types/

  agents/
    skills/
    subagents/

  .codex/
    agents/

  tool-contracts/
  migrations/
  config.example.yaml
  .env.example
```

---

## 5. Package Responsibilities

## 5.1. `packages/shared-types`

Chứa type dùng chung:

```ts
Ticket
TicketSnapshot
FetchRun
TicketChange
TicketContext
Translation
AiArtifact
ToolRun
AgentRun
ToolResult<T>
```

Không chứa business logic.

## 5.2. `packages/config`

Vai trò:

- Load `config.yaml`.
- Load `.env`.
- Validate config.
- Resolve data dir.
- Resolve Redmine base URL.
- Resolve browser profile path.
- Resolve working hours.

Output:

```ts
AppConfig
RedmineConfig
WatcherConfig
TranslationConfig
CodexConfig
```

## 5.3. `packages/db`

Vai trò:

- SQLite connection.
- Migrations.
- Repositories.
- Transaction helper.
- Index/constraint definitions.

Không chứa browser/AI logic.

Repositories:

```text
TicketRepository
SnapshotRepository
FetchRunRepository
ChangeRepository
ContextRepository
TranslationRepository
AiArtifactRepository
ToolRunRepository
AgentRunRepository
```

## 5.4. `packages/redmine-browser`

Vai trò:

- Open/reuse browser.
- Login Redmine.
- Build ticket URL.
- Fetch ticket page.
- Extract main content HTML.
- Save raw HTML.

Public API:

```ts
fetchTicketHtml(ticketId: string): Promise<RedmineFetchResult>
openTicket(ticketId: string): Promise<void>
loginIfNeeded(): Promise<void>
```

Không làm:

- Không convert Markdown.
- Không lưu snapshot.
- Không gọi AI.

## 5.5. `packages/normalizer`

Vai trò:

- Clean HTML.
- Remove layout noise.
- Convert HTML → Markdown.
- Convert HTML → blocks JSON.
- Extract metadata.
- Generate stable block IDs.

Public API:

```ts
normalizeTicketHtml(input: RawTicketHtml): NormalizedTicket
htmlToMarkdown(html: string): string
htmlToBlocks(html: string): TicketBlocks
```

Output chính:

```text
ticket.md
blocks.json
metadata.json
```

## 5.6. `packages/diff-engine`

Vai trò:

- Compute content hash.
- Compute metadata hash.
- Compare latest snapshot.
- Generate raw Markdown diff.
- Detect structural events.

Public API:

```ts
computeContentHash(blocks: TicketBlocks): string
computeMetadataHash(metadata: TicketMetadata): string
createRawDiff(oldMd: string, newMd: string): string
detectChangeEvents(oldBlocks: TicketBlocks, newBlocks: TicketBlocks): ChangeEvent[]
```

Events:

```text
comment_added
comment_edited
comment_deleted
description_edited
metadata_changed
attachment_added
attachment_deleted
```

## 5.7. `packages/core`

Orchestrates deterministic workflows.

Services:

```text
TicketService
RedmineFetchService
SnapshotService
DiffService
ContextService
TranslationService
SchedulerService
NotificationService
ContextPackageService
AiArtifactService
```

Ví dụ `RedmineFetchService`:

```text
fetch HTML
→ normalize
→ hash
→ create snapshot if changed
→ create fetch_run
→ create ticket_changes
```

## 5.8. `packages/cli`

Implements `rlaw`.

Responsibilities:

- Parse commands.
- Call core services.
- Return JSON result.
- Log tool runs.
- Normalize errors.

Command groups:

```text
ticket
redmine
snapshot
diff
context
translation
scheduler
context-package
ai-artifact
agent
```

## 5.9. `packages/agent-runner`

Vai trò:

- Build context package.
- Spawn Codex CLI.
- Pass input path.
- Capture output.
- Validate JSON schema.
- Save AI artifact via core service.
- Save agent run log.

Public API:

```ts
runSkill(skillName, input): Promise<AiArtifact>
runAgent(agentName, input): Promise<AiArtifact>
```


## 5.10. `packages/document-converter`

Vai trò:

- Optional adapter cho MarkItDown.
- Convert attachment/local document sang Markdown.
- Chỉ nhận file nằm trong allowed workspace roots.
- Không cho remote URL mặc định.
- Không bật plugin/cloud OCR mặc định.
- Lưu output Markdown và metadata vào `document_conversions`.

Public API:

```ts
convertDocument(input: DocumentConvertInput): Promise<DocumentConvertResult>
listDocuments(ticketId: string): Promise<DocumentConversion[]>
getDocumentMarkdown(documentId: string): Promise<string>
```

Implementation MVP:

```text
Node.js service
  ↓ spawn CLI
markitdown input-file -o output.md
  ↓
validate output
  ↓
save metadata
```

Không làm:

- Không convert Redmine ticket HTML core.
- Không cho Codex gọi MarkItDown trực tiếp.
- Không đọc file ngoài workspace.
- Không fetch remote URL mặc định.


## 5.11. `packages/notification`

Vai trò:

- Build notification summary.
- Desktop notification.
- No per-ticket spam.

Public API:

```ts
notifyBatchScanResult(result: BatchScanResult): Promise<void>
```

---

## 6. Data Architecture

## 6.1. Source of truth

| Data | Source of truth |
|---|---|
| Ticket list | SQLite `tickets` |
| Redmine raw HTML | File system |
| Normalized Markdown | File system |
| Blocks JSON | File system |
| Snapshot metadata | SQLite `ticket_snapshots` |
| Fetch history | SQLite `fetch_runs` |
| Changes | SQLite `ticket_changes` |
| External context | SQLite `ticket_contexts` |
| Translation metadata/result | SQLite `translations` |
| AI artifact metadata | SQLite `ai_artifacts` |
| Converted documents | SQLite `document_conversions` + file system Markdown |
| Tool logs | SQLite `tool_runs` |
| Agent logs | SQLite `agent_runs` |

## 6.2. File system layout

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
        summary-2026-07-03T100000.json
        actions-2026-07-03T100000.json
        risks-2026-07-03T100000.json

      documents/
        api_spec.xlsx
        api_spec.md

      packages/
        context-package-2026-07-03T100000.json

  exports/
    daily-digest-2026-07-03.md
```

## 6.3. Database ownership rule

Only deterministic app code writes DB.

Allowed DB writers:

```text
Backend API
Worker
rlaw CLI
Core services
```

Forbidden direct DB writers:

```text
Codex agent
Skill
Sub-agent
Random shell command
```

Agents save output through:

```bash
rlaw ai-artifact save ...
rlaw translation save ...
```

---

## 7. API Architecture

## 7.1. Backend REST API

Suggested endpoints:

```http
GET    /api/tickets
POST   /api/tickets
GET    /api/tickets/:id
PATCH  /api/tickets/:id
POST   /api/tickets/:id/fetch

GET    /api/tickets/:id/snapshots
GET    /api/tickets/:id/diff/latest
GET    /api/tickets/:id/changes

GET    /api/tickets/:id/contexts
POST   /api/tickets/:id/contexts
PATCH  /api/contexts/:contextId
DELETE /api/contexts/:contextId

GET    /api/tickets/:id/translations
POST   /api/tickets/:id/translations/run

GET    /api/tickets/:id/ai-artifacts
POST   /api/tickets/:id/agents/:agentName/run

GET    /api/tickets/:id/documents
POST   /api/tickets/:id/documents/convert
GET    /api/documents/:documentId/markdown

GET    /api/scheduler/status
POST   /api/scheduler/run-batch
POST   /api/scheduler/run-daily

GET    /api/settings
PATCH  /api/settings
```

## 7.2. API should not expose secrets

Never return:

- `REDMINE_PASSWORD`.
- raw `.env`.
- full browser cookies.
- auth session values.

---

## 8. Tool Gateway Architecture

## 8.1. Tool result contract

All tools return:

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

Failure:

```json
{
  "ok": false,
  "tool": "redmine.fetch",
  "run_id": "run_20260703_080000",
  "error_code": "LOGIN_FAILED",
  "message": "Cannot login Redmine",
  "retryable": false,
  "warnings": [],
  "errors": [
    {
      "code": "LOGIN_FAILED",
      "message": "Cannot login Redmine"
    }
  ]
}
```

## 8.2. Tool categories

```text
rlaw ticket ...
rlaw redmine ...
rlaw snapshot ...
rlaw diff ...
rlaw context ...
rlaw translation ...
rlaw scheduler ...
rlaw context-package ...
rlaw ai-artifact ...
rlaw document ...
rlaw agent ...
```

## 8.3. Tool run logging

Every CLI command logs:

```text
run_id
tool_name
input_json
output_json
status
started_at
finished_at
error_message
```

This helps debug both app and agent behavior.


---

## 9. Document Conversion Architecture

## 9.1. Purpose

Document conversion is optional and is not part of the Redmine core snapshot/diff path.

It exists to support:

- Redmine attachments.
- Local files manually attached to a ticket.
- Future Document Center.
- Future document-aware Q&A.

## 9.2. MarkItDown adapter boundary

```text
Attachment/local file
  ↓
rlaw document convert
  ↓
DocumentConverterService
  ↓
MarkItDown CLI adapter
  ↓
converted Markdown
  ↓
document_conversions DB record
  ↓
context package when enabled
```

## 9.3. Security boundary

Allowed:

```text
~/.redmine-ai-workspace/tickets/<ticketId>/attachments/
~/.redmine-ai-workspace/tickets/<ticketId>/documents/
```

Denied by default:

```text
remote URL
file outside workspace
~/.ssh/*
/etc/*
plugins
cloud OCR
```

## 9.4. Agent rule

Codex agents do not call `markitdown`.

Agents may only consume converted Markdown already produced by:

```bash
rlaw document convert
```

## 9.5. Why this is not the Redmine normalizer

Redmine core needs structured blocks and stable evidence references.

MarkItDown output is useful for LLM ingestion, but it is not designed to preserve Redmine-specific entities such as journals/comments with stable IDs.


---

## 10. Codex Agent Architecture

## 10.1. Agent execution model

```text
Backend/UI/Worker
  ↓
rlaw agent run <agentName> --ticket <id>
  ↓
agent-runner builds context package
  ↓
agent-runner spawns Codex CLI
  ↓
Codex reads package
  ↓
Codex runs skill/sub-agent
  ↓
Codex writes artifact JSON
  ↓
agent-runner validates schema
  ↓
agent-runner saves artifact through core service
  ↓
UI renders artifact
```

## 10.2. Context package

A context package is the only input the agent should read.

Example:

```json
{
  "package_type": "ticket_context_package",
  "ticket_id": "12345",
  "ticket_ref": "ticket:12345",
  "snapshot_ref": "snapshot:100",
  "markdown_path": ".../ticket.md",
  "blocks_path": ".../blocks.json",
  "latest_diff_path": ".../diff.md",
  "change_refs": ["change:201", "change:202"],
  "contexts": [
    {
      "ref": "context:10",
      "type": "clarification",
      "source_type": "chat",
      "source_url": null,
      "content": "PRE2だけ修正してください。本番はまだ不要です。"
    }
  ],
  "allowed_tools": [
    "rlaw ai-artifact save",
    "rlaw translation save"
  ]
}
```

## 10.3. Skill vs Sub-agent

| Type | Use case | Tool access |
|---|---|---|
| Skill | Một tác vụ hẹp | Thường không cần tool hoặc rất ít tool |
| Sub-agent | Workflow nhiều bước | Có whitelist tool |

Examples:

| Skill | Purpose |
|---|---|
| `translate-ja-vi` | Dịch block Nhật → Việt |
| `summarize-ticket` | Tóm tắt ticket |
| `classify-change` | Diễn giải mức độ thay đổi |
| `classify-context` | Gợi ý type/source/language |
| `extract-actions` | Rút checklist |
| `detect-risks` | Phát hiện rủi ro |

| Sub-agent | Purpose |
|---|---|
| `ticket-analyst` | Summary + action + risk |
| `translation-agent` | Dịch pending blocks |
| `daily-digest-agent` | Tổng hợp batch/day |
| `ticket-qa` | Q&A theo ticket |

## 10.4. Agent guardrails

Each sub-agent has:

```text
AGENT.md
allowed-tools.json
output-schema.json
```

Example:

```json
{
  "allowed_tools": [
    "rlaw context-package build",
    "rlaw diff latest",
    "rlaw ai-artifact save"
  ],
  "forbidden": [
    "sqlite3",
    "rm",
    "curl",
    "git",
    "rlaw redmine fetch",
    "rlaw snapshot delete"
  ]
}
```

## 10.5. Artifact validation

Before saving AI output:

1. Parse JSON.
2. Validate schema.
3. Check artifact type.
4. Check ticket ID.
5. Check input hash.
6. Check evidence refs if required.
7. Save artifact.
8. Log agent run.

Invalid output is saved only as failed agent run, not as valid artifact.

---

## 11. Core Workflows

## 11.1. Manual ticket fetch

```text
User clicks Fetch
  ↓
Backend API: POST /api/tickets/:id/fetch
  ↓
Core RedmineFetchService
  ↓
redmine-browser fetch HTML
  ↓
normalizer create Markdown/blocks
  ↓
diff-engine compute content_hash
  ↓
SnapshotService compare latest snapshot
  ↓
If changed:
    save raw.html / ticket.md / blocks.json / metadata.json
    insert ticket_snapshots
    generate raw diff
    detect change events
    insert ticket_changes
    insert fetch_runs success_changed
  Else:
    insert fetch_runs success_no_change
  ↓
Return result to UI
```

## 11.2. Scheduled batch scan

```text
Scheduler tick
  ↓
Check working hours
  ↓
Find due tickets
  ↓
Run tickets one by one
  ↓
Apply rate-limit between tickets
  ↓
Retry failed fetch up to 3 times
  ↓
Aggregate results
  ↓
Save fetch_runs
  ↓
Send one batch notification
  ↓
Update Today Focus dashboard
```

## 11.3. Missed daily scan catch-up

```text
App start/resume
  ↓
Check today's daily scan marker
  ↓
If daily scan not run:
    run one catch-up daily scan
  ↓
Do not replay all missed intervals
```

## 11.4. Add external context

```text
User opens Context tab
  ↓
Paste context
  ↓
Select type/source or ask AI classify-context
  ↓
Save context via Backend API
  ↓
Insert ticket_contexts
  ↓
Context appears in ticket detail
  ↓
Context becomes part of future context package
```

## 11.5. Translation flow

```text
Snapshot created
  ↓
TranslationService checks blocks
  ↓
Find Japanese blocks without translation or stale
  ↓
rlaw agent run translation-agent
  ↓
Agent gets pending blocks through rlaw translation pending
  ↓
Codex skill translate-ja-vi translates
  ↓
Agent saves via rlaw translation save
  ↓
Translation appears in UI
```

## 11.6. Ticket analysis flow

```text
User clicks Analyze or watcher triggers AI
  ↓
ContextPackageService builds package
  ↓
AgentRunner runs ticket-analyst
  ↓
Codex reads ticket.md + diff + contexts
  ↓
Codex outputs summary/actions/risks JSON
  ↓
AgentRunner validates output
  ↓
AiArtifactService saves artifact
  ↓
UI renders AI tab
```

---

## 12. Scheduler Architecture

## 12.1. Scheduling model

Scheduler uses DB/config to calculate due tickets.

Inputs:

```text
watch_enabled
watch_profile
last_checked_at from fetch_runs
dailyScanTime
workingHours
profile interval
paused/archived state
```

## 12.2. Due calculation

```text
if profile is paused/archived:
  not due

if outside working hours:
  not due for interval scan

if daily_only:
  due only for daily scan or catch-up

if active/waiting/managing:
  due if now - last_successful_fetch >= intervalHours
```

## 12.3. Batch run model

MVP:

```text
maxConcurrentFetches = 1
minDelayBetweenTicketsMs = 3000
retry maxAttempts = 3
```

This avoids stressing Redmine and reduces browser instability.

---

## 13. UI Architecture

## 13.1. Pages

```text
/pages
  TodayFocusPage
  TicketsPage
  TicketDetailPage
  SettingsPage
```

## 13.2. Ticket Detail tabs

```text
Overview
Diff
Translation
Context
History
AI
```

## 13.3. UI data flow

```text
React Query / TanStack Query
  ↓
Backend REST API
  ↓
Core services
  ↓
SQLite/file system
```

Use polling lightly or explicit refresh after actions.

Do not let frontend read files directly from `~/.redmine-ai-workspace`.

---

## 14. Security Architecture

## 14.1. Credential handling

- `.env` local only.
- `.env` must be gitignored.
- `.env.example` only has placeholder.
- Password never returned to UI.
- Password never logged.
- Browser cookies stay in `browser-profile`.

## 14.2. Agent restrictions

Agent cannot directly:

- write DB.
- delete snapshots.
- fetch Redmine.
- run raw git.
- run raw curl.
- post Redmine.
- modify code.

Agent must use `rlaw`.

## 14.3. Output validation

No invalid AI JSON enters `ai_artifacts`.

Invalid outputs go to `agent_runs.status = failed`.

---

## 15. Error Handling Architecture

## 15.1. Error code categories

```text
CONFIG_ERROR
ENV_MISSING
BROWSER_LAUNCH_FAILED
LOGIN_FAILED
TICKET_NOT_FOUND
PERMISSION_DENIED
PAGE_LOAD_TIMEOUT
EXTRACT_FAILED
NORMALIZE_FAILED
SNAPSHOT_WRITE_FAILED
DB_ERROR
CODEX_FAILED
ARTIFACT_VALIDATION_FAILED
```

## 15.2. Retryable errors

Retry:

- `PAGE_LOAD_TIMEOUT`
- `BROWSER_DISCONNECTED`
- temporary network error
- selector wait timeout

Do not retry:

- `LOGIN_FAILED`
- `PERMISSION_DENIED`
- `TICKET_NOT_FOUND`
- `CONFIG_ERROR`

---

## 16. Observability Architecture

## 16.1. Logs

Need structured logs:

```text
logs/app.log
logs/worker.log
logs/rlaw.log
logs/agent.log
```

## 16.2. DB logs

- `fetch_runs`
- `tool_runs`
- `agent_runs`

## 16.3. UI diagnostics

Settings/Diagnostics page should show:

- config path.
- data dir.
- browser profile path.
- Redmine login status.
- last fetch status.
- last scheduler run.
- failed fetches.
- failed agent runs.

---

## 17. Recommended Tech Stack

## 17.1. MVP stack

| Layer | Tech |
|---|---|
| Language | TypeScript |
| Frontend | React + Vite |
| UI kit | Ant Design or shadcn/ui |
| Backend | Fastify |
| DB | SQLite |
| DB access | Drizzle ORM or better-sqlite3 |
| Browser automation | Playwright |
| Diff | jsdiff |
| Scheduler | node-cron or custom interval loop |
| CLI | Commander.js |
| Config | yaml + dotenv + zod |
| Validation | zod / ajv |
| AI orchestration | Codex CLI |
| Notifications | node-notifier or OS-specific wrapper |
| Document conversion | Optional MarkItDown CLI adapter via Python environment |

## 17.2. Why TypeScript-first

- Same language for UI/backend/CLI.
- Good Playwright support.
- Easy Codex CLI process integration.
- Easier monorepo.
- Less moving parts for MVP.

---

## 18. MVP Architecture Roadmap

## v0.1 — Core local tool + manual fetch

```text
apps/web
apps/server
packages/cli
packages/db
packages/redmine-browser
packages/normalizer
packages/diff-engine
```

Deliver:

- Add ticket.
- Manual fetch.
- Browser login/fetch.
- Markdown/blocks.
- Snapshot if changed.
- Raw diff.
- Fetch runs.

## v0.2 — Worker scheduler

Add:

```text
apps/worker or server scheduler module
packages/notification
```

Deliver:

- Daily scan.
- Working hours.
- Catch-up.
- Retry.
- Rate-limit.
- Batch notification.

## v0.3 — Context

Add:

```text
ContextService
ContextPackageService
Context tab UI
```

Deliver:

- Manual context CRUD.
- Context package builder.

## v0.4 — Codex skills

Add:

```text
packages/agent-runner
agents/skills/*
.codex/agents/*
tool-contracts/*
```

Deliver:

- Translation.
- Summary.
- Change classify.
- Actions.
- Risks.

## v0.5 — Sub-agents

Deliver:

- ticket-analyst.
- translation-agent.
- daily-digest-agent.
- ticket-qa.
- whitelist tools.
- agent run logs.

## v0.6 — Document conversion via MarkItDown

Deliver:

- `packages/document-converter`.
- MarkItDown CLI adapter.
- `rlaw document convert`.
- `document_conversions` table.
- Workspace-only input validation.
- Converted Markdown included in context package only when enabled.

## v0.7 — Git/commit tools

Deliver later:

- repo registry.
- git scan.
- commit explainer.

---

## 19. Architecture Decision Records

## ADR-001 — Browser automation primary, Redmine API not required

Decision:

```text
Use browser automation in debug/headful mode as primary Redmine integration.
```

Reason:

- User will provide base URL and credential.
- Browser can work even if API unavailable.
- Redmine HTML is what user sees.

Tradeoff:

- More fragile than API.
- Needs robust normalizer and fixture tests.

## ADR-002 — Snapshot only when content hash changes

Decision:

```text
Only create snapshot when content_hash changes.
```

Reason:

- Avoid noisy snapshot history.
- Match user expectation: snapshot means real diff.

Tradeoff:

- Need fetch_runs to know no-change checks happened.

## ADR-003 — Exclude updated timestamp from content hash

Decision:

```text
Exclude Redmine updated timestamp from content_hash.
```

Reason:

- Prevent false positive changes.
- Keep dashboard signal clean.

Tradeoff:

- Store metadata_hash separately for diagnostics.

## ADR-004 — Codex CLI as orchestration, not core engine

Decision:

```text
Use Codex CLI for AI skills/sub-agents only.
```

Reason:

- User already has Codex CLI.
- AI should handle soft reasoning, not mechanical operations.

Tradeoff:

- Need tool gateway and schema validation.

## ADR-005 — MarkItDown optional for document conversion only

Decision:

```text
Use MarkItDown only as an optional document/attachment converter.
```

Reason:

- It can reduce effort converting common document formats to Markdown.
- It fits the future Document Center and document-aware AI context.

Tradeoff:

- Requires Python/CLI dependency in a TypeScript-first app.
- Must be sandboxed by path allowlist and disabled for remote URLs by default.
- Must not replace custom Redmine ticket normalization.

## ADR-006 — `rlaw` as deterministic tool gateway

Decision:

```text
All stateful/mechanical operations are exposed through rlaw.
```

Reason:

- Prevent agent from doing unsafe direct operations.
- Make operations testable and auditable.

Tradeoff:

- More upfront tool design work.

---

## 20. References

- Microsoft MarkItDown GitHub: https://github.com/microsoft/markitdown
- PyPI MarkItDown: https://pypi.org/project/markitdown/

---

## 21. Final Architecture Summary

```text
React UI
  ↓
Fastify API
  ↓
Core services
  ↓
rlaw deterministic tool gateway
  ↓
SQLite + file system + browser automation
  ↓
Optional MarkItDown document conversion
  ↓
Context package
  ↓
Codex CLI skills/sub-agents
  ↓
Validated AI artifacts
  ↓
UI display
```

The system should be built around one strict boundary:

```text
Anything mechanical or stateful must be a deterministic tool.
Anything interpretive or linguistic can be AI.
```

This keeps the app safe, debuggable, and practical for daily Redmine work.
