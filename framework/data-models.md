# Data Models

This document defines the core data entities in the KYC API Framework and their relationships.

---

## Core Entities

### KYCApplication

The central entity representing a single KYC verification workflow.

| Field | Type | Required | Description |
|---|---|---|---|
| kyc_id | UUID | Yes | System-generated unique identifier |
| institution_id | UUID | Yes | Identifies the financial institution |
| applicant_id | UUID | Yes | Links to the Applicant entity |
| product_type | Enum | Yes | CHECKING, SAVINGS, LOAN_PERSONAL, LOAN_SECURED, LOAN_BUSINESS, BUSINESS_ACCOUNT |
| channel | Enum | Yes | WEB, MOBILE, BRANCH, API |
| status | Enum | Yes | See Status Lifecycle below |
| risk_tier | Enum | Yes | LOW, MEDIUM, HIGH |
| overall_confidence_score | Float | No | Aggregate confidence 0.0–1.0, computed after all steps complete |
| institution_reference_id | String | No | Institution's own reference number |
| created_at | Timestamp | Yes | ISO8601 |
| updated_at | Timestamp | Yes | ISO8601 |
| expires_at | Timestamp | Yes | When incomplete application expires (default: 72 hours) |

**Status Lifecycle:**
```
INITIATED → IN_PROGRESS → APPROVED
                       → DENIED
                       → PENDING_REVIEW → APPROVED
                                       → DENIED
                       → INCOMPLETE (customer did not complete within expiry)
                       → EXPIRED
```

---

### Applicant

Identity information collected during onboarding. Stored separately from KYCApplication to support multiple applications per individual.

| Field | Type | Required | Description |
|---|---|---|---|
| applicant_id | UUID | Yes | System-generated |
| first_name | String | Yes | — |
| last_name | String | Yes | — |
| date_of_birth | Date | Yes | YYYY-MM-DD |
| ssn_hash | String | Conditional | SHA-256 hash of SSN; never store plaintext |
| ssn_last4 | String | Conditional | Last 4 digits of SSN, used for simplified products |
| address_street | String | Yes | — |
| address_street2 | String | No | — |
| address_city | String | Yes | — |
| address_state | String | Yes | 2-letter state code |
| address_zip | String | Yes | — |
| address_country | String | Yes | ISO 3166-1 alpha-2; default US |
| email | String | Yes | — |
| phone | String | No | E.164 format |
| ip_address | String | No | Captured at application initiation |
| created_at | Timestamp | Yes | — |

**Data Security Note:** SSN must be stored as a one-way hash (SHA-256 with institution-specific salt at minimum; bcrypt recommended). The plaintext SSN is used for verification calls to identity services and then discarded. Never log or store plaintext SSN.

---

### VerificationStep

Records the result of each individual verification service call.

| Field | Type | Required | Description |
|---|---|---|---|
| step_id | UUID | Yes | System-generated |
| kyc_id | UUID | Yes | Parent KYCApplication |
| step_type | Enum | Yes | IDENTITY, DOCUMENT, SANCTIONS, PROGRESSIVE_OPEN_BANKING, PROGRESSIVE_UTILITY, PROGRESSIVE_EMPLOYER |
| status | Enum | Yes | PENDING, PASS, FAIL, TIMEOUT, NOT_REQUIRED, SKIPPED |
| confidence_score | Float | No | 0.0–1.0; returned by service |
| vendor_name | String | No | Name of integration vendor used |
| vendor_reference_id | String | No | Vendor's own transaction ID |
| raw_response_hash | String | No | SHA-256 hash of vendor response; for integrity verification |
| initiated_at | Timestamp | Yes | — |
| completed_at | Timestamp | No | Null if not yet complete |
| timeout_at | Timestamp | Yes | When this step will be timed out |

---

### SanctionsMatch

Stores details of any watchlist matches returned by the sanctions screening service.

| Field | Type | Required | Description |
|---|---|---|---|
| match_id | UUID | Yes | System-generated |
| step_id | UUID | Yes | Parent VerificationStep |
| kyc_id | UUID | Yes | Parent KYCApplication |
| list_name | String | Yes | E.g., OFAC_SDN, FINCEN_314A, EU_CONSOLIDATED |
| matched_name | String | Yes | Name on the watchlist that matched |
| match_score | Float | Yes | 0.0–1.0; service-returned match confidence |
| match_type | Enum | Yes | EXACT, PARTIAL, PHONETIC, FUZZY |
| resolution | Enum | No | CONFIRMED_MATCH, FALSE_POSITIVE, PENDING_REVIEW |
| resolved_by | String | No | Examiner ID who resolved |
| resolved_at | Timestamp | No | — |

---

### ConsentRecord

Records each instance of customer consent for data use. Required for CFPB Section 1033 compliance.

| Field | Type | Required | Description |
|---|---|---|---|
| consent_id | UUID | Yes | System-generated |
| kyc_id | UUID | Yes | Parent KYCApplication |
| applicant_id | UUID | Yes | — |
| data_source | Enum | Yes | OPEN_BANKING, UTILITY, EMPLOYER_PAYROLL, CREDIT_BUREAU, STANDARD_KYC |
| purpose | Enum | Yes | KYC_IDENTITY_VERIFICATION, KYC_IDENTITY_SUPPLEMENT |
| consent_given | Boolean | Yes | Must be true to proceed |
| consent_method | Enum | Yes | DIGITAL_CHECKBOX, ESIGNATURE, VERBAL_BRANCH |
| ip_address | String | Conditional | Required for digital methods |
| agent_id | String | Conditional | Required for VERBAL_BRANCH |
| consent_text_version | String | Yes | Version identifier for the consent language shown |
| status | Enum | Yes | ACTIVE, REVOKED, EXPIRED |
| created_at | Timestamp | Yes | — |
| expires_at | Timestamp | Yes | Default 90 days |
| revoked_at | Timestamp | No | Populated on revocation |

---

### AuditEvent

Immutable record of every action taken in the KYC system.

| Field | Type | Required | Description |
|---|---|---|---|
| event_id | UUID | Yes | System-generated |
| kyc_id | UUID | Yes | Parent KYCApplication |
| event_type | Enum | Yes | INITIATED, STEP_COMPLETED, STEP_FAILED, STEP_TIMEOUT, DECISION_MADE, ESCALATED, REVIEWED, CONSENT_RECORDED, CONSENT_REVOKED |
| timestamp | Timestamp | Yes | Immutable; set at event creation |
| actor | String | Yes | SYSTEM or examiner ID |
| step_type | Enum | No | Which verification step this event relates to |
| result | Enum | No | PASS, FAIL, TIMEOUT, ESCALATED, APPROVED, DENIED |
| confidence_score | Float | No | If applicable |
| decision_rationale | String | No | Human-readable explanation of decision |
| model_version | String | No | AI model version if AI-assisted; required for NIST AI RMF |
| input_feature_set | String | No | Hashed feature set used by AI model; required for NIST AI RMF |

**Immutability requirement:** The AuditEvent table must not support UPDATE or DELETE operations. Implement at the database level (append-only table or write-once permissions). Any corrections must be recorded as new events with a `CORRECTION` event type referencing the original event_id.

---

## Entity Relationships

```
Institution (1)
  └── KYCApplication (many)
        ├── Applicant (1)
        ├── VerificationStep (1–4)
        │     └── SanctionsMatch (0–many)
        ├── ConsentRecord (1–many)
        └── AuditEvent (many)
```

---

## Data Retention Policy

| Entity | Minimum Retention | Reference |
|---|---|---|
| KYCApplication | 5 years | BSA record retention requirements |
| Applicant | 5 years | BSA CIP requirements |
| VerificationStep | 5 years | BSA examination readiness |
| SanctionsMatch | 5 years | OFAC recordkeeping |
| ConsentRecord | 5 years or lifetime of relationship | CFPB guidance |
| AuditEvent | 5 years | BSA recordkeeping; NIST AI RMF |

All retention periods measured from date of account closure or application denial, whichever is later. Consult your BSA officer for institution-specific retention policy.
