# LTI-IGR — System & High-Level Architecture

> **Document type:** High-level architecture design
> **Scope:** Whole LTI-IGR platform (AI-native ATS)
> **Companion documents:** [`../AGENTS.md`](../AGENTS.md) · [`README.md`](README.md) · domain `spec.md` and `design.md` files under [requisitions/](requisitions/), [candidates/](candidates/), [interviews/](interviews/)
> **Last updated:** 2026-05-23
> **Status:** v0.1 — Initial architecture baseline

---

## 1. Architectural Goals & Drivers

Derived from the strategic positioning in [`../AGENTS.md`](../AGENTS.md) (AI-native ATS for high-growth tech, 50–500 employees) and the MVP use cases in [`README.md`](README.md).

| # | Driver | Architectural implication |
|---|---|---|
| G1 | **AI-native** (copilot, matching, agentic sourcing) | First-class AI service layer; async pipelines; vector store; cost controls (caching, multi-model) |
| G2 | **Multi-tenant SaaS** | Tenant isolation enforced at data + auth layer; per-tenant config and branding |
| G3 | **Open ecosystem** (HRIS, calendars, job boards, assessments) | Public API + webhooks + integration adapters; marketplace-ready |
| G4 | **Compliance** (SOC2, GDPR, EU AI Act, NYC LL144) | Audit log, consent management, RBAC, data residency, AI explainability |
| G5 | **Recruiter-grade UX** (keyboard-first, fast) | SPA front-end, optimistic UI, real-time updates, sub-200ms p95 read latency |
| G6 | **Scalability** (10× volume without linear headcount) | Stateless services, horizontal scaling, async workers, event-driven core |
| G7 | **Time-to-market** (MVP in Q1) | Modular monolith first, extract services only where justified by scale or team boundaries |

---

## 2. Architectural Style

LTI-IGR follows a **modular-monolith-with-satellite-services** style, evolving toward selective microservices:

- **Core API** as a modular monolith organized by the functional domains defined in [`README.md`](README.md) §2: `auth`, `organization`, `requisitions`, `candidates`, `sourcing`, `assessments`, `interviews`, `communications`, `offers`, `onboarding-handoff`, `analytics`, `integrations`, `automation`. Each module owns its tables and exposes an internal contract; cross-module calls only via published interfaces.
- **Satellite services** for workloads with different scaling, security, or technology profiles: AI/ML service, parsing service, integrations gateway, async workers, and the public career site.
- **Event bus** for cross-module side-effects (notifications, analytics, AI triggers, integration outbox) — the backbone of the [Spec Conventions §7 "Published events"](README.md) listed in every domain spec.
- **CQRS-lite** in read-heavy areas (pipeline boards, analytics): write through the domain module, read from purpose-built projections.

Rationale: optimizes for G7 (speed) without painting the team into a corner on G6 (scale) and G3 (ecosystem).

---

## 3. System Context (C4 — Level 1)

```mermaid
flowchart LR
    subgraph Users
        REC[Recruiter / Sourcer]
        HM[Hiring Manager / Interviewer]
        ADM[Admin / Talent Ops]
        AUD[Auditor / DPO]
        CAND[Candidate]
    end

    subgraph LTI[LTI-IGR Platform]
        WEB[Recruiter Web App]
        CAREER[Career Site / Apply Flow]
        API[Core API]
        AI[AI / ML Service]
    end

    subgraph External[External Systems]
        JB[Job Boards<br/>LinkedIn · Indeed · niche]
        CAL[Calendars<br/>Google · Microsoft 365]
        VID[Video<br/>Zoom · Teams · Meet]
        MAIL[Email / SMS<br/>SendGrid · Twilio]
        ASS[Assessment Vendors<br/>HackerRank · Codility]
        BG[Background Check<br/>Checkr · Sterling]
        ESIGN[E-Signature<br/>DocuSign]
        HRIS[HRIS / HRMS<br/>Workday · Rippling · BambooHR]
        SSO[Identity Providers<br/>Google · Microsoft · Okta · SAML]
        LLM[LLM Providers<br/>Anthropic · OpenAI]
        ANALYTICS[BI / Warehouse<br/>Snowflake · BigQuery]
    end

    REC --> WEB
    HM --> WEB
    ADM --> WEB
    AUD --> WEB
    CAND --> CAREER

    WEB --> API
    CAREER --> API
    API <--> AI
    AI --> LLM

    API <--> JB
    API <--> CAL
    API <--> VID
    API --> MAIL
    API <--> ASS
    API <--> BG
    API <--> ESIGN
    API <--> HRIS
    API <--> SSO
    API --> ANALYTICS
```

**Actors** map 1:1 to the roles in [`README.md`](README.md) §1.
**External systems** are the table-stakes integrations called out in [`../AGENTS.md`](../AGENTS.md) §10 ("Minimum Viable Feature Set" and "Key Integrations").

---

## 4. Container View (C4 — Level 2)

```mermaid
flowchart TB
    subgraph Edge
        CDN[CDN / WAF<br/>CloudFront]
        APIGW[API Gateway<br/>auth · rate limit · routing]
    end

    subgraph Frontends
        WEBAPP[Recruiter SPA<br/>React + TypeScript]
        CAREERSITE[Career Site<br/>SSR · per-tenant branding]
        CANDPORTAL[Candidate Portal<br/>apply · self-schedule · e-sign]
    end

    subgraph Backend["Core API (modular monolith)"]
        AUTH[Auth & RBAC]
        ORG[Organization]
        REQ[Requisitions]
        CANDM[Candidates]
        SRC[Sourcing]
        INT[Interviews]
        COMM[Communications]
        OFF[Offers]
        ONB[Onboarding Handoff]
        AN[Analytics]
        INTG[Integrations Gateway]
        AUTO[Automation Engine]
    end

    subgraph Async[Async Workloads]
        BUS[(Event Bus<br/>Kafka / Kinesis)]
        OUTBOX[(Transactional Outbox)]
        WORKERS[Workers<br/>BullMQ · SQS]
        SCHED[Scheduler<br/>cron · delayed jobs]
    end

    subgraph AI[AI Platform]
        PARSE[Resume Parsing]
        MATCH[Candidate-Job Matching]
        COPILOT[Recruiter Copilot]
        AGENT[Agentic Sourcing]
        EMB[Embeddings Pipeline]
    end

    subgraph Data
        PG[(PostgreSQL<br/>OLTP · per-tenant RLS)]
        OS[(OpenSearch<br/>candidate search)]
        VEC[(Vector DB<br/>pgvector / Pinecone)]
        S3[(Object Storage<br/>resumes · attachments)]
        REDIS[(Redis<br/>cache · sessions · queues)]
        DWH[(Warehouse<br/>Snowflake / BigQuery)]
    end

    subgraph Observability
        LOG[Logs · Metrics · Traces<br/>Datadog / OTel]
        AUDIT[Audit Log Store]
    end

    CDN --> WEBAPP
    CDN --> CAREERSITE
    CDN --> CANDPORTAL
    WEBAPP --> APIGW
    CAREERSITE --> APIGW
    CANDPORTAL --> APIGW

    APIGW --> AUTH
    APIGW --> ORG
    APIGW --> REQ
    APIGW --> CANDM
    APIGW --> SRC
    APIGW --> INT
    APIGW --> COMM
    APIGW --> OFF
    APIGW --> ONB
    APIGW --> AN
    APIGW --> INTG
    APIGW --> AUTO

    Backend --> PG
    Backend --> REDIS
    Backend --> S3
    CANDM --> OS
    CANDM --> VEC
    AN --> DWH

    Backend --> OUTBOX
    OUTBOX --> BUS
    BUS --> WORKERS
    SCHED --> WORKERS
    WORKERS --> AI
    WORKERS --> INTG
    WORKERS --> COMM

    AI --> VEC
    AI --> S3
    AI --> PG

    Backend --> LOG
    Backend --> AUDIT
    AI --> LOG
    WORKERS --> LOG
```

---

## 5. Component Distribution

### 5.1 Frontends

| Component | Audience | Tech | Notes |
|---|---|---|---|
| **Recruiter SPA** | Internal users (R1–R6, R9) | React + TypeScript, Tanstack Query, Tailwind | Keyboard-first; optimistic UI; WebSocket for live pipeline updates |
| **Career Site** | Candidates (R7) | Next.js SSR | Per-tenant theming, SEO, programmatic landing pages |
| **Candidate Portal** | Candidates (R7) | Next.js | Apply flow, self-scheduling, status, e-sign, GDPR self-service |

### 5.2 Edge Layer

- **CDN/WAF** — static asset delivery, DDoS protection, geo-routing.
- **API Gateway** — TLS termination, JWT/session validation, rate limiting, per-tenant quotas, request routing to backend modules.

### 5.3 Core API (modular monolith)

Each module owns the entities defined in the corresponding domain `design.md`. Module-to-module communication is **synchronous via internal interfaces** for read paths and **asynchronous via domain events** for write side-effects. See §6.

| Module | Owns | Key external touchpoints |
|---|---|---|
| **Auth & RBAC** | `User`, `Role`, `Permission`, `Session`, `Invitation` | SSO IdPs (SAML/OIDC), MFA providers |
| **Organization** | `Tenant`, `Team`, `Brand`, `Settings` | Domain DNS, asset CDN |
| **Requisitions** | `Requisition`, `JobPosting`, `KnockoutQuestion`, `ApprovalChain` | Job boards (via Integrations Gateway) |
| **Candidates** | `Candidate`, `Application`, `Resume`, `MatchScore`, `TalentPool`, `Consent` | AI Platform (parsing, matching), Search & Vector stores |
| **Sourcing** | `SourcingCampaign`, `Lead`, `Referral` | LinkedIn, Chrome extension, AI agent |
| **Interviews** | `InterviewSchedule`, `InterviewKit`, `Scorecard`, `HireDecision` | Calendars, video, email |
| **Communications** | `Template`, `Message`, `Thread`, `NurtureCampaign` | Email/SMS providers |
| **Offers** | `Offer`, `OfferApproval`, `Signature` | E-signature, background check |
| **Onboarding Handoff** | `OnboardingPacket` | HRIS/HRMS |
| **Analytics** | Read models, dashboards | Warehouse |
| **Integrations Gateway** | OAuth tokens, connector state, webhooks | All third parties |
| **Automation Engine** | `WorkflowRule`, `Trigger`, `Action`, `Task` | Event bus, Scheduler |

### 5.4 AI Platform (satellite service)

Isolated from the Core API for independent scaling, GPU/inference cost control, and clean swap-in/out of providers.

| Component | Purpose | UC mapping |
|---|---|---|
| **Resume Parsing** | Extract structured skills/experience/education from CV | UC-CAND-02 |
| **Candidate-Job Matching** | Vector + outcome-trained ranking with explainability | UC-SCR-02 |
| **Recruiter Copilot** | JD drafting, candidate summaries, outreach drafting | Cross-cutting |
| **Agentic Sourcing** | Autonomous discovery + outreach with human-in-the-loop | Roadmap Q4 |
| **Embeddings Pipeline** | Backfill + incremental embeddings of candidates and jobs | Foundational |

Guardrails: every AI decision that affects a candidate (rank, reject, flag) writes to the `AIDecision` audit table with model version, prompt hash, inputs, outputs, and explanation — required by EU AI Act and NYC LL144 ([`../AGENTS.md`](../AGENTS.md) §10 risks).

### 5.5 Async Workloads

- **Event Bus** (Kafka or Kinesis): persistent, partitioned by `tenantId`, source of truth for domain events.
- **Transactional Outbox** in PostgreSQL: guarantees at-least-once delivery from write transactions to the bus.
- **Workers** (BullMQ on Redis for short jobs; SQS-backed workers for long-running): consume events, perform side-effects (send email, call AI, push to job boards, generate reports).
- **Scheduler**: cron-style jobs (digest emails, SLA escalations, GDPR purge sweeps, interview reminders — UC-INT-06).

### 5.6 Data Stores

| Store | Use | Why |
|---|---|---|
| **PostgreSQL** | OLTP for all modules | Strong consistency; row-level security for tenant isolation |
| **OpenSearch** | Candidate / job full-text + faceted search (UC-CAND-04) | Boolean + relevance ranking at scale |
| **Vector DB** (pgvector → Pinecone at scale) | Semantic match, similarity search | AI matching, talent rediscovery |
| **Object Storage** (S3) | Resumes, attachments, generated offers | Cheap, durable, signed-URL delivery |
| **Redis** | Cache, sessions, rate limits, BullMQ queues | Low-latency hot paths |
| **Data Warehouse** (Snowflake/BigQuery) | Analytics, DEI reports, customer BI | Decoupled from OLTP; long retention |
| **Audit Log Store** | Append-only audit trail | Compliance (SOC2/GDPR) — UC-AN-04 |

---

## 6. Communication Patterns

LTI-IGR uses four communication patterns, chosen by use case:

### 6.1 Synchronous Request/Response — REST + JSON over HTTPS

Used between frontends and Core API, and for module-to-module read paths inside the monolith. OpenAPI-described, versioned (`/v1`), idempotency keys on mutating endpoints.

### 6.2 Real-time Push — WebSocket / Server-Sent Events

Used for pipeline board updates, interview scheduling collision warnings, copilot streaming responses. Connection terminated at API Gateway, fanned out via Redis pub/sub.

### 6.3 Asynchronous Domain Events — Event Bus

Every domain spec under [`README.md`](README.md) §2 publishes events (see "Published events" section). Examples:

| Event | Producer | Consumers |
|---|---|---|
| `RequisitionApproved` | Requisitions | Integrations (post to boards), Communications (notify recruiter), Analytics |
| `ApplicationSubmitted` | Candidates | AI (parse + match), Communications (ack email), Automation |
| `MatchScored` | AI Platform | Candidates (persist), Automation (auto-advance/knockout) |
| `InterviewScheduled` | Interviews | Calendars, Communications (invites + reminders), Analytics |
| `OfferAccepted` | Offers | Onboarding Handoff (HRIS), Communications, Analytics |

Delivery: at-least-once via transactional outbox; consumers are idempotent.

### 6.4 Third-party Integration — Adapters & Webhooks

The **Integrations Gateway** is the only module that talks to external SaaS. Inbound webhooks (e.g., assessment completed, e-sign event, calendar update) hit a public endpoint, are verified (HMAC), normalized to internal events, and dropped on the bus. Outbound calls go through provider-specific adapters with retry, circuit breaker, and per-tenant OAuth token storage.

```mermaid
sequenceDiagram
    autonumber
    participant Recruiter
    participant SPA as Recruiter SPA
    participant API as Core API
    participant Bus as Event Bus
    participant AI as AI Platform
    participant Mail as Email Provider

    Recruiter->>SPA: Submit candidate application (manual add)
    SPA->>API: POST /v1/applications
    API->>API: Persist Application + write Outbox
    API-->>SPA: 201 Created (optimistic ack)
    API->>Bus: ApplicationSubmitted
    par Async fan-out
        Bus->>AI: Parse resume
        AI->>API: PUT /v1/resumes/{id}/parsed
        AI->>Bus: ResumeParsed
        Bus->>AI: Score match (UC-SCR-02)
        AI->>API: PUT /v1/applications/{id}/match
        AI->>Bus: MatchScored
    and
        Bus->>Mail: Send acknowledgement email
    end
    Bus->>SPA: Push update (WebSocket)
```

---

## 7. External System Integrations

Aligned with the integration roadmap in [`../AGENTS.md`](../AGENTS.md) §10.

| Category | Providers (priority) | Pattern | Direction |
|---|---|---|---|
| **Identity** | Google · Microsoft · Okta · generic SAML | OIDC / SAML | Inbound auth |
| **Calendars** | Google Calendar · Microsoft 365 | OAuth2 + webhooks | Bi-directional (UC-INT-01) |
| **Video** | Zoom · Microsoft Teams · Google Meet | OAuth2 + API | Outbound (create meeting) |
| **Job boards** | LinkedIn · Indeed · niche boards | XML feed + API | Outbound (UC-REQ-04) |
| **Email / SMS** | SendGrid · Postmark · Twilio | API + inbound webhooks (replies) | Bi-directional |
| **Assessments** | HackerRank · Codility · Codesignal | API + webhooks | Bi-directional (UC-SCR-03) |
| **Background checks** | Checkr · Sterling | API + webhooks | Bi-directional |
| **E-signature** | DocuSign · Dropbox Sign | API + webhooks | Bi-directional (UC-OFF-03) |
| **HRIS / HRMS** | Workday · Rippling · BambooHR · Gusto (via Merge.dev or Finch) | API + webhooks | Outbound on hire (UC-OFF-04) |
| **LLM providers** | Anthropic · OpenAI · self-hosted small models | API (with caching) | Outbound |
| **Analytics / BI** | Snowflake · BigQuery · Looker | Reverse-ETL + native connectors | Outbound |

**Public API & webhooks** — same surface the Integrations Gateway uses internally is exposed to customers and marketplace partners (G3). REST + OpenAPI, OAuth2 client credentials, per-tenant scoped tokens, signed outbound webhooks.

---

## 8. Cross-cutting Concerns

| Concern | Approach |
|---|---|
| **Multi-tenancy** | `tenantId` on every row, enforced by PostgreSQL Row-Level Security; tenant claim baked into every JWT and event |
| **Authentication & RBAC** | Central Auth module; resource-action-scope permissions; role templates per [`README.md`](README.md) §1 |
| **Audit & Compliance** | Append-only `AuditEvent` stream for every state change; AI decisions logged with explainability; GDPR right-to-be-forgotten cascades via soft-delete + scheduled purge (UC-CAND-09) |
| **Observability** | OpenTelemetry traces across API → workers → AI; structured logs with `tenantId` + `requestId`; metrics for funnel SLOs |
| **Secrets** | Cloud KMS-backed secret manager; per-tenant OAuth tokens encrypted at rest |
| **Data residency** | Tenant-pinned region (US / EU) selected at signup; events partitioned per region |
| **Idempotency** | Client-provided `Idempotency-Key` on mutating endpoints; deduplication keys on consumers |
| **Rate limiting & quotas** | At API Gateway, per-tenant and per-user; AI calls have a separate quota with bill-shock guardrails |
| **Feature flags** | Per-tenant flags for staged rollouts and pilot AI features |

---

## 9. Deployment Topology

- **Cloud:** AWS (primary), with portability through Kubernetes + Terraform.
- **Runtime:** EKS clusters per region (us-east-1, eu-west-1). Stateless services scale horizontally; AI Platform on GPU node pools.
- **Environments:** `dev` → `staging` → `prod`, with ephemeral preview environments per PR for the SPA + API contract.
- **CI/CD:** GitHub Actions → image registry → ArgoCD; database migrations gated by review and run before service rollout.
- **DR:** Multi-AZ active-active; cross-region async replication for OLTP; RPO ≤ 5 min, RTO ≤ 1 hour.

```mermaid
flowchart LR
    subgraph US[AWS us-east-1]
        EKSUS[EKS Cluster<br/>API · Workers · AI]
        PGUS[(RDS Postgres<br/>Multi-AZ)]
        S3US[(S3)]
        REDISUS[(ElastiCache)]
    end
    subgraph EU[AWS eu-west-1]
        EKSEU[EKS Cluster<br/>API · Workers · AI]
        PGEU[(RDS Postgres<br/>Multi-AZ)]
        S3EU[(S3)]
        REDISEU[(ElastiCache)]
    end
    CFRONT[CloudFront + Route53 geo-routing]
    CFRONT --> EKSUS
    CFRONT --> EKSEU
    EKSUS --> PGUS
    EKSUS --> S3US
    EKSUS --> REDISUS
    EKSEU --> PGEU
    EKSEU --> S3EU
    EKSEU --> REDISEU
    PGUS -. async DR replica .- PGEU
```

---

## 10. Tech Stack (proposed)

| Layer | Choice | Rationale |
|---|---|---|
| Frontend | React + TypeScript, Next.js for SSR sites | Ecosystem, hiring pool, SSR for SEO |
| Backend | TypeScript (Node/NestJS) **or** Go for hot paths | Type-safe, share types with FE; Go reserved for ingestion/matching |
| API style | REST + OpenAPI; GraphQL gateway for SPA aggregation (later) | Predictable; partner-friendly |
| Datastore | PostgreSQL · Redis · S3 · OpenSearch · pgvector | Battle-tested; defer Pinecone until scale |
| Event bus | Kafka (MSK) or Kinesis | Durable, partitioned, replay |
| Workers | BullMQ (short) · SQS + workers (long) | Mix matches workload profile |
| AI | Anthropic Claude (default) + OpenAI fallback + fine-tuned small models | Multi-model strategy ([`../AGENTS.md`](../AGENTS.md) §10 risks) |
| Auth | Cloud-hosted IdP (WorkOS / Auth0) + SAML/OIDC | SSO/SCIM out-of-the-box for enterprise |
| Observability | OpenTelemetry · Datadog | Single pane of glass |
| Infra | AWS · EKS · Terraform · ArgoCD | Standard cloud-native |

Stack choices are **proposals, not commitments** — they will be revisited per module once team and load profile are known.

---

## 11. Architecture Evolution Plan

| Stage | Trigger | Change |
|---|---|---|
| **MVP (Q1–Q2)** | Launch | Modular monolith + AI Platform + workers. Single region (US). pgvector. |
| **Scale-up (Q3–Q4)** | First enterprise + EU customer | EU region; extract Integrations Gateway and AI Platform if independent scaling required; move to Pinecone if recall/latency degrades. |
| **Platform (Year 2)** | Marketplace + 100+ partner integrations | Public API hardening; integrations marketplace; extract Communications and Analytics into independent services. |
| **Enterprise (Year 2+)** | Multi-entity + global compliance | Data residency tiers; pluggable AI providers per tenant; dedicated single-tenant deploy option. |

---

## 12. Open Questions

1. **Vector store** — pgvector long-term, or commit to managed Pinecone/Weaviate at MVP for talent-pool scale?
2. **Eventing** — Kafka (MSK) vs Kinesis: trade off operational complexity vs replay/tooling.
3. **HRIS connector strategy** — build directly, or use unified API (Merge.dev / Finch) — speed vs margin.
4. **Single-tenant vs pooled** for enterprise — required tier or sales narrative only?
5. **AI provider lock-in** — how deeply do we couple to Anthropic features (e.g., tool use, computer use) vs maintain provider neutrality?
6. **GraphQL gateway** — introduce in Q3 for SPA aggregation, or stay REST-only?

---

## 13. Deep Dive — AI Platform (C4 Level 3)

This section zooms from the container-level **AI Platform** box in §4 into its internal components. The AI Platform is selected because it is the platform's primary strategic differentiator ([`../AGENTS.md`](../AGENTS.md) §5, §10) and the riskiest subsystem (cost, compliance, model drift). All other modules treat it as a black box behind a stable contract.

### 13.1 Scope & Responsibilities

The AI Platform owns:

1. **Resume understanding** — extract structured `ParsedSkill` / `ParsedExperience` / `ParsedEducation` from raw CVs (UC-CAND-02).
2. **Candidate–job matching** — produce a `MatchScore` with explainability for every `Application` (UC-SCR-02).
3. **Recruiter copilot** — interactive drafting (JD, outreach, summaries, scorecard hints) backing the SPA.
4. **Agentic sourcing** — autonomous loops that propose passive candidates with human-in-the-loop approval (roadmap Q4).
5. **Embeddings lifecycle** — keep vector representations of candidates and jobs current.
6. **Responsible-AI guardrails** — bias checks, explainability records, decision logging required by EU AI Act and NYC LL144.

Out of scope: business workflow state (owned by Candidates / Requisitions / Interviews modules) and any direct user authentication (delegated to Auth).

### 13.2 Component Diagram

```mermaid
flowchart TB
    subgraph External["External callers"]
        APIC[Core API<br/>Candidates · Requisitions · Sourcing]
        WK[Async Workers]
        SPA[Recruiter SPA<br/>streaming copilot]
    end

    subgraph AI[AI Platform]
        GW[AI API Gateway<br/>auth · quotas · cost meter]
        ORCH[Inference Orchestrator<br/>routing · retries · timeouts]
        ROUTER[Model Router<br/>provider + size selection]
        PROMPT[Prompt & Template Registry<br/>versioned · A/B tagged]
        CACHE[Semantic Cache<br/>Redis + embeddings]
        EVAL[Evaluation Harness<br/>online + offline]
        GUARD[Safety & Bias Guardrails<br/>PII redaction · toxicity · fairness checks]
        EXPL[Explainability Service<br/>feature attribution · rationale]
        LEDGER[AI Decision Ledger<br/>append-only audit]

        subgraph Pipelines["Domain pipelines"]
            PARSE[Resume Parsing Pipeline]
            MATCH[Matching Pipeline<br/>candidate ↔ job]
            COPILOT[Copilot Service<br/>chat · drafting · summarize]
            AGENT[Agent Runtime<br/>sourcing loop · tool calls]
            EMB[Embeddings Worker]
        end

        FEAT[Feature Store<br/>candidate · job · outcome features]
        MODEL[Model Registry<br/>versioned weights · prompts · evals]
    end

    subgraph DataPlane["Shared data plane"]
        VEC[(Vector DB<br/>pgvector / Pinecone)]
        S3[(Object Storage<br/>resumes · prompt logs)]
        PG[(PostgreSQL<br/>read-only views: applications · outcomes)]
        DWH[(Warehouse<br/>training datasets)]
    end

    subgraph Providers["External providers"]
        ANTH[Anthropic Claude]
        OAI[OpenAI]
        SLM[Self-hosted Small Models<br/>Llama · Qwen]
    end

    APIC -->|HTTPS / gRPC| GW
    WK -->|event triggers| GW
    SPA -->|SSE streaming| GW

    GW --> ORCH
    ORCH --> PARSE
    ORCH --> MATCH
    ORCH --> COPILOT
    ORCH --> AGENT
    ORCH --> EMB

    PARSE --> ROUTER
    MATCH --> ROUTER
    COPILOT --> ROUTER
    AGENT --> ROUTER
    EMB --> ROUTER

    ROUTER --> CACHE
    ROUTER --> PROMPT
    ROUTER --> ANTH
    ROUTER --> OAI
    ROUTER --> SLM

    MATCH --> FEAT
    MATCH --> VEC
    EMB --> VEC
    PARSE --> S3

    Pipelines --> GUARD
    GUARD --> LEDGER
    Pipelines --> EXPL
    EXPL --> LEDGER

    FEAT --> PG
    FEAT --> DWH
    MODEL --> ROUTER
    EVAL --> MODEL
    EVAL --> DWH
```

### 13.3 Components

| # | Component | Responsibility | Notes |
|---|---|---|---|
| C1 | **AI API Gateway** | Authenticates callers (mTLS for internal, JWT for SPA), enforces per-tenant quotas, meters token cost, exposes one versioned contract (`/ai/v1/...`) | Single entry point — all AI traffic is observable here |
| C2 | **Inference Orchestrator** | Dispatches each request to the correct pipeline; owns retry, timeout, and circuit-breaker policy; emits trace spans | Stateless; horizontally scaled |
| C3 | **Model Router** | Chooses model + provider per request (cost, latency SLO, capability, tenant policy); supports cascading fallback (e.g., Claude → OpenAI → small local model) | Driven by config in Model Registry; per-tenant overrides for compliance |
| C4 | **Prompt & Template Registry** | Versioned prompts with metadata (model, locale, eval scores); A/B test tags | Prompts are code: PR-reviewed, immutable once tagged |
| C5 | **Semantic Cache** | Two-level cache: exact-match (Redis) + semantic similarity (embedding lookup) | Bounded TTL; tenant-scoped keys; major lever for G1 cost control |
| C6 | **Resume Parsing Pipeline** | OCR (when needed) → LLM extraction → schema validation → confidence scoring | Persists parsed JSON back to `Resume` via Core API (UC-CAND-02) |
| C7 | **Matching Pipeline** | Hybrid: vector similarity (semantic) + structured rules (must-haves, knockouts) + outcome-trained ranker; outputs `MatchScore` + rationale | Implements UC-SCR-02; explainability mandatory |
| C8 | **Copilot Service** | Streaming chat/drafting backend for the SPA (SSE); supports tools (read candidate, search jobs, draft email) | Tool calls go back through Core API with the caller's RBAC scope, never the AI's |
| C9 | **Agent Runtime** | Long-running, checkpointed agent loops for sourcing; bounded by step / cost / time budget; every external action requires human approval token | Q4 roadmap; high-blast-radius — strict guardrails |
| C10 | **Embeddings Worker** | Computes and refreshes vector embeddings on `Resume`, `JobPosting`, and `Candidate` profile changes | Triggered by domain events `ResumeParsed`, `JobPostingPublished` |
| C11 | **Feature Store** | Online + offline features (skills, recency, outcome labels) for the matching ranker | Online via Redis; offline via warehouse for training |
| C12 | **Model Registry** | Versioned models, prompts, eval results, deployment stages (canary, prod) | Source of truth for what is "live" |
| C13 | **Evaluation Harness** | Offline benchmarks before promotion; online shadow eval; drift detection on production traffic | Blocks promotion if regressions on hire-outcome metrics |
| C14 | **Safety & Bias Guardrails** | PII redaction before logging; toxicity filter; demographic parity / adverse-impact checks on matching outcomes | Hard fail closed for matching pipeline; soft warn for copilot |
| C15 | **Explainability Service** | Generates human-readable rationale (feature attributions for ranker, key spans for LLM decisions) | Required for every candidate-affecting decision (UC-CAND-09, EU AI Act, LL144) |
| C16 | **AI Decision Ledger** | Append-only store: input hash, prompt version, model version, output, explanation, guardrail verdicts, cost | Joined to `AuditEvent` for UC-AN-04 auditor exports |

### 13.4 Internal Contract (selected endpoints)

All endpoints are tenant-scoped via the JWT subject, idempotent on `Idempotency-Key`, and return a `decisionId` that resolves to the AI Decision Ledger entry.

| Endpoint | Caller | Purpose |
|---|---|---|
| `POST /ai/v1/resumes/{id}/parse` | Candidates module (after upload) | Run resume parsing pipeline; result also pushed back via event |
| `POST /ai/v1/applications/{id}/match` | Candidates module on `ApplicationSubmitted` | Compute `MatchScore` for one application |
| `POST /ai/v1/jobs/{id}/rematch` | Requisitions on JD edit | Re-rank existing applications |
| `POST /ai/v1/copilot/messages` (SSE) | Recruiter SPA | Streaming copilot turns with tool use |
| `POST /ai/v1/agents/sourcing/runs` | Sourcing module / scheduler | Start a bounded sourcing agent run |
| `GET  /ai/v1/decisions/{decisionId}` | Audit, Candidates UI ("why this score?") | Retrieve full explanation + provenance |

### 13.5 Matching Request — End-to-End Flow

```mermaid
sequenceDiagram
    autonumber
    participant Bus as Event Bus
    participant GW as AI API Gateway
    participant ORCH as Inference Orchestrator
    participant CACHE as Semantic Cache
    participant MATCH as Matching Pipeline
    participant FEAT as Feature Store
    participant VEC as Vector DB
    participant RT as Model Router
    participant LLM as LLM Provider
    participant GUARD as Guardrails
    participant EXPL as Explainability
    participant LED as Decision Ledger
    participant API as Core API

    Bus->>GW: ApplicationSubmitted(applicationId)
    GW->>ORCH: match(applicationId)
    ORCH->>CACHE: lookup(applicationId, jobId, modelHash)
    alt cache hit
        CACHE-->>ORCH: cached MatchScore + decisionId
    else cache miss
        ORCH->>MATCH: run()
        MATCH->>FEAT: fetch candidate + job features
        MATCH->>VEC: top-K similar candidates / skills
        MATCH->>RT: rerank(candidate, job, features)
        RT->>LLM: scoring + rationale prompt
        LLM-->>RT: score, rationale
        RT-->>MATCH: structured score
        MATCH->>GUARD: fairness + PII check
        GUARD-->>MATCH: ok / blocked
        MATCH->>EXPL: build explanation
        EXPL-->>MATCH: rationale + feature attributions
        MATCH->>LED: append decision (inputs hash, prompt ver, model ver, output, cost)
        MATCH-->>ORCH: MatchScore + decisionId
        ORCH->>CACHE: store
    end
    ORCH-->>GW: result
    GW->>API: PUT /v1/applications/{id}/match
    GW->>Bus: MatchScored(applicationId, score, decisionId)
```

### 13.6 Data Ownership

| Asset | Owner | Notes |
|---|---|---|
| Prompts, model configs, eval reports | AI Platform (Model Registry) | Versioned in Git + registry |
| Embeddings | AI Platform | Tenant-partitioned indices |
| Decision ledger | AI Platform | Read by Auth-scoped audit endpoints |
| Parsed resume structured data | Candidates module | AI writes via Core API |
| `MatchScore` row | Candidates module | AI writes via Core API; `decisionId` links to ledger |
| Outcome labels (hire, reject, retention) | Analytics module | Streamed into the Feature Store for training |

This split keeps the AI Platform free of domain-state ownership: business modules remain authoritative, AI is a stateless decision producer plus its own observability/provenance state.

### 13.7 Cross-cutting Behaviors

| Concern | Implementation |
|---|---|
| **Cost control** | Token meter at C1; semantic cache (C5); cascading router (C3) prefers small/local models when SLO permits; per-tenant monthly cost budgets with throttle |
| **Latency budgets** | Copilot p95 ≤ 1.5s first token, ≤ 8s full turn; matching p95 ≤ 4s; parsing p95 ≤ 10s per CV |
| **Provider neutrality** | Single internal `LLMRequest` schema; all provider-specific quirks live in adapters under C3 |
| **Privacy** | PII redaction before any prompt leaves the VPC (C14); per-tenant opt-in for provider training data sharing — default off |
| **Fairness audits** | Nightly job in C13 computes adverse-impact ratios per protected class on `MatchScored` events; alerts on threshold breach |
| **Reproducibility** | Every ledger entry stores enough to replay: input hash, prompt version, model version, sampling params |
| **Kill switch** | Per-tenant flag disables AI features without redeploy; matching pipeline degrades to rule-based fallback |

### 13.8 Failure Modes & Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| Provider outage | Copilot, matching unavailable | Router cascade to alternate provider; small-model fallback for matching |
| Prompt regression | Quality drop | Eval harness blocks promotion; canary rollout per tenant; one-click rollback in Model Registry |
| Cost spike (loop / abuse) | Bill shock | Token meter hard cap per tenant + per user; agent runs are step-bounded |
| Bias drift | Compliance breach | Fairness audit (C14) alerts; auto-quarantine model version |
| Hallucinated tool call by Copilot | Wrong data shown | Tool calls re-enter Core API under caller RBAC; AI cannot bypass authorization |
| Stale embeddings after JD edit | Bad matches | `JobPostingPublished` re-triggers Embeddings Worker; rematch job for active applications |

### 13.9 Open Questions Specific to This Component

1. **In-house ranker vs LLM-as-judge** — train a dedicated learning-to-rank model on outcome data, or rely on LLM scoring with structured features? Affects C7 and the entire training pipeline.
2. **Per-tenant model fine-tuning** — offer as enterprise feature, or stay with prompt-level personalization only?
3. **Agent sandboxing** — run C9 tool calls inside a separate process / VM, or trust process-level isolation?
4. **Feature Store buy vs build** — Feast/Tecton vs custom on Postgres+Redis at MVP.

---

## 14. Changelog

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-05-23 | Initial high-level architecture: context, containers, components, communication patterns, integrations, deployment. |
| 0.2 | 2026-05-23 | Added §13 — Deep Dive on the AI Platform component (C4 Level 3): components, internal contract, matching sequence, data ownership, cross-cutting behaviors, failure modes. |
