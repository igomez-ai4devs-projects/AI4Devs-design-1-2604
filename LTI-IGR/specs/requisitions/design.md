# Domain: Requisitions & Job Posting — `design.md`

> **Companion to:** [`spec.md`](spec.md)
> **Scope:** data model (entities, attributes, relationships) for the Requisitions & Job Posting domain.
> **Last updated:** 2026-05-23

---

## 1. Conventions

- **Identifiers:** every entity has `id : UUID` (PK). Foreign keys named `<entity>Id`.
- **Multi-tenancy:** every top-level entity carries `tenantId : UUID`.
- **Timestamps:** `createdAt`, `updatedAt` (TIMESTAMPTZ, UTC).
- **Enums:** rendered as inline comments; persisted as constrained strings or DB enums.
- **Cardinality (Mermaid ER):** `||--o{` 1-to-many · `||--||` 1-to-1 · `||--|{` 1-to-one-or-many.
- **PII** is flagged in field comments.

---

## 2. Referenced Cross-Cutting Entities

Owned by the platform, referenced (FK only) by this domain.

| Entity | Why referenced |
|---|---|
| `Tenant` | Multi-tenancy boundary for every requisition. |
| `User` | Creator, approver, hiring-team member. |

---

## 3. Domain Entities

| Entity | Description |
|---|---|
| `Requisition` | Internal, approved request to hire for a role. |
| `JDTemplate` | Reusable job-description template. |
| `HiringTeamMember` | User assigned to a requisition (recruiter, hiring manager, panel). |
| `ApprovalStep` | One node in the approval chain — snapshot taken at submission time. |
| `JobPosting` | Public-facing publication derived from a requisition. |
| `KnockoutQuestion` | Mandatory pre-screening question attached to a posting. |
| `BoardDistribution` | Per-job-board syndication record (LinkedIn, Indeed, etc.). |

---

## 4. ER Diagram

```mermaid
erDiagram
    Tenant ||--o{ Requisition : "owns"
    Tenant ||--o{ JDTemplate : "owns"

    JDTemplate ||--o{ Requisition : "instantiates"
    User ||--o{ Requisition : "creates (hiringManager)"

    Requisition ||--|{ ApprovalStep : "has approval chain"
    User ||--o{ ApprovalStep : "approves"

    Requisition ||--o{ HiringTeamMember : "has team"
    User ||--o{ HiringTeamMember : "participates"

    Requisition ||--o{ JobPosting : "publishes as"
    JobPosting ||--o{ KnockoutQuestion : "screens with"
    JobPosting ||--o{ BoardDistribution : "syndicated to"

    Tenant {
        UUID id PK
        string name
        string plan
    }
    User {
        UUID id PK
        UUID tenantId FK
        string email "PII"
        string role
    }
    Requisition {
        UUID id PK
        UUID tenantId FK
        UUID jdTemplateId FK "nullable"
        UUID createdByUserId FK
        string title
        string department
        string costCenter
        string location
        string workMode "REMOTE|HYBRID|ONSITE"
        string employmentType "FULL_TIME|PART_TIME|CONTRACT|INTERN"
        string level "JUNIOR|MID|SENIOR|STAFF|LEAD"
        int salaryMin
        int salaryMax
        string currency "ISO-4217"
        text justification
        date targetStartDate
        string status "DRAFT|PENDING_APPROVAL|APPROVED|OPEN|ON_HOLD|CLOSED|CANCELLED"
        timestamptz createdAt
        timestamptz updatedAt
    }
    JDTemplate {
        UUID id PK
        UUID tenantId FK
        string name
        text contentMarkdown
        int version
    }
    HiringTeamMember {
        UUID id PK
        UUID requisitionId FK
        UUID userId FK
        string role "RECRUITER|HIRING_MANAGER|INTERVIEWER|OBSERVER"
    }
    ApprovalStep {
        UUID id PK
        UUID requisitionId FK
        UUID approverUserId FK
        int order
        string status "PENDING|APPROVED|REJECTED|SKIPPED"
        text comments "nullable"
        timestamptz decidedAt "nullable"
    }
    JobPosting {
        UUID id PK
        UUID requisitionId FK
        text jobDescriptionMarkdown
        string employerBrandTheme
        UUID applicationFormId
        UUID pipelineTemplateId
        string status "DRAFT|LIVE|UNPUBLISHED"
        string publicUrl
        string locale "en-US, es-ES, ..."
        timestamptz publishedAt "nullable"
        timestamptz closedAt "nullable"
    }
    KnockoutQuestion {
        UUID id PK
        UUID jobPostingId FK
        text question
        string answerType "BOOLEAN|NUMBER|CHOICE|TEXT"
        json expectedAnswer
        boolean isRequired
        int order
    }
    BoardDistribution {
        UUID id PK
        UUID jobPostingId FK
        string board "LINKEDIN|INDEED|GLASSDOOR|XING|NICHE_X"
        string externalPostingId
        string status "PENDING|LIVE|FAILED|UNPUBLISHED"
        int applicantCount
        timestamptz lastSyncedAt
        text errorMessage "nullable"
    }
```

---

## 5. Key Cardinality Rules

| Relation | Cardinality | Note |
|---|---|---|
| `Requisition → ApprovalStep` | 1 : N | Snapshot at submit — later org changes don't break in-flight reqs. |
| `Requisition → JobPosting` | 1 : N | One req can have N postings (locales, regions). |
| `JobPosting → KnockoutQuestion` | 1 : N | Optional pre-screening rules. |
| `JobPosting → BoardDistribution` | 1 : N | One row per syndicated board. |

---

## 6. Lifecycle Invariants

1. A `JobPosting` can only exist when its `Requisition.status = APPROVED` (or later).
2. `Requisition.status` transitions are linear: `DRAFT → PENDING_APPROVAL → APPROVED → OPEN → (ON_HOLD | CLOSED | CANCELLED)`.
3. Closing a `Requisition` cascades: every `JobPosting.status` becomes `UNPUBLISHED` within 24h and `BoardDistribution` rows are unpublished externally.
4. `ApprovalStep` rows are append-only after creation; comments can be added but `status` is immutable once decided.

---

## 7. Boundary with Other Domains

- **Outbound:** `JobPosting.id` is the FK consumed by `Application` (see [candidates/design.md](../candidates/design.md)) — this is the join point with the Candidates domain.
- **Outbound:** `KnockoutQuestion.id` is consumed by `KnockoutResponse` in the Candidates domain.

---

## 8. Open Questions

- `PipelineTemplate` as first-class entity vs. JSON on `JobPosting`? Proposal: first-class once templates are reused across postings.
- Programmatic job advertising (auto-bid) — needs a `BoardCampaign` entity in P2.
- Localization: model translations as `JobPostingLocale` child entity or one `JobPosting` row per locale? Currently modeled as one row per locale.
