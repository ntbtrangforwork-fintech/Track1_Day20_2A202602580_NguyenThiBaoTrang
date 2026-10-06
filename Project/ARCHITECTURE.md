# ViVi Architecture

Tài liệu này tách rõ hai trạng thái:

- **Current / as-built:** frontend prototype đang chạy trong repo.
- **Target / tuần backend:** kiến trúc cần triển khai để nối hai luồng Buyer và
  Sales bằng dữ liệu bền vững, quyền truy cập và AI có kiểm soát.

## 1. Product boundaries

ViVi là một sản phẩm với hai experience chạy song song:

```mermaid
flowchart TB
    subgraph Buyer[Buyer experience]
        B1[Catalog / vehicle detail]
        B2[ViVi conversation]
        B3[Needs review]
        B4[Recommendation / compare / TCO]
        B5[Quote status / test drive]
        B1 --> B2 --> B3 --> B4 --> B5
    end

    subgraph Core[Shared journey state]
        C1[Conversation]
        C2[Confirmed needs]
        C3[Consent record]
        C4[Lead + assignment]
        C5[Recommendation + review]
        C6[Test drive]
        C1 --> C2 --> C3 --> C4 --> C5 --> C6
    end

    subgraph Sales[Sales experience]
        S1[Lead queue / Kanban]
        S2[Lead detail]
        S3[Conversation summary]
        S4[Copilot suggestions]
        S5[Human decision / follow-up]
        S1 --> S2 --> S3 --> S4 --> S5
    end

    B2 --> C1
    B3 --> C2
    B3 --> C3
    C4 --> S1
    S5 --> C5
    C5 --> B5
    C6 --> B5
    C6 --> S5
```

Điểm handoff chỉ xảy ra sau khi Buyer xem lại nhu cầu và đồng ý chia sẻ. Sales
chỉ thấy hồ sơ được phân công hoặc thuộc phạm vi được cấp quyền.

## 2. Current architecture — frontend prototype

### 2.1 Runtime component diagram

```mermaid
flowchart TB
    subgraph Actors[Actors]
        Buyer[Buyer]
        Advisor[Sales / Advisor]
    end

    subgraph Web[Next.js 16 + React 19]
        Shell[DemoShell<br/>role + view + selected profile]

        subgraph Views[UI components]
            Consult[Consultation + ViVi chat]
            Compare[Comparison + TCO]
            Ops[Metrics + lead list + Kanban]
            Insights[Buyer signal + anonymous trend]
            Calendar[Test-drive calendar]
        end

        subgraph Services[Async service contracts]
            Demo[demo.service.ts]
            Lead[lead-pipeline.service.ts]
            Cal[calendar.service.ts]
        end

        subgraph Domain[Deterministic domain logic]
            Recommend[Recommendation rules]
            TCO[TCO calculator]
            Pipeline[Lead stage / feedback rules]
            Conflict[Booking validation / conflict checks]
        end

        subgraph State[Local data]
            Seeds[Catalog + scenarios + analytics + calendar seeds]
            Memory[Module-level memory]
            Local[localStorage<br/>lead stage + feedback]
        end
    end

    Buyer --> Shell
    Advisor --> Shell
    Shell --> Views
    Views --> Services
    Services --> Domain
    Services --> Seeds
    Services --> Memory
    Lead --> Local
```

### 2.2 Current responsibilities

| Component | Responsibility |
|---|---|
| `DemoShell` | Bootstrap data, đổi role/view, giữ profile đang dùng |
| Buyer components | Catalog, chat, discovery, recommendation, compare, TCO, booking |
| Advisor components | Metrics, anonymous trend, lead/profile, buyer signal, review, Kanban |
| Calendar components | Availability, create/update/confirm/cancel appointment |
| `demo.service.ts` | Profile store, rule-based chat, recommendation, review, basic booking |
| `lead-pipeline.service.ts` | Role checks, stage persistence, customer feedback |
| `calendar.service.ts` | Calendar store, conflicts, role-filtered results, profile sync |
| `src/data/*` | Seed catalog, questions, profiles, analytics and appointments |
| `src/lib/*` | TCO, lead stages, formatting and result helpers |

### 2.3 Current persistence and trust boundary

| Data | Storage | Lifetime / limitation |
|---|---|---|
| Profiles, messages, needs, recommendations | Module memory | Reset theo frontend runtime |
| Appointments | Module memory | Không đồng bộ giữa browser/user |
| Lead stage, customer feedback | Browser `localStorage` | Chỉ cùng browser |
| Catalog, AI signals, anonymous trend | Source/seed files | Static, không phải AI output thật |
| Auth/role | UI role switch | Không phải security boundary |

Root `src/main.py` đã khai báo FastAPI nhưng import `src.api.routes`, file này
chưa tồn tại. Vì vậy backend hiện chưa nằm trong runtime và Docker health check
cũng chưa thể pass.

## 3. Target architecture — backend and AI

```mermaid
flowchart LR
    subgraph Clients[Clients]
        BuyerUI[Buyer UI]
        SalesUI[Sales workspace]
    end

    subgraph API[FastAPI application]
        Auth[Auth + RBAC]
        Conversation[Conversation API]
        Profiles[Profile / needs / consent]
        Leads[Lead / assignment / review]
        Booking[Booking / availability]
        Copilot[AI orchestration API]
        Knowledge[Sales knowledge API]
        Audit[Audit + feedback]
    end

    subgraph Domain[Domain services]
        Needs[Needs state machine]
        Recommendation[Recommendation ranker]
        TCO[TCO engine]
        Handoff[Consented handoff]
        Scheduling[Conflict-safe scheduling]
        Prompt[Prompt / policy / output validation]
    end

    subgraph AI[AI capabilities]
        Extract[Intent + structured extraction]
        RAG[Grounded Q&A / retrieval]
        Explain[Trade-off explanation]
        SalesAI[Summary / missing info / next question / draft]
        SalesKB[Sales knowledge assistant]
    end

    subgraph Data[Data layer]
        PG[(PostgreSQL)]
        Vector[(Vector index)]
        Sources[(Versioned catalog<br/>price + public policy sources)]
        InternalKB[(Sales playbook<br/>training + internal policy)]
        Queue[(Optional job queue)]
    end

    subgraph External[External services]
        LLM[LLM provider]
        Notify[Email / SMS / CRM<br/>future]
    end

    BuyerUI --> API
    SalesUI --> API
    API --> Domain
    Conversation --> Copilot
    Profiles --> Handoff
    Leads --> Handoff
    Booking --> Scheduling
    Copilot --> AI
    Knowledge --> SalesKB
    AI --> Prompt
    AI --> LLM
    SalesKB --> Prompt
    SalesKB --> LLM
    RAG --> Vector
    SalesKB --> Vector
    Vector --> Sources
    Vector --> InternalKB
    Domain --> PG
    Audit --> PG
    Booking -. async .-> Queue
    Queue -. approved action only .-> Notify
```

### Architectural rules

1. Frontend chỉ gọi typed API; không gọi model provider trực tiếp.
2. Recommendation constraints và TCO là deterministic services; LLM chỉ diễn
   giải, không tự tạo giá hay thông số.
3. Mọi factual answer về xe/chính sách phải grounded trên nguồn có version và
   ngày hiệu lực.
4. Handoff sang Sales cần consent record và audit event.
5. AI-generated summary/inference/draft không được ghi đè customer-confirmed
   facts.
6. Follow-up chỉ là draft cho tới khi Sales sửa và bấm duyệt/gửi.
7. RBAC, tenant/showroom scope và assignment được kiểm tra ở server, không dựa
   vào role switch phía client.
8. Sales knowledge retrieval phải lọc tài liệu theo quyền của người hỏi trước
   khi đưa context cho model; citation không được làm lộ tài liệu ngoài scope.

## 4. End-to-end data flow

```mermaid
sequenceDiagram
    actor B as Buyer
    participant W as Next.js
    participant API as FastAPI
    participant AI as AI orchestrator
    participant DB as PostgreSQL
    actor S as Sales

    B->>W: Gửi câu hỏi / cập nhật nhu cầu
    W->>API: POST message + conversation version
    API->>AI: classify + extract structured needs
    AI-->>API: answer draft + facts/inferences + citations
    API->>DB: Save message, needs revision, audit event
    API-->>W: Answer + needs needing confirmation
    B->>W: Xác nhận needs + consent handoff
    W->>API: PATCH needs + POST consent
    API->>DB: Create/merge lead and assignment
    API-->>S: Lead appears in authorized queue
    S->>API: Request summary / missing info / next question
    API->>AI: Use consented conversation + structured facts
    AI-->>S: Labeled draft/inference
    S->>API: Edit + approve recommendation/follow-up
    API->>DB: Save human decision and audit event
    API-->>W: Approved/revision status
    B->>W: Chọn lịch lái thử
    W->>API: Create appointment
    API->>DB: Atomic conflict check + save
    API-->>W: Pending/confirmed appointment
```

### Concurrency/versioning

- `Conversation`, `NeedsState`, `Recommendation` và `Lead` nên có `version`.
- Client gửi `expected_version`; conflict trả `409` thay vì ghi đè thay đổi mới.
- Booking dùng transaction/unique constraint theo vehicle/showroom/time range.
- Các action dễ retry dùng idempotency key: handoff, create booking, approval.

## 5. Data model

Các entity P0:

| Entity | Fields quan trọng |
|---|---|
| `User` | id, role, showroom/tenant scope, status |
| `CustomerProfile` | contact, vehicle interest, source, consent scope |
| `Conversation` / `Message` | actor, text, timestamp, model/prompt version, citations |
| `NeedFact` | key, value, provenance, confidence, status, revision |
| `ConsentRecord` | purpose, data scope, granted/revoked time, policy version |
| `Lead` | profile, assignee, stage, source, latest activity, version |
| `Recommendation` | ranked vehicles, constraints, reasons, assumptions, status |
| `TcoSnapshot` | inputs, formula version, values, currency, created_at |
| `SalesNote` | author, body, visibility, created_at |
| `AiArtifact` | type, input refs, output, confidence, review status, reviewer |
| `TestDrive` | customer, vehicle, showroom, salesperson, time, status |
| `AuditEvent` | actor, action, object, before/after refs, timestamp |

`NeedFact.provenance` tối thiểu có bốn loại:

```text
customer_confirmed   Khách đã xác nhận; nguồn sự thật ưu tiên
sales_observed       Ghi chú/quan sát của tư vấn viên
ai_inferred          Suy luận của model; không được hiển thị như fact
anonymous_aggregate  Chỉ số nhóm; không gắn ngược về cá nhân
```

## 6. Proposed API surface

Đây là contract đề xuất để thay mock services mà không phải viết lại UI:

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/health` | Liveness/readiness cơ bản |
| `POST` | `/api/v1/conversations` | Mở hoặc tiếp tục session |
| `POST` | `/api/v1/conversations/{id}/messages` | Chat + structured extraction |
| `GET/PATCH` | `/api/v1/profiles/{id}/needs` | Đọc/sửa/xác nhận needs state |
| `POST` | `/api/v1/profiles/{id}/consents` | Ghi nhận phạm vi handoff |
| `POST` | `/api/v1/leads` | Create/merge lead idempotently |
| `GET` | `/api/v1/leads` | Queue có filter và RBAC |
| `GET` | `/api/v1/leads/{id}` | Shared customer context |
| `POST` | `/api/v1/recommendations` | Rank + reasons + TCO snapshot |
| `POST` | `/api/v1/recommendations/{id}/review` | Approve/revision by Sales |
| `POST` | `/api/v1/copilot/lead-summary` | Labeled summary draft |
| `POST` | `/api/v1/copilot/missing-information` | Gap detection |
| `POST` | `/api/v1/copilot/next-question` | Suggested question + rationale |
| `POST` | `/api/v1/copilot/follow-up-draft` | Draft only; never auto-send |
| `POST` | `/api/v1/sales-knowledge/chat` | RAG chat trên knowledge base được phân quyền |
| `GET` | `/api/v1/sales-knowledge/sources/{id}` | Mở source/citation nếu user có quyền |
| `GET` | `/api/v1/test-drives/availability` | Available slots |
| `POST/PATCH` | `/api/v1/test-drives` | Create/update appointment |
| `POST` | `/api/v1/test-drives/{id}/confirm` | Advisor confirmation |
| `POST` | `/api/v1/ai-artifacts/{id}/review` | Accept/edit/reject AI output |

Response errors nên thống nhất:

```json
{
  "error": {
    "code": "VERSION_CONFLICT",
    "message": "Dữ liệu đã thay đổi, vui lòng tải lại.",
    "request_id": "req_...",
    "details": {}
  }
}
```

## 7. AI orchestration and human review

```mermaid
flowchart LR
    Input[Message / lead context] --> Scope{In scope?}
    Scope -->|No| Redirect[Safe redirect]
    Scope -->|Yes| Extract[Structured extraction]
    Extract --> Retrieve[Retrieve official facts]
    Retrieve --> Generate[Generate answer / draft]
    Generate --> Validate[Schema + citation + policy validation]
    Validate --> Label[Label fact vs inference]
    Label --> Review{Needs human review?}
    Review -->|Buyer answer| Output[Return with sources/assumptions]
    Review -->|Sales action| Draft[Draft only]
    Draft --> Human[Sales edit / approve / reject]
    Human --> Audit[Persist decision + audit]
```

Không dùng một prompt lớn cho toàn bộ hành trình. Tách capability và schema:

- `intent_and_scope`
- `extract_needs`
- `grounded_vehicle_qa`
- `explain_recommendation`
- `summarize_lead`
- `detect_missing_information`
- `suggest_next_question`
- `draft_follow_up`
- `sales_knowledge_chat`

Mỗi artifact lưu `model`, `prompt_version`, `input_refs`, `output_schema`,
`latency`, `token_usage`, `citations` và trạng thái review.

## 8. Implementation order for this week

### P0 — phải có để hai luồng thật sự nối nhau

1. Làm backend boot được: `src/api/routes.py`, config validation, health/readiness,
   error envelope và test client.
2. PostgreSQL + migration cho profile, conversation, needs, consent, lead,
   recommendation/review, appointment và audit.
3. API conversation/needs + handoff có consent; nối `demo.service.ts` sang adapter
   HTTP nhưng giữ mock adapter cho story/demo.
4. Recommendation/TCO deterministic ở server, có formula/source version.
5. Lead queue/detail, assignment và review gate với RBAC.
6. Availability/booking dùng transaction và conflict handling.

### P1 — AI value trên cùng luồng

1. Intent/scope + structured extraction; Buyer xác nhận trước khi lưu fact.
2. RAG Q&A từ catalog/chính sách đã kiểm duyệt.
3. Sales copilot: summary, missing info, next question, follow-up draft.
4. Human review UI + audit cho mọi AI artifact dẫn tới hành động.
5. Eval dataset và regression tests cho groundedness, extraction và refusal.

### P2 — làm sau cùng, chỉ bắt đầu khi toàn bộ P0/P1 đã hoàn tất

- CRM/notification integration.
- Async jobs, retries và dead-letter handling.
- Advanced buyer signals, model monitoring và analytics mở rộng.
- Multi-showroom/tenant hardening.
- **Hai feature cuối — Sales Knowledge Assistant:** RAG trên playbook, tài liệu
  đào tạo và chính sách nội bộ; citation, access filter và xử lý `không tìm thấy`.
- **Hai feature cuối — Anonymous interest aggregation:** tổng hợp mối quan tâm theo
  nhóm, không định danh cá nhân; có privacy threshold và không truy ngược về
  từng session.

Hai mục in đậm ở trên là hai feature cuối roadmap. Seed anonymous trend hiện có
trong frontend chỉ dùng để thể hiện ý tưởng UI, không phải dấu hiệu rằng pipeline
analytics này đã được ưu tiên triển khai.

## 9. Test strategy and release gates

| Layer | Minimum gate |
|---|---|
| Domain | Unit tests cho TCO, recommendation constraints, stage transitions |
| API | Auth/RBAC, consent, idempotency, version conflict, validation |
| Database | Migration test, rollback, booking concurrency |
| Contract | Frontend adapter vs OpenAPI schema |
| AI extraction | Exact/partial field accuracy và provenance correctness |
| RAG answer | Citation validity, groundedness, stale-source handling |
| Sales knowledge | Document-level authorization, citation access, no-answer behavior |
| Sales copilot | Không biến inference thành fact; output luôn là draft |
| E2E | Buyer → consent → Sales → approval → booking |
| Security | Secret scan, PII-safe logs, unauthorized access tests |

Không gọi backend/AI là hoàn thành chỉ vì endpoint trả `200`. Release gate cần
có: happy path, negative cases, authorization, persistence qua restart, audit
trail và ít nhất một deployed smoke test.

## 10. Deployment target

```mermaid
flowchart LR
    Git[GitHub] --> CI[lint + typecheck + tests + build]
    CI --> Web[Next.js host]
    CI --> API[FastAPI container]
    API --> DB[(Managed PostgreSQL)]
    API --> Vec[(Managed/local vector index)]
    API --> Obs[Logs + traces + metrics]
    Web --> API
```

Production requirements: HTTPS, secret manager, database backups, PII redaction,
request IDs, structured logs, rate limiting, CORS allowlist, model timeouts,
fallback behavior và documented data-retention policy.
