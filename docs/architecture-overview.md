# Architecture Overview

## System Design

The KYC API Framework is organized into four layers that work together to deliver a real-time, modular KYC workflow.

```
┌─────────────────────────────────────────────────────────────────┐
│                     PRESENTATION LAYER                          │
│         Customer-facing onboarding UI / Mobile / Web            │
└─────────────────────────┬───────────────────────────────────────┘
                          │  REST API
┌─────────────────────────▼───────────────────────────────────────┐
│                    ORCHESTRATION LAYER                          │
│   KYC Workflow Engine │ Risk Scoring │ Decision Router          │
└──────┬─────────────────┬──────────────────────┬────────────────-┘
       │                 │                      │
┌──────▼──────┐  ┌───────▼──────┐   ┌──────────▼──────────┐
│  IDENTITY   │  │  DOCUMENT    │   │  SANCTIONS &         │
│VERIFICATION │  │  PROCESSING  │   │  WATCHLIST           │
│  SERVICE    │  │  SERVICE     │   │  SCREENING           │
└──────┬──────┘  └───────┬──────┘   └──────────┬──────────-┘
       │                 │                      │
┌──────▼─────────────────▼──────────────────────▼────────────────┐
│                      DATA LAYER                                 │
│    Customer Profile Store │ Audit Log │ Consent Registry       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Layer Descriptions

### Presentation Layer
The customer-facing interface where the onboarding journey begins. This framework is UI-agnostic — it works with web applications, mobile apps, or third-party onboarding portals. The framework defines the API contract; the UI implementation is left to each institution.

### Orchestration Layer

This is the core of the framework. Three components work together:

**KYC Workflow Engine**
Manages the state machine of a KYC application from initiation through approval, manual review, or denial. Supports parallel processing of verification steps (identity, document, sanctions) rather than sequential processing — this is the primary source of latency reduction.

States: `INITIATED` → `IN_PROGRESS` → `PENDING_REVIEW` (if escalated) → `APPROVED` / `DENIED` / `INCOMPLETE`

**Risk Scoring Module**
Assigns a risk tier to each applicant based on available signals (transaction type, geography, product requested, applicant profile). Risk tier determines which verification path is required:

| Risk Tier | Verification Path | Typical Latency |
|---|---|---|
| Low | Identity verification only | < 2 minutes |
| Medium | Identity + document verification | 5–15 minutes |
| High | Full KYC + manual review | 1–3 business days |

Risk tier assignment uses configurable rules — institutions define their own thresholds. Where ML-assisted scoring is used, NIST AI RMF governance controls apply (see [Compliance Alignment](compliance-alignment.md)).

**Decision Router**
Routes completed verification results to the appropriate downstream outcome: auto-approve, auto-deny, or escalate to human review queue. Decision rationale is always logged for audit purposes.

### Integration Services

The framework defines integration contracts for three external service categories. Institutions plug in their preferred vendors.

**Identity Verification Service**
Verifies that the applicant is who they claim to be. Typical methods: knowledge-based authentication (KBA), device biometrics, one-time passcodes.

Integration contract: POST `/verify/identity` → `{verified: boolean, confidence_score: float, method: string}`

**Document Processing Service**
Captures and validates government-issued ID documents. The framework supports both synchronous (real-time OCR) and asynchronous (upload + callback) processing patterns, allowing institutions to use lower-cost batch processing for lower-risk applications.

Integration contract: POST `/verify/document` → `{extracted_fields: object, authenticity_score: float, processing_mode: string}`

**Sanctions and Watchlist Screening**
Screens applicants against OFAC SDN, FinCEN 314(a), and institution-specific watchlists. This step runs in parallel with identity and document verification rather than sequentially — a key latency reduction in this framework.

Integration contract: POST `/screen/sanctions` → `{cleared: boolean, match_results: array, screening_timestamp: string}`

### Data Layer

**Customer Profile Store**
Canonical record of the applicant's KYC data, verification history, and current status. Structured to support CFPB Section 1033 data portability requirements — customers can request their own KYC data in machine-readable format.

**Audit Log**
Immutable append-only record of every action taken in the KYC workflow: who requested what, when, with what result, and what decision was made. Required for BSA/AML examination readiness and NIST AI RMF accountability requirements.

**Consent Registry**
Tracks customer consent for each data use, particularly relevant for consumer-permissioned data flows under CFPB Section 1033. Before any third-party data is used to assist KYC (e.g., open banking transaction history to supplement thin-file applicants), consent must be recorded here.

---

## Key Architectural Decisions

### Parallel vs. Sequential Verification

Legacy KYC architectures run verification steps sequentially: identity check completes, then document check begins, then sanctions screening begins. Total latency is the sum of all steps.

This framework runs all three verification steps in parallel. Total latency is the maximum of any single step — typically 60–70% faster for low-to-medium risk applications.

### Progressive Verification for Thin-File Applicants

A common failure mode for underserved populations: applicants with limited credit history or non-standard documentation are immediately routed to manual review, where they wait days and often abandon the process.

This framework supports progressive verification: applicants who cannot complete standard verification are offered alternative verification paths (e.g., consumer-permissioned bank transaction history via CFPB 1033, utility bill verification, employer verification) rather than being denied outright. The progressive path is risk-tiered — it only activates for applicants in low-to-medium risk categories.

### Event-Driven Status Updates

Rather than requiring the customer to check back for status, the framework uses a webhook/event pattern to push status updates to the institution's system of record and, optionally, to the customer's preferred notification channel. This eliminates the "application black hole" experience that causes abandonment.

---

## Technology Assumptions

This framework is designed to be technology-stack-agnostic. The following assumptions are minimal:

- RESTful HTTP/S API communication between components
- JSON request and response bodies
- OAuth 2.0 or API key authentication between services
- Relational or document database for profile storage (specific implementation left to institution)
- Standard webhook/callback pattern for asynchronous notifications

No specific programming language, cloud provider, or database vendor is required.
