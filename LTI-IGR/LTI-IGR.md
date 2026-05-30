# LTI-IGR — Product Requirements Document (Index)

> **Product:** LTI-IGR — AI-native Applicant Tracking System
> **Document type:** Master index / PRD entry point
> **Audience:** Product, Engineering, Design, GTM, Compliance
> **Last updated:** 2026-05-23

This document is the entry point to the LTI-IGR product specification. It links every artifact under the `LTI-IGR/` folder and orders them as a standard Product Requirements Document (PRD): from product vision and market context, through functional and data requirements, down to system architecture and supporting material.

---

## 1. Product Vision & Strategic Context

Foundational documents that establish *why* the product exists, *who* it is for, and *how* it competes.

| # | Document | Purpose |
|---|---|---|
| 1.1 | [AGENTS.md](AGENTS.md) | **Living strategic specification.** Defines what an ATS is, the core feature catalog, end-to-end recruitment workflow, customer value proposition by segment, competitive landscape, market opportunity, Lean Canvas, revenue model analysis, and strategic recommendations (MVP scope, pricing, GTM, 24-month roadmap, risks). The source of truth for business and product strategy. |

---

## 2. Scope & Functional Requirements

Definition of the functional surface of the MVP: roles, domains, and use cases.

| # | Document | Purpose |
|---|---|---|
| 2.1 | [specs/README.md](specs/README.md) | **Use case catalog & domain map.** Enumerates user roles (R1–R9), functional domains, the master MVP use-case catalog (with P0/P1/P2 priorities), inter-domain dependency map, the three core business flows, and the conventions every domain spec must follow. The functional-requirements anchor of the PRD. |

---

## 3. Detailed Functional Specifications (per domain)

One `spec.md` per functional domain. Each follows the structure defined in [specs/README.md §6](specs/README.md): domain summary, roles, use-case list, detailed flows, Mermaid diagrams, business rules, published events, and open questions.

| # | Domain | Document | Purpose |
|---|---|---|---|
| 3.1 | **Requisitions** | [specs/requisitions/spec.md](specs/requisitions/spec.md) | Functional spec for the requisition lifecycle: creation, approval workflow, career-site publication, and multi-board distribution. Covers UC-REQ-01 through UC-REQ-05. Origin of the hiring funnel. |
| 3.2 | **Candidates** | [specs/candidates/spec.md](specs/candidates/spec.md) | Functional spec for the candidate journey: application, CV parsing, knockout screening, AI candidate–job matching, talent database, pipeline management, and GDPR consent. Covers UC-CAND-01 through UC-CAND-09 plus UC-SCR-01/02. Heart of the funnel. |
| 3.3 | **Interviews** | [specs/interviews/spec.md](specs/interviews/spec.md) | Functional spec for interview scheduling, interview kits, structured scorecards, collaborative debrief, and the hire decision. Covers UC-INT-01 through UC-INT-06. Where applications convert into hires. |

---

## 4. Data Model & Domain Design (per domain)

One `design.md` per functional domain, defining the entities, attributes, relationships, and Mermaid ER diagrams that back the spec.

| # | Domain | Document | Purpose |
|---|---|---|---|
| 4.1 | **Requisitions** | [specs/requisitions/design.md](specs/requisitions/design.md) | Data model for the Requisitions domain: `Requisition`, `JobPosting`, `KnockoutQuestion`, approval chain entities, and their relationships. |
| 4.2 | **Candidates** | [specs/candidates/design.md](specs/candidates/design.md) | Data model for the Candidates domain: `Candidate`, `Consent`, `Resume` and parsed sub-entities, `Application`, `KnockoutResponse`, `MatchScore`, `Tag`, `RejectionReason`, `TalentPool`. |
| 4.3 | **Interviews** | [specs/interviews/design.md](specs/interviews/design.md) | Data model for the Interviews domain: `InterviewSchedule`, `InterviewKit`, `Scorecard`, `HireDecision`, and related supporting entities. |

---

## 5. System & Technical Architecture

How the platform is built to satisfy the functional and non-functional requirements above.

| # | Document | Purpose |
|---|---|---|
| 5.1 | [specs/design.md](specs/design.md) | **High-level system & architecture design.** Architectural drivers, chosen style (modular monolith + satellite services + event bus), C4 Level 1 (system context) and Level 2 (containers), component distribution, communication patterns (sync REST, real-time push, async events, third-party adapters), external system integrations, cross-cutting concerns (multi-tenancy, RBAC, audit, observability, residency), deployment topology, proposed tech stack, evolution plan, and a C4 Level 3 deep dive on the AI Platform component. |

---

## 6. Appendices & Process Artifacts

Working material that supports the documents above but is not part of the formal PRD.

| # | Document | Purpose |
|---|---|---|
| 6.1 | [prompts-IGR.md](prompts-IGR.md) | **Prompt log.** Captures the prompts used to drive the iterative generation of the specifications above. Useful for traceability, reproducibility, and auditing how each artifact was produced. |

---

## 7. Reading Order

| Reader | Recommended path |
|---|---|
| Executive / Investor | §1.1 → §2.1 → §5.1 (skim) |
| Product Manager | §1.1 → §2.1 → §3.x (all) → §5.1 |
| Engineer / Architect | §2.1 → §5.1 → §4.x (all) → §3.x for the domain you own |
| Designer | §1.1 §4 (segments) → §2.1 (roles) → §3.x (flows) |
| Compliance / Legal | §1.1 §7 + §10 (risks) → §5.1 §8 (cross-cutting) → §3.2 (GDPR use cases) |
| Sales / GTM | §1.1 §4 → §1.1 §6 → §1.1 §9 → §1.1 §10 |

---

## 8. Changelog

| Version | Date | Change |
|---|---|---|
| 0.1 | 2026-05-23 | Initial PRD index linking strategy, functional specs, data models, architecture, and appendices. |
