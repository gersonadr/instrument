# Feature Research: Proposals
> Source: ignitionapp.com — official product, help center (support.ignitionapp.com), and learning center.
> Purpose: Reference document for a planner agent writing software development epics.

---

## Table of Contents

1. [Feature Overview](#1-feature-overview)
2. [User Personas](#2-user-personas)
3. [Proposal Lifecycle & Statuses](#3-proposal-lifecycle--statuses)
4. [Firm-Side Screen Flow (Proposal Editor)](#4-firm-side-screen-flow-proposal-editor)
   - 4.1 [Step 1 — General Settings](#41-step-1--general-settings)
   - 4.2 [Step 2 — Services & Billing](#42-step-2--services--billing)
   - 4.3 [Step 3 — Proposal Options (Tiered Packages)](#43-step-3--proposal-options-tiered-packages)
   - 4.4 [Step 4 — Presentation](#44-step-4--presentation)
   - 4.5 [Step 5 — Automation / Workflow](#45-step-5--automation--workflow)
   - 4.6 [Step 6 — Send](#46-step-6--send)
5. [Client-Side Acceptance Flow](#5-client-side-acceptance-flow)
6. [Data Model: Proposal Fields](#6-data-model-proposal-fields)
7. [Services, Price Types & Billing Rules](#7-services-price-types--billing-rules)
8. [Templates & Library System](#8-templates--library-system)
9. [Placeholders / Merge Fields](#9-placeholders--merge-fields)
10. [Internal Review & Approval Workflow](#10-internal-review--approval-workflow)
11. [Proposals Tab: List View & Management](#11-proposals-tab-list-view--management)
12. [Notifications & Reminders](#12-notifications--reminders)
13. [Branding & Customization](#13-branding--customization)
14. [Renewals](#14-renewals)
15. [Bulk Operations](#15-bulk-operations)
16. [User Roles & Permissions](#16-user-roles--permissions)
17. [Integrations Touchpoints](#17-integrations-touchpoints)
18. [Business Rules & Constraints](#18-business-rules--constraints)
19. [Edge Cases & Known Limitations](#19-edge-cases--known-limitations)
20. [Key Metrics & Success Indicators](#20-key-metrics--success-indicators)

---

## 1. Feature Overview

The **Proposal** is the core commerce object in Ignition. It is a structured, digitally signed client engagement document that combines:
- A **scope of work** (services offered, described with pricing)
- **Terms & conditions** (legal engagement letter)
- **Billing rules** (when and how each service is invoiced)
- **Payment method capture** (client card/bank details collected upfront at signing)
- **eSignature** (client signs electronically to accept)

The central value proposition: when a client signs a proposal, billing and payment collection **trigger automatically** — eliminating manual invoicing, chasing, and scope creep. The proposal is therefore simultaneously a sales document, a legal contract, and a billing instruction.

**Entry points to create a proposal:**
- Red `+` button (global, visible from anywhere in the app) → select "Proposal"
- From a client's profile page → "Create Proposal"
- From the Proposals tab → "+ New Proposal"
- From Deals (pipeline) → convert a Deal to a Proposal
- Bulk creation from a template via the Clients tab

---

## 2. User Personas

### 2.1 Firm Owner / Practice Principal
- **Who:** Accountant, CPA, or bookkeeper who owns the firm. Account creator and "Principal User" in Ignition.
- **Goals:** Win new engagements, standardize pricing, eliminate late payments, grow recurring revenue.
- **Proposal behaviors:** Sets up templates and service library; may review/approve proposals before sending; signs off on pricing strategy.
- **Pain points addressed:** Previously emailed PDFs or Word docs; chased payments manually; no audit trail.
- **Access level:** Administrator (full access to all settings, templates, services, payments, billing).

### 2.2 Senior Accountant / Partner
- **Who:** Senior staff who manages client relationships and creates proposals.
- **Goals:** Send professional proposals quickly; meet deadlines; minimize admin.
- **Proposal behaviors:** Creates proposals from templates; customizes intro/next-steps messages; sends to clients; monitors acceptance.
- **Access level:** Administrator or Member (depending on firm configuration).

### 2.3 Junior Staff / Team Member
- **Who:** Staff accountant, admin assistant, or bookkeeper on the team.
- **Goals:** Create draft proposals using standardized templates; hand off to a senior for review.
- **Proposal behaviors:** Drafts proposals; may be an assignee in the review workflow; limited ability to change templates or settings.
- **Access level:** Member (restricted — cannot access Settings, Service Library, Templates, Payments, or Subscription & Billing).

### 2.4 Client (End User — Proposal Recipient)
- **Who:** Business owner or finance person at the accounting firm's client company.
- **Goals:** Understand what they're agreeing to; sign easily; know when and how much they'll be charged.
- **Proposal behaviors:** Receives email with proposal link; reviews intro message, scope, and pricing; enters payment details; signs digitally.
- **Pain points addressed:** Previously received PDF invoices via email and paid by bank transfer days later; now signs and pays in one session on mobile or desktop.
- **Does NOT have an Ignition account** — interacts only via the client acceptance page (public URL).

### 2.5 Reviewer / Approver (Internal)
- **Who:** A partner or senior who is assigned to review a draft before it's sent to a client.
- **Goals:** Catch errors in pricing or scope; ensure compliance with firm standards.
- **Proposal behaviors:** Receives in-app notification when assigned; reviews, adds internal notes, approves or requests changes.
- **Access level:** Any user with access to Proposals (the review feature does not require a specific role beyond Proposals access).

---

## 3. Proposal Lifecycle & Statuses

A proposal progresses through the following statuses. Transitions are triggered by user actions or system events.

```
[Draft] ──────────────────────────────────────────────────────►[Lost]
   │                                                               ▲
   │ (review requested)                                            │
   ▼                                                               │
[In Review] ──► (approved) ──► back to [Draft] ──► (send/move)   │
                                    │                              │
                                    ▼                              │
                            [Awaiting Acceptance] ──► (mark lost) ┘
                                    │
                                    │ (client views)
                                    ▼
                                [Viewed]
                                    │
                                    │ (client signs)
                                    ▼
                                [Accepted]
                                    │
                                    │ (all services end / complete)
                                    ▼
                                [Completed]
```

| Status | Description | Who Triggers | Deletable? |
|---|---|---|---|
| **Draft** | Created but not yet sent. Fully editable. | Firm user (auto on creation) | Yes |
| **In Review** | Draft submitted for internal approval. | Firm user (manual) | No — must revoke to draft first |
| **Awaiting Acceptance** | Sent to client or moved live; not yet signed. | Firm user (send or move) | No — must revoke first |
| **Viewed** | Client has opened the proposal link. | System (auto, on client open) | No |
| **Accepted** | Client has signed and payment details captured. | Client (sign) | No |
| **Ending Soon** | Active proposal whose end date is within next 90 days. | System (auto) | No |
| **Completed** | All services on the proposal have ended/expired. | System (auto) | No |
| **Lost** | Manually marked as not proceeding. Applies from Draft or Awaiting Acceptance. | Firm user (manual) | Yes |

**Key rules:**
- A proposal that is Awaiting Acceptance must first be **Revoked** (→ returns to Draft) before it can be edited or deleted.
- Once Accepted, the proposal itself is **locked** — changes are made via editing the resulting Active Services, not the proposal.
- Only proposals in **Accepted** or **Completed** status can be Renewed.

---

## 4. Firm-Side Screen Flow (Proposal Editor)

The New Proposal Editor is a multi-step wizard with a persistent sidebar/stepper. Steps are navigated sequentially but can be revisited. The editor auto-saves drafts.

### 4.1 Step 1 — General Settings

**Purpose:** Identify who this proposal is for and set top-level engagement parameters.

**Fields:**

| Field | Type | Required | Notes |
|---|---|---|---|
| **Proposal Name** | Text | Yes | Internal only — clients never see this. Max ~100 chars. Used in Proposals tab list view. |
| **Client** | Lookup / Create | Yes | Search existing clients or create inline (requires: Client Name, Contact Name, Contact Email). Pre-filled if initiated from a client record. |
| **Contact** | Lookup (sub-field of Client) | Yes | The specific person who will receive and sign the proposal. A client can have multiple contacts. |
| **Effective Start Date** | Date picker | Yes | When the engagement begins. Sets the basis for date-relative billing rules. Displays as "On Client Acceptance" until accepted. Recommended: date when services start, or earlier. Format: YYYY-MM-DD. |
| **Minimum Contract Length** | Number + unit (months) | No | Sets a minimum term. Does NOT determine when billing ends (billing rules govern that). Services should start within this term but can continue beyond. |
| **Proposal Template** | Dropdown | No | Load a pre-built template to pre-populate services, billing, terms, and presentation. Two types: Provided (system) and My Templates (firm-custom). |

**Inline client creation fields** (when creating a new client from within the editor):

| Field | Required |
|---|---|
| Client Name (business name) | Yes |
| Contact Name (individual) | Yes |
| Contact Email | Yes |
| Phone, Address, ABN/Tax ID, etc. | No (optional, used by placeholders) |

**Actions available:**
- Edit Client (opens client record in-line for updating contact info)
- Save & Continue to next step

---

### 4.2 Step 2 — Services & Billing

**Purpose:** Define the scope of work — what services are being offered and how/when each will be billed.

**Concepts:**
- A proposal contains one or more **Service Groups**. Each group has one **Billing Rule** (i.e., all services in the group are billed at the same time and frequency).
- Each group contains one or more individual **Services** (from the Service Library).
- Services within a group can have different **Price Types** but share the same billing trigger.
- Optionally, groups can be organized under **Projects** (section headers for client-facing grouping).

**Adding a service:**
1. Click `+ Add Service`
2. Select service type (from Service Library — 40+ pre-loaded; firm can create custom)
3. Set Price Type (Fixed, Variable: Unit/Minimum/Price Range/Included)
4. Set price / rate
5. Assign to a Service Group (new or existing), which carries the Billing Rule

**Billing Rule fields per Service Group:**

| Field | Options | Notes |
|---|---|---|
| **Billing Type** | Recurring / One-off / Deposit | Top-level selection |
| **One-off: When** | On Acceptance / On Completion | On Acceptance = proposal start date; On Completion = proposal end date |
| **Recurring: Frequency** | # of days / weeks / months | e.g., every 1 month = monthly |
| **Recurring: Start** | On Acceptance / On Proposal Start Date | Optional delay (e.g., "+1 month") can be set for both |
| **Recurring: End** | No end date / After N periods / Specific date | Continuous billing available as option |
| **Deposit: Split** | Default 50%/50%; customizable percentage | Only available for Fixed Price, one-time services. Balance always manually billed. |
| **Billing Mode** | Automatic / Manual | Fixed Price only; Variable Price is always Manual. Automatic = system raises invoice when rule triggers with no further action. Manual = firm user must confirm before billing. |

**Per-service fields:**

| Field | Type | Notes |
|---|---|---|
| Service Name | Text (from library) | Can be renamed per-proposal |
| Service Description | Rich text | Displayed to client on proposal; supports formatting |
| Price Type | Dropdown | Fixed / Unit / Minimum / Price Range / Included |
| Price / Rate | Currency | Fixed: exact amount. Unit: per-unit rate. Min: floor amount. Range: min–max. Included: $0 / no price shown. |
| Unit Label | Text (for Unit Price type) | e.g., "per Hour", "per Employee", "per Return" — or custom |
| Optional toggle | Boolean | If ON, service appears as a client-selectable add-on (client can choose to include or exclude) |
| Service Terms | Rich text | Service-specific sub-terms; appended to the main terms template |
| Quantity | Number | Only shown for Unit Price; firm enters quantity at billing time |

**Projects (optional grouping):**
- Projects are section headers within the services list that group Service Groups visually
- Example uses: "Initial Setup Work" vs. "Ongoing Monthly Services"
- Client sees these headings on the pricing/scope page
- Do NOT affect billing logic — only presentation

**Manual Billing Mode:**
- Available for Fixed Price services
- System will NOT auto-raise an invoice; firm user must manually initiate billing from the Billing Hub
- Use case: when final scope or sign-off is needed before charging

---

### 4.3 Step 3 — Proposal Options (Tiered Packages)

**Purpose:** Allow a single proposal to present up to 3 alternative service packages (e.g., Basic / Standard / Premium), from which the client chooses one.

**Rules:**
- Maximum **3 options** per proposal
- Each option is a complete, independent set of services with its own billing rules
- One option can be marked as **Recommended** (visually highlighted to the client)
- Client selects one option, then proceeds to payment and signing
- Once an option is selected, the client can also select any **Optional add-on services** within that option

**Building options workflow:**
1. Build the first/ideal option in the Services step
2. In "Show proposal options" → `+ Add Option`
3. Duplicate the existing option → edit services/pricing for each variant
4. Toggle "Mark as recommended" on the preferred option
5. Client sees Option A / Option B / Option C on acceptance page with prices

**Fields per option:**
- Option name / label (e.g., "Essentials", "Growth", "Premium")
- Services list (same service fields as main proposal)
- Total price calculation (sum of all service groups in that option)

---

### 4.4 Step 4 — Presentation

**Purpose:** Configure the client-facing experience: the visual layout, branding, intro message, next-steps message, attached documents, and custom terms.

**Fields and settings:**

| Setting | Description |
|---|---|
| **Intro Message** | First "page" the client sees when opening the proposal. Supports rich text + video embed. Pulled from an Intro Message Template or written ad hoc. |
| **Next Steps Message** | Shown immediately after client accepts. Used for onboarding instructions, what to expect, bank statement descriptor note. Supports rich text + video embed. |
| **Terms Template** | The legal engagement letter/contract. Pulled from the Terms Template library. Liquid-based dynamic content supported. Placeholders auto-populate client/service details. |
| **Brochure / Attachment** | Optional PDF attached to the proposal. Can be set as a default in Branding settings (auto-attaches to all new proposals). |
| **Show Line Item Prices** | Toggle — if enabled, clients see individual service prices (not just totals). |
| **Show Proposal Minimum Value** | Toggle — shows the minimum total value of the engagement over its term. |
| **Email Notification Template** | Select which email template the client receives when the proposal is sent (Pro+ plans). Multiple templates supported. |
| **Expiry Date** | Optional date after which the proposal auto-expires. Drives urgency; included in reminder emails if set. |
| **Pricing Page Currency display** | Inherited from account settings (AUD, USD, GBP, CAD, NZD). |

**Live Preview:**
- Right-hand panel in the Presentation step shows a live, real-time preview of the client-facing proposal as settings change.
- The preview reflects branding colors, logo, service list layout, and messaging.

---

### 4.5 Step 5 — Automation / Workflow

**Purpose:** Configure post-acceptance automation — specifically, which jobs/projects should be created in connected practice management tools (e.g., Xero Practice Manager, Financial Cents, Karbon) when the proposal is accepted.

**Applicable integrations:** Xero Practice Manager (XPM), Karbon, Financial Cents, CCH Axcess, Thomson Reuters Onvio.

**Fields (XPM example):**

| Field | Description |
|---|---|
| **Job Name** | Name of the job to create in the connected tool |
| **Job Category** | Groups different jobs for reporting in the practice management tool |
| **Job Budget** | Sets an estimate/budget in the PM tool; eliminates a manual step post-acceptance |
| **Deploy Mode** | Automatic (default — deployed 1 day before job start date) or Manual (firm deploys per-proposal) |
| **Assigned Staff** | Team member(s) allocated to the job |

**Automation triggers:**
- When proposal is accepted → jobs created automatically in connected PM tool
- Billing trigger fires → invoice raised in connected accounting software (Xero, QBO)
- AutoCollect: invoices created externally in QBO/Xero can be pulled into Ignition for collection

---

### 4.6 Step 6 — Send

**Purpose:** Final review and dispatch to the client.

**Pre-send actions:**
- **Preview button** — opens a full client-perspective view of the proposal (separate browser tab) before sending
- Review contact/email address shown

**Send options:**

| Option | Behavior |
|---|---|
| **Send via Email** | Ignition emails the client with a branded email containing the proposal link. Standard flow. |
| **Move to Awaiting Acceptance** | Proposal goes live (link becomes active) without sending an email via Ignition. Useful if firm wants to share the link manually, via their own email, or SMS. |
| **Copy Link** | Copies the proposal URL for manual sharing |
| **CC recipients** | Adds read-only email recipients who receive a copy of the proposal email but CANNOT accept the proposal |
| **Resend / Revoke** | Available after sending; Revoke returns proposal to Draft status |

**Post-send state:**
- Proposal status → **Awaiting Acceptance**
- Audit trail records timestamp of: Created, Sent, Viewed (by client), Accepted
- Firm user sees the proposal in the client record with the audit trail

---

## 5. Client-Side Acceptance Flow

The client receives an email (branded with firm logo and colors) containing a button/link to their proposal. All interaction happens on a public web page — no account creation required.

### Screen 1 — Intro Page
- Firm's intro message (text + optional video)
- Firm logo and branding color
- "View Proposal" / Continue button

### Screen 2 — Scope / Services Page
- List of services to be provided
- Service descriptions
- Pricing displayed per service (if line-item pricing is enabled) and/or total
- If Proposal Options (A/B/C) are enabled: client selects one option
- If Optional add-on services exist: client can toggle them on/off
- Pricing updates dynamically as options are selected

### Screen 3 — Payment Schedule Page
- Full billing schedule showing each payment event:
  - **Billed on acceptance** — one-off immediate charge
  - **Recurring** — frequency, amount, start date
  - **Billed on completion** — charged when firm manually triggers
  - **Deposit** — upfront amount + balance-to-be-confirmed
  - **Price to be confirmed** — variable price service, amount TBD
  - **Catch-up billing** — if start date is in the past at time of signing, outstanding amounts shown in a **yellow box**

### Screen 4 — Secure Payment Details
- Firm can make payment method entry **required** or optional
- Supported payment methods:
  - **Credit / Debit Cards:** Visa, Mastercard, American Express (and others)
  - **Direct Debit / ACH:** Bank account details
  - **Digital Wallets:** Apple Pay, Google Pay
- Powered by Stripe
- Multiple proposals can use different payment methods — client selects per-proposal or uses saved method
- Client can also add/update payment details later via the Client Portal (link sent separately)
- Payment availability: Australia, Canada, New Zealand, United Kingdom, United States only

**Surcharge option (firm setting):**
- Firm can enable credit card surcharge pass-through — client who pays by card sees the processing fee added to their total

### Screen 5 — Terms & Conditions
- Full terms template rendered (Liquid-processed, placeholders filled)
- Client must scroll/review before signing
- Checkbox: "I agree to the terms and conditions"

### Screen 6 — eSignature
- Client enters their name and draws/types signature
- Up to **10 signatures** can be captured per proposal (for multi-signer engagements)
- "Accept Proposal" / "Sign & Accept" confirmation button

### Screen 7 — Thank You / Confirmation
- Customizable thank-you message (from General Settings → Library → Templates)
- Next Steps message displayed (from Presentation step)
- Optional video in Next Steps message

**Post-acceptance system actions (automatic):**
1. Proposal status → **Accepted**
2. Active Services created (one per service group) — these are now the live billing records
3. First automatic billing event fires (for services set to "On Acceptance" with Automatic billing mode)
4. Invoice created in connected accounting software (Xero / QBO) if integration is active
5. Automation/workflow jobs created in connected PM tool (if Automation step was configured)
6. Firm receives notification email / in-app notification
7. Client receives confirmation email

**Accepting on client's behalf (firm-side override):**
- Firm admin can accept a proposal on the client's behalf (e.g., verbal authorization)
- Requires entering a mandatory **Reason for Acceptance** (audit trail)
- Can select from saved payment methods on file for the client, or add new
- Produces same downstream effects as client self-signing

---

## 6. Data Model: Proposal Fields

This section enumerates all known data fields associated with a Proposal object, suitable for database schema design.

### Proposal Header

| Field | Data Type | Notes |
|---|---|---|
| `id` | UUID | Internal identifier |
| `name` | String | Internal name; not shown to client |
| `status` | Enum | Draft, InReview, AwaitingAcceptance, Viewed, Accepted, Completed, Lost, EndingSoon |
| `client_id` | FK → Client | |
| `contact_id` | FK → Contact | The signing contact |
| `created_by_user_id` | FK → User | |
| `assigned_reviewer_id` | FK → User | Nullable; set during review workflow |
| `effective_start_date` | Date | Null = On Acceptance |
| `minimum_contract_length` | Integer (months) | Nullable |
| `expiry_date` | Date | Nullable; proposal auto-expires if set |
| `terms_template_id` | FK → TermsTemplate | |
| `email_template_id` | FK → EmailTemplate | Nullable (defaults to account default) |
| `intro_message_template_id` | FK → MessageTemplate | |
| `next_steps_message_id` | FK → MessageTemplate | |
| `brochure_attachment_id` | FK → File | Nullable |
| `show_line_item_prices` | Boolean | |
| `show_minimum_value` | Boolean | |
| `payment_required` | Boolean | Whether client MUST enter payment method to accept |
| `surcharge_enabled` | Boolean | Pass credit card fees to client |
| `proposal_option_mode` | Boolean | Whether A/B/C options are enabled |
| `accepted_option_id` | FK → ProposalOption | Nullable; set on acceptance |
| `sent_at` | Timestamp | Nullable |
| `viewed_at` | Timestamp | Nullable (first client view) |
| `accepted_at` | Timestamp | Nullable |
| `completed_at` | Timestamp | Nullable |
| `lost_at` | Timestamp | Nullable |
| `lost_reason` | String | Nullable |
| `internal_note` | Text | Nullable; used in review workflow |
| `cc_recipients` | String[] | Read-only CC email addresses |
| `created_at` | Timestamp | |
| `updated_at` | Timestamp | |
| `account_id` | FK → Account | Multi-tenant identifier |

### Proposal Option (for A/B/C tiered packages)

| Field | Data Type | Notes |
|---|---|---|
| `id` | UUID | |
| `proposal_id` | FK → Proposal | |
| `label` | String | e.g., "Essentials", "Growth", "Premium" |
| `is_recommended` | Boolean | Highlighted to client |
| `sort_order` | Integer | Display order (1, 2, 3) |

### Service Group (Billing Rule container)

| Field | Data Type | Notes |
|---|---|---|
| `id` | UUID | |
| `proposal_id` | FK → Proposal | |
| `proposal_option_id` | FK → ProposalOption | Nullable (if not using options) |
| `project_id` | FK → Project | Nullable (optional visual grouping) |
| `billing_type` | Enum | Recurring, OneOff, Deposit |
| `one_off_trigger` | Enum | OnAcceptance, OnCompletion — null if Recurring or Deposit |
| `recurring_period_value` | Integer | e.g., 1 |
| `recurring_period_unit` | Enum | Day, Week, Month — null if not Recurring |
| `recurring_start_trigger` | Enum | OnAcceptance, OnStartDate — null if not Recurring |
| `recurring_start_delay_value` | Integer | Nullable; optional delay |
| `recurring_start_delay_unit` | Enum | Day, Week, Month — nullable |
| `recurring_end_type` | Enum | NoEnd, AfterNPeriods, SpecificDate — null if not Recurring |
| `recurring_end_n_periods` | Integer | Nullable |
| `recurring_end_date` | Date | Nullable |
| `continuous_billing` | Boolean | Allow billing to continue after end date |
| `billing_mode` | Enum | Automatic, Manual |
| `deposit_percentage` | Decimal | e.g., 0.50 = 50%; null if not Deposit |
| `sort_order` | Integer | |

### Proposal Service (line item)

| Field | Data Type | Notes |
|---|---|---|
| `id` | UUID | |
| `service_group_id` | FK → ServiceGroup | |
| `service_library_id` | FK → ServiceLibraryItem | Source template |
| `name` | String | Can be overridden per-proposal |
| `description` | RichText | Client-visible description |
| `service_terms` | RichText | Service-specific sub-terms |
| `price_type` | Enum | Fixed, UnitPrice, MinimumPrice, PriceRange, Included |
| `price_fixed` | Decimal | Null if not Fixed |
| `price_unit_rate` | Decimal | Null if not UnitPrice |
| `price_unit_label` | String | e.g., "per Hour"; null if not UnitPrice |
| `price_minimum` | Decimal | Null if not MinimumPrice or PriceRange |
| `price_maximum` | Decimal | Null if not PriceRange |
| `is_optional` | Boolean | Client can toggle on/off as add-on |
| `sort_order` | Integer | |

### Proposal Audit Event

| Field | Data Type | Notes |
|---|---|---|
| `id` | UUID | |
| `proposal_id` | FK → Proposal | |
| `event_type` | Enum | Created, Sent, Viewed, Accepted, Revoked, MarkedLost, ReviewRequested, ReviewApproved, ReviewChangesRequested, AcceptedOnClientBehalf, Renewed, Completed |
| `actor_type` | Enum | FirmUser, Client, System |
| `actor_id` | FK → User or null | |
| `notes` | Text | e.g., "Accepted on behalf: verbal authorization from call 2024-03-15" |
| `timestamp` | Timestamp | |

---

## 7. Services, Price Types & Billing Rules

### 7.1 Service Library

The Service Library is a firm-level catalog of reusable service definitions. All proposals draw services from this library (though names/descriptions can be overridden per-proposal).

**Pre-loaded example services (40+):** Accounts Payable, Accounts Receivable, Annual Tax Return, Audit Support, Bank Reconciliation, Bookkeeping, Business Activity Statement (BAS), Business Advisory, Cash Flow Forecasting, Company Tax Return, Financial Reporting, FBT Return, GST Filing, Individual Tax Return, Payroll Processing, Payroll Tax, SMSF Compliance, Superannuation, Tax Planning, etc.

**Service Library item fields:**

| Field | Notes |
|---|---|
| `name` | Display name |
| `description` | Default client-visible description (overridable per-proposal) |
| `default_price_type` | Preset type |
| `default_price` | Preset price (overridable per-proposal) |
| `default_billing_type` | Preset billing rule type |
| `service_terms` | Default sub-terms (overridable) |
| `category` | Optional grouping within the library |

### 7.2 Price Types Summary

| Price Type | Description | Billing Mode | Use Case |
|---|---|---|---|
| **Fixed** | Pre-determined amount shown to client | Auto or Manual | Tax return at $500; monthly bookkeeping at $800/mo |
| **Unit Price** | Rate × quantity (firm confirms quantity at billing) | Always Manual | Hourly work at $200/hr; payroll at $10/employee |
| **Minimum Price** | A floor amount; final bill may be more | Always Manual | "From $500" engagements |
| **Price Range** | Min–max bracket shown to client | Always Manual | "Estimated $800–$1,200" |
| **Included** | Service included at no charge (no price shown) | N/A | Add-on included in a bundle |

### 7.3 Billing Rule Types Summary

| Billing Type | Trigger | Recurs? | Use Case |
|---|---|---|---|
| **One-off: On Acceptance** | Proposal signed | No | Setup fee, one-time consultation |
| **One-off: On Completion** | Firm manually triggers | No | Project with deliverable at end |
| **Recurring** | Periodically per schedule | Yes | Monthly retainer, annual tax prep |
| **Deposit** | 50%/50% split: first on acceptance, balance on completion | No (2 events) | Large projects requiring upfront deposit |
| **Catch-up** | System-calculated back-billing when start date is in the past | No | Backdated engagement start |

---

## 8. Templates & Library System

Ignition has a multi-layered template system that separates reusable content from per-proposal customization.

### 8.1 Proposal Templates

**What they are:** A saved blueprint of a full proposal — pre-populated services, billing rules, terms, presentation settings, and messaging — that can be applied when creating a new proposal.

**Types:**
- **Provided templates:** Ignition-supplied out-of-the-box templates (e.g., "Bookkeeping Package", "Tax Preparation"). Anyone can use. Not editable.
- **My Templates (firm-custom):** Created and owned by the firm. Editable by Administrators only. Available on Core, Pro, Pro+, and Scale plans.

**Key behaviors:**
- When a provided template is used, the services inside the template are imported into the firm's Service Library automatically.
- Template name: max 40 characters. Template description: max 60 characters.
- Only Administrators can create, edit, duplicate, or delete My Templates.
- Members can use (apply) templates but cannot edit them.
- Using **relative dates** in billing rules (e.g., "On Acceptance", "+1 month") makes templates reusable year-over-year without editing start dates.

**Proposal Template fields:**
| Field | Notes |
|---|---|
| `name` | 40 char max |
| `description` | 60 char max |
| `services` | Full list of services with billing rules |
| `terms_template_id` | Default terms |
| `intro_message_template_id` | Default intro |
| `next_steps_message_id` | Default next steps |
| `presentation_settings` | Line item visibility, show minimum value, etc. |

### 8.2 Other Template Types

| Template Type | Location | Purpose |
|---|---|---|
| **Terms Templates** | Library → Terms | Legal engagement letters. Liquid-based. Placeholders supported. Multiple per account. |
| **Email Templates** | Library → Emails (Templates → Emails) | Proposal send email, reminder emails, etc. Multiple per type. |
| **Message Templates** | Library → Messages | Intro messages, Next Steps messages. Rich text + video. |
| **Notification Templates** | Settings → Notifications | In-app and email notification copy. |
| **Authority Templates** | Library | ATO-specific authority letters (STP, TPAR, BAS) — AU market. |

---

## 9. Placeholders / Merge Fields

Placeholders are dynamic merge tokens that auto-populate from the client record and proposal data. They function like mail merge fields.

**Syntax:** `{{placeholder_name}}`

**Supported contexts:** Message templates, Email templates, Terms templates, Service Terms, Notification templates. **NOT** supported in Service Descriptions.

**Visual feedback:**
- Placeholder with missing data → displayed in **yellow** (data gap warning)
- Placeholder with an error (invalid syntax) → displayed in **red** (renders nothing; must be removed)

**Common placeholder categories:**

| Category | Examples |
|---|---|
| **Client** | `{{client.name}}`, `{{client.address}}`, `{{client.abn}}`, `{{client.phone}}` |
| **Contact** | `{{contact.first_name}}`, `{{contact.last_name}}`, `{{contact.email}}` |
| **Proposal** | `{{proposal.name}}`, `{{proposal.start_date}}`, `{{proposal.minimum_contract_length}}` |
| **Firm** | `{{firm.name}}`, `{{firm.address}}`, `{{firm.phone}}`, `{{firm.signatory_name}}` |
| **Pricing (Classic only)** | `{{proposal_service.summary}}`, `{{proposal_price.summary}}` |

**Advanced: Liquid templating in Terms**

The Terms template engine supports Liquid (the templating language used by Shopify/Jekyll). Features:
- `{% if condition %}...{% endif %}` — conditional blocks for relevance-dependent clauses
- Loops over services
- Filters for formatting dates, currencies
- This allows a single terms template to cover multiple service scenarios (e.g., only show the "hourly billing" clause if there is a Unit Price service on the proposal)

---

## 10. Internal Review & Approval Workflow

Available for firms that want to gate proposal sending with a senior partner review before the client sees the proposal.

### Flow

```
Draft → [Request Review] → InReview (+ assignee + internal note)
                               │
              ┌────────────────┴────────────────┐
              │                                 │
       [Approve → back to Draft]       [Request Changes → back to Draft]
              │
       [Send proposal to client]
```

### Review Statuses (sub-states of Draft)

| Review State | Description |
|---|---|
| No review | Default draft state |
| **Review Requested** | Submitted for review; assignee notified |
| **Changes Requested** | Reviewer has requested edits; back in draft |
| **Approved** | Reviewer approved; ready to send |

### Key behaviors
- Any user can move a proposal to any review state (not gated by role)
- The review filter in the Proposals tab shows all proposals by review state
- Can be filtered by **Assignee** — useful bookmark: `Proposals → Review → Assignee = [self]`
- In-app badge notification appears for assignee when review is requested
- **Email notification is NOT currently sent to assignee** — only in-app notification
- Once proposal is Sent, Moved to Awaiting Acceptance, or Marked Lost — the review state is cleared
- Internal notes from the review flow are stored on the proposal (visible to all firm users, not clients)

---

## 11. Proposals Tab: List View & Management

The Proposals tab is the primary management screen for all proposals across all clients.

### View Structure

**Quick Filters (predefined preset views):**
- Awaiting Acceptance
- Viewed
- Accepted
- Ending Soon (end date within 90 days)
- Completed
- Review (custom filter — shows proposals in review workflow)
- Lost

**Table Columns (visible in list view):**
- Client Name
- Proposal Name
- Status (badge)
- Effective Start Date
- Assignee (if review workflow active)
- Created date / Sent date
- Total value

**More Filters (advanced — stackable):**
- Status
- Assignee
- Date range (sent, created, start date)
- Client
- Template used
- Custom views can be saved as browser bookmark URLs

**Sorting:** Sortable by Effective Start Date and other columns.

**Row Actions (⋮ menu, varies by status):**
- Draft: Edit, Duplicate, Delete, Mark as Lost, Request Review
- Awaiting Acceptance: Preview, Revoke, Mark as Lost, Resend
- Accepted: View, Renew, Edit Active Services
- Completed: View, Renew

**Export:**
- Click Export → generates CSV of current filtered view
- Delivered to user's email
- Fields include: Proposal name, client, status, start date, end date, services, billing totals

**Search:**
- Global search bar searches by Proposal name and Service name

---

## 12. Notifications & Reminders

### Proposal Reminders (for unsigned proposals)

**Purpose:** Automated emails sent to clients whose proposals are in Awaiting Acceptance status.

**Configuration (General Settings → Proposal Settings):**
- Enable/disable toggle
- Number of days between reminders (e.g., 3 days)
- Total number of reminder emails to send (best practice: 3 reminders, 3 days apart)
- If a proposal has been awaiting acceptance for fewer than 3 days at time of enabling, reminders begin once the day threshold is reached

**Content:** Same email template as the original proposal send email (subject line prefixed with "Reminder: ")

**Rules:**
- Reminders only sent for proposals in Awaiting Acceptance status
- Proposals with an Expiry Date: expiry date is included in reminder emails
- To stop reminders for a specific proposal without affecting others: Mark that proposal as Lost

### Firm-Side Notifications

**Notification levels (per user, per account):**
- **All clients and proposals** — notified about all activity
- **Clients and proposals I'm associated with** — only those where user is the assigned team member
- **Nothing** — all notifications off

**Notification triggers:**
| Event | Channel |
|---|---|
| Proposal Accepted | In-app + Email |
| Proposal Viewed (client opened) | In-app |
| Proposal Expired | In-app + Email |
| Payment failed | Email |
| Review Requested | In-app (no email) |
| Proposal Completed | In-app |

**Additional notification recipients:**
- Account-level setting: add extra email addresses to receive "Proposal Accepted" notifications (Settings → General → Proposal Settings)

**Weekly Summary Email:**
- Optional; toggle in notification settings
- Aggregated view of activity for the week

---

## 13. Branding & Customization

### Account-Level Branding (Settings → Branding)

| Setting | Description |
|---|---|
| **Logo** | Uploaded image; displayed top-left on all client-facing pages and emails |
| **Primary Brand Color** | Auto-detected from logo or manually set (HEX value). Used as accent color on acceptance page, buttons, and email headers. |
| **Default Brochure** | PDF auto-attached to every new proposal created. Can be overridden per-proposal. |
| **Custom Domain** | Proposals served from the firm's own domain (e.g., `proposals.youragency.com`). DNS CNAME setup required. Up to 48h propagation (usually ~1h). |

### Per-Proposal Overrides

- Brochure: can be changed or removed per-proposal in the Presentation step
- Intro/Next Steps messages: can be customized per-proposal in Presentation step
- Email template: can be selected per-proposal (on Pro+ plans, multiple templates available)
- Terms template: can be selected per-proposal

### Email Templates (customizable)

Available email template types:
- New Proposal (client receives when proposal is sent)
- Proposal Reminder (auto-sent while awaiting acceptance)
- Payment Request / AutoCollect
- Renewal notification
- Payment confirmation
- Failed payment notice

All support placeholders for dynamic client/firm name insertion. Subject line customizable.

---

## 14. Renewals

**Purpose:** Re-engage an existing client by creating a new proposal based on an accepted or completed previous proposal — commonly used for annual renewals of recurring engagements.

**Trigger:** Proposals whose end date is approaching appear in the "Ending Soon" filter (end date within 90 days).

**Renew workflow:**
1. From Proposals tab → select accepted/completed proposal → action: **Renew**
2. A new Draft proposal is created, pre-populated with the services and settings from the original
3. Firm can adjust pricing, terms, start/end date, and services before sending
4. **Bulk Price Update:** When renewing, the firm can update pricing across multiple services in bulk (launched June 2025):
   - Edit price in the "New Price" column
   - Search/filter specific services
   - See percentage increase per service
   - Progress tracker shows how many services have been updated

**Renewal features:**
- Ignition will remind you when agreements are due for a pricing review
- AI Price Insights (add-on): benchmark data surfaced within the renewal workflow to inform pricing decisions
- Supporting email templates for communicating price increases to clients
- Bulk renewal: select multiple proposals in the Proposals tab → **Renew** → creates multiple draft renewal proposals simultaneously

**Post-renewal:**
- Client receives the new proposal and must sign again (new eSignature captured)
- New payment method optionally re-confirmed
- Original proposal remains in Accepted/Completed status (history preserved)

---

## 15. Bulk Operations

### Bulk Proposal Creation (from Template)

**Use case:** Engage multiple clients at once (e.g., annual renewal season, price increase rollout).

**Flow:**
1. Clients tab → select clients (checkbox; can select all on page or all clients)
2. Click "+ Create Proposal" → select template from library → Next
3. Confirm/edit proposal details:
   - Proposal name (editable per client)
   - Proposal start date: "On Acceptance" or "Specific Date"
4. Update service pricing (bulk price override):
   - Edit per-service price in "New Price" column
   - Search/filter services
   - View percentage change
   - Top-left tracker shows progress
5. Generate → creates N individual Draft proposals (one per selected client)
6. Each can be individually reviewed/sent or sent in bulk

**Preparation requirements:**
- Each client must have a valid contact email
- Template must be defined in advance

### Bulk Renewals
- Proposals tab → select multiple proposals → Renew
- Creates renewal drafts in batch

### Bulk Price Updates (standalone, June 2025)
- Applied across active services (not just proposals) — adjusts ongoing billing amounts for multiple clients simultaneously
- Use: annual price increase across entire client base

---

## 16. User Roles & Permissions

### User Types

| Role | Access Scope | Proposal Permissions |
|---|---|---|
| **Principal User** | Everything | Full access; receives all system error/billing notifications; linked to account billing |
| **Administrator** | Everything including Settings, Service Library, Templates, Payments, Subscription & Billing | Create, edit, send, approve, delete proposals and templates; accept on client's behalf |
| **Member** | Proposals, Clients, Reporting (NOT Settings, Service Library, Templates, Payments, Subscription & Billing) | Create and edit proposals; cannot create or edit templates; cannot accept on client's behalf* |

*Capability varies by plan and specific configuration.

### Key Permission Differences

| Action | Administrator | Member |
|---|---|---|
| Create proposal | ✅ | ✅ |
| Edit Draft proposal | ✅ | ✅ |
| Use proposal templates | ✅ | ✅ |
| Create/edit/delete proposal templates | ✅ | ❌ |
| Send proposal to client | ✅ | ✅ |
| Revoke sent proposal | ✅ | ✅ (usually) |
| Mark proposal as Lost | ✅ | ✅ |
| Accept on client's behalf | ✅ | ❌ (usually) |
| Edit Service Library | ✅ | ❌ |
| Edit Terms Templates | ✅ | ❌ |
| Access Payment Settings | ✅ | ❌ |
| View Billing/Subscription | ✅ | ❌ |
| Assign reviewers | ✅ | ✅ (any user can assign) |
| Create/manage users | ✅ | ❌ |

**Invite behavior:**
- When adding a user, "Invite user to Ignition" toggle can be disabled — creates a staff record for workflow/assignment purposes without giving system login access

---

## 17. Integrations Touchpoints

The proposal feature is the trigger point for most downstream integration workflows.

### On Proposal Acceptance → Auto-triggered

| Integration | Action |
|---|---|
| **Xero** | Invoice created for billing events; payment synced when collected |
| **QuickBooks Online (QBO)** | Invoice created; payment synced |
| **Xero Practice Manager (XPM)** | Jobs created per configured workflow; staff assigned |
| **Karbon** | Client and Contact created (or linked); Practice job triggered |
| **Financial Cents** | Client/contact created or linked; project triggered |
| **Thomson Reuters Onvio** | Engagement created |
| **Wolters Kluwer CCH Axcess** | Engagement created |
| **Intuit ProConnect Tax** | Return and e-file status synced back to Ignition |
| **Slack** | Notification message posted to configured channel |
| **Zapier** | Triggers available: "Proposal Accepted", "Proposal Completed" — connect to 5,000+ apps |

### AutoCollect (reverse flow)

- Unpaid invoices created externally in QBO or Xero can be imported into Ignition
- Ignition collects payment via the client's stored payment method
- No separate proposal needed — works on existing client relationships

### Zapier Events (Proposal-related)

| Trigger | Description |
|---|---|
| Proposal Accepted | Fires when any proposal is accepted (New or Classic editor) |
| Proposal Completed | Fires when a proposal completes |
| New Client | Fires when a new client is created (often triggered by proposal creation) |
| Invoice Paid | Fires when payment collected |

---

## 18. Business Rules & Constraints

- A proposal must have at least **1 service** before it can be sent
- Proposal must have a **client with a valid contact email** before sending
- Maximum **3 proposal options** (A/B/C) per proposal
- Maximum **10 eSignatures** per proposal
- Deposit billing only available for **Fixed Price, one-time** services
- Variable Price services are always **Manual billing** — cannot be set to Automatic
- Once a proposal is **Accepted**, the proposal document is locked (cannot edit services, billing rules, or terms retroactively). Edits are made to **Active Services** (the live billing records derived from the accepted proposal)
- **Revoke** is the only way to edit a sent (Awaiting Acceptance) proposal — returns it to Draft; client link becomes inactive
- Catch-up billing is automatically calculated when a backdated start date is set and the proposal is accepted after the start date
- Payment processing only available in: **AU, CA, NZ, UK, US**
- The Principal User cannot be deleted (they are the account owner and billing contact)
- Template creation and management restricted to **Administrator** role
- Custom proposal templates only available on **Core, Pro, Pro+, Scale** plans (not Solo)
- Proposal Reminders apply to ALL awaiting-acceptance proposals globally; to exempt a specific proposal, it must be marked Lost

---

## 19. Edge Cases & Known Limitations

| Scenario | Behavior / Limitation |
|---|---|
| Client opens proposal on mobile | Fully responsive — no app install needed; Apple Pay / Google Pay available if device supports it |
| Client has no payment method and payment is required | Cannot accept proposal — blocked at payment step |
| Firm enables reminders when old proposals are already waiting | Reminders start for all awaiting-acceptance proposals that are >3 days old, immediately |
| Proposal start date in the past at time of signing | System calculates and displays "catch-up" billing in yellow box on payment schedule page |
| Placeholders reference empty client fields | Placeholder renders in yellow in the firm editor; client sees the placeholder token unfilled (firm should fix before sending) |
| Classic Proposal Editor vs. New Proposal Editor | Two co-existing editors; features described here are the New Editor. Classic editor is legacy. Proposals can't be migrated between editors. |
| Proposals without payment enabled | Client still signs but no card/bank details captured; firm must bill manually outside Ignition or use AutoCollect |
| Continuous billing: past end date | Services continue billing past the contractual end date if continuous billing is enabled; firm must manually stop |
| Bulk creation: client missing email | That client's proposal cannot be created/sent; skipped or shows error |
| Multiple contacts on one client | Proposal is linked to a specific contact; if contact changes, the proposal link may not reach the right person |
| Reviewing firm accepts on client's behalf | Audit reason required; stored in audit trail; legally, this means the firm is recording verbal/written authorization received outside Ignition |
| Terms template Liquid errors | If Liquid syntax is invalid, terms render partially or throw errors visible to firm in preview; not shown to client until preview confirms correct rendering |
| Email delivery failures | Ignition does not provide delivery/bounce tracking natively — firms should monitor client email deliverability separately |
| Surcharge pass-through | Not available in all regions; check Stripe/Ignition terms per country |

---

## 20. Key Metrics & Success Indicators

The following metrics are referenced by Ignition in marketing/product materials and indicate what the platform optimizes for. These are useful for defining KPIs in epic acceptance criteria.

| Metric | Reported Figure | Notes |
|---|---|---|
| Payment auto-collection rate | 91% of payments collected automatically | Platform-wide; 2025 |
| Reduction in late payments | 78% of customers report reduced late payments | 2025 customer survey |
| Reduction in scope creep | 85% of customers report reduced scope creep | 2025 customer survey |
| Total payment transactions | ~3.7M transactions processed | 2025 |
| Total revenue managed | $3.1B in 2025; $9B+ all-time | |
| Time to payment (anecdotal) | "Same day, some within 5–10 minutes" (vs. 1–2 weeks via EFT) | Customer testimonial |
| Proposal acceptance rate | Not publicly disclosed | Tracked per firm in account |
| Client relationships managed | ~900K in 2025; 1.9M+ total | |

---

*Sources: [ignitionapp.com](https://www.ignitionapp.com), [support.ignitionapp.com Help Center](https://support.ignitionapp.com), [ignitionapp.com/product/online-proposal-management](https://www.ignitionapp.com/product/online-proposal-management), [ignitionapp.com/how-it-works](https://www.ignitionapp.com/how-it-works), [ignitionapp.com/blog/2025-ignition-product-launch](https://www.ignitionapp.com/blog/2025-ignition-product-launch), [G2 Ignition Reviews](https://www.g2.com/products/ignition/reviews), [Capterra Ignition Listing](https://www.capterra.com/p/140861/Practice-Ignition/) — compiled February 2026.*
