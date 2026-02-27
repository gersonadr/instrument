# Feature Research: Automated Billing
> Source: ignitionapp.com — official product, help center (support.ignitionapp.com), learning center, Stripe documentation, and regulatory frameworks.
> Purpose: Reference document for a planner agent writing software development epics.

---

## Table of Contents

1. [Feature Overview](#1-feature-overview)
2. [How Billing Consent is Captured](#2-how-billing-consent-is-captured)
   - 2.1 [Proposal Acceptance as the Billing Mandate](#21-proposal-acceptance-as-the-billing-mandate)
   - 2.2 [What the Client Consents To](#22-what-the-client-consents-to)
   - 2.3 [Payment Method Capture at Signing](#23-payment-method-capture-at-signing)
3. [Payment Methods by Country](#3-payment-methods-by-country)
4. [Stripe Integration Architecture](#4-stripe-integration-architecture)
   - 4.1 [How Ignition Stores Payment Data](#41-how-ignition-stores-payment-data)
   - 4.2 [PCI DSS Scope & Tokenization](#42-pci-dss-scope--tokenization)
   - 4.3 [Stripe Connect vs. Direct Integration](#43-stripe-connect-vs-direct-integration)
5. [Billing Types: Automatic vs. Manual](#5-billing-types-automatic-vs-manual)
6. [Billing Engine: How Invoices Are Generated](#6-billing-engine-how-invoices-are-generated)
   - 6.1 [Billing Schedule Tab (Per-Client View)](#61-billing-schedule-tab-per-client-view)
   - 6.2 [Billing Hub (Cross-Client View)](#62-billing-hub-cross-client-view)
   - 6.3 [Payment Collection Timing Settings](#63-payment-collection-timing-settings)
7. [Collections Pipeline](#7-collections-pipeline)
   - 7.1 [Collections Tab: States and Queues](#71-collections-tab-states-and-queues)
   - 7.2 [Invoice Lifecycle](#72-invoice-lifecycle)
8. [Failed Payment Handling & Retry Logic](#8-failed-payment-handling--retry-logic)
   - 8.1 [Automatic Retry](#81-automatic-retry)
   - 8.2 [Manual Retry & Rescheduling](#82-manual-retry--rescheduling)
   - 8.3 [Client Self-Service Retry](#83-client-self-service-retry)
9. [Client Notifications](#9-client-notifications)
   - 9.1 [All Billing Email Events](#91-all-billing-email-events)
   - 9.2 [Per-Client Email Recipients Configuration](#92-per-client-email-recipients-configuration)
   - 9.3 [Upcoming Payment Notifications](#93-upcoming-payment-notifications)
10. [Firm Notifications](#10-firm-notifications)
11. [AutoCollect: Importing Unpaid External Invoices](#11-autocollect-importing-unpaid-external-invoices)
12. [Instant Bill: Ad Hoc Billing Without a Proposal](#12-instant-bill-ad-hoc-billing-without-a-proposal)
13. [Review + Pay: Manual Payment Collection Link](#13-review--pay-manual-payment-collection-link)
14. [Accounting Software Sync](#14-accounting-software-sync)
15. [Transaction Fees & Surcharge Pass-Through](#15-transaction-fees--surcharge-pass-through)
    - 15.1 [Processing Fee Structure](#151-processing-fee-structure)
    - 15.2 [Surcharge Pass-Through to Client](#152-surcharge-pass-through-to-client)
    - 15.3 [How Fees Are Billed to the Firm](#153-how-fees-are-billed-to-the-firm)
16. [Legal & Regulatory Compliance by Payment Method](#16-legal--regulatory-compliance-by-payment-method)
    - 16.1 [US: ACH / NACHA Rules](#161-us-ach--nacha-rules)
    - 16.2 [UK: Bacs Direct Debit Mandate](#162-uk-bacs-direct-debit-mandate)
    - 16.3 [AU/NZ: Direct Debit Rules](#163-aunz-direct-debit-rules)
    - 16.4 [CA: PAD (Pre-Authorized Debit)](#164-ca-pad-pre-authorized-debit)
    - 16.5 [EU/UK: PSD2 & Strong Customer Authentication (SCA)](#165-euuk-psd2--strong-customer-authentication-sca)
    - 16.6 [Chargebacks & Dispute Protection](#166-chargebacks--dispute-protection)
17. [Data Model: Billing Objects](#17-data-model-billing-objects)
18. [Business Rules & Constraints](#18-business-rules--constraints)
19. [Edge Cases & Known Limitations](#19-edge-cases--known-limitations)
20. [Key Metrics](#20-key-metrics)

---

## 1. Feature Overview

Ignition's automated billing system is the back half of its core value loop: the **proposal captures consent and payment details upfront** so that every subsequent billing event fires automatically — no manual invoice creation, no payment chasing, no reconciliation lag.

The system handles:
- **Invoice generation** — triggered automatically by billing rules defined in the accepted proposal
- **Payment collection** — Stripe charges the stored payment method when the invoice is due
- **Accounting sync** — invoices and payments pushed to Xero / QuickBooks / MYOB in real time
- **Failure handling** — automatic retry, firm notifications, and client self-service recovery
- **Ad hoc billing** — Instant Bill and AutoCollect extend billing capability beyond proposal-driven services

**Core design principle:** The firm never touches the billing cycle for automatically billed services. Once a proposal is accepted, the machine runs. The firm only intervenes for manually-billed items (variable-price services that need confirmation before invoicing).

---

## 2. How Billing Consent is Captured

### 2.1 Proposal Acceptance as the Billing Mandate

The proposal acceptance flow serves as the **legally binding billing authorization mandate**. When a client accepts a proposal in Ignition, they are simultaneously:

1. Signing the engagement letter (terms and conditions)
2. Agreeing to the scope of services
3. **Authorizing Ignition to debit their stored payment method per the billing schedule**

This single act covers what would otherwise require a separate recurring billing authorization form. The signed proposal PDF — which includes the payment schedule and payment authority section — is the auditable record of this consent.

**What the signed PDF records as payment authority:**
- The payment method type the client has authorized (credit card / direct debit / ACH)
- The billing schedule (amounts, frequency, start/end dates)
- The trigger conditions for each service group
- The Ignition Terms of Use acceptance (which includes Ignition's payment collection terms)
- The date, time, and IP address of the authorization (same as e-signature capture)

### 2.2 What the Client Consents To

By accepting the proposal, the client specifically consents to:

| Authorization | Description |
|---|---|
| **Recurring charges** | Ignition may automatically charge the stored payment method on the dates specified in the billing schedule, at the agreed amounts |
| **On-acceptance charges** | For services billed on acceptance, the charge may occur immediately or within the payment collection window set by the firm |
| **Variable price confirmation** | For variable-price services (Unit, Minimum, Range), the client acknowledges that the actual amount will be confirmed by the firm before billing |
| **Deposit + balance** | For deposit billing, the initial % is charged on acceptance; the balance will be charged at a later date confirmed by the firm |
| **Future price updates** | When a renewal proposal is accepted, the new pricing supersedes the previous authorization |
| **Ignition Terms of Use** | Client agrees to Ignition's platform payment terms, which govern data handling, refund policies, and dispute procedures |

### 2.3 Payment Method Capture at Signing

Payment details are collected at the **Secure Payment step** of the client acceptance flow (step 4 of 7), before the signature is captured. This ordering is critical: payment details must be on file before the signature triggers downstream billing.

**Payment entry UI (client-facing):**
- Hosted by Stripe (Stripe.js / Stripe Elements) — the card fields render inside an iframe that originates from Stripe's servers
- Ignition's servers never see or handle raw card numbers or bank account details
- The client enters their details directly into Stripe's hosted fields

**Payment method storage:**
- Stripe creates a `PaymentMethod` object (tokenized reference to the card/bank account)
- Stripe stores the raw payment details in its PCI-compliant vault
- Ignition stores only the Stripe `PaymentMethod ID` (a token like `pm_1ABC...`) and metadata: last 4 digits, card brand, expiry, payment type
- The token is reused for every future billing event on that client's account
- Clients can add multiple payment methods and designate one per proposal (or per service group in some configurations)

**When payment entry is optional:**
- The firm can toggle "Payment required" OFF on the proposal
- In this case, the client can proceed through signing without entering payment details
- Billing items will remain unpaid until: the client adds payment details later (via Client Portal or payment method request email), or the firm uses Review + Pay / AutoCollect to collect manually

---

## 3. Payment Methods by Country

Ignition payments are powered by Stripe and available in five countries. Each country supports specific payment methods.

| Country | Credit/Debit Cards | Bank Debit | Digital Wallets |
|---|---|---|---|
| **Australia (AUD)** | Visa, Mastercard, Amex, Discover, JCB, UnionPay | **Direct Debit** (bank account) | Apple Pay, Google Pay |
| **United States (USD)** | Visa, Mastercard, Amex, Diners Club, Discover, JCB, UnionPay | **ACH Debit** (bank account) | Apple Pay, Google Pay |
| **United Kingdom (GBP)** | Visa, Mastercard, Amex, Diners Club (Intl), Discover | **Bacs Direct Debit** | Apple Pay, Google Pay |
| **Canada (CAD)** | Visa, Mastercard, Amex, Discover, JCB, UnionPay | **PAD (Pre-Authorized Debit)** | Apple Pay, Google Pay |
| **New Zealand (NZD)** | Visa, Mastercard, Amex, Discover, JCB | **Direct Debit** | Apple Pay, Google Pay |

**Not currently supported:** All other countries (including South Africa). Single currency per account — multi-currency not supported.

**Processing time differences:**
- Credit / Debit Card: typically **1–2 business days** to payout
- Direct Debit / ACH / PAD / Bacs: typically **3–7 business days** to payout (varies by country and bank)

**Important behavior for bank debit methods:**
- For direct debit: payment receipt email to client is sent **after bank confirmation** (3–5 business days), not on submission
- For credit card: payment receipt is sent **immediately** on successful charge
- This means the invoice may show as "Collecting" for several days for bank debit clients

---

## 4. Stripe Integration Architecture

### 4.1 How Ignition Stores Payment Data

Ignition uses Stripe as an exclusive payment processor. The integration architecture follows the **tokenization model**:

```
Client enters card/bank details
         │
         ▼ (directly to Stripe via Stripe.js iframe — Ignition servers never see raw data)
[Stripe] Creates PaymentMethod object → returns PaymentMethod ID token
         │
         ▼
[Ignition DB] Stores:
  - stripe_customer_id (Stripe Customer object)
  - stripe_payment_method_id (token reference)
  - payment_method_type (card / us_bank_account / bacs_debit / acct_debit / pad)
  - card_last4, card_brand, card_expiry (display metadata only)
  - billing_details (name, email, address — for receipt purposes)
```

**When billing fires:**
```
[Billing Engine] Billing rule trigger met → Invoice created in Ignition
         │
         ▼
[Ignition → Stripe API] Creates PaymentIntent or charges Customer via saved PaymentMethod
         │
         ▼
[Stripe] Processes charge → webhook sent to Ignition on success/failure
         │
         ▼
[Ignition] Invoice status updated → accounting software synced → notifications sent
```

### 4.2 PCI DSS Scope & Tokenization

**Ignition's PCI DSS posture:**
- Ignition does **not** store, process, or transmit raw cardholder data (PANs, CVVs, full bank account numbers)
- All card capture uses **Stripe Elements** (hosted iframes) — raw card data flows directly from client browser to Stripe's PCI DSS-validated servers
- Ignition systems are **outside the cardholder data environment (CDE)** for PCI purposes
- Ignition's PCI scope is reduced to **SAQ A** (the lightest self-assessment questionnaire) or equivalent for merchant customers
- Stripe holds **PCI DSS Level 1** certification (the highest tier — for processors handling >6M transactions/year)

**What Ignition stores vs. what Stripe stores:**

| Data | Ignition DB | Stripe Vault |
|---|---|---|
| Full card number (PAN) | ❌ Never | ✅ Encrypted at rest (AES-256) |
| CVV / CVC | ❌ Never | ❌ Not retained even by Stripe post-auth |
| Card expiry | Display metadata only (MM/YY) | ✅ Full |
| Last 4 digits | ✅ (display only) | ✅ |
| Card brand | ✅ (display only) | ✅ |
| Full bank account number | ❌ Never | ✅ Tokenized |
| Bank account routing number | ❌ Never | ✅ |
| PaymentMethod token | ✅ (Stripe `pm_xxx` ID) | ✅ |
| Customer token | ✅ (Stripe `cus_xxx` ID) | ✅ |

### 4.3 Stripe Connect vs. Direct Integration

Ignition operates as a **Stripe Connect platform**. This means:

- Ignition has a Stripe platform account
- Each Ignition firm has a **Stripe Connected Account** linked to their Ignition account
- Payment collection flows through: Client → Ignition's Stripe Connect platform → firm's Connected Account
- Funds are disbursed to the firm's bank account; processing fees are collected separately in arrears
- This architecture allows Ignition to facilitate payments on behalf of many firms without each firm needing to independently manage Stripe integration

**Firm onboarding:**
- Firms must complete Stripe identity verification (Know Your Customer / KYC) as part of enabling Ignition Payments
- First payment can take up to **7 business days** after account is fully verified
- Stripe handles KYC, anti-money-laundering (AML) checks, and regulatory compliance for each connected firm

---

## 5. Billing Types: Automatic vs. Manual

Every billing item in Ignition is classified as either **Automatic** or **Manual**. This classification determines whether Ignition fires the invoice and charge without firm intervention.

| Dimension | Automatic | Manual |
|---|---|---|
| **Invoice generation** | System generates invoice when trigger fires | Firm user must click "Invoice Now" or "Schedule Invoice" |
| **Payment collection** | Stripe charged automatically per payment collection schedule | Firm must schedule collection or client must click "Pay Now" / "Review + Pay" |
| **When used** | Recurring billing, On-Acceptance billing with Fixed Price | On-Completion billing, Variable-Price services, Deposit balance, any service the firm wants to confirm before charging |
| **Billing Mode in proposal** | Fixed Price + Automatic | Fixed Price + Manual, OR any Variable Price type |
| **Overdue badge** | N/A (system handles it) | Yellow "Overdue" badge appears if billing item was not invoiced by its expected date |
| **Firm action needed** | None (fully hands-off) | User must action each billing item in Billing Schedule → Billed Manually section |

**Switching between modes:**
- A firm can flip an upcoming auto-billing item to manual (to pause it) and vice versa
- Done via: client file → Billing Schedule → select billing item → change mode
- Useful for: client disputes, scope changes, seasonal pauses, or partial invoicing

---

## 6. Billing Engine: How Invoices Are Generated

### 6.1 Billing Schedule Tab (Per-Client View)

The Billing Schedule tab is the per-client view of all future billing. It is divided into two sections:

**Billed Automatically (top section):**
- All upcoming invoices that the system will generate without firm action
- Shows: service name, amount, next billing date, billing frequency
- Sorted by upcoming date
- Firm can: view, pause (flip to manual), or modify the next billing date

**Billed Manually (bottom section):**
- All billing items awaiting firm action before invoicing
- Each item has a Billing Date (the date the invoice should be issued)
- Yellow **Overdue** badge appears when the billing date has passed without the firm issuing the invoice
- Overdue items float to the top of the manual section
- Available actions per item: Invoice Now, Schedule Invoice (future date), Edit Price/Quantity, Skip (mark as not billable)

**Invoice Now flow (for manual items):**
1. Select billing item(s) in Billed Manually section
2. Click "Invoice Now" → review dialog appears
3. Review service name, quantity, and price (editable for variable-price services)
4. Click "Issue Invoice" → invoice is created
5. If payment method is on file: Ignition schedules payment collection per the payment collection schedule
6. If no payment method: invoice is created but flagged as outstanding (firm must collect separately)

**Editing a billing item:**
- Firms can edit: service name, price, quantity, billing date (reschedule)
- Cannot change billing type (Recurring/One-off) after proposal is accepted — must create a new proposal or service for that

### 6.2 Billing Hub (Cross-Client View)

The Billing Hub is the firm-level command center for all manual billing items across all clients.

**Columns shown:**
- Client name
- Service name
- Amount
- Billing date (scheduled or overdue)
- Status badge (upcoming / overdue)
- Proposal reference

**Available actions from Billing Hub (⋮ menu per row):**
- Invoice Now
- Schedule Invoice
- Skip billing item

**Filtering / searching:**
- Filter by billing date range
- Filter by overdue status
- Search by client name or service name
- Useful for end-of-month batch invoicing runs

### 6.3 Payment Collection Timing Settings

Settings → Payments → Payment Collection Schedule controls the delay between invoice creation and payment collection attempt.

| Setting | Description |
|---|---|
| **Automatic Payment Terms** (days) | Days after invoice creation before Stripe charges the stored payment method. Default: 0 (same day). Common: 0–3 days. |
| **Manual Payment Terms** (days) | Days after invoice creation before the invoice is considered "due" (shown in Xero/QBO as due date). Common: 7–14 days. |
| **Same Day** | Setting both to 0 = collect payment on the same day invoice is raised |

**Effect on invoice due dates in accounting software:**
- Automatic Terms controls: when the scheduled payment appears in Collections → Scheduled
- Manual Terms controls: the "due date" field on invoices exported to Xero / QuickBooks

**Changing the collection date per-invoice:**
- Can be done on a per-invoice basis from the invoice detail view
- Options: Collect Now, Collect on Original Terms, Collect on a Future Date

---

## 7. Collections Pipeline

### 7.1 Collections Tab: States and Queues

The Collections tab is the real-time view of all payment transactions across the firm's entire client base.

**Sub-tabs (queues):**

| Queue | Contents |
|---|---|
| **Scheduled** | Invoices with a payment method and a scheduled collection date (not yet attempted) |
| **Collecting** | Charges submitted to Stripe and awaiting bank confirmation (primarily for bank debit methods: 3–7 days) |
| **Paid** | Successfully collected payments |
| **Failed** | Charges rejected by the client's bank or card issuer |
| **Outstanding** | Invoices without a payment method or with payment not yet scheduled — includes AutoCollect-imported invoices |

**Progress status bar:**
- Each row in Collections shows a visual progress bar (grey = not started, green = progressing, complete = full green)
- Steps represented: Invoice Created → Payment Scheduled → Collecting → Paid

**Columns in Collections tab:**
- Client Name
- Invoice Reference / Description
- Amount
- Billing Date
- Scheduled Collection Date
- Estimated Payout Date
- Status badge
- Source (Ignition-created or Ledger-imported)

**Export:**
- Full CSV export of the current Collections view

### 7.2 Invoice Lifecycle

```
[Billing Rule Trigger Met]
        │
        ▼
[Invoice Created] ─── Status: Issued
        │
        │ (payment method on file + auto billing mode)
        ▼
[Payment Scheduled] ─── Status: Scheduled
        │
        │ (after Payment Collection Schedule delay)
        ▼
[Charge Submitted to Stripe] ─── Status: Collecting
        │
        ├──── [SUCCESS] ─────────────────────────────► [Paid] → Receipt to client
        │                                                      → Sync to accounting software (marked paid)
        │                                                      → Payout to firm bank account
        │
        └──── [FAILURE] ─────────────────────────────► [Failed]
                                                              → Email to client (failed payment)
                                                              → Email to firm notification recipients
                                                              → Auto-retry queued (3 business days, if enabled)
```

**For bank debit methods, "Collecting" state:**
- Charge is submitted but bank confirmation takes 3–7 business days
- Invoice shows as "Collecting" during this window
- If bank returns a failure (NSF, account closed, etc.) during this window, invoice moves to Failed
- Receipt email sent to client only after successful bank confirmation

---

## 8. Failed Payment Handling & Retry Logic

### 8.1 Automatic Retry

**Default behavior:** Enabled by default on all Ignition accounts.

**Retry rule:**
- 1 automatic retry
- Triggered **3 business days** after the initial failure
- Retry uses the **same payment method** that failed
- If the retry also fails: invoice returns to Failed; no further automatic retries
- Total automatic attempts: 2 (initial + 1 retry)

**Configuring auto-retry:**
- Settings → Payments → Auto-retry payments toggle
- Enable or disable globally (applies to all clients)
- Cannot be set per-client or per-invoice

**Retry audit trail:**
- Each retry attempt is logged in the invoice's Activity section
- Visible in: invoice detail view → scroll to Activity section
- Shows: timestamp, attempt number, outcome

**Retry in Collections tab:**
- While retry is pending: appears in Collections → Scheduled (with a future date = original failure date + 3 business days)
- If retry succeeds: moves to Paid
- If retry fails: returns to Failed

### 8.2 Manual Retry & Rescheduling

After a failed payment (and after auto-retry has also failed, or if auto-retry is disabled):

**Firm-side manual retry options:**
1. Navigate to Collections → Failed
2. Click into the failed payment record → drawer opens
3. Options:
   - **Reschedule → Collect Now:** Attempts to charge immediately
   - **Reschedule → Original Terms:** Collects on the original payment terms schedule
   - **Reschedule → Future Date:** Pick a specific date for the next attempt
4. Once rescheduled, invoice moves from Failed → Scheduled

**Changing payment method for retry:**
1. Go to client record → payment method settings
2. Update or add a new payment method (or request client to update via Client Portal)
3. Assign the new payment method to the failed invoice
4. Then reschedule the collection

### 8.3 Client Self-Service Retry

**From the failed payment email (client receives):**
- Email subject: "Action required: Your payment to [Firm Name] failed"
- Email contains:
  - Amount that failed
  - Invoice description
  - CTA button: "View payment details" or "Retry payment"
- Clicking the link takes the client to a Stripe-hosted or Ignition-hosted payment page
- Client options on that page:
  1. **Retry with same payment method** — attempts charge immediately
  2. **Use a different payment method** — enter new card/bank details; new method is saved and charge attempted

**If client updates payment method:**
- New method is stored in Stripe and linked to the client record in Ignition
- Firm receives notification that client has retried (email to notification recipients)
- New payment method can be used for future billing

---

## 9. Client Notifications

### 9.1 All Billing Email Events

The following emails are sent to the client at various billing lifecycle events. All emails use the firm's branding (logo, primary color, firm name, custom domain if configured).

| Event | Trigger | Client Receives |
|---|---|---|
| **Proposal sent** | Firm sends proposal | Email with proposal link; contains billing schedule summary |
| **Proposal accepted confirmation** | Client accepts proposal | Confirmation email + signed PDF attachment (includes full payment schedule) |
| **Upcoming payment notification** | Configurable days before billing date | Preview of upcoming charge (amount, date, service description) |
| **Invoice created (manual billing)** | Firm issues invoice | Invoice email with PDF link and "Pay Now" button (if no saved payment method) |
| **Payment receipt (card)** | Stripe confirms card charge | Receipt with amount, date, service description, and invoice PDF link |
| **Payment receipt (bank debit)** | Bank confirms debit (3–5 days after submission) | Same as card receipt, but delayed by bank confirmation window |
| **Payment failed** | Stripe webhook: charge.failed | Notification with amount, description, CTA to retry or update payment method |
| **Auto-retry scheduled** | Auto-retry queued after failure | "Your payment failed; we'll try again in 3 business days" |
| **Payment method request** | Firm sends payment method request | Link to Client Portal to add payment details (for clients with no saved method) |
| **Payment method added confirmation** | Client adds payment method via portal | Confirmation that payment details have been saved |

### 9.2 Per-Client Email Recipients Configuration

The email addresses that receive billing notifications are configurable per-client (not just per-contact).

**How it works:**
- Client record → Email Preferences tab
- All contacts added under the client's Details tab can be nominated as billing notification recipients
- Firm selects which contacts receive: invoices, payment receipts, and/or other billing emails
- At least **one email recipient is required** — cannot have zero billing recipients
- Additional non-contact email addresses can also be added (e.g., a client's accountant, bookkeeper, or accounts payable email — useful for Receipt Bank / Hubdoc integration)

**Use case:** A business client might want invoices sent to their accounts@company.com inbox (not the personal email of the signing contact).

### 9.3 Upcoming Payment Notifications

**Special compliance status:** Upcoming payment notification emails **cannot be disabled** for UK clients — this is a regulatory requirement under Bacs Direct Debit rules (10 working days advance notice for changes; general advance notice is best practice).

**For all other countries:** Upcoming payment notifications are optional and configurable.

**Configuration:**
- Settings → General → Proposal Settings → Upcoming Payment Notifications
- Set the number of days before billing date to send the notification

---

## 10. Firm Notifications

The following events generate notifications to firm-side users:

| Event | Channel | Recipients |
|---|---|---|
| **Proposal accepted** | Email | Configured notification recipients (Settings → General → Proposal Settings) + default recipients |
| **Payment collected** | In-app | User(s) associated with the client |
| **Payment failed** | Email + In-app | Notification email addresses (Settings → Payments → Notifications Email) |
| **Payment retried by client** | Email | Notification email addresses |
| **Payment method added by client** | Email | Notification email addresses |
| **Billing item overdue** | In-app | Badge on Billing Hub + overdue filter |

**Firm notification email configuration:**
- Settings → Payments → Notifications Email
- One or more email addresses
- Separate from the proposal notification recipients (though can be the same addresses)

---

## 11. AutoCollect: Importing Unpaid External Invoices

**What it is:** AutoCollect allows firms to pull unpaid invoices **created outside of Ignition** (in Xero or QuickBooks Online) into Ignition's payment collection pipeline — without creating a new proposal.

**Primary use case:** Firms that:
- Have existing client relationships with invoices already in their accounting software
- Want to collect payment via Ignition's payment methods without rebuilding the whole engagement as a proposal
- Need to collect on ad hoc billable hours, expenses, or out-of-scope work invoiced directly in Xero/QBO

**Setup:**
1. Navigate to Apps tab → select the connected ledger (Xero or QBO)
2. Open Invoice Settings → enable "Sync unpaid invoices" toggle → Save
3. Ignition immediately begins importing qualifying invoices

**Import rules:**
- Only imports invoices in **Issued / Approved + Awaiting Payment** status
- Only imports invoices billed within the **last 90 days**
- Ignition does **not** import draft, void, or fully paid invoices
- Xero sync frequency: every **1–2 hours**
- QuickBooks sync frequency: **daily** (not real-time)
- First sync after enabling may take several hours

**Manual import option:**
- Apps tab → select ledger → click Import in Invoices section
- Firm can review all unpaid ledger invoices and selectively import chosen ones

**Where imported invoices appear:**
- Collections → Outstanding tab (alongside Ignition-originated invoices)
- Use Source filter (Ledger vs. Ignition) to distinguish

**Collecting on imported invoices:**
- **If client has a saved payment method:** Firm can click Schedule → choose collection date → Ignition charges the stored Stripe method automatically
- **If no payment method on file:** Firm sends a "Request Payment" to the client (individual or bulk)
  - Client receives email → clicks link → enters payment details on Ignition-hosted page → can opt-in to auto-collect for future invoices
  - Pay Now button on invoice: client can pay one-time without saving details

**Disabling AutoCollect:**
- Toggle off stops future imports; already-imported invoices remain visible
- To remove an imported invoice: mark it as paid

---

## 12. Instant Bill: Ad Hoc Billing Without a Proposal

**What it is:** A lightweight way to create and send a one-off or recurring invoice to a client without going through the full proposal flow.

**Use cases:**
- Out-of-scope work discovered mid-engagement
- Quick ad hoc charge that doesn't warrant a new proposal
- Services provided outside the normal engagement cadence

**How it works:**
1. Click `+` button (global) → "Instant Bill"
2. Select client
3. Select service(s) from library (or enter free-text description)
4. Set price and billing type (one-off or recurring)
5. Confirm and submit

**Billing behavior:**
- Uses the same payment collection schedule as automatic billing
- If client has a saved payment method → Stripe charges automatically per the schedule
- If no payment method → invoice sent to client with Pay Now link

**Accounting sync:**
- Invoice appears in Xero / QBO the same way as proposal-generated invoices

---

## 13. Review + Pay: Manual Payment Collection Link

**What it is:** A payment link mechanism for collecting on invoices where the client has no saved payment method (or where the firm wants to offer one-time payment without storing details).

**How to generate:**
- Client record → Invoices tab → select an invoice → Enable Pay Now button
- Or send a "Review + Pay" email from the invoice

**Client experience:**
- Client receives an email with a "Pay Invoice" link
- Clicks through to a secure Ignition-hosted payment page
- Reviews the invoice
- Enters payment details (one-time) — or opts in to save for automatic future payments
- Pays

**Opt-in to automatic future billing:**
- On the Review + Pay page, client can check "Use this payment method for future invoices automatically"
- If checked: payment method is saved to their Stripe customer record and linked to all future billing items
- This brings them into the same automated pipeline as clients who entered payment details at proposal acceptance

---

## 14. Accounting Software Sync

Ignition syncs billing and payment events bidirectionally with connected accounting software.

### Invoice Sync (Ignition → Accounting Software)

| Event in Ignition | Action in Accounting Software |
|---|---|
| Invoice generated (auto or manual) | Invoice created in Xero / QBO / MYOB |
| Invoice marked as paid | Invoice status updated to Paid |
| Payment partially collected | Partial payment recorded against the invoice |
| Invoice voided / cancelled | Invoice voided |

### Payment Status Sync (Accounting Software → Ignition)

| Event in Accounting Software | Action in Ignition |
|---|---|
| Invoice marked as paid in Xero / QBO | Corresponding Ignition invoice updated to Paid |
| Invoice deleted in Xero / QBO | Ignition invoice flagged (does not auto-delete) |

**Configuration options per integration (example: Xero):**
- Default contact for invoices (map Ignition client to Xero contact)
- Default account code for services
- Default tax code
- Invoice reference format (e.g., include proposal name, date)
- Whether to send invoices to Xero before or after payment collection
- Whether Ignition should create Xero invoices for manual billing items (or only auto-billing)

**Billing assignment levels:**
- **Client level:** All invoices for a client map to a single Xero job/contact
- **Proposal level:** Individual proposals map to specific Xero jobs — allows multiple billing streams per client with separate reconciliation

**Xero Practice Manager (XPM) integration:**
- Jobs created in XPM (on proposal acceptance) can be linked to billing at the job level
- Invoices are associated with the job for time/cost tracking

---

## 15. Transaction Fees & Surcharge Pass-Through

### 15.1 Processing Fee Structure

Fees vary by country and payment method. Stripe's underlying rates plus Ignition's margin are blended into a single per-transaction fee charged to the firm.

**Credit / Debit Card:**

| Country | Rate Range | Notes |
|---|---|---|
| **Australia** | ~1.3% – 2.9% + AUD fee | Standard vs. premium card rates differ |
| **United States** | ~1.5% – 3.6% + $0.30 | Premium US card rate: 3.6% + $0.30 |
| **United Kingdom** | ~1.5% – 3.5% + £0.20 | Varies by card type |
| **Canada** | ~1.5% – 3.5% + CAD fee | Varies by card type |
| **New Zealand** | ~1.5% – 2.9% + NZD fee | Varies by card type |

> Card type (standard vs. premium/business) is unknown until the transaction settles — Ignition/Stripe cannot determine in advance which rate will apply.

**Bank Debit (ACH / Direct Debit / PAD / Bacs):**
- Starting from **~1% + $0.30** per transaction
- **Capped at $5 (USD) / £4 (GBP)** per transaction
- An additional **0.3%** applies to payments above $3,000 / £2,000

**No fees for:**
- Failed transactions (no charge on unsuccessful attempts)
- Chargebacks / NSF returns (no chargeback fee from Ignition)

### 15.2 Surcharge Pass-Through to Client

Firms can elect to pass credit card processing fees to clients rather than absorbing them.

**Availability:** United States, Canada, Australia only. Not available in UK or NZ.

**How it works:**
1. Firm enables surcharging in Settings → Payments → Surcharges
2. When client selects credit card as payment method: a surcharge fee is added to the invoice total (displayed to client before they confirm)
3. Client sees: base invoice amount + surcharge amount + total
4. Surcharge amounts:
   - **Australia:** Maximum 2.5%
   - **Canada:** Maximum 2.4%
   - **United States:** Maximum 4% (option to exclude debit cards from surcharging)
5. Surcharge is applied as a credit against the firm's processing fees in Ignition billing

**Direct debit / ACH / PAD / Bacs cannot be surcharged.**

**Refunds with surcharging:** If a surcharged payment is refunded, the surcharge fee is also returned to the client.

### 15.3 How Fees Are Billed to the Firm

- Ignition **deposits 100% of the invoice total** to the firm (gross settlement — not net-of-fees)
- Processing fees are collected **separately, in arrears**, charged to the firm's Ignition subscription credit card
- This means: client invoices reconcile cleanly in accounting software (no fee deduction from invoice amount)
- Firms can see their fee accruals in Settings → Billing

---

## 16. Legal & Regulatory Compliance by Payment Method

### 16.1 US: ACH / NACHA Rules

**What is required:**

| Requirement | Implementation in Ignition |
|---|---|
| Written Proof of Authorization (POA) | Proposal acceptance = the written authorization. The signed proposal PDF (including payment authority section) serves as the POA document |
| "Clear and readily understandable terms" | Payment schedule displayed to client in step 3 of acceptance flow before payment details are entered |
| Recurring authorization disclosure | Billing schedule shows each future recurring event, amount, and dates |
| Identity verification | Signatory email + IP address + typed name captured at signing |
| Record retention (2 years post last transaction) | Signed PDF stored permanently; audit trail stored in Ignition |
| 7-day notice for date changes | Upcoming payment notification emails; firms should also notify clients before rescheduling |
| 10-day notice for amount changes | Renewal proposal with updated pricing; client must re-sign |
| Revocation mechanism | Client can request cancellation; firm marks proposal/services as complete and removes payment method |

**NACHA WEB transaction rules (internet-initiated debit):**
- Ignition's ACH collection qualifies as WEB transactions (initiated via internet/online)
- WEB requires: validation of bank account information (Stripe performs micro-deposit verification or instant verification via Plaid for ACH setup)
- Secure transmission: all data transmitted over TLS/HTTPS

### 16.2 UK: Bacs Direct Debit Mandate

**What is required:**

| Requirement | Implementation in Ignition |
|---|---|
| Signed Direct Debit mandate (DDI) | Client enters bank details via Stripe's Bacs setup flow; this creates a Bacs mandate with Bacs/bank |
| Sort code + Account number + Name + Address | Collected via Stripe's hosted Bacs setup element |
| Advance notice (10 working days before first collection) | Ignition's Upcoming Payment notification emails; non-disableable in UK |
| Advance notice before any change | New amount/date changes require firm to send notice; Upcoming Payment notifications serve this purpose |
| Direct Debit Guarantee disclosure | Displayed on Stripe's hosted payment page at time of mandate setup |
| Customer's right to immediate cancellation | Client can cancel mandate via their bank; Ignition processes removal of payment method on firm's action |

**Direct Debit Guarantee (client protection):**
- If an error is made in payment amount or date: immediate refund available from the bank
- If payment is taken without advance notice: immediate refund available
- Firm must be a Bacs-approved Service User (handled via Stripe's Bacs setup)

**Upcoming payment notification is mandatory in UK and cannot be disabled.**

### 16.3 AU/NZ: Direct Debit Rules

**Australian framework:**
- Governed by APCA (Australian Payments Council) and the ePayments Code
- Customer authorization is required before debiting; proposal acceptance + payment method entry = authorization
- Customer can dispute unauthorized debits within 120 days
- 14-day advance notice for changes to amount or frequency is best practice (not always mandated for B2B)

**New Zealand framework:**
- Similar to AU; governed by Payments NZ direct debit rules
- Authorization obtained at proposal signing
- Advance notice for changes

### 16.4 CA: PAD (Pre-Authorized Debit)

**Canadian framework:**
- Governed by Payments Canada Rule H1 (Pre-Authorized Debits)
- Written PAD agreement required before first debit
- Must include: amount (or method of determining amount), timing/frequency, payor's right to revoke, process for handling errors
- Ignition's proposal acceptance (including payment schedule display + payment details entry) constitutes the PAD agreement
- Payor can cancel PAD at any time with written notice
- 10-business-day notification required before first debit in some cases
- Records retained for 7 years (longer than NACHA's 2 years)

### 16.5 EU/UK: PSD2 & Strong Customer Authentication (SCA)

**What SCA requires:**
- For card payments in EU/EEA: 2 of 3 authentication factors required: Something you know (PIN, password), Something you have (mobile device OTP), Something you are (biometric)
- Stripe handles 3D Secure (3DS) authentication to comply with SCA for card payments

**Recurring payment exemptions:**
- SCA is only required on the **first payment** in a recurring series
- Subsequent merchant-initiated transactions (MITs) are **exempt from SCA** if flagged correctly
- This is why Ignition (via Stripe) requires the initial card setup to go through SCA authentication — all future automatic billing is then exempt

**How Stripe handles this for Ignition:**
1. Initial card capture: Stripe triggers 3DS challenge if issuer requires it (client may see 2FA prompt)
2. Card saved with SCA-compliant authorization token
3. Future charges: flagged as MITs (off-session, merchant-initiated) → SCA exemption applies → no client friction

**Low-value exemption:**
- Transactions under €30 / £25 may be exempt from SCA
- Banks can apply this exemption up to 5 times consecutively or until cumulative amount exceeds €100/£85

### 16.6 Chargebacks & Dispute Protection

**What Ignition provides for dispute evidence:**
- Signed proposal PDF (proof of agreement)
- Audit trail (proof of when client viewed and accepted)
- IP address and timestamp of acceptance (geographic/identity context)
- Email correspondence trail (proposal sent, viewed, accepted emails)
- Payment schedule page (proof client was shown billing terms before signing)

**What the firm should maintain:**
- Records of communications authorizing the work
- Delivery evidence for services rendered
- Any modification agreements (renewal proposals or amendment emails)

**Chargeback handling:**
- Chargebacks are processed by Stripe; firm is notified
- Firm must submit dispute evidence to Stripe within the dispute window
- Ignition does not charge NSF or chargeback fees on top of Stripe's dispute fees

**Direct Debit Guarantee (UK):**
- Provides clients an automatic refund right for unauthorized or erroneous debits
- Firm must be registered as a Bacs Service User via Stripe's setup — Stripe handles this requirement

---

## 17. Data Model: Billing Objects

### Invoice

| Field | Data Type | Notes |
|---|---|---|
| `id` | UUID | |
| `account_id` | FK → Account | Multi-tenant |
| `client_id` | FK → Client | |
| `proposal_id` | FK → Proposal | Nullable (null for Instant Bill or AutoCollect) |
| `service_group_id` | FK → ServiceGroup | The billing rule that generated this invoice |
| `billing_item_id` | FK → BillingItem | The specific billing item being invoiced |
| `source` | Enum | ProposalBilling, InstantBill, AutoCollect (Ledger Import) |
| `status` | Enum | Draft, Issued, Scheduled, Collecting, Paid, Failed, Void |
| `amount` | Decimal | Invoice amount in account currency |
| `currency` | String (ISO 4217) | e.g., "USD", "AUD" |
| `invoice_date` | Date | Date invoice was issued |
| `due_date` | Date | Based on manual payment terms settings |
| `collection_date` | Date | Scheduled Stripe charge date |
| `invoice_reference` | String | Reference number (formatted per firm settings) |
| `line_items` | JSON | Array of service line items with descriptions and amounts |
| `ledger_invoice_id` | String | Xero/QBO invoice ID (after sync) |
| `ledger_status` | Enum | Synced, Pending, Error |
| `payment_method_id` | FK → PaymentMethod | The stored method used/to be used for collection |
| `stripe_payment_intent_id` | String | Stripe PaymentIntent ID |
| `stripe_charge_id` | String | Stripe Charge ID |
| `paid_at` | Timestamp | When payment confirmed |
| `failed_at` | Timestamp | When last failure occurred |
| `retry_count` | Integer | Number of attempts (0 = not yet attempted, 1 = attempted once, 2 = retried) |
| `next_retry_at` | Timestamp | Scheduled auto-retry timestamp |
| `created_at` | Timestamp | |
| `updated_at` | Timestamp | |

### PaymentMethod

| Field | Data Type | Notes |
|---|---|---|
| `id` | UUID | |
| `client_id` | FK → Client | |
| `stripe_customer_id` | String | Stripe `cus_xxx` ID |
| `stripe_payment_method_id` | String | Stripe `pm_xxx` ID |
| `type` | Enum | Card, USBankAccount, BacsDebit, AUBECSDebit, CABankAccount, NZBankAccount |
| `card_brand` | String | e.g., "visa", "mastercard" — null if not card |
| `card_last4` | String | e.g., "4242" — null if not card |
| `card_expiry_month` | Integer | null if not card |
| `card_expiry_year` | Integer | null if not card |
| `bank_last4` | String | Last 4 digits of bank account — null if not bank debit |
| `bank_sort_code` | String | UK sort code — null if not Bacs |
| `bank_routing_number` | String | US routing number — null if not ACH |
| `billing_name` | String | Cardholder / account holder name |
| `billing_email` | String | |
| `is_default` | Boolean | Whether this is the default method for new billing |
| `created_at` | Timestamp | |

### BillingItem

| Field | Data Type | Notes |
|---|---|---|
| `id` | UUID | |
| `service_group_id` | FK → ServiceGroup | Parent billing rule |
| `proposal_id` | FK → Proposal | |
| `client_id` | FK → Client | |
| `billing_mode` | Enum | Automatic, Manual |
| `billing_date` | Date | When this item should be invoiced |
| `status` | Enum | Upcoming, Overdue, Invoiced, Skipped |
| `amount` | Decimal | Pre-set amount (may be null for variable price — confirmed at invoice time) |
| `invoice_id` | FK → Invoice | Null until invoiced |
| `is_overdue` | Boolean | Computed: billing_date < today and status = Upcoming |
| `overdue_since` | Date | Nullable |

### PaymentEvent (audit log for billing)

| Field | Data Type | Notes |
|---|---|---|
| `id` | UUID | |
| `invoice_id` | FK → Invoice | |
| `event_type` | Enum | InvoiceCreated, PaymentScheduled, CollectionSubmitted, PaymentSucceeded, PaymentFailed, RetryScheduled, RetryAttempted, RefundIssued, Void |
| `stripe_event_id` | String | Stripe webhook event ID |
| `amount` | Decimal | Amount involved in this event |
| `failure_code` | String | Stripe failure code (e.g., "insufficient_funds") |
| `failure_message` | String | Human-readable failure reason |
| `actor` | Enum | System, FirmUser, Client, Stripe |
| `actor_id` | UUID | Nullable — user ID if actor = FirmUser |
| `timestamp` | Timestamp | |
| `metadata` | JSON | Additional event-specific data |

---

## 18. Business Rules & Constraints

- Automated billing only fires if: (a) the proposal is in Accepted status, (b) the billing item has a saved payment method, and (c) the billing mode is Automatic
- An invoice is never automatically created for a variable-price (Unit, Minimum, Range) service — these are always Manual billing mode by definition
- The balance of a Deposit billing is **always Manual** regardless of the main billing mode setting — this is a system safeguard against accidental billing of the final balance
- Payment collection cannot be scheduled for an invoice that has no payment method attached — invoice enters Outstanding state instead
- Changing a client's default payment method does NOT retroactively update already-scheduled invoices — only future invoices use the new method
- Automatic retry fires once, 3 business days after failure — there is only one automatic retry attempt; all subsequent retries must be manually triggered
- Surcharging (pass-through) is not available for bank debit payment methods under any circumstances
- Ignition deposits 100% of invoice amounts gross; fees are collected separately — this is by design for clean accounting reconciliation
- Firms must maintain at least one notification email address for payment failure notifications (Settings → Payments)
- Each client must have at least one email recipient set for invoice and receipt notifications
- The first payment after enabling Ignition Payments for a firm may take up to 7 business days (Stripe's initial payout hold for new connected accounts)
- Multi-currency is not supported — a single Ignition account operates in one currency only
- AutoCollect imports invoices from the last 90 days only; older invoices must be manually imported

---

## 19. Edge Cases & Known Limitations

| Scenario | Behavior / Implication |
|---|---|
| Card expires between proposal acceptance and a future billing date | Stripe may decline the charge; invoice moves to Failed; client receives failed payment email and must update card. Firm is also notified. Card expiry is not proactively flagged to firms in advance. |
| Client cancels their bank account (ACH/Direct Debit) | Next debit attempt returns a "bank account closed" failure code; invoice moves to Failed. Firm must request new payment details from client. |
| Client disputes a charge with their bank (chargeback) | Stripe notifies firm via webhook; invoice status may update; firm must submit dispute evidence within Stripe's window (typically 7–14 days). |
| Bank debit "collecting" state: 5-day window passes without confirmation | Stripe will eventually return the debit status (succeeded or failed). If bank returns NSF after several days, invoice moves to Failed despite having been "collecting" for days. |
| Invoice amount is $0 (e.g., an "Included" service is the only item) | System should skip payment collection (nothing to charge). This edge case requires explicit handling — zero-amount invoices should be created for accounting sync purposes but not submitted to Stripe. |
| Client removes payment method from portal while invoices are scheduled | Scheduled invoices will fail when Stripe attempts to charge a detached payment method. Firm should re-attach a payment method or reschedule with a new method. |
| Firm changes invoice amount after it is scheduled but before collection date | Stripe PaymentIntent must be updated or cancelled and recreated with new amount. Accounting software invoice must also be updated. |
| QuickBooks sync latency (daily) | AutoCollect-imported QBO invoices may lag up to 24 hours. Real-time collection is not available for QBO as it is for Xero (1–2 hours). |
| Stripe Connect payout failure (firm's bank account invalid) | Payments are still collected from clients but payouts to the firm fail. Stripe notifies the firm. Funds accumulate in the connected account until bank details are corrected. |
| Surcharge display on mobile | The surcharge amount must be clearly displayed on the payment confirmation step before the client confirms. Mobile rendering must ensure visibility. |
| Proposal accepted from a country where Ignition Payments is not supported | If the firm's account country is one of the 5 supported, payments work regardless of where the client is located. The key is the firm's account country, not the client's location. |
| Repeat AutoCollect: same invoice imported twice | System should deduplicate by ledger invoice ID. If the same invoice ID is already imported, do not create a duplicate. |

---

## 20. Key Metrics

| Metric | Value | Source |
|---|---|---|
| % payments auto-collected | **91%** | Ignition 2025 platform data |
| Payment transactions processed (2025) | **~3.7 million** | Ignition 2025 platform data |
| Total revenue processed (2025) | **$3.1 billion** | Ignition 2025 announcement |
| % customers reporting reduced late payments | **78%** | Ignition 2025 customer survey |
| Automatic retry policy | 1 retry, 3 business days after failure | Ignition Help Center |
| Max automatic retry attempts | 2 (initial + 1 retry) | Ignition Help Center |
| Credit card payout time | ~1–2 business days | Ignition documentation |
| Bank debit payout time | ~3–7 business days | Ignition documentation |
| First payment hold (new firm account) | Up to 7 business days | Ignition documentation |
| AutoCollect import window | Last 90 days of unpaid invoices | Ignition Help Center |
| Xero sync frequency (AutoCollect) | Every 1–2 hours | Ignition Help Center |
| QuickBooks sync frequency (AutoCollect) | Daily | Ignition Help Center |
| NACHA record retention requirement | 2 years post last transaction | NACHA Rules |
| CA PAD record retention requirement | 7 years | Payments Canada Rule H1 |
| SCA: recurring exemption applies after | 1st SCA-authenticated payment | Stripe / PSD2 |

---

*Sources: [support.ignitionapp.com — Setting up Payments](https://support.ignitionapp.com/en/articles/3767726-setting-up-ignition-payments), [support.ignitionapp.com — Payments Overview](https://support.ignitionapp.com/en/articles/600819-payments-overview), [support.ignitionapp.com — Billing Schedule Tab](https://support.ignitionapp.com/en/articles/4373185-the-billing-schedule-tab), [support.ignitionapp.com — Collections Tab](https://support.ignitionapp.com/en/articles/9458405-the-collections-tab), [support.ignitionapp.com — Automatic Retries](https://support.ignitionapp.com/en/articles/9625741-automatic-payment-collection-retries), [support.ignitionapp.com — Failed Payments](https://support.ignitionapp.com/en/articles/1091256-how-to-handle-failed-payments), [support.ignitionapp.com — Surcharges](https://support.ignitionapp.com/en/articles/8264575-passing-on-card-processing-fees-to-clients-surcharges), [support.ignitionapp.com — Client Emails](https://support.ignitionapp.com/en/articles/1332439-client-emails), [support.ignitionapp.com — AutoCollect](https://support.ignitionapp.com/en/articles/11362280-autocollect-automate-invoice-payments), [support.ignitionapp.com — Payment Fees](https://support.ignitionapp.com/en/articles/9489207-how-payment-fees-are-calculated), [Stripe — ACH Direct Debit](https://docs.stripe.com/payments/ach-direct-debit), [Stripe — Bacs Direct Debit](https://docs.stripe.com/payments/payment-methods/bacs-debit), [Stripe — SCA Guide](https://stripe.com/guides/strong-customer-authentication), [Stripe — PCI DSS](https://stripe.com/guides/pci-compliance), [NACHA — ACH Authorization Requirements](https://www.nacha.org), [Payments Canada Rule H1 — PAD](https://www.payments.ca) — compiled February 2026.*
