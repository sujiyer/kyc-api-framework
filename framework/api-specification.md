# API Specification

This document defines the complete REST API contract for the KYC API Framework orchestration layer.

All requests require an `Authorization` header using Bearer token (OAuth 2.0) or API key depending on deployment configuration.

Base URL: `https://{your-domain}/api/v1`

---

## KYC Workflow Endpoints

### POST /kyc/initiate
Initiates a new KYC verification workflow.

**Request**
```json
{
  "applicant": {
    "first_name": "string (required)",
    "last_name": "string (required)",
    "date_of_birth": "YYYY-MM-DD (required)",
    "ssn": "string (required for full SSN products) OR",
    "ssn_last4": "string (required for simplified products)",
    "address": {
      "street": "string (required)",
      "street2": "string (optional)",
      "city": "string (required)",
      "state": "2-letter state code (required)",
      "zip": "string (required)",
      "country": "ISO 3166-1 alpha-2 (default: US)"
    },
    "email": "string (required)",
    "phone": "string E.164 format (optional)",
    "ip_address": "string (recommended for fraud scoring)"
  },
  "product_type": "CHECKING | SAVINGS | LOAN_PERSONAL | LOAN_SECURED | LOAN_BUSINESS | BUSINESS_ACCOUNT",
  "channel": "WEB | MOBILE | BRANCH | API",
  "consent": {
    "terms_accepted": "boolean (required: true)",
    "consent_timestamp": "ISO8601 datetime (required)",
    "electronic_consent": "boolean (required: true for digital channels)",
    "ip_address": "string (required for digital channels)"
  },
  "institution_reference_id": "string (your internal application ID, optional)"
}
```

**Response 201 — Created**
```json
{
  "kyc_id": "uuid",
  "status": "IN_PROGRESS",
  "risk_tier": "LOW | MEDIUM | HIGH",
  "verification_steps_initiated": ["IDENTITY", "DOCUMENT", "SANCTIONS"],
  "estimated_completion_seconds": 90,
  "webhook_registered": "boolean",
  "created_at": "ISO8601 datetime"
}
```

**Response 400 — Bad Request**
```json
{
  "error": "VALIDATION_ERROR",
  "message": "string",
  "missing_fields": ["field_name"]
}
```

---

### GET /kyc/{kyc_id}/status
Retrieves the current status of a KYC workflow.

**Response 200 — OK**
```json
{
  "kyc_id": "uuid",
  "status": "IN_PROGRESS | APPROVED | DENIED | PENDING_REVIEW | INCOMPLETE | EXPIRED",
  "risk_tier": "LOW | MEDIUM | HIGH",
  "verification_steps": {
    "identity": {
      "status": "PASS | FAIL | PENDING | TIMEOUT",
      "confidence_score": "float 0.0–1.0",
      "completed_at": "ISO8601 datetime or null"
    },
    "document": {
      "status": "PASS | FAIL | PENDING | NOT_REQUIRED | TIMEOUT",
      "confidence_score": "float 0.0–1.0 or null",
      "completed_at": "ISO8601 datetime or null"
    },
    "sanctions": {
      "status": "CLEARED | FLAGGED | PENDING | TIMEOUT",
      "match_count": "integer",
      "completed_at": "ISO8601 datetime or null"
    }
  },
  "progressive_verification_available": "boolean",
  "progressive_verification_methods": ["OPEN_BANKING", "UTILITY", "EMPLOYER"] ,
  "decision": {
    "outcome": "APPROVED | DENIED | PENDING_REVIEW | null",
    "decided_at": "ISO8601 datetime or null",
    "decided_by": "AUTO | HUMAN | null",
    "denial_reason_code": "string or null",
    "denial_reason_text": "string (plain language) or null"
  },
  "overall_confidence_score": "float 0.0–1.0",
  "updated_at": "ISO8601 datetime"
}
```

---

### POST /kyc/{kyc_id}/progressive-verify
Initiates an alternative verification path for an applicant who could not complete standard verification.

**Request**
```json
{
  "method": "OPEN_BANKING | UTILITY | EMPLOYER_PAYROLL",
  "consent": {
    "data_source_consent": "boolean (required: true)",
    "consent_timestamp": "ISO8601 datetime",
    "specific_purpose": "KYC_IDENTITY_SUPPLEMENT"
  },
  "open_banking_params": {
    "institution_id": "string (FDX institution identifier)",
    "account_types": ["CHECKING", "SAVINGS"]
  }
}
```

**Response 202 — Accepted**
```json
{
  "kyc_id": "uuid",
  "progressive_verification_id": "uuid",
  "method": "OPEN_BANKING",
  "status": "INITIATED",
  "redirect_url": "string (for open banking consent flow, if applicable)",
  "estimated_completion_seconds": 120
}
```

---

### POST /kyc/{kyc_id}/document
Submits a document for verification (for MEDIUM and HIGH risk workflows).

**Request**
```json
{
  "document_type": "DRIVERS_LICENSE | PASSPORT | STATE_ID | MILITARY_ID",
  "document_side": "FRONT | BACK | SINGLE",
  "image_data": "base64-encoded image string",
  "image_format": "JPEG | PNG | PDF",
  "capture_method": "CAMERA | UPLOAD | SCANNER"
}
```

**Response 202 — Accepted**
```json
{
  "document_submission_id": "uuid",
  "kyc_id": "uuid",
  "processing_mode": "REALTIME | ASYNC",
  "estimated_completion_seconds": 15,
  "webhook_will_notify": "boolean"
}
```

---

## Webhook Events

Register a webhook URL in your institution's configuration to receive real-time status updates. The framework posts to your endpoint for each of the following events.

### kyc.step_completed
```json
{
  "event_type": "kyc.step_completed",
  "kyc_id": "uuid",
  "step": "IDENTITY | DOCUMENT | SANCTIONS",
  "result": "PASS | FAIL | TIMEOUT",
  "confidence_score": "float",
  "timestamp": "ISO8601 datetime"
}
```

### kyc.decision_made
```json
{
  "event_type": "kyc.decision_made",
  "kyc_id": "uuid",
  "decision": "APPROVED | DENIED | PENDING_REVIEW",
  "decided_by": "AUTO | HUMAN",
  "denial_reason_code": "string or null",
  "institution_reference_id": "string or null",
  "timestamp": "ISO8601 datetime"
}
```

### kyc.escalated
```json
{
  "event_type": "kyc.escalated",
  "kyc_id": "uuid",
  "escalation_reason": "LOW_CONFIDENCE | SANCTIONS_FLAG | TIMEOUT | MANUAL_REQUEST",
  "queue_position": "integer",
  "sla_due_by": "ISO8601 datetime",
  "timestamp": "ISO8601 datetime"
}
```

---

## Consent Management Endpoints

### POST /consent/record
Records customer consent for a specific data use.

**Request**
```json
{
  "kyc_id": "uuid",
  "applicant_id": "uuid",
  "data_source": "OPEN_BANKING | UTILITY | EMPLOYER_PAYROLL | CREDIT_BUREAU | STANDARD_KYC",
  "purpose": "KYC_IDENTITY_VERIFICATION | KYC_SUPPLEMENT",
  "consent_given": "boolean",
  "consent_timestamp": "ISO8601 datetime",
  "consent_method": "DIGITAL_CHECKBOX | ESIGNATURE | VERBAL_BRANCH",
  "ip_address": "string (for digital methods)",
  "agent_id": "string (for branch/verbal methods)"
}
```

**Response 201 — Created**
```json
{
  "consent_id": "uuid",
  "kyc_id": "uuid",
  "data_source": "OPEN_BANKING",
  "status": "ACTIVE",
  "created_at": "ISO8601 datetime",
  "expires_at": "ISO8601 datetime (90 days default)"
}
```

### DELETE /consent/{consent_id}
Revokes a previously granted consent. Required for CFPB Section 1033 compliance.

**Response 200 — OK**
```json
{
  "consent_id": "uuid",
  "status": "REVOKED",
  "revoked_at": "ISO8601 datetime",
  "data_deletion_scheduled": "boolean"
}
```

---

## Audit Endpoints

### GET /kyc/{kyc_id}/audit-log
Returns the complete audit trail for a KYC workflow. Requires elevated permissions.

**Response 200 — OK**
```json
{
  "kyc_id": "uuid",
  "events": [
    {
      "event_id": "uuid",
      "event_type": "INITIATED | STEP_COMPLETED | DECISION_MADE | ESCALATED | REVIEWED",
      "timestamp": "ISO8601 datetime",
      "actor": "SYSTEM | {examiner_id}",
      "step": "IDENTITY | DOCUMENT | SANCTIONS | WORKFLOW",
      "result": "PASS | FAIL | TIMEOUT | ESCALATED | APPROVED | DENIED",
      "confidence_score": "float or null",
      "vendor_reference_id": "string or null",
      "decision_rationale": "string"
    }
  ],
  "total_events": "integer",
  "workflow_duration_seconds": "integer"
}
```

---

## Error Codes

| Code | HTTP Status | Description |
|---|---|---|
| VALIDATION_ERROR | 400 | Required field missing or malformed |
| APPLICANT_NOT_FOUND | 404 | kyc_id not found |
| DUPLICATE_APPLICATION | 409 | Matching application already in progress |
| SERVICE_TIMEOUT | 503 | Upstream verification service did not respond |
| CONSENT_REQUIRED | 403 | Required consent not recorded before data access |
| INSUFFICIENT_PERMISSIONS | 403 | Caller lacks permission for requested operation |
| RATE_LIMIT_EXCEEDED | 429 | Too many requests; implement exponential backoff |
