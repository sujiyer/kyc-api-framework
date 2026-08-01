# Mobile-First and Accessibility Design for KYC Onboarding

This document extends the KYC API Framework with mobile-first design patterns and accessibility standards specifically for underserved populations — the customers most likely to be lost at the onboarding step.

---

## Why Mobile-First Matters for Financial Inclusion

Among unbanked and underbanked Americans, smartphone ownership significantly outpaces computer ownership. A person without a bank account is far more likely to attempt account opening on a phone than on a desktop. If the KYC onboarding experience is not genuinely designed for mobile — not just responsive, but mobile-first — the institutions that most need to serve this population will continue to lose them at the door.

Mobile-first KYC design is not a UX nicety. It is a financial inclusion requirement.

---

## Mobile-First Design Principles for KYC

### One Task Per Screen
Each step in the KYC flow occupies a single screen with a single action. Never combine identity entry, document upload, and consent on one screen. The cognitive load of completing multiple tasks simultaneously on a small screen is a primary cause of abandonment.

**Not this:**
Screen shows: personal information form (8 fields) + document upload + consent checkbox + submit button

**This:**
Screen 1: Enter your name and date of birth
Screen 2: Enter your address
Screen 3: Upload the front of your ID
Screen 4: Review and confirm

### Progress Visibility
The applicant always knows where they are in the process and how much remains. A progress indicator — "Step 3 of 5" — reduces abandonment by eliminating uncertainty about how long the process takes.

### Async-Friendly Design
Mobile users are interrupted. The KYC session must survive:
- Phone calls received mid-application
- App going to background and returning
- Loss of connectivity followed by reconnection
- Session resume from a different device

All partial application data is saved automatically. The applicant should never have to restart from the beginning because of an interruption.

### Camera-First Document Capture
For document capture on mobile, native camera integration outperforms file upload. Do not ask the user to upload a file — ask them to take a photo. Provide a camera overlay showing the correct framing for the document type.

**API guidance for mobile document capture:**
```
POST /api/v1/kyc/{kyc_id}/document/capture
  Request: {
    "capture_method": "CAMERA",
    "document_type": "DRIVERS_LICENSE",
    "image_stream": "base64",
    "device_type": "MOBILE"
  }
  Response: {
    "capture_quality": "ACCEPTABLE | RETRY",
    "retry_reason": "BLUR | LIGHTING | FRAMING | null",
    "retry_guidance": "Plain language instruction for retake"
  }
```

The response includes real-time quality feedback — if the image is blurry or poorly lit, tell the user immediately with plain-language guidance rather than failing the document check silently.

---

## Accessibility Standards

### Plain Language at Every Step

Every instruction, error message, and status update uses plain language. Target a reading level accessible to a general audience.

**Not acceptable:** "Your identity verification request could not be processed due to insufficient documentation confidence levels."

**Acceptable:** "We couldn't verify your ID from the photo. Please try again in better lighting or use a different ID."

Plain language checklist for each screen:
- [ ] No financial or technical jargon
- [ ] Sentences under 20 words where possible
- [ ] Active voice — "Upload your ID" not "Your ID should be uploaded"
- [ ] One idea per sentence

### Screen Reader Compatibility
All KYC UI components must be screen reader compatible:
- All images have descriptive alt text
- Form fields have clear labels associated programmatically
- Error messages are announced by screen readers when they appear
- Status updates are announced when they change

### Low-Bandwidth Compatibility
In underserved geographic areas, mobile bandwidth may be limited:
- Compress all images before transmission where possible
- Document upload accepts images up to 10MB but compresses them client-side before sending
- API timeout windows are extended for mobile (30 seconds per service call vs 10 seconds desktop default)
- Partial uploads are resumable — if connection drops during document upload, the upload can resume

### Multi-Language Support
The KYC API supports multi-language error messages and consumer explanations through an `Accept-Language` header:

```
GET /api/v1/kyc/{kyc_id}/status
Headers: Accept-Language: es  (Spanish)
         Accept-Language: zh  (Chinese)
         Accept-Language: vi  (Vietnamese)

Response includes:
{
  "consumer_explanation": "[Plain language in requested language]",
  "retry_guidance": "[Retry instructions in requested language]"
}
```

Supported languages are institution-configurable. The framework provides the interface; institutions supply the translation content.

---

## Mobile Abandonment Prevention

Track these mobile-specific metrics to identify abandonment causes:

| Metric | Flag Threshold | Common Cause |
|---|---|---|
| Document capture retry rate | > 30% | Poor camera guidance or lighting instructions |
| Step abandonment by screen | > 20% on any single screen | That screen has a UX or content problem |
| Session resume rate | < 40% | Session save is not working or not communicated |
| Time on document capture screen | > 3 minutes average | Camera instructions are unclear |
| Mobile vs desktop pass rate gap | > 15% | Mobile experience has unaddressed friction |

A high retry rate on document capture almost always means the framing overlay or lighting guidance is insufficient. Fix the guidance before assuming the document verification model has a problem.
