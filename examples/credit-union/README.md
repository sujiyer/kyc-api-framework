# Example: Credit Union Implementation

This example covers implementation for a regional credit union with the following profile:

- **Asset size:** $800M
- **Member base:** Employer-chartered, serving members of a defined geographic community
- **Products in scope:** Membership accounts, auto loans, HELOCs, personal loans
- **Current state:** In-branch KYC for loans; online KYC for membership accounts routes to 2-day manual review
- **Goal:** Reduce HELOC and auto loan time-to-KYC-complete from 4 days to same-day; maintain member experience quality

---

## Key Differences from Community Bank Pattern

Credit unions have two specific considerations this configuration accounts for:

**1. Membership eligibility gate**
Members must qualify for membership (field of membership) before KYC is triggered. The framework is invoked after membership eligibility is confirmed, not before.

**2. Secured product risk tier**
Auto loans and HELOCs require document verification as a baseline (not just for medium-risk applicants) because loan-type products carry higher inherent risk than deposit accounts.

---

## Configuration

### Risk Tier Thresholds (Credit Union)

```yaml
risk_tier_config:
  low_risk:
    conditions:
      - product_type: [CHECKING, SAVINGS]
      - member_status: [EXISTING_MEMBER]  # Existing members on deposit products
      - identity_confidence_threshold: 0.80
    outcome: AUTO_APPROVE_ELIGIBLE

  medium_risk:
    conditions:
      - product_type: [CHECKING, SAVINGS]
      - member_status: [NEW_MEMBER]
      - identity_confidence_threshold: 0.75
      - OR:
        - product_type: [LOAN_PERSONAL]
        - identity_confidence_threshold: 0.80
    outcome: DOCUMENT_VERIFY_REQUIRED

  high_risk:
    conditions:
      - product_type: [LOAN_SECURED, HELOC]  # All secured products require full KYC
      - OR:
        - identity_confidence_below: 0.75
        - any_sanctions_flag: true
    outcome: FULL_KYC_REQUIRED
```

Note: For secured loan products (HELOC, auto), Document Verification runs for ALL applicants regardless of risk tier — this is a credit union policy decision based on loan product risk, not an identity confidence gate.

---

## Workflow: HELOC Application

```
Step 1: Member initiates HELOC application (online or branch)
Step 2: Membership eligibility confirmed (pre-framework)
Step 3: KYC Framework initiated with product_type: LOAN_SECURED

Parallel execution:
  ├── Identity Verification Service
  ├── Document Processing Service (always required for LOAN_SECURED)
  └── Sanctions Screening Service

Step 4: Decision routing
  ├── All three PASS + confidence ≥ 0.80: Route to underwriting with complete KYC package
  ├── Identity or document confidence 0.65–0.79: Route to loan officer review (same-day SLA)
  └── Any sanctions flag OR confidence < 0.65: Route to BSA officer review

Step 5: Underwriting receives structured KYC package (not raw documents)
```

**Target:** KYC package delivered to underwriting within 30 minutes of application submission for 80%+ of HELOC applications.

---

## Member Communication Templates

Status webhook events should trigger member-facing communications. Suggested language:

**On initiation:**
> "We've received your application and are verifying your information. This typically takes less than 5 minutes. We'll notify you by email when complete."

**On auto-approval:**
> "Your identity has been verified. Your application has been sent to our lending team for review. You'll hear from us within [X] business days."

**On progressive verification trigger:**
> "We need a little more information to complete your verification. You have two options: [connect your existing bank account] or [upload a utility bill or other document]. This takes about 2 minutes."

**On manual review escalation:**
> "Your application is being reviewed by our team. We'll contact you within [SLA] business hours. If you have questions, call [number]."

---

## Integration with Core Banking System

The credit union's core banking system (e.g., Symitar, FiServ, DNA) needs to receive the KYC decision to proceed with account or loan setup.

Recommended integration point: webhook on `kyc.decision_made` event.

```
On event: kyc.decision_made
  If decision = APPROVED:
    POST to core banking API: {
      member_id: applicant.member_id,
      kyc_status: "VERIFIED",
      kyc_date: decision.decided_at,
      kyc_method: decision.decided_by,
      kyc_reference: kyc_id
    }
  If decision = PENDING_REVIEW:
    Update loan origination system status to "KYC_PENDING"
    Notify loan officer queue
```

---

## Examination Documentation for NCUA

For NCUA examinations, produce the following:

1. **Written KYC/BSA policy** (not provided by this framework — must be institution-authored)
2. **CIP implementation evidence:** Export from this framework showing identity verification date, method, vendor, and confidence score for any member account
3. **Risk-based approach evidence:** Show examiner the risk tier configuration. For credit unions, NCUA examiners will ask specifically about how member relationship (existing vs. new) affects your CIP approach
4. **Sanctions screening evidence:** Show that all members are screened against OFAC SDN at account opening and on an ongoing basis
5. **Progressive verification rationale:** Document why you configured alternative verification paths and the criteria for offering them

**NCUA-specific note:** NCUA examiners may ask about your field of membership verification separately from your CIP process. Membership eligibility and identity verification are distinct compliance requirements — make sure your documentation treats them separately.
