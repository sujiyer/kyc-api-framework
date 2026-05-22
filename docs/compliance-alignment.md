# Compliance Alignment

This document maps the KYC API Framework's design decisions to the specific regulatory and policy requirements that financial institutions must satisfy.

---

## CFPB Section 1033 — Personal Financial Data Rights

**Rule:** CFPB Final Rule on Personal Financial Data Rights, published October 2024. Requires covered financial institutions to provide consumers with standardized, machine-readable access to their financial data through secure APIs upon request.

**Relevance to KYC:** Section 1033 creates a new consumer right to share their own financial data with third parties. Within the KYC context, this right can be used constructively: a consumer applying to a new institution can permission access to their existing financial account data to supplement or replace traditional identity documentation. This is particularly valuable for applicants from underserved populations who may have limited credit file history but do have existing banking relationships.

**Framework Implementation:**

| 1033 Requirement | Framework Implementation |
|---|---|
| Consumer authorization before data access | Consent Registry records explicit consent before any 1033 data call |
| Machine-readable data format | API responses use standardized JSON schema |
| Consumer right to revoke access | Consent revocation endpoint: DELETE `/api/v1/consent/{consent_id}` |
| Data minimization | KYC workflow requests only fields necessary for verification, not full transaction history |
| Third-party authorization records | Consent Registry maintains auditable record of all consumer permissions |

**Key limitation:** Section 1033 data flows are available only for applicants who consent and who have accounts at covered institutions. This is a supplementary verification path, not a replacement for standard KYC.

---

## BSA/AML — Bank Secrecy Act and Anti-Money Laundering Requirements

**Relevant Requirements:**
- Customer Identification Program (CIP): 31 CFR 1020.220 — requires collection and verification of customer identity at account opening
- Customer Due Diligence (CDD): FinCEN May 2018 Final Rule — requires beneficial ownership identification for legal entities
- Recordkeeping: 5-year retention of identity verification records

**Framework Implementation:**

| BSA/AML Requirement | Framework Implementation |
|---|---|
| Collect name, DOB, address, ID number | Applicant schema captures all required CIP fields |
| Verify identity through documentary or non-documentary methods | Identity and Document Verification Services satisfy this requirement |
| OFAC SDN screening | Sanctions Screening Service covers OFAC SDN as a baseline |
| FinCEN 314(a) compliance | Sanctions Screening Service includes 314(a) list |
| Audit-ready records | Audit Log captures all verification steps, timestamps, results |
| 5-year record retention | Data Layer retention policy section of implementation guide |

**Important Note:** This framework provides the technical architecture for CIP/CDD compliance. It does not replace your institution's written CIP policy, BSA officer oversight, or FinCEN examination preparation. Your BSA officer must review framework configuration to ensure alignment with your institution's specific risk-based approach.

---

## Executive Order 14110 — Safe, Secure, and Trustworthy AI

**Signed:** October 30, 2023

**Relevance:** E.O. 14110 directs federal agencies to develop standards and guidance for AI systems used in consequential decisions. For financial institutions, this applies to any AI or machine learning component that assists in KYC decisions — document authenticity scoring, fraud risk scoring, identity confidence scoring.

**Framework Implementation:**

Where AI assists KYC decisions, this framework applies the following governance controls aligned with E.O. 14110 and NIST AI RMF 1.0:

| Governance Control | Implementation |
|---|---|
| Transparency | Decision rationale field in every AI-assisted outcome; human-readable explanation required |
| Human oversight | No AI-only denial path; all AI-flagged denials route to human review queue |
| Bias monitoring | Demographic parity monitoring across applicant risk tier assignments; quarterly review recommended |
| Audit trail | All AI model decisions logged with model version, confidence score, input feature set |
| Right to explanation | Denial reason communicated to applicant in plain language, regardless of AI involvement |

**Compliance Note:** Institutions using AI-assisted components in production must document their AI risk management approach under NIST AI RMF 1.0 (Govern, Map, Measure, Manage functions). This framework's governance controls are designed to satisfy those requirements.

---

## NIST AI Risk Management Framework 1.0

**Published:** January 2023

**Framework Functions and Implementation Mapping:**

**GOVERN**
> Establish policies, accountability, and organizational practices for AI risk management.

Implementation: The Risk Scoring Module's threshold configuration and escalation rules should be governed by a documented policy approved by senior management. The Audit Log provides accountability for all AI-assisted decisions.

**MAP**
> Categorize AI risks and understand context, including impacts on affected populations.

Implementation: KYC AI components that affect account access decisions have direct impacts on financially underserved populations. Institutions should document this context formally. The progressive verification pathway is a risk mitigation specifically designed to reduce disparate impact on thin-file applicants.

**MEASURE**
> Analyze and assess AI risks through quantitative and qualitative methods.

Implementation: The monitoring metrics in Phase 4 of the Implementation Guide (auto-approval rate by demographic, false positive rate, model confidence drift) constitute the measurement program. These should be reviewed at minimum quarterly.

**MANAGE**
> Prioritize and address AI risks through response plans and ongoing monitoring.

Implementation: The human review escalation path is the primary management response to AI risk. Any AI-assisted decision with confidence score below the institution's defined threshold must be routed to human review rather than auto-decided.

---

## Fair Lending Considerations

KYC architecture can create fair lending exposure if:
- Risk tier assignment correlates with protected class characteristics
- Alternative verification paths are inconsistently offered
- Manual review queues have disparate processing times by applicant demographics

**Framework mitigations:**
- Risk tier criteria are documented and consistently applied
- Progressive verification paths are offered based on objective criteria (confidence score below threshold), not subjective reviewer judgment
- Manual review queue metrics include demographic parity monitoring as a standard metric

Institutions should have fair lending counsel review the risk tier configuration and progressive verification trigger criteria before go-live.

---

## Summary Compliance Matrix

| Regulation / Standard | Directly Addressed | Partially Addressed | Out of Scope |
|---|---|---|---|
| CFPB Section 1033 | Consumer consent management, data portability | Covered institution connectivity | 1033 provider certification |
| BSA/AML CIP requirements | Data collection, verification, recordkeeping | Risk-based CIP policy | Suspicious Activity Reporting |
| OFAC SDN screening | Real-time sanctions check | — | OFAC penalty response |
| FinCEN 314(a) | Screening integration | — | Law enforcement response |
| Executive Order 14110 | AI governance controls, human oversight | — | Agency-specific AI policy |
| NIST AI RMF 1.0 | Govern, Map, Measure, Manage | — | Enterprise AI governance program |
| Fair Lending (ECOA, FHA) | Consistent criteria, bias monitoring | — | Legal compliance program |
