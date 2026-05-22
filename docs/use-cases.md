# Use Cases

This document describes the primary implementation scenarios for the KYC API Framework, organized by institution type.

---

## Use Case 1: Community Bank — Retail Account Onboarding

**Institution Profile:** Community bank, $2–8B in assets, serving a mixed urban/suburban customer base including first-generation banking customers and recent immigrants.

**Problem:** The bank's existing account opening process requires in-branch visits for identity verification. Customers who work multiple jobs or lack transportation cannot complete the process. Online account opening has a 60% abandonment rate because document upload and manual review takes 3–5 business days.

**Framework Application:**

The bank implements the full framework with emphasis on real-time identity verification and progressive verification for thin-file applicants.

*Workflow for a standard retail checking account:*

1. Customer initiates online application
2. Workflow Engine assigns LOW or MEDIUM risk tier (retail checking, new customer)
3. Identity Verification and Sanctions Screening run in parallel (< 90 seconds total)
4. If identity_confidence_score ≥ 0.85: auto-approve, account opened immediately
5. If identity_confidence_score 0.65–0.84: trigger progressive verification — customer offered option to connect existing bank account via Section 1033 or upload utility bill
6. If identity_confidence_score < 0.65: route to manual review queue with 4-hour SLA

**Expected Outcomes (based on validated production patterns):**
- 65–70% of applicants reach auto-approval without any manual intervention
- Average completion time for auto-approved applications: under 3 minutes
- Manual review queue volume: 15–20% of applications (vs. 100% in branch-only model)
- Abandonment rate reduction: estimated 35–45% improvement

---

## Use Case 2: Credit Union — Member Onboarding with HELOC Origination

**Institution Profile:** Regional credit union, $500M–$2B in assets, community-chartered serving a defined geographic field of membership.

**Problem:** HELOC and personal loan origination requires full KYC, but the credit union's existing process runs identity, document, and credit checks sequentially. Total time-to-decision averages 4–6 business days. Members frequently go to competing banks that offer faster decisions.

**Framework Application:**

The credit union implements the orchestration layer with parallel verification, emphasizing the Document Processing Service for loan-type products.

*Workflow for a HELOC application:*

1. Member initiates application (web or branch)
2. Workflow Engine assigns MEDIUM or HIGH risk tier (secured credit product)
3. Identity Verification, Document Processing (government ID + income documentation), and Sanctions Screening run in parallel
4. MEDIUM risk: if all three services return clean results, route to underwriting with complete KYC package within 30 minutes
5. HIGH risk or adverse signals: route to manual BSA review with full audit package

**Expected Outcomes:**
- Time from application to complete KYC package delivered to underwriting: under 1 hour for MEDIUM risk (vs. 4–6 days)
- Manual review volume reduction: approximately 40%
- Member satisfaction improvement through status webhooks (member receives real-time updates rather than waiting)

---

## Use Case 3: CDFI — Small Business Lending KYC

**Institution Profile:** Community Development Financial Institution (CDFI) providing small business loans to minority-owned businesses and businesses in low-to-moderate income census tracts.

**Problem:** Small business KYC requires both individual identity verification (for business owners) and entity-level verification (beneficial ownership, business registration). The CDFI's borrowers frequently have limited business credit history, non-standard business structures, or businesses that have operated informally. Standard KYC processes reject or heavily delay these applications.

**Framework Application:**

The CDFI implements the framework with custom risk tier logic specifically designed for their borrower population, and an extended progressive verification path that accommodates informal business documentation.

*Custom risk tier for CDFI small business:*

```
LOW RISK (eligible for streamlined path):
  - Loan amount < $50,000
  - Business operating > 2 years (any documentation)
  - Owner identity verified with confidence ≥ 0.75
  - No adverse sanctions results

MEDIUM RISK (enhanced verification, not automatic denial):
  - Business operating < 2 years OR informal documentation
  - Owner identity confidence 0.55–0.74
  - Trigger: progressive verification with CDFI-approved alternatives
    (community references, alternative business records, church/community organization letters)

HIGH RISK (manual review, not denial):
  - Complex beneficial ownership structure
  - Sanctions screening flags (high match threshold to avoid false positives)
  - Route to loan officer with full documentation package
```

**Key Design Decision:** The CDFI configured the progressive verification threshold at 0.55 (lower than the default 0.65) based on their mission population's documentation patterns. This is a legitimate, documented configuration choice — not a compliance shortcut.

**Expected Outcomes:**
- Applications that previously auto-denied (low confidence identity, informal business docs) now route to progressive verification or manual review rather than outright denial
- Loan officers receive structured KYC packages rather than raw document uploads
- BSA examination readiness: full audit trail for every application decision

---

## Use Case 4: FinTech Overlay — White-Label KYC for Banking-as-a-Service

**Institution Profile:** FinTech company providing banking-as-a-service infrastructure to neobanks and embedded finance partners. The FinTech's banking partner (a sponsor bank) requires standardized KYC across all FinTech-originated accounts.

**Problem:** Each neobank partner has different customer populations and risk profiles. A one-size-fits-all KYC configuration generates too many false positives for some partners and insufficient scrutiny for others. The sponsor bank needs consistent audit documentation across all partners.

**Framework Application:**

The FinTech deploys the framework as a multi-tenant service. Each partner bank configures their own risk tier thresholds and progressive verification triggers, but all share the same Audit Log format and Consent Registry schema — satisfying the sponsor bank's requirement for standardized examination documentation.

*Multi-tenant configuration:*

- Shared: Identity Verification and Sanctions Screening services (cost efficiency)
- Per-partner: Risk tier thresholds, progressive verification triggers, webhook endpoints
- Shared: Audit Log schema (sponsor bank can run examination queries across all tenants)
- Per-partner: Consent Registry (partner-specific consent language and purposes)

**Key Benefit:** Sponsor bank has a single audit interface across all FinTech partners. Regulatory examinations require one set of documentation rather than separate packages per partner.
