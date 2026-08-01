# Financial Literacy Integration

This document extends the KYC API Framework with patterns for delivering plain-language financial education at the moment of onboarding — turning the KYC process from a compliance gate into a financial literacy touchpoint.

---

## The Opportunity in the Onboarding Moment

The KYC onboarding moment is when a person is most engaged with a financial institution. They have made the decision to open an account or access a product. Their attention is focused. They are motivated.

Most institutions use this moment purely for compliance — collecting the information required for CIP and CDD. This framework adds a second layer: using the onboarding moment to deliver the plain-language explanations that help a first-time banking customer understand what they are signing up for and why each step exists.

This is not financial advice. It is financial context — the difference between "enter your Social Security Number" and "enter your Social Security Number — we use this to verify your identity and meet federal requirements. We do not share it without your consent."

---

## Contextual Explanation API

The Financial Literacy API provides plain-language explanations for KYC-related concepts, delivered in context — at the exact moment the applicant encounters them.

### Endpoint

```
GET /api/v1/explain/{concept}
  Query params:
    language: ISO 639-1 language code (default: en)
    reading_level: BASIC | STANDARD | DETAILED (default: STANDARD)
    product_type: CHECKING | SAVINGS | LOAN | BUSINESS
  
  Response:
  {
    "concept": "string",
    "explanation": "string — plain language",
    "why_we_ask": "string — institution-specific reason",
    "your_rights": "string — what the applicant controls",
    "learn_more_url": "string or null"
  }
```

### Concept Library

| Concept Key | What It Explains |
|---|---|
| `kyc` | What Know Your Customer means and why it is required |
| `ssn_collection` | Why we collect your Social Security Number |
| `identity_verification` | What identity verification is and how it works |
| `document_upload` | Why we need to see your ID document |
| `sanctions_screening` | What a sanctions check is in plain terms |
| `progressive_verification` | Why we are asking for additional information |
| `open_banking_consent` | What it means to connect your existing bank account |
| `manual_review` | What happens when an application goes to manual review |
| `adverse_action` | What your rights are if your application is not approved |
| `data_retention` | How long we keep your information and why |

### Example Response — SSN Collection

```
GET /api/v1/explain/ssn_collection?language=en&reading_level=BASIC

Response:
{
  "concept": "ssn_collection",
  "explanation": "Your Social Security Number is a unique number the US government uses to identify you. We use it to confirm you are who you say you are when you open an account.",
  "why_we_ask": "Federal law requires us to verify the identity of everyone who opens an account. Your Social Security Number is one of the ways we do this.",
  "your_rights": "We keep your Social Security Number secure and do not share it without your permission, except as required by law.",
  "learn_more_url": "https://[institution-domain]/help/ssn-and-your-account"
}
```

---

## Adverse Action Plain-Language Standards

Under ECOA and the Fair Credit Reporting Act, applicants who are denied or who receive less favourable terms have the right to know why. The adverse action explanation must be in plain language.

### Adverse Action Explanation API

```
GET /api/v1/kyc/{kyc_id}/adverse-action-explanation
  Response:
  {
    "outcome": "DENIED | ESCALATED | RESTRICTED",
    "reasons": [
      {
        "reason_code": "string",
        "plain_language_reason": "string",
        "is_disputable": "boolean",
        "dispute_path": "string or null"
      }
    ],
    "your_rights": "string — ECOA and FCRA rights in plain language",
    "next_steps": [
      {
        "action": "string",
        "description": "string",
        "url": "string or null"
      }
    ],
    "regulatory_notice": "string — required regulatory disclosure"
  }
```

### Plain Language Reason Examples

| Reason Code | Plain Language Version |
|---|---|
| `IDENTITY_NOT_VERIFIED` | "We were not able to confirm your identity using the information provided. You can try again with a different form of ID." |
| `ADDRESS_MISMATCH` | "The address you provided does not match our records. You can update your address or provide a document showing your current address." |
| `SANCTIONS_MATCH` | "Your information matched a name on a federal watchlist. This sometimes happens because of a name that is similar to someone else on the list. You can contact us to provide additional information." |
| `INSUFFICIENT_DOCUMENTATION` | "We need one more piece of information to verify your identity. You can upload a utility bill, bank statement, or other document showing your name and address." |

---

## Financial Concept Glossary Integration

For first-time banking customers, terms that are routine to banking professionals are completely unfamiliar. The KYC onboarding flow should surface definitions at the point of encounter.

### Implementation Pattern

When a financial term appears in the onboarding UI, render it as a tappable tooltip that calls the Financial Literacy API:

```
User sees: "We need to complete your CDD verification."
User taps "CDD verification"
System calls: GET /api/v1/explain/cdd
System shows: "CDD stands for Customer Due Diligence. It is a process banks 
               use to understand who their customers are and make sure their 
               accounts are used safely. It is required by federal law."
```

This pattern requires no separate screens or navigation — the explanation appears inline at the moment of need and dismisses when the user continues.

### Priority Terms for Glossary

These terms consistently cause confusion for first-time banking customers and should be prioritised for glossary integration:

- Know Your Customer (KYC)
- Identity verification
- Beneficial owner
- Sanctions screening
- OFAC
- Adverse action
- Credit bureau
- Hard inquiry vs soft inquiry
- Annual Percentage Rate (APR)
- FDIC insurance

---

## Multi-Language Implementation

The Financial Literacy API serves all content in the language specified in the request. Institutions must supply translations for their customer-facing content; the API provides the interface and the delivery mechanism.

**Priority languages for US financial inclusion:**
- Spanish (es)
- Chinese Simplified (zh-Hans)
- Vietnamese (vi)
- Korean (ko)
- Tagalog (tl)
- Arabic (ar)
- Haitian Creole (ht)

The language used in each explanation is logged in the Audit Trail — institutions can track which languages their customers are using and prioritise translation resources accordingly.
