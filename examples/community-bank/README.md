# Example: Community Bank Implementation

This example walks through a complete KYC API Framework implementation for a community bank with the following profile:

- **Asset size:** $3.5B
- **Customer base:** Mixed urban/suburban, including first-generation banking customers, recent immigrants, and thin-file applicants
- **Products in scope:** Consumer checking accounts, savings accounts, personal loans
- **Current state:** Branch-only KYC; online account opening exists but routes 100% of applications to manual review
- **Goal:** Reduce manual review to <20% of applications; reduce average completion time from 3 business days to <5 minutes for auto-approved applications

---

## Configuration

### Risk Tier Thresholds

```yaml
risk_tier_config:
  low_risk:
    conditions:
      - product_type: [CHECKING, SAVINGS]
      - channel: [WEB, MOBILE, BRANCH]
      - identity_confidence_threshold: 0.85
      - no_sanctions_flags: true
    outcome: AUTO_APPROVE_ELIGIBLE

  medium_risk:
    conditions:
      - product_type: [CHECKING, SAVINGS, LOAN_PERSONAL]
      - identity_confidence_range: [0.65, 0.84]
      - no_sanctions_flags: true
    outcome: PROGRESSIVE_VERIFY_ELIGIBLE

  high_risk:
    conditions:
      - product_type: [LOAN_SECURED, LOAN_PERSONAL]
        loan_amount_above: 25000
      - any_sanctions_flag: true
      - identity_confidence_below: 0.65
        progressive_verify_failed: true
    outcome: MANUAL_REVIEW_REQUIRED
```

### Progressive Verification Configuration

```yaml
progressive_verification:
  trigger_threshold: 0.65
  eligible_risk_tiers: [LOW, MEDIUM]
  available_methods:
    - OPEN_BANKING:
        enabled: true
        min_account_age_days: 90
        data_requested: [account_holder_name, address, account_status]
    - UTILITY:
        enabled: true
        accepted_providers: [electric, gas, water, internet]
        recency_required_days: 90
    - EMPLOYER_PAYROLL:
        enabled: false  # Enable when payroll API integration complete
  max_attempts: 2
  expiry_hours: 48
```

### Vendor Integrations

```yaml
vendors:
  identity_verification:
    provider: "[YOUR_PROVIDER]"
    endpoint: "https://api.[provider].com/v2/verify"
    timeout_seconds: 10
    confidence_field: "result.score"

  document_processing:
    provider: "[YOUR_PROVIDER]"
    endpoint: "https://api.[provider].com/verify/id"
    timeout_seconds: 15
    processing_mode: REALTIME  # Switch to ASYNC for MEDIUM/HIGH risk cost savings
    confidence_field: "authenticity.score"

  sanctions_screening:
    provider: "[YOUR_PROVIDER]"
    endpoint: "https://api.[provider].com/screen"
    timeout_seconds: 8
    lists: [OFAC_SDN, FINCEN_314A]
    match_threshold: 0.85  # Scores above this treated as matches
```

---

## Workflow Sequence Diagram

```
Customer          Onboarding UI     KYC Engine      Services
   │                   │                │                │
   │  Fill form        │                │                │
   │──────────────────>│                │                │
   │                   │  POST /initiate│                │
   │                   │───────────────>│                │
   │                   │                │  [parallel]    │
   │                   │                │──Identity─────>│
   │                   │                │──Sanctions────>│
   │                   │                │                │
   │                   │   kyc_id       │                │
   │                   │<───────────────│                │
   │  "Verifying..."   │                │  Results back  │
   │<──────────────────│                │<───────────────│
   │                   │                │                │
   │                   │  [if score ≥ 0.85]              │
   │                   │  webhook: APPROVED              │
   │                   │<───────────────│                │
   │  "Account open!"  │                │                │
   │<──────────────────│                │                │
   │                   │                │                │
   │                   │  [if 0.65–0.84]│                │
   │                   │  webhook: PROGRESSIVE_AVAILABLE │
   │                   │<───────────────│                │
   │  "Connect bank?"  │                │                │
   │<──────────────────│                │                │
```

---

## Metrics to Track

After go-live, review these metrics weekly for the first 90 days:

| Metric | Target | Action if Missed |
|---|---|---|
| Auto-approval rate (LOW risk) | ≥ 70% | Review identity vendor confidence calibration |
| Progressive verify completion rate | ≥ 50% of offered | Review UX of progressive verify flow |
| Manual review rate (all applications) | ≤ 20% | Review risk tier thresholds |
| Avg completion time (auto-approve) | ≤ 3 minutes | Review vendor latency; switch to parallel calls |
| Avg completion time (manual review) | ≤ 4 business hours | Review queue staffing |
| Abandonment at document upload | ≤ 15% | Review document upload UX |
| False positive rate (sanctions) | ≤ 3% | Raise match threshold; review name normalization |

---

## BSA Examination Readiness

For your next BSA examination, you can produce the following from this framework:

1. **CIP documentation:** Export Applicant records for any account showing identity verification date, method, and result
2. **Sanctions screening records:** Export VerificationStep and SanctionsMatch records for any account
3. **Decision audit trail:** Export AuditEvent log showing every action taken on any application
4. **Risk-based approach documentation:** Show examiner the risk tier configuration and explain the rationale

Keep the risk tier configuration under version control. Examiners may ask when thresholds were changed and why.
