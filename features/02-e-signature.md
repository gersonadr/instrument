# Feature Research: E-Signature
> Source: ignitionapp.com — official product, help center (support.ignitionapp.com), learning center, and open-source ecosystem research.
> Purpose: Reference document for a planner agent writing software development epics.

---

## Table of Contents

1. [Feature Overview](#1-feature-overview)
2. [User Personas & Their Signing Roles](#2-user-personas--their-signing-roles)
3. [How Ignition Implements E-Signature](#3-how-ignition-implements-e-signature)
   - 3.1 [Signature Method: Typed Name](#31-signature-method-typed-name)
   - 3.2 [Data Captured at Signing](#32-data-captured-at-signing)
   - 3.3 [Audit Trail](#33-audit-trail)
   - 3.4 [Signed PDF Generation](#34-signed-pdf-generation)
4. [Multi-Signer Workflow](#4-multi-signer-workflow)
   - 4.1 [Roles: Primary vs. Secondary Signatories](#41-roles-primary-vs-secondary-signatories)
   - 4.2 [Sequential Signing Flow](#42-sequential-signing-flow)
   - 4.3 [Plan Limits on Signers](#43-plan-limits-on-signers)
   - 4.4 [Proposal Status During Multi-Sign](#44-proposal-status-during-multi-sign)
   - 4.5 [Signer Links & Reminder Logic](#45-signer-links--reminder-logic)
5. [Firm-Side Signature in Engagement Letters](#5-firm-side-signature-in-engagement-letters)
6. [Additional Document Signatures](#6-additional-document-signatures)
7. [Accepting on Client's Behalf](#7-accepting-on-clients-behalf)
8. [E-Signature in the Proposal Workflow](#8-e-signature-in-the-proposal-workflow)
   - 8.1 [Where Signature Sits in the Acceptance Flow](#81-where-signature-sits-in-the-acceptance-flow)
   - 8.2 [Payment Capture & Signature Interaction](#82-payment-capture--signature-interaction)
   - 8.3 [Downstream Triggers on Signature Completion](#83-downstream-triggers-on-signature-completion)
9. [Data Model: E-Signature Objects](#9-data-model-e-signature-objects)
10. [Legal Compliance Framework](#10-legal-compliance-framework)
    - 10.1 [US: ESIGN Act & UETA](#101-us-esign-act--ueta)
    - 10.2 [EU: eIDAS](#102-eu-eidas)
    - 10.3 [AU/NZ/CA/UK Frameworks](#103-aunzcauk-frameworks)
    - 10.4 [Industry-Specific: IRC Section 7216](#104-industry-specific-irc-section-7216)
    - 10.5 [What Makes an E-Signature Legally Valid](#105-what-makes-an-e-signature-legally-valid)
11. [Open-Source E-Signature Solutions](#11-open-source-e-signature-solutions)
    - 11.1 [DocuSeal](#111-docuseal)
    - 11.2 [OpenSign (OpenSignLabs)](#112-opensign-opensignlabs)
    - 11.3 [Documenso](#113-documenso)
    - 11.4 [LibreSign](#114-libresign)
    - 11.5 [signature_pad (JS Library — UI only)](#115-signature_pad-js-library--ui-only)
    - 11.6 [Comparison Matrix](#116-comparison-matrix)
    - 11.7 [Build vs. Integrate Decision Guide](#117-build-vs-integrate-decision-guide)
12. [Business Rules & Constraints](#12-business-rules--constraints)
13. [Edge Cases & Known Limitations](#13-edge-cases--known-limitations)
14. [Key Product Metrics](#14-key-product-metrics)

---

## 1. Feature Overview

E-signature in Ignition is not a standalone product — it is **tightly embedded within the proposal acceptance flow** and serves as the legally binding confirmation of a client's agreement to:

1. The **scope of work** (services listed in the proposal)
2. The **terms and conditions** (engagement letter / legal contract)
3. The **billing and payment authority** (authorization for Ignition to charge the stored payment method)

This coupling of signature with payment authority capture is Ignition's core differentiator: **one act of signing simultaneously authorizes the contract and the recurring payment collection**. There is no separate invoicing or payment authorization step.

**Secondary function (as of 2024–2025):** Ignition extended e-signature to support **additional documents** beyond the main proposal — such as regulatory consents (IRC §7216), NDAs, and powers of attorney — signed within the same workflow session.

---

## 2. User Personas & Their Signing Roles

| Persona | Role in Signing | Actions Available |
|---|---|---|
| **Firm Owner / Administrator** | Sets up the proposal; defines who signs | Configures required payment toggle; adds signatories; accepts on client's behalf; views all audit trails |
| **Senior Accountant / Partner** | May be the designated firm signatory on engagement letters (via placeholder in Terms template) | Name appears in terms via placeholder; does not "sign" in the UI — it's a merge field |
| **Client — Primary Signatory** | The main client contact who receives the proposal | Selects proposal option; enters payment details; agrees to terms; types name as signature |
| **Client — Secondary Signatory** | Additional client contact (partner, director, co-owner) who must also sign | Views proposal as already accepted by primary; cannot change options or payment; types name as signature only |
| **Firm User (Proxy)** | Acts on behalf of the client (with verbal/written authorization) | Accepts the proposal, selects payment method, provides a mandatory written reason (audit trail) |

---

## 3. How Ignition Implements E-Signature

### 3.1 Signature Method: Typed Name

Ignition uses **typed name** as the electronic signature method — not a drawn, biometric, or cryptographic signature.

**Implementation:**
- Client reaches the signature step of the acceptance page
- A text input field prompts: "Type your full name to sign"
- Client types their name (e.g., "Jane Smith")
- By typing and clicking "Accept Proposal", the client confirms:
  - Agreement to the proposal terms (engagement letter)
  - Agreement to Ignition's own Terms of Use
  - Authorization for the billing/payment schedule
- The typed name string is stored as the `signature_text` field on the signature record

**Why typed name (not drawn)?**
- Legally equivalent to a drawn signature under ESIGN/UETA — the law defines e-signature as "any electronic sound, symbol, or process" attached to a record with intent to sign
- Removes friction: works on any device with no touchscreen, stylus, or mouse drawing required
- Faster adoption rate for clients unfamiliar with digital signing tools
- The evidentiary weight comes from the audit trail metadata (IP, timestamp), not from the visual fidelity of the signature mark

### 3.2 Data Captured at Signing

At the moment of signing, Ignition captures and permanently stores:

| Data Point | Description | Purpose |
|---|---|---|
| **Typed Name** (`signature_text`) | The exact string typed by the signer | The e-signature itself; identifies intent |
| **Timestamp** (`signed_at`) | UTC date and time of the signing action | Proves when agreement was made |
| **IP Address** (`signer_ip`) | The public IP of the device used to sign | Geolocation context; supports attribution |
| **Signatory Email** | The email address the proposal link was sent to | Links signature to a specific person |
| **Proposal ID** | The ID of the proposal being signed | Links signature to the specific document version |
| **Signer Role** | Primary or Secondary | Identifies which contact in the multi-signer flow |
| **User Agent (inferred)** | Browser/device type | Contextual metadata for dispute resolution |

> **Note on IP Address and Mobile Devices:** The IP captured is the public-facing IP of the network the client is on (e.g., home router IP, mobile carrier IP). It is the same methodology used by all major e-signature platforms (DocuSign, Adobe Sign). For mobile devices connecting via carrier data, the IP is the carrier's NAT address — not the device's private IP.

### 3.3 Audit Trail

Ignition maintains a **digital audit trail** for every signature event on every proposal.

**Contents of the audit trail (per proposal):**

| Event | Recorded Data |
|---|---|
| Proposal Created | Firm user ID, timestamp |
| Proposal Sent | Firm user ID, recipient email, timestamp |
| Proposal Viewed | Recipient email, timestamp, IP address |
| Proposal Signed (Primary) | Typed name, timestamp, IP address, signatory email |
| Proposal Signed (Secondary N) | Typed name, timestamp, IP, signatory email — repeated per signer |
| Payment Details Entered | Timestamp (payment method type — not card number) |
| Proposal Accepted (all signed) | Final status timestamp |
| Accepted on Client's Behalf | Firm user ID, reason text, timestamp |

**Audit trail access:**
- Available to firm users from the proposal record in the Ignition UI
- Embedded in the signed PDF (see §3.4) as the "Agreement Summary" section
- Available in exported CSV from the Proposals tab

### 3.4 Signed PDF Generation

Upon completion of all signatures, Ignition generates a **signed PDF document** and distributes it:

- **Emailed automatically** to: all client signatories, and optionally firm notification recipients
- **Accessible** from the firm's Ignition dashboard (client record → proposal → PDF link)
- **Accessible** to the client via the Client Portal

**Structure of the signed PDF:**

| Section | Content |
|---|---|
| Personalised Message | The intro message configured by the firm (with placeholders resolved) |
| Services Summary | Complete list of services, descriptions, and prices agreed to |
| Payment Schedule | Full billing timeline — all billing events, amounts, and triggers |
| Payment Authority | Client's authorization to charge the stored payment method |
| Terms & Conditions | The full engagement letter / terms template (with all placeholders resolved) |
| Service Terms | Any per-service sub-terms |
| **Audit Trail of Acceptance** | Timestamped log of all events (sent, viewed, signed, by whom, IP, time) |
| **Signature Block(s)** | Typed name of each signatory, their email, date/time signed |

The PDF is generated server-side at the moment of final acceptance and is **tamper-evident** (any modification to the PDF after generation would be detectable).

---

## 4. Multi-Signer Workflow

### 4.1 Roles: Primary vs. Secondary Signatories

| Capability | Primary Signatory | Secondary Signatory(ies) |
|---|---|---|
| Receives initial proposal email | ✅ Yes | ⚠️ "Heads-up" preview email only |
| Selects proposal option (A/B/C) | ✅ Yes | ❌ No (sees what primary selected) |
| Enters payment details | ✅ Yes | ❌ No |
| Signs the proposal | ✅ Yes (first) | ✅ Yes (after primary) |
| Receives signing invitation email | On proposal send | After primary has signed |
| Receives reminder emails | ✅ Yes (until signed) | ✅ Yes (after primary has signed) |
| Receives signed PDF | ✅ Yes | ✅ Yes |

**Primary Signatory:** The designated main client contact. Required on every proposal. Has full decision-making responsibilities: option selection, payment authorization, and signature.

**Secondary Signatories:** Additional contacts from the same client record added to the proposal. They can view the proposal at any time but can only sign after the primary has completed their portion. They do not re-select options or enter payment details — they are confirming the same agreement.

### 4.2 Sequential Signing Flow

```
[Firm sends proposal]
       │
       ▼
[Primary Signatory receives proposal email]
       │
       │ (Primary opens → selects option → enters payment → types name → accepts)
       ▼
[Primary signs] → Ignition sends signing invitation to ALL Secondary Signatories simultaneously
       │
       │ (Each secondary: opens → reviews → types name → accepts)
       ▼
[Last secondary signs] → Proposal status changes to ACCEPTED
                       → Billing events triggered
                       → Automation/workflow jobs created
                       → Signed PDF emailed to all parties
```

**Heads-up email to secondaries:**
- Sent at the same time as the primary's proposal email (on proposal dispatch)
- Content: "A proposal is being sent for your signature. You will receive a signing invitation once [Primary Name] has signed."
- Purpose: Maximizes awareness and speed to completion by giving secondaries advance notice

### 4.3 Plan Limits on Signers

| Plan | Maximum Signatories per Proposal |
|---|---|
| Solo | 1 (primary only; no multi-signer) |
| Core | 1 (primary only; no multi-signer) |
| Pro | **2** (1 primary + 1 secondary) |
| Pro+ / Scale | **10** (1 primary + 9 secondaries) |
| Trial | **10** (full access during trial) |

> Multi-signature is only available in the **New Proposal Editor**. The Classic Proposal Editor does not support it.

### 4.4 Proposal Status During Multi-Sign

- Proposal remains in **Awaiting Acceptance** until ALL signatories have signed
- Firm receives a "proposal accepted" notification email **after each individual signature** (not just at the end)
- Once the final signatory signs, the proposal transitions to **Accepted/Active**
- If any secondary signatory does not sign (e.g., leaves the firm), the firm must revoke the proposal (returns to Draft) to update the signatory list, then resend

### 4.5 Signer Links & Reminder Logic

**Individual signer links:**
- Each signatory gets their own **unique proposal link** — the primary's link and each secondary's link are distinct
- Firm can access each signatory's live link from the proposal record → left-hand signatory list → copy link
- These links can be shared via external email, SMS, or any channel if the firm-sent email is not received

**Reminder sequencing for multi-signer:**
1. Primary: receives reminders per the global reminder schedule (Settings → General → Proposal Reminders)
2. Secondaries: receive no reminders until primary has signed; after primary signs, secondaries start receiving reminders on the same schedule
3. Reminders stop for a signatory as soon as they have signed

---

## 5. Firm-Side Signature in Engagement Letters

Ignition does not require the firm to sign the proposal in the UI — the **firm's signature is embedded in the Terms template** as a static or dynamic element.

### Static Firm Signature in Terms
- Firm prepares the Terms template (engagement letter)
- The partner's name and title are added as static text at the bottom of the terms
- Every proposal using that template shows the same signatory name
- This is appropriate when one partner signs all engagements

### Dynamic Firm Signature via Placeholders
- More flexible: uses Ignition placeholders to insert the **assigned partner's** name and title dynamically
- Requires: each Client record to have a Partner assigned; each User record to have a Job Title set
- Placeholder syntax (in Terms template):

```
{{user.name}}
{{user.job_title}}
{{account.name}}
```

- Result: The engagement letter shows "Sarah Johnson, Managing Partner, Acme CPA Firm" for any proposal assigned to Sarah Johnson
- No UI signing step for the firm — the firm's "signature" is the act of sending the proposal

### Implication for Development
- No "firm signature" object needs to be stored — only client signatures are captured
- The firm's identity and authorization to engage the client is established by their account credentials and the act of sending
- If a two-sided signing flow is required (firm signs too), this would need to be built as a new capability

---

## 6. Additional Document Signatures

**Introduced:** 2024 (free until May 5, 2025; paid thereafter on eligible plans)

**Purpose:** Collect signatures for supplementary documents alongside the main proposal in the same signing session.

### Supported Document Types

| Document Type | Description |
|---|---|
| **IRC §7216 Consent** | US-specific; required for tax preparers who wish to share client return information with third parties (e.g., for additional services, marketing, referrals). Failure to obtain written, signed consent before disclosure is a federal criminal offense. |
| **NDA / Confidentiality Agreement** | Non-disclosure agreement where the client consents to confidentiality terms |
| **Power of Attorney** | Authorization for the firm to act on the client's behalf with a government agency (e.g., ATO, IRS, HMRC) |
| **Custom Disclosures** | Any additional document requiring client sign-off |

### How It Works in the Acceptance Flow

1. Firm attaches the additional document(s) to the proposal in the Presentation step (or as a library item)
2. During client acceptance, after the main proposal is signed, the client is presented with any additional documents requiring signatures
3. Client signs each document in turn (same typed-name mechanism)
4. All signatures are captured in the same audit trail
5. All signed documents included in the confirmation PDF

### Development Implications
- Each additional document is a separate **document entity** with its own signature record
- A proposal can have N additional documents (limits TBD per plan)
- Each document has its own: content (PDF/rich text), signature capture, and audit event
- The "proposal is complete" trigger should wait for ALL additional documents to be signed (or define whether additional docs are optional vs. required)

---

## 7. Accepting on Client's Behalf

Firm administrators can accept a proposal on a client's behalf when the client has given verbal or written authorization outside of Ignition (e.g., via phone call, physical paperwork).

### Flow

1. Firm admin opens the proposal (in Awaiting Acceptance status)
2. Clicks "Accept on client's behalf"
3. A confirmation dialog appears with:
   - **Payment method selection**: choose from saved payment methods on file for the client, or add a new one
   - **Mandatory reason field**: firm must enter the reason for accepting on behalf (e.g., "Client verbally authorized during call on 2025-03-15; email confirmation pending")
4. Firm admin clicks "Accept"
5. System records the acceptance with the firm admin's user ID, the reason text, timestamp, and admin's IP

### Audit Trail Entry
```
Event: AcceptedOnClientBehalf
Actor: FirmUser (user_id: xxx)
Reason: "Verbal authorization given by John Smith on phone call, March 15, 2025"
Timestamp: 2025-03-15T14:32:00Z
IP: [admin's IP]
```

### Key Rules
- Only Administrators can perform this action (Members cannot)
- The reason text is mandatory — cannot be bypassed
- Stored permanently in the audit trail on the proposal record
- The accepted-on-behalf flag is visible in the proposal record and exported CSV
- No client email notification is sent (since the firm is acting for the client) — firm should notify the client separately
- The signed PDF is still generated and available; it notes the proxy acceptance

---

## 8. E-Signature in the Proposal Workflow

### 8.1 Where Signature Sits in the Acceptance Flow

The signature step is the **final gate** in the client acceptance sequence. Nothing downstream triggers until signing is complete.

```
Client opens proposal link
        │
        ▼
[Screen 1] Intro / Personalised Message
        │
        ▼
[Screen 2] Scope / Services / Proposal Options (client selects package)
        │
        ▼
[Screen 3] Payment Schedule (review billing timeline)
        │
        ▼
[Screen 4] Payment Details (enter card/bank — required or optional per firm setting)
        │
        ▼
[Screen 5] Terms & Conditions (read engagement letter; checkbox: "I agree")
        │
        ▼
[Screen 6] E-SIGNATURE ← (type full name → click "Accept Proposal")
        │
        ▼
[Screen 7] Thank You / Next Steps
```

**Key coupling:** The signature step appears AFTER payment details are captured but BEFORE the "Thank You" confirmation. This means:
- The client cannot sign without first (optionally) entering payment details
- If payment is marked as required, entering payment details is a prerequisite to reaching the signature screen
- The act of signing is the final confirmation — it authorizes both the engagement AND the payment

### 8.2 Payment Capture & Signature Interaction

| Payment Setting | Behavior at Signature Step |
|---|---|
| Payment required = true | Client MUST enter card/bank before signature step is reachable. Cannot skip. |
| Payment required = false | Client can optionally enter payment details or skip. Signature step is still reachable without payment details. |
| No payment integration set up | Payment step skipped entirely; client goes directly to Terms → Signature |

**What the client is consenting to by signing:**
1. The engagement/scope of services
2. The payment terms and schedule as displayed
3. Authority for Ignition to debit the payment method per the billing schedule
4. Ignition's Terms of Use
5. Any additional documents (IRC §7216, NDAs, etc.) presented in the flow

### 8.3 Downstream Triggers on Signature Completion

When the final signature event completes (all required signatories have signed):

| Action | Timing | Condition |
|---|---|---|
| Proposal status → Accepted | Immediate | Always |
| Active Services created | Immediate | Always — one service record per service group |
| Billing event: "On Acceptance" services | Immediate | Automatic billing mode + On Acceptance trigger |
| Invoice created in Xero / QBO | Immediate (sync) | Integration active + Automatic billing mode |
| Practice management jobs created | Immediate or scheduled | Automation step configured (deploy mode: Auto) |
| Firm notification email sent | Immediate | Notification settings enabled |
| Signed PDF generated | Immediate | Always |
| Signed PDF emailed to all signatories | Immediate | Always |
| Signed PDF emailed to firm notification recipients | Per settings | Notification settings |
| Slack notification | Immediate | Slack integration enabled |
| Zapier "Proposal Accepted" trigger | Immediate | Zapier connected |

---

## 9. Data Model: E-Signature Objects

### Signature Record

| Field | Data Type | Notes |
|---|---|---|
| `id` | UUID | |
| `proposal_id` | FK → Proposal | |
| `additional_document_id` | FK → AdditionalDocument | Nullable — null if this is the main proposal signature |
| `contact_id` | FK → Contact | The signing contact |
| `signatory_role` | Enum | Primary, Secondary |
| `signatory_order` | Integer | 1 = primary, 2–10 = secondaries |
| `email` | String | Email at time of signing (in case contact email changes later) |
| `signature_text` | String | The typed name string |
| `signed_at` | Timestamp | UTC timestamp of signing |
| `signer_ip` | String | Public IP address of signer's device |
| `user_agent` | String | Browser/device user agent (optional but recommended) |
| `is_proxy_acceptance` | Boolean | True if accepted by firm admin on behalf of client |
| `proxy_user_id` | FK → User | Nullable — set if is_proxy_acceptance = true |
| `proxy_reason` | Text | Nullable — mandatory reason text for proxy acceptance |
| `status` | Enum | Pending, Signed |
| `invitation_sent_at` | Timestamp | When the signing email was sent to this signatory |
| `first_viewed_at` | Timestamp | When this signatory first opened the proposal link |
| `created_at` | Timestamp | |

### Additional Document

| Field | Data Type | Notes |
|---|---|---|
| `id` | UUID | |
| `proposal_id` | FK → Proposal | |
| `document_type` | Enum | IRC7216Consent, NDA, PowerOfAttorney, Custom |
| `title` | String | Display name |
| `content` | RichText or FileRef | The document content (may be a PDF file or inline rich text) |
| `is_required` | Boolean | Whether client must sign this document to complete proposal acceptance |
| `sort_order` | Integer | Presentation order in signing flow |
| `created_at` | Timestamp | |

### Signature Invitation

| Field | Data Type | Notes |
|---|---|---|
| `id` | UUID | |
| `signature_id` | FK → Signature | |
| `email_template_id` | FK → EmailTemplate | Template used |
| `sent_at` | Timestamp | |
| `delivery_status` | Enum | Sent, Delivered, Bounced, Unknown |
| `unique_token` | String | The token embedded in the signing URL (one per signatory) |
| `link_url` | String | The full signing link for this signatory |
| `link_expires_at` | Timestamp | Nullable — if proposal has an expiry date set |

---

## 10. Legal Compliance Framework

### 10.1 US: ESIGN Act & UETA

**ESIGN Act (Electronic Signatures in Global and National Commerce Act, 2000):**
- Federal law that grants legal recognition to electronic signatures and electronic records
- Core rule: an e-signature "may not be denied legal effect, validity, or enforceability solely because it is in electronic form"
- Applies to transactions affecting interstate or foreign commerce

**UETA (Uniform Electronic Transactions Act, 1999):**
- Adopted by 49 US states + DC, Puerto Rico, US Virgin Islands
- New York has its own equivalent (ESRA)
- Provides that when a law requires a writing or signature, an electronic record or signature can satisfy that requirement when parties have agreed to proceed electronically

**Combined effect:** Electronic signatures are legally equivalent to wet ink signatures in commerce across the US.

**Requirements Ignition satisfies:**

| Requirement | How Ignition Meets It |
|---|---|
| Intent to sign | Client actively types name and clicks "Accept Proposal" button |
| Consent to transact electronically | Client receives proposal via email link and proceeds through acceptance flow — consent is demonstrated by action |
| Attribution | Typed name + signatory email + IP address + timestamp |
| Record retention | Signed PDF generated; stored in Ignition; emailed to all parties; firm can export |
| Tamper evidence | Signed PDF generated at completion; any post-signing modification is detectable |

### 10.2 EU: eIDAS

**eIDAS (Electronic Identification, Authentication and Trust Services, 2014):**

Three tiers of e-signatures recognized:

| Type | Description | Ignition's Implementation |
|---|---|---|
| **Simple Electronic Signature (SES)** | Any electronic data logically associated with a record and used to sign. Lowest standard. | ✅ Ignition's typed-name + metadata qualifies as SES |
| **Advanced Electronic Signature (AdES)** | Must uniquely identify the signer, allow detection of any changes post-signing, be linked to signer data, and be created under signer's sole control | ⚠️ Ignition's implementation is borderline — does not use a PKI certificate per signer |
| **Qualified Electronic Signature (QES)** | AdES + created using a qualified signature device + based on a qualified certificate. Highest standard; legally equivalent to wet ink in all EU member states | ❌ Ignition does not implement QES |

**Practical implication:** Ignition's signatures qualify as SES under eIDAS. For most professional services engagements (accounting, bookkeeping, consulting), SES is legally sufficient. QES is generally required only for specific regulated instruments (government filings, real estate transfers, etc.) and is not commonly used in the Ignition target market.

**When building for EU clients:** The product should clearly document that signatures are SES-level, and provide guidance to firms in regulated EU sectors that require AdES or QES.

### 10.3 AU/NZ/CA/UK Frameworks

| Jurisdiction | Framework | Ignition Compliance |
|---|---|---|
| **Australia** | Electronic Transactions Act 1999 (federal); state equivalents | ✅ Typed-name signatures are valid for commercial contracts |
| **New Zealand** | Electronic Transactions Act 2002 | ✅ Same — typed name satisfies the signature requirement for most contracts |
| **Canada** | PIPEDA (federal); UETA-equivalent laws by province | ✅ Valid for commercial contracts; some provinces have higher requirements for specific document types |
| **United Kingdom** | Electronic Communications Act 2000; EU eIDAS retained post-Brexit | ✅ SES is valid; QES required only for specific instruments (e.g., land registry) |

### 10.4 Industry-Specific: IRC Section 7216

**Relevant only for US-based tax preparers.**

**What it is:**
- Internal Revenue Code §7216 prohibits tax preparers from knowingly disclosing or using a taxpayer's tax return information for any purpose other than preparing their return — without written, signed consent.
- Violating §7216 is a **federal criminal offense** (misdemeanor, up to 12 months imprisonment + fines).

**When it applies:**
- Firm offers services beyond tax preparation: financial planning, investment advice, insurance referrals, bank products, audit protection, identity theft monitoring
- Firm wishes to disclose client tax information to a third party (e.g., referring to a wealth manager)
- Firm uses client data for internal marketing or cross-selling

**Requirements for a valid §7216 consent:**
- Must be in writing (electronic signature qualifies)
- Must be separate from other agreements — cannot be embedded in an engagement letter
- Must specify: the tax return information to be disclosed, the purpose, the recipient, the time period
- Client must sign specifically for this consent (hence Ignition's "additional documents" feature)

**Ignition's solution:** The "Additional Document Signatures" feature allows firms to present a pre-built IRC §7216 consent form during the proposal signing session, ensuring clients sign it separately before the firm uses their data for non-preparation purposes.

### 10.5 What Makes an E-Signature Legally Valid

For any implementation (custom-built or open-source), the following elements are required for legal enforceability:

| Element | Implementation Requirement |
|---|---|
| **Intent** | Affirmative action by the signer (click, type, draw) with clear label indicating they are signing |
| **Identity Attribution** | Capture: signer's email (matches the address the invite was sent to), typed name, IP address, timestamp |
| **Consent to Transact Electronically** | Signer must have had the opportunity to consent to electronic transactions — typically implicit if they received a link via email and proceeded |
| **Document Integrity** | The document content must be fixed at the time of signing; any subsequent modification must be detectable |
| **Record Retention** | The signed record (document + metadata + audit trail) must be stored and retrievable for the duration required by applicable law |
| **Accessibility** | Signer must be able to receive a copy of the signed record (emailed PDF or download) |

---

## 11. Open-Source E-Signature Solutions

This section evaluates open-source alternatives for building e-signature functionality in a product similar to Ignition, rather than building from scratch.

---

### 11.1 DocuSeal

**What it is:** A fully-featured, open-source document signing platform. The closest open-source equivalent to DocuSign in terms of feature completeness and developer tooling.

**GitHub:** [github.com/docusealco/docuseal](https://github.com/docusealco/docuseal) — ~10k+ stars

**Tech Stack:** Ruby on Rails (backend), JavaScript/Stimulus (frontend), PostgreSQL or SQLite (database), Active Storage (file storage)

**License:** AGPLv3 (free for self-hosting; commercial cloud pricing also available)

**Key Features:**

| Feature | Details |
|---|---|
| PDF form building | Drag-and-drop field placement or `{{FieldName;type=signature}}` text tags in PDFs |
| Field types | Text, Signature, Date, Initials, Number, Image, Checkbox, Radio, Select, File, Cells, Stamp, Payment, Phone, Verification (KBA), Strikethrough |
| Multi-signer | Sequential or parallel signing; define order per submitter |
| Embedding | Web component `<docuseal-form>` that embeds signing UI into any web page |
| API | Full REST API + SDKs (JS/TS, Ruby, Python, PHP) |
| Audit trail | Timestamps, IP addresses, email of each signer; embedded in signed PDF |
| Compliance | ESIGN, UETA, eIDAS (Simple/Advanced) |
| Storage | Local disk, AWS S3, Google Cloud Storage, Azure Blob Storage |
| Database | SQLite (default for dev), PostgreSQL or MySQL (production) |
| Deployment | Docker (single container); one-click deploys for Heroku, Render, Railway, DigitalOcean |

**Embedding example (drop into any HTML page):**

```html
<script src="https://cdn.docuseal.com/js/form.js"></script>
<docuseal-form
  id="docusealForm"
  data-src="https://YOUR_DOCUSEAL_HOST/d/{{ submission_slug }}"
  data-email="{{ signer_email }}">
</docuseal-form>
<script>
  window.docusealForm.addEventListener('completed', (e) => {
    // e.detail contains completion data
    console.log('Signed!', e.detail);
  });
</script>
```

**API — Create a signing request:**

```javascript
const { createSubmission } = require("@docuseal/api");

const submission = await createSubmission({
  template_id: 1000001,
  send_email: true,
  submitters: [
    { role: "Client", email: "client@example.com", order: 0 },
    { role: "Witness", email: "witness@example.com", order: 1 }
  ]
});
// Returns: submission ID + signer links
```

**Field placement via API (coordinates):**

```json
{
  "fields": [
    {
      "submitter_uuid": "...",
      "name": "ClientSignature",
      "type": "signature",
      "required": true,
      "areas": [{ "x": 0.2, "y": 0.8, "w": 0.3, "h": 0.05, "page": 1 }]
    }
  ]
}
```

**Strengths:**
- Most complete feature set among open-source options
- Best developer experience (documented API, multiple SDKs, embeddable component)
- SQLite makes it easy to get started; Postgres for production scale
- Self-hosted means documents never leave your infrastructure

**Weaknesses:**
- AGPLv3 license: if you modify DocuSeal and run it as a service, you must open-source your modifications
- Ruby on Rails stack — requires Ruby expertise for customization
- No built-in SOC 2 or HIPAA compliance (must achieve this independently)
- No qualified e-signature (QES) support out of the box

---

### 11.2 OpenSign (OpenSignLabs)

**What it is:** An open-source DocuSign alternative focused on unlimited free signing with a strong community.

**GitHub:** [github.com/OpenSignLabs/OpenSign](https://github.com/OpenSignLabs/OpenSign) — ~3k+ stars

**Tech Stack:** React (frontend), Node.js (backend), MongoDB (database), Parse Server (BaaS layer)

**License:** AGPLv3

**Key Features:**

| Feature | Details |
|---|---|
| Signature types | Drawn (canvas), typed, uploaded image, saved signature |
| Multi-signer | Sequential and parallel workflows; OTP verification per signer |
| Document management | "OpenSign Drive" — a built-in document vault |
| Audit trail | Timestamps, IP addresses, email IDs, phone numbers; completion certificate PDF |
| API | REST API (self-hosted API access requires paid plan; sandbox available free) |
| Integrations | Zapier, cloud storage (S3, etc.) |
| Deployment | Docker (single command); DigitalOcean one-click |

**Important limitation:** **The free self-hosted version does not support API token generation for production use.** Only a sandbox API (for development/testing) is available. For production API access in a self-hosted setup, a paid plan is required. This is a significant constraint for embedding OpenSign into another product.

**Strengths:**
- Supports drawn signatures (more visually "traditional")
- OTP verification adds identity validation layer
- Free for unlimited documents in cloud version

**Weaknesses:**
- MongoDB dependency (versus more common Postgres)
- API production access requires paid plan on self-hosted
- Smaller developer ecosystem than DocuSeal
- Parse Server layer adds architectural complexity

---

### 11.3 Documenso

**What it is:** A modern, developer-friendly open-source e-signature platform built with contemporary web technologies.

**GitHub:** [github.com/documenso/documenso](https://github.com/documenso/documenso) — 10k+ stars

**Tech Stack:** Next.js 14, TypeScript, Prisma ORM, PostgreSQL, Tailwind CSS, shadcn/ui

**License:** AGPLv3 (Community); Enterprise license available ($30,000/yr — unlimited users/volume)

**Founded:** 2023, Germany. Pre-seed funded ($1.54M). SOC 2 compliance published.

**Key Features:**

| Feature | Details |
|---|---|
| Document templates | Reusable templates with pre-placed fields |
| Field types | Signature (drawn/typed), Initials, Date, Text, Number, Checkbox, Dropdown, Radio, Image |
| Multi-signer | Sequential workflows; team collaboration |
| Audit trail | Per-event logging; embedded in signed PDF |
| White-labeling | Custom logo, domain, color scheme |
| SSO | Yes (enterprise) |
| API | Full open API with webhook support |
| Compliance | GDPR, eIDAS, 21 CFR Part 21 (FDA), SOC 2 |
| Database | PostgreSQL (required) |
| Deployment | Docker, Railway ($5–10/mo), any VPS with Node.js |

**Strengths:**
- Most modern tech stack (Next.js/TypeScript/Prisma) — easiest for teams with web dev expertise to extend
- Best compliance posture (SOC 2, GDPR, 21 CFR Part 21, eIDAS)
- EU-based — strong GDPR alignment by design
- PostgreSQL — fits cleanly into a Postgres-primary stack
- White-labeling included

**Weaknesses:**
- AGPLv3 applies; enterprise license at $30k/yr for non-AGPL usage
- Newer project (2023) — less battle-tested than DocuSeal
- Community edition described as suitable for "smaller teams and non-critical deployments"
- Smaller ecosystem and fewer pre-built integrations than DocuSeal

---

### 11.4 LibreSign

**What it is:** An e-signature application built as a Nextcloud app, making it ideal for organizations already running Nextcloud.

**GitHub:** [github.com/LibreSign/libresign](https://github.com/LibreSign/libresign)

**Tech Stack:** PHP (Nextcloud app), Vue.js (frontend), PostgreSQL or MySQL

**License:** AGPLv3

**Key Features:**
- Deeply integrated with Nextcloud (files, users, teams)
- End-to-end encryption; multi-factor authentication
- Digital signatures using PDF standard cryptographic signatures (not just typed name)
- Supports legally binding e-signatures across multiple countries

**Strengths:**
- Strong cryptographic PDF signatures (goes beyond typed-name SES)
- Ideal if Nextcloud is already in the infrastructure
- Strong privacy focus

**Weaknesses:**
- Tightly coupled to Nextcloud — not practical without it
- Not suitable for embedding in a third-party app without significant work
- Smaller community than DocuSeal or Documenso

---

### 11.5 signature_pad (JS Library — UI only)

**What it is:** A JavaScript library for capturing hand-drawn signatures on an HTML5 canvas. This is a **UI component only** — not a full e-signature platform. No audit trail, storage, or document management.

**GitHub:** [github.com/szimek/signature_pad](https://github.com/szimek/signature_pad) — 13k+ stars

**License:** MIT

**Install:**

```bash
npm install signature_pad
```

**Basic usage:**

```javascript
import SignaturePad from 'signature_pad';

const canvas = document.querySelector('canvas');
const signaturePad = new SignaturePad(canvas, {
  backgroundColor: 'rgba(255, 255, 255, 0)',
  penColor: 'rgb(0, 0, 0)'
});

// Get signature as PNG data URL
const dataURL = signaturePad.toDataURL();

// Get signature as SVG
const svg = signaturePad.toSVG();

// Check if empty
if (signaturePad.isEmpty()) { alert('Please sign first'); }

// Clear
signaturePad.clear();

// Export raw point data (for replay/animation)
const data = signaturePad.toData();
```

**Strengths:**
- MIT license — no AGPL obligations
- Tiny and dependency-free
- Works on all modern browsers and touch devices (mobile-optimized)
- Uses Bézier curve interpolation for smooth natural signatures
- TypeScript declarations included
- Output: PNG, JPEG, SVG, or raw point data

**Weaknesses:**
- UI only — must build all backend logic (storage, audit trail, PDF embedding, email, notifications) separately
- No audit trail, identity capture, or compliance features out of the box
- Needs substantial surrounding infrastructure to achieve legal validity

**Use case in a custom build:** Use `signature_pad` as the draw-capture UI component within a custom signing page, combined with backend logic for: UUID-linked signing sessions, IP capture, timestamp storage, PDF generation (e.g., pdf-lib or PDFKit), and email delivery.

---

### 11.6 Comparison Matrix

| | DocuSeal | OpenSign | Documenso | LibreSign | signature_pad |
|---|---|---|---|---|---|
| **Type** | Full platform | Full platform | Full platform | Full platform | UI component only |
| **License** | AGPLv3 | AGPLv3 | AGPLv3 / Enterprise | AGPLv3 | MIT |
| **Self-hosted** | ✅ | ✅ | ✅ | ✅ (via Nextcloud) | N/A |
| **Tech stack** | Ruby on Rails | Node.js/React/MongoDB | Next.js/TypeScript/Postgres | PHP/Vue.js | Vanilla JS |
| **Database** | SQLite / Postgres / MySQL | MongoDB | PostgreSQL | Postgres / MySQL | N/A |
| **REST API** | ✅ Full | ⚠️ Paid for production | ✅ Full | ⚠️ Limited | N/A |
| **Embed signing UI** | ✅ Web component | ⚠️ Custom work | ⚠️ Custom work | ❌ | ✅ (canvas only) |
| **Multi-signer** | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Drawn signatures** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Typed signatures** | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Audit trail** | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Signed PDF generation** | ✅ | ✅ | ✅ | ✅ | ❌ |
| **ESIGN/UETA compliant** | ✅ | ✅ | ✅ | ✅ | ❌ alone |
| **eIDAS level** | SES/AdES | SES | SES/AdES | AdES | N/A |
| **SOC 2** | ❌ | ❌ | ✅ (hosted) | ❌ | N/A |
| **Docker deploy** | ✅ | ✅ | ✅ | ✅ | N/A |
| **White-label** | ✅ | ✅ | ✅ | Limited | N/A |
| **GitHub Stars (approx.)** | ~10k | ~3k | ~10k | ~1k | ~13k |
| **Community maturity** | High | Medium | High (growing) | Low | Very High |

---

### 11.7 Build vs. Integrate Decision Guide

The choice between building from scratch, using an open-source platform, or licensing a commercial API depends on:

**Use DocuSeal (self-hosted) if:**
- You want the most complete OSS feature set with the best embedding story
- Your team is comfortable with or can hire for Ruby on Rails
- You want typed + drawn + initial field support out of the box
- You need multi-signer sequential workflows immediately
- You're OK with AGPLv3 (or plan to open-source the integration)

**Use Documenso if:**
- Your team is TypeScript/Next.js native
- You need modern compliance posture (SOC 2, GDPR, 21 CFR)
- You're building for EU markets (eIDAS considerations)
- You have budget for the enterprise license to avoid AGPLv3 obligations ($30k/yr)

**Use OpenSign if:**
- You want free unlimited signing in the cloud (without API integration)
- You need drawn signature as a first-class UI
- You're using it primarily via its web UI (not embedded)

**Use signature_pad only if:**
- You are building a fully custom e-signature flow and only need the canvas drawing UI
- You will build all backend infrastructure: session management, audit logging, PDF generation, email delivery, storage
- You want MIT license with zero AGPL obligation

**Build from scratch if:**
- Your e-signature needs are very simple (single signer, typed name, internal only)
- You need precise control over the data model and no external dependencies
- Minimum requirements: typed-name capture, IP + timestamp logging, PDF snapshot at signing time, signed PDF storage + email delivery

**Use a commercial API if:**
- Compliance certification (SOC 2, HIPAA, FedRAMP) is required from day one
- You need QES (Qualified Electronic Signature) for EU regulated documents
- You cannot take on AGPL obligations or infrastructure overhead

**Commercial API options (for reference):**

| Service | Notes |
|---|---|
| **DocuSign API** | Market leader; $10–$40+ per envelope; full compliance |
| **Adobe Acrobat Sign** | Strong PDF integration; enterprise pricing |
| **HelloSign (Dropbox Sign)** | Simpler API; better pricing for mid-market |
| **Zoho Sign** | Low cost; API available |
| **eSignatures.com** | Pay-per-envelope; simple API |

---

## 12. Business Rules & Constraints

- A proposal cannot move from Awaiting Acceptance to Accepted with any unsigned signature records outstanding
- Secondary signatories **cannot** change the option selection or payment details — these are locked after primary signs
- The proposal status remains Awaiting Acceptance for the entire duration of multi-signer flow (even after primary signs)
- The "Proposal Accepted" trigger (billing, automation, PDF) fires only when the **last required signature** is captured
- Accepting on a client's behalf requires an Administrator role AND a non-empty reason text field
- The signed PDF is immutable — it is generated at the moment of final acceptance and cannot be regenerated or replaced (it reflects the exact document the client signed)
- If an additional document signature is required, the proposal is not considered complete until all additional documents are signed
- Revoking a proposal (to edit it) **invalidates all existing signature records** for that proposal — when resent, new signature records are created
- A proposal with an expired expiry date cannot be accepted — the client sees an "expired" message and must contact the firm to renew or resend
- Signer links are **one-time use per signing session** — once signed, the link shows the completed state and cannot be used to re-sign

---

## 13. Edge Cases & Known Limitations

| Scenario | Behavior / Implication |
|---|---|
| Client signs from a VPN or proxy | IP captured is the VPN/proxy exit IP — not the client's actual location IP. Standard industry limitation. |
| Shared email address (multiple people at one email) | Ignition cannot distinguish which individual signed; attribution relies on the client's own access controls. Firm should ensure each signatory has their own email address. |
| Secondary signatory's email becomes invalid before they sign | Proposal stays in Awaiting Acceptance indefinitely. Firm must revoke, update contact email, and resend. |
| Client's device cannot render the signing page (very old browser) | No documented fallback. Firm should offer to accept on client's behalf or direct client to a modern browser. |
| Typo in typed name | There is no validation that the typed name matches the contact name on file — any string is accepted. This is by design (same as wet ink — you can sign with any mark). |
| Client changes their mind after signing (wants to withdraw) | Must contact the firm. Firm can mark the proposal as Lost (for billing purposes) but the signed PDF audit trail remains permanent. |
| Power outage / session drop mid-signing | If the client has not clicked "Accept Proposal", no signature is recorded. The proposal remains in Awaiting Acceptance. Client must restart from the link. |
| Proposal has no payment method + billing is set to Automatic | Billing will fail when triggered (no payment method on file). Firm must follow up to collect payment details via AutoCollect or manual invoice. |
| Multi-signer: primary accepts on behalf of client | Secondary signatories are then notified to sign. The proxy acceptance covers only the primary role; secondaries must still sign normally. |
| Additional documents: client exits after signing main proposal but before additional docs | System must define: are additional doc signatures required to complete the proposal? If yes, proposal should remain in a sub-state (e.g., "Partially Signed") until all docs are signed. Ignition's current handling of this edge case is not publicly documented. |
| Plan downgrade after proposals with 10 signers are sent | Existing sent proposals retain their signer count. New proposals created after downgrade are subject to new plan limits. |

---

## 14. Key Product Metrics

| Metric | Value | Source |
|---|---|---|
| Max signatories per proposal (Pro+/Scale) | 10 | Ignition Help Center |
| Data captured per signature | Name, IP, Timestamp, Email | Official documentation |
| Acceptance page: mobile-compatible | Yes — no app install required | Official documentation |
| Signed PDF delivery | Immediate on final signature | Official documentation |
| E-signature legal standard (US) | ESIGN Act + UETA compliant | Inferred from metadata captured |
| E-signature legal standard (EU) | SES under eIDAS | Inferred from implementation |
| Time from proposal send to acceptance (anecdotal) | "Minutes to hours" for digital vs. "days to weeks" for paper | Customer testimonials |
| % payments auto-collected post-signature | 91% | Ignition 2025 platform data |
| % customers reporting reduced late payments | 78% | Ignition 2025 customer survey |
| % customers reporting reduced scope creep | 85% | Ignition 2025 customer survey |

---

*Sources: [ignitionapp.com](https://www.ignitionapp.com), [support.ignitionapp.com — Multiple Signatures](https://support.ignitionapp.com/en/articles/5518741-multiple-signatures), [support.ignitionapp.com — Ignition for Clients](https://support.ignitionapp.com/en/articles/9830758-ignition-for-clients), [support.ignitionapp.com — Accept on Client's Behalf](https://support.ignitionapp.com/en/articles/8594099-accept-on-your-client-s-behalf), [support.ignitionapp.com — Adding a Partner Signature](https://support.ignitionapp.com/en/articles/600785-adding-a-partner-signature-to-your-templates), [DocuSeal](https://www.docuseal.com), [DocuSeal GitHub](https://github.com/docusealco/docuseal), [OpenSign GitHub](https://github.com/OpenSignLabs/OpenSign), [Documenso](https://documenso.com), [signature_pad GitHub](https://github.com/szimek/signature_pad), [ESIGN Act / UETA — DocuSign](https://www.docusign.com/products/electronic-signature/learn/esign-act-ueta), [eIDAS — Signaturit](https://www.signaturit.com/blog/electronic-signature-legislation-in-the-united-states-ueta-act-and-e-sign-act/), [IRC §7216 — CPA Journal](https://www.cpajournal.com/2019/12/03/getting-taxpayers-consent-to-disclose-or-use-tax-return-information-under-irc-section-7216/) — compiled February 2026.*
