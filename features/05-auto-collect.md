# Feature Research: AutoCollect
> Source: ignitionapp.com — official product, help center (support.ignitionapp.com), Ignition newsroom, and third-party coverage of the launch announcement (May 8, 2025).
> Purpose: Reference document for a planner agent writing software development epics.

---

## Table of Contents

1. [Feature Overview and Purpose](#1-feature-overview-and-purpose)
2. [Accounting Software Integration Scope](#2-accounting-software-integration-scope)
   - 2.1 [Supported Platforms](#21-supported-platforms)
   - 2.2 [Eligible Invoice Statuses](#22-eligible-invoice-statuses)
   - 2.3 [Invoice Fields Imported](#23-invoice-fields-imported)
   - 2.4 [Credit Notes, Partial Payments, and Prepayments](#24-credit-notes-partial-payments-and-prepayments)
3. [Invoice Import Mechanics](#3-invoice-import-mechanics)
   - 3.1 [Import Trigger Modes](#31-import-trigger-modes)
   - 3.2 [Sync Schedule](#32-sync-schedule)
   - 3.3 [Filtering Invoices](#33-filtering-invoices)
   - 3.4 [Post-Import Modifications in Ledger](#34-post-import-modifications-in-ledger)
   - 3.5 [Invoices Already Paid in Ledger](#35-invoices-already-paid-in-ledger)
   - 3.6 [Bulk Import and Bulk Actions](#36-bulk-import-and-bulk-actions)
   - 3.7 [Rate Limits](#37-rate-limits)
4. [Client and Payment Method Matching](#4-client-and-payment-method-matching)
   - 4.1 [Client Matching Logic](#41-client-matching-logic)
   - 4.2 [No Payment Method on File](#42-no-payment-method-on-file)
   - 4.3 [Bulk Payment Method Requests](#43-bulk-payment-method-requests)
5. [Payment Collection Mechanics](#5-payment-collection-mechanics)
   - 5.1 [When Payment Is Charged](#51-when-payment-is-charged)
   - 5.2 [Payment Terms and Due Dates](#52-payment-terms-and-due-dates)
   - 5.3 [Scheduling Future Collection](#53-scheduling-future-collection)
   - 5.4 [Collection Engine Shared with Proposal Billing](#54-collection-engine-shared-with-proposal-billing)
6. [Fee Structure](#6-fee-structure)
   - 6.1 [Transaction Fees](#61-transaction-fees)
   - 6.2 [Active Client Quota](#62-active-client-quota)
   - 6.3 [Plan Tier Restrictions](#63-plan-tier-restrictions)
7. [Reconciliation and Accounting Sync](#7-reconciliation-and-accounting-sync)
   - 7.1 [How Ignition Marks Invoices as Paid](#71-how-ignition-marks-invoices-as-paid)
   - 7.2 [Xero Reconciliation](#72-xero-reconciliation)
   - 7.3 [QuickBooks Online Reconciliation](#73-quickbooks-online-reconciliation)
   - 7.4 [Void Sync Direction](#74-void-sync-direction)
8. [Status Tracking and Collections Dashboard](#8-status-tracking-and-collections-dashboard)
   - 8.1 [UI Location](#81-ui-location)
   - 8.2 [Collections Sub-Tabs](#82-collections-sub-tabs)
   - 8.3 [Actions per Invoice](#83-actions-per-invoice)
9. [Failed Payments and Retry Logic](#9-failed-payments-and-retry-logic)
10. [Limitations and Edge Cases](#10-limitations-and-edge-cases)
11. [Relationship to Other Ignition Features](#11-relationship-to-other-ignition-features)
12. [Data Model: AutoCollect-Relevant Objects](#12-data-model-autocollect-relevant-objects)
13. [Research Gaps](#13-research-gaps)

---

## 1. Feature Overview and Purpose

**Official branding:** AutoCollect (also written "auto collect" in older help articles)
**Public launch:** May 8, 2025
**Tagline (at launch):** "End the business chase for late payments"

### Problem Statement

AutoCollect addresses a fundamental gap in Ignition's payment automation: prior to its launch, Ignition could only collect payments on invoices that originated from an accepted Ignition proposal. Any invoice billed directly from Xero, QuickBooks Online, or a practice management system — without a preceding Ignition proposal — was outside Ignition's reach.

According to Ignition's own research cited at launch: 94% of accountants and bookkeepers report needing to chase clients for late payments; in the US, over half of small businesses are regularly paid late. CEO Greg Strickland: *"We need to stop the cycle of service businesses having to negotiate twice; first for the contract, and then to get paid."*

### End-to-End Workflow (Firm Perspective)

1. **Connect accounting software** — Xero or QuickBooks Online connected under the Apps tab.
2. **Enable sync** — Apps tab → open ledger tile → Invoice Settings → toggle on "Sync unpaid invoices" → Save.
3. **Automatic import** — Ignition continuously polls the ledger and imports qualifying invoices (Issued/Approved + Awaiting Payment + billed within last 90 days) without further firm action. Xero polls every 1–2 hours; QBO polls daily.
4. **Review in Outstanding tab** — Payments → Collections → Outstanding. Filter Source = "Ledger" to isolate AutoCollect invoices from proposal-originated ones.
5. **Collect payment** — Two paths:
   - **Client has no payment method on file:** firm sends a payment request email (individually or in bulk); client follows link to Ignition client portal, adds a payment method, and pays.
   - **Client has a payment method on file:** firm schedules collection (sets a date) from the Outstanding tab; Ignition charges the stored method on that date.
6. **Automatic opt-in for future invoices** — when a client pays via the portal, they can opt in to automatic payment, authorising future imported invoices to be charged automatically.
7. **Reconciliation** — within 24 hours of the payout clearing the firm's bank, Ignition marks the ledger invoice as paid in Xero/QBO via the standard clearing account mechanism.

### Client Experience

- Client receives an email with a copy of the invoice and a "Pay Now" link.
- Link opens the Ignition client portal (`go.ignitionapp.com/`).
- Client adds a payment method (card or bank account/direct debit per region) or uses one already on file.
- Payment is processed immediately on submission.
- Client receives a receipt email.
- Client can opt in during payment to authorise automatic future collection.

---

## 2. Accounting Software Integration Scope

### 2.1 Supported Platforms

| Platform | Sync Type | Sync Frequency | Regions |
|---|---|---|---|
| **Xero** | Automatic API sync | Every 1–2 hours | US, AU, UK, NZ, CA, Philippines |
| **QuickBooks Online (QBO)** | Automatic API sync | Daily | US, AU, UK, CA |
| **Practice CS** (Thomson Reuters) | Manual Excel file upload | On-demand only | US only |
| **CCH Axcess** | Import-based (API announced Apr 2025) | Not publicly documented | US only |

The help center explicitly groups these as "connected apps," and the import flow for Practice CS requires invoices to be uploaded manually in bulk via spreadsheet — there is no webhook or polling connection.

### 2.2 Eligible Invoice Statuses

**Confidence: Confirmed**

Only invoices meeting **all** of the following criteria are imported:

| Criterion | Value |
|---|---|
| Ledger status | Issued **or** Approved |
| Payment status | Awaiting Payment (not partially paid, not voided, not draft) |
| Age | Billed within the last **90 days** |
| Currency | Single currency only (multi-currency not supported) |

In Xero terminology, "Issued or Approved" = "Authorised." Draft invoices, voided invoices, and fully paid invoices are excluded.

### 2.3 Invoice Fields Imported

**Confidence: Inferred from UI display and reconciliation documentation**

| Field | Source | Notes |
|---|---|---|
| Invoice number / reference | Ledger | Used for matching and reconciliation |
| Client name | Ledger | Primary field for Ignition client matching |
| Invoice total (amount due) | Ledger | Gross amount; fees collected separately |
| Due date | Ledger | Used as reference for scheduling; see Section 5.2 |
| Invoice date | Ledger | |
| Line item descriptions | Ledger | Referenced in repeating invoice docs; inferred for unpaid invoices |
| Ledger invoice ID | Ledger | Internal reference stored by Ignition for sync back |

### 2.4 Credit Notes, Partial Payments, and Prepayments

**Confidence: Unknown — not documented in public sources**

No public documentation exists confirming how Ignition handles:
- Xero/QBO invoices with a credit note partially offsetting the balance
- Invoices with a partial payment already recorded (outstanding balance < original amount)
- Prepayments applied against an invoice

The documented eligibility criterion is "Awaiting Payment" — in Xero, an invoice with a partial payment applied may still carry "Awaiting Payment" status for the remaining balance. Whether Ignition imports the full original amount or the outstanding balance is undocumented. **This is a significant data integrity risk requiring investigation before implementation.**

---

## 3. Invoice Import Mechanics

### 3.1 Import Trigger Modes

**Confidence: Confirmed**

**Automatic (recommended):**
- Toggle: Apps tab → ledger tile → Invoice Settings → "Sync unpaid invoices" → on → Save.
- Ignition continuously polls the ledger on the schedule defined in Section 3.2.
- No firm action required per invoice.

**Manual (on-demand):**
- Apps tab → ledger tile → click "Import" in the Invoices section.
- Firm sees all unpaid invoices in the ledger and selects which to import.
- Useful for firms wanting to review before importing, or for one-off imports without enabling automatic sync.

### 3.2 Sync Schedule

| Platform | Schedule | Notes |
|---|---|---|
| Xero | Every **1–2 hours** | Near real-time polling |
| QBO | **Daily** | Batch sync; not real-time |
| Practice CS | Manual only | Excel upload; no automated schedule |

### 3.3 Filtering Invoices

**Confidence: Partial**

For **automatic sync**: the only filters applied are the system eligibility criteria (Section 2.2). No firm-controlled filters (by client, date range, or amount) are documented for automatic mode.

For **manual import**: the documentation states the firm can "view all the unpaid invoices in your ledger and select the ones you would like to import." The repeating invoice import feature (a related but distinct feature) explicitly provides search and filter by client name and line item — similar filtering for the unpaid invoice import view is reasonable to infer but not explicitly confirmed.

### 3.4 Post-Import Modifications in Ledger

**Confidence: Partial**

| Change in ledger after import | Ignition behaviour |
|---|---|
| Invoice **voided** in Xero/QBO | Ignition automatically syncs the void; invoice removed from Outstanding |
| Invoice **marked as paid** in Xero/QBO | Ignition syncs paid status (post-Sept 2024 only); invoice removed from Outstanding |
| Invoice **amount modified** in Xero/QBO | **Not documented** — behaviour unknown |
| Invoice **due date changed** in Xero/QBO | **Not documented** — behaviour unknown |

Amount and due date modification sync are unresolved edge cases for engineering.

### 3.5 Invoices Already Paid in Ledger

**Confidence: Confirmed**

- If an invoice is paid in Xero/QBO, Ignition automatically syncs the paid status and removes it from Outstanding — **but only for invoices created after September 2024** when the "Mark as Paid" sync feature was introduced.
- Invoices paid externally before September 2024 may appear as unpaid in Ignition's Outstanding tab. Firms must manually bulk-select and mark them as paid using the Outstanding tab's bulk "Mark as Paid" action.
- Documented recommendation: *"Do not mark the invoice as paid directly in Ignition. Instead, record the payment in QuickBooks/Xero — if connected, it will automatically sync the invoice status back to Ignition."*

### 3.6 Bulk Import and Bulk Actions

**Confidence: Confirmed**

| Action | Available? | Entry Point |
|---|---|---|
| Bulk payment request (email) | Yes | Outstanding tab → select multiple → Request Payment |
| Bulk mark as paid | Yes | Outstanding tab → select multiple → Mark as Paid |
| Bulk schedule collection | Not explicitly documented | — |
| Bulk payment method request to clients | Yes | Clients tab → filter by Payment Method: None → select → Request Payment Method |

### 3.7 Rate Limits

**Confidence: Confirmed**

- Maximum of **2,000 ledger events per account within a 24-hour window**.
- Accounts with large historical backlogs (>2,000 unpaid invoices) will have invoices imported over multiple days.
- Self-resolving: no manual intervention required; the backlog processes automatically over subsequent sync cycles.

---

## 4. Client and Payment Method Matching

### 4.1 Client Matching Logic

**Confidence: Confirmed (primary mechanism); Inferred (fallback behaviour)**

Ignition matches imported invoices to existing Ignition clients using **exact client name matching**:

- *"For any invoice you create against a client in your cloud ledger software, Ignition will first check to see if that client is already synced with Ignition."*
- Match field: client name (same string, case-insensitive matching behaviour not documented).
- For QBO: *"If the Client Name exactly matches a Client Name in QuickBooks, then Ignition will assign all invoices to that existing client."*
- Manual override: firms can manually re-link an Ignition client record to a ledger contact if automatic name matching fails.
- **Unmatched clients**: behaviour is not explicitly documented. Invoice may appear in Outstanding without a linked Ignition client, or may remain unimported. **Engineering risk: requires investigation.**

### 4.2 No Payment Method on File

**Confidence: Confirmed**

When a client has no payment method saved in Ignition, two pathways exist:

1. **Automatic payment request (if email setting enabled):** If the firm has enabled "Unpaid invoices are imported from my accounting software" under Client Emails settings, Ignition automatically sends the client a payment request email upon invoice import.
2. **Manual payment request:** Firm triggers from Outstanding tab (3-dot menu → Request Payment) or in bulk.

In both cases, the client follows a link to the Ignition client portal (`go.ignitionapp.com/`), adds a payment method, and pays immediately. Ignition cannot automatically charge a client without a stored payment method.

**Important guidance from documentation:** *"It is recommended to advise the client that you intend to use their stored payment method for these invoices"* — implying the firm should proactively notify clients before silently charging a stored method.

### 4.3 Bulk Payment Method Requests

**Confidence: Confirmed**

- Clients tab → filter by Payment Method: None → select filtered clients → Request Payment Method.
- Sends a branded email with a link to the client portal to add payment details.
- Automated follow-up: **3 reminder emails** sent automatically, spaced **3 days apart**, until the client responds.
- Same mechanism used for payment method requests in other Ignition contexts (not AutoCollect-specific).

---

## 5. Payment Collection Mechanics

### 5.1 When Payment Is Charged

**Confidence: Confirmed for client-initiated and manually scheduled; Inferred for automatic future invoices**

| Scenario | Charge Timing |
|---|---|
| Client has no payment method → payment request sent | Charged immediately when client submits payment in the portal |
| Client has payment method on file → firm schedules collection | Charged on the firm-specified date (set via Outstanding tab → Schedule) |
| Automatic future invoices (client has opted in) | Inferred: on import or on due date; exact behaviour not fully documented |

### 5.2 Payment Terms and Due Dates

**Confidence: Inferred**

Ignition does not automatically read the due date from the imported ledger invoice and charge on that date without firm intervention. The firm controls collection timing by:

1. Using Ignition's own Payment Terms (Settings → Payment Gateway Settings), which governs days after invoice creation for automatic collection.
2. Manually scheduling collection per invoice in the Outstanding tab.

Ignition documentation advises aligning Ignition Payment Terms settings with the due date settings in Xero/QBO: *"If you are using Ignition Payments, these settings should be set to match your Payment Terms settings... This means that payments will always be deducted from your client's account or card on the due date of the invoice."* Alignment is achieved by matching configurations in both systems, not by Ignition dynamically reading the ledger due date.

### 5.3 Scheduling Future Collection

**Confidence: Confirmed**

For clients with a payment method on file:
- Outstanding tab → select invoice → **Schedule** action.
- Firm sets the collection date.
- Scheduled payment appears in Payments → Collections → **Scheduled** sub-tab with status "Not started."
- Payment transitions to "Collecting" on the scheduled date and "Collected" on success.

### 5.4 Collection Engine Shared with Proposal Billing

**Confidence: Confirmed**

AutoCollect invoices use the same payment collection engine, Stripe integration, retry logic, and payout mechanism as regular Ignition proposal-based billing. There is no separate payment processing path. The Outstanding tab displays all unpaid invoices (both sources) in one view; the Source filter isolates ledger-imported ones.

---

## 6. Fee Structure

### 6.1 Transaction Fees

**Confidence: Confirmed (fees apply); Inferred (same rates as standard)**

No documentation states AutoCollect carries different fees from standard Ignition payments. The same rate structure applies:

| Method | Rate | Cap / Notes |
|---|---|---|
| Credit card (standard) | 1.3%–3.6% + $0.30 | Tier determined after processing (cannot predict upfront) |
| Credit card (premium) | 3.6% + $0.30 | Business/corporate cards and rewards-linked consumer cards |
| ACH (US bank) | 1% + $0.30 | Capped at $5.00; +0.3% on amount above $3,000 |
| BECS (AU direct debit) | 1% + $0.30 | Capped at AUD $5.00; +0.3% on amount above $3,000 |
| BACS (UK direct debit) | 1% + £0.30 | Capped at £4.00; +0.3% on amount above £2,000 |
| PAD (CA pre-authorised debit) | Per pricing page | |

**Payout model:** 100% gross payout to the firm's bank. Transaction fees billed separately, monthly in arrears, from the firm's Ignition subscription card.

**Surcharging:** Available (card payments only) in US, AU, CA. Not available in UK or NZ. Bank/direct debit payments cannot be surcharged in any region.

### 6.2 Active Client Quota

**Confidence: Confirmed**

This is a significant billing distinction:

> *"Collecting payment on unpaid invoices via Ignition will not count clients as 'active clients' against your subscription — clients only count as active if there are active services set up via an accepted proposal or instant bill."*

AutoCollect clients do **not** increment the active client count against the firm's subscription tier limit (Solo: 20, Core: 50, Pro: higher). A firm can collect on invoices for hundreds of ledger clients via AutoCollect without affecting their active client quota.

### 6.3 Plan Tier Restrictions

**Confidence: Partial / Inferred**

Not explicitly documented in publicly available sources. AutoCollect requires:
1. Ignition Payments to be enabled (requires KYC onboarding)
2. A connected Xero or QBO account

The Solo plan requires connecting Xero or QBO as a condition of the plan. AutoCollect therefore likely works on all paid plans (Solo, Core, Professional, Professional+). Plan-specific gating is not confirmed.

---

## 7. Reconciliation and Accounting Sync

### 7.1 How Ignition Marks Invoices as Paid

**Confidence: Confirmed**

Documentation states: *"Invoice payment status from Ignition will be synced back to your ledger — it works the same as the existing Ignition and Ledger sync functionality."*

AutoCollect uses the **identical reconciliation mechanism** as regular Ignition billing:
- Timing: within **24 hours** of funds clearing the firm's bank account.
- Xero: payment posted to the IGNPayments clearing account; invoice marked paid.
- QBO: payment posted to Undeposited Funds; invoice marked paid.

### 7.2 Xero Reconciliation

**Confirmed**

| Component | Value |
|---|---|
| Clearing account name | **IGNPayments - Ignition Clearing Account** (legacy name: "PI Clearing Account" for pre-April 2022 accounts) |
| Account type | Current Asset |
| Payment posting | Debit to clearing account linked to Xero invoice → invoice status = Paid |
| Bank feed payout identifier | **IGNITIONPAY** |
| Bank rule | Matches IGNITIONPAY bank feed entries and codes to the clearing account |
| Reconciliation timing | Within 24 hours of funds clearing the firm's bank |
| Auto-reconciliation | Yes; no manual matching required in normal cases |
| NZ-specific requirement | GST Exempt tax code must be created in Xero for the bank rule |

### 7.3 QuickBooks Online Reconciliation

**Confirmed**

| Component | Value |
|---|---|
| Clearing account | **Undeposited Funds** (built-in QBO account) |
| Payment posting | Invoice marked paid via Undeposited Funds on collection |
| Bank feed identifier | **IGNITIONPAY** |
| Transfer rule | Created in QBO to move IGNITIONPAY bank deposits from bank account to Undeposited Funds |
| Reconciliation timing | Within 24 hours of funds clearing |
| Known limitation | *"The bank rule is only suitable if the Undeposited Funds account in QuickBooks is being used solely to record payment of invoices from Ignition."* Mixed payment environments (some payments outside Ignition going to Undeposited Funds) cause reconciliation issues. |

### 7.4 Void Sync Direction

**Confidence: Confirmed**

| Action | Direction | Behaviour |
|---|---|---|
| Invoice voided in Xero/QBO | Ledger → Ignition | Ignition automatically syncs the void; invoice removed from Outstanding |
| Invoice voided in Ignition | Ignition → Ledger | Corresponding Xero/QBO invoice must be **manually voided** in the ledger (one-way sync only) |

**Pre-void requirement in Ignition:** Collection must be cancelled before voiding. Voiding is only possible before payment begins collecting.

---

## 8. Status Tracking and Collections Dashboard

### 8.1 UI Location

**Confirmed:** Payments → Collections

AutoCollect invoices live within the existing Collections interface — there is no standalone AutoCollect section. The **Source filter** within the Outstanding tab distinguishes ledger-imported invoices from Ignition-created ones.

### 8.2 Collections Sub-Tabs

| Sub-tab | Contents |
|---|---|
| **Outstanding** | All unpaid invoices (both Ignition-created and ledger-imported). Filter Source = "Ledger" to isolate AutoCollect invoices. |
| **Scheduled** | Payments scheduled for future collection, including auto-retry attempts. Status: "Not started" → "Collecting" → "Collected". |
| **Failed** | Payments that failed collection; awaiting retry or manual action. |
| **Rejected** | Payments rejected by the client's bank or card provider; shown with red highlight in the invoice record. |

### 8.3 Actions per Invoice (Outstanding Tab)

| Action | Availability | Notes |
|---|---|---|
| Request Payment | Individual or bulk | Sends branded email with payment link to client |
| Schedule | Clients with payment method on file | Firm sets a collection date |
| Mark as Paid | Individual or bulk | For manually or externally paid invoices |
| Export | Account-level | CSV export of all collections data sent to firm's email |

---

## 9. Failed Payments and Retry Logic

**Confidence: Confirmed — same engine as all Ignition payments**

AutoCollect uses the same retry and dunning logic as proposal-based billing:

| Parameter | Value |
|---|---|
| Auto-retry default state | **Enabled** on all accounts |
| Retry count | **Once** |
| Retry timing | **3 business days** after first failure |
| Toggle location | Settings → Payments tab → "Auto-retry payments" |
| Retry log | Invoice → Activity section (audit trail) |
| Retry visibility | Collections → Scheduled sub-tab |

**Client notifications on failure:**
- Client receives automated email on payment failure.
- Email includes: failure reason, option to retry immediately, option to add a new payment method, notice that auto-retry will occur in 3 business days.

**Firm notifications on failure:**
- Email sent to address(es) configured under Settings → Payment Gateway → Notifications email.
- Multiple notification addresses supported.

**After client self-retry:**
- Firm receives a notification email confirming the client's successful retry.
- Platform data: clients self-retry in ~50% of failed payment cases.

**Rejected payments (distinct from failures):**
- Appear in the Rejected tab under Collections.
- Highlighted red in the Invoices & Payments tab.
- Both firm and client receive email notifications.

---

## 10. Limitations and Edge Cases

| Limitation | Detail | Confidence |
|---|---|---|
| **Invoice age limit** | Only invoices billed within the last 90 days are imported | Confirmed |
| **No multi-currency support** | Multi-currency invoices are not supported | Confirmed (repeating invoices; inferred for unpaid invoices) |
| **No negative prices** | Negative line items not supported | Confirmed (repeating invoices; inferred for unpaid invoices) |
| **No discount field** | Ignition has no discounts feature; discount line items from ledger may not import correctly | Inferred |
| **Ledger event rate limit** | 2,000 ledger events per 24-hour window per account | Confirmed |
| **Historical paid status sync gap** | Invoices paid externally before September 2024 must be manually marked as paid in Ignition | Confirmed |
| **QBO Undeposited Funds exclusivity** | QBO bank rule only works if Undeposited Funds is used solely for Ignition payments | Confirmed |
| **No partial refunds** | Refunds must be for the full invoice amount (applies platform-wide) | Confirmed |
| **Refund accounting sync** | Refunds are not auto-synced to Xero/QBO; must be manually reconciled as a Credit Note | Confirmed |
| **Void sync direction** | Voiding in Ignition does not cascade to the ledger; must void manually in Xero/QBO | Confirmed |
| **Practice CS is US-only, manual** | No API sync; Excel upload required; clients must pre-exist in Ignition | Confirmed |
| **Credit note / partial payment handling** | Undocumented — significant data integrity risk | Unknown |
| **Unmatched client handling** | Behaviour when ledger invoice client has no Ignition match is undocumented | Unknown |
| **Invoice modification after import** | Amount or due date changes in ledger after import — sync behaviour unknown | Unknown |

---

## 11. Relationship to Other Ignition Features

### Proposal-Based Billing (Core Flow)

The standard Ignition flow (client signs proposal → invoice auto-generated → payment collected per billing schedule) and AutoCollect are **parallel, independent paths**. AutoCollect is explicitly described as requiring "no proposals." A single client can simultaneously have:
- Active services billing via accepted proposals
- Ledger-imported invoices collected via AutoCollect

Both appear in the same Outstanding tab; the Source filter distinguishes them.

### Import Repeating Invoices (Separate Feature)

A distinct feature that converts Xero/QBO **repeating invoice templates** into **draft Ignition proposals**. This migrates recurring billing setup forward into Ignition's proposal system. AutoCollect, by contrast, collects on **already-issued, one-time unpaid invoices** without creating any proposal. The two features serve different migration strategies:

| Feature | Use case | Output |
|---|---|---|
| Import Repeating Invoices | Migrate recurring billing setup to Ignition going forward | Draft proposals |
| AutoCollect | Collect on existing outstanding A/R immediately | No proposal; payment collection only |

### Review + Pay (Related Feature)

Review + Pay adds a "Pay Now" button to invoice notification emails sent from Ignition for invoices where no payment method has been arranged. When a firm sends a payment request from the AutoCollect Outstanding tab, the client experience (email + link → client portal → payment) is functionally the same as the Review + Pay flow for Ignition-created invoices. They share the same client portal infrastructure.

### Collecting Debt via Proposals (Legacy Alternative)

Before AutoCollect, the documented approach for collecting outstanding A/R was to create a special Ignition proposal using an "Outstanding Payment" service type (or "Outstanding Payment (Instalments)" for payment plans). This approach still exists. AutoCollect is the newer, simpler alternative that avoids creating a proposal per debtor.

---

## 12. Data Model: AutoCollect-Relevant Objects

Reconstructed from documentation. Field names are descriptive, not necessarily API-exact.

### LedgerInvoice (Imported from Xero/QBO)

| Field | Type | Stored At | Notes |
|---|---|---|---|
| `id` | UUID | Ignition | Ignition-internal identifier |
| `ledger_invoice_id` | String | Ignition | External ID from Xero/QBO |
| `ledger_source` | Enum | Ignition | xero, quickbooks, practice_cs, cch_axcess |
| `client_id` | FK | Ignition | Matched Ignition client; null if unmatched |
| `ledger_client_name` | String | Ignition | Raw name from ledger (pre-match) |
| `invoice_number` | String | Ignition | Reference from ledger |
| `invoice_date` | Date | Ignition | |
| `due_date` | Date | Ignition | From ledger; used for scheduling reference |
| `amount` | Decimal | Ignition | Gross amount (whether full or outstanding balance — see Section 2.4) |
| `currency` | String | Ignition | ISO currency code; multi-currency unsupported |
| `status` | Enum | Ignition | outstanding, scheduled, collecting, collected, failed, rejected, paid, voided |
| `payment_method_id` | FK | Ignition | Linked Stripe payment method token; null if none on file |
| `collection_date` | Date | Ignition | Firm-set scheduled collection date; null if not scheduled |
| `payment_intent_id` | String | Ignition | Stripe PaymentIntent reference |
| `paid_at` | Timestamp | Ignition | |
| `failed_at` | Timestamp | Ignition | |
| `retry_scheduled_at` | Timestamp | Ignition | Set 3 business days after failure if auto-retry is on |
| `ledger_paid_synced_at` | Timestamp | Ignition | When paid status was written back to Xero/QBO |
| `imported_at` | Timestamp | Ignition | When Ignition first imported this invoice |
| `sync_source` | Enum | Ignition | automatic, manual |

### LedgerSyncConfig (per firm, per connected ledger)

| Field | Type | Notes |
|---|---|---|
| `firm_id` | FK | |
| `ledger_type` | Enum | xero, quickbooks |
| `auto_sync_enabled` | Boolean | "Sync unpaid invoices" toggle |
| `auto_email_on_import` | Boolean | "Unpaid invoices are imported from my accounting software" email setting |
| `last_synced_at` | Timestamp | |
| `next_sync_at` | Timestamp | |
| `rate_limit_window_events` | Integer | Events used in current 24-hour window (max 2,000) |

### SyncEvent (per import cycle)

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | |
| `firm_id` | FK | |
| `ledger_type` | Enum | |
| `triggered_at` | Timestamp | |
| `trigger_type` | Enum | scheduled, manual |
| `invoices_found` | Integer | Qualifying invoices discovered |
| `invoices_imported` | Integer | New invoices added to Ignition |
| `invoices_updated` | Integer | Status changes synced (paid, voided) |
| `rate_limited` | Boolean | Whether 2,000 event cap was hit |

---

## 13. Research Gaps

The following items are unknown or unconfirmed and represent engineering risks requiring investigation before sprint planning:

| Gap | Risk Level | Recommended Action |
|---|---|---|
| **Credit notes / partial payments:** Does Ignition import the full original invoice amount or the outstanding balance when a credit note or partial payment exists in Xero? | High — data integrity | Test in Xero sandbox; contact Ignition support |
| **Unmatched client handling:** What happens when an imported invoice's ledger client name does not match any Ignition client? Is the invoice dropped, queued, or imported without a client link? | High — silent data loss risk | Contact Ignition support; test with a deliberate mismatch |
| **Invoice modification after import:** If the invoice amount or due date is changed in Xero/QBO after import (but before collection), does Ignition sync the change? | High — incorrect charge amounts | Contact Ignition support |
| **Automatic collection timing for future invoices:** After a client opts in to automatic collection, how does Ignition determine when to charge each subsequent imported invoice? (On import? On due date? Per Payment Terms setting?) | High — operational ambiguity | Test with a client opt-in scenario |
| **Plan tier restriction:** Is AutoCollect gated to specific subscription plans (Core, Pro, etc.) or available on Solo? | Medium — product scope | Review Ignition pricing page at ignitionapp.com/pricing |
| **QBO sync: Invoice modification detection:** Does QBO's daily batch sync detect amount changes, or only status changes (paid, voided)? | Medium — reconciliation risk | Contact Ignition support |
| **Practice CS / CCH Axcess: client matching:** Are the matching rules (name-based?) the same as for Xero/QBO, or different? | Medium | Review CCH Axcess help article |
| **Prepayments in Xero:** A Xero prepayment applied to an invoice — does this affect import eligibility? | Low–Medium | Test or contact Ignition support |
| **Minimum invoice amount:** Is there a floor (e.g., < $1.00) below which AutoCollect will not import an invoice? | Low | Review Ignition Payments FAQ |

---

*Sources: [AutoCollect: Automate Invoice Payments](https://support.ignitionapp.com/en/articles/11362280-autocollect-automate-invoice-payments), [Import unpaid invoices from your connected apps](https://support.ignitionapp.com/en/articles/10553614-import-unpaid-invoices-from-your-connected-apps), [Send invoice payment requests](https://support.ignitionapp.com/en/articles/10968746-send-invoice-payment-requests), [The Collections tab](https://support.ignitionapp.com/en/articles/9458405-the-collections-tab), [Automatic payment collection retries](https://support.ignitionapp.com/en/articles/9625741-automatic-payment-collection-retries), [How to handle failed payments](https://support.ignitionapp.com/en/articles/1091256-how-to-handle-failed-payments), [Mark invoices as paid](https://support.ignitionapp.com/en/articles/9824402-mark-invoices-as-paid), [Reconciling payouts with a bank rule in Xero](https://support.ignitionapp.com/en/articles/601684-reconciling-payouts-with-a-bank-rule-in-xero), [Reconciling Payouts with a Transfer Rule in QuickBooks](https://support.ignitionapp.com/en/articles/7000069-reconciling-payouts-with-a-transfer-rule-in-quickbooks), [Void or Delete Invoices in Ignition](https://support.ignitionapp.com/en/articles/13134076-void-or-delete-invoices-in-ignition), [Collecting debt and invoice management](https://support.ignitionapp.com/en/articles/4666310-collecting-debt-and-invoice-management-when-using-ignition-and-xero-quickbooks), [Import repeating Xero and QuickBooks invoices](https://support.ignitionapp.com/en/articles/8366031-import-repeating-xero-and-quickbooks-invoices), [How to use Review + Pay](https://support.ignitionapp.com/en/articles/4697854-how-to-use-review-pay-to-collect-payment-on-your-invoices), [Request payment methods in bulk](https://support.ignitionapp.com/en/articles/7104348-request-payment-methods-in-bulk), [Ignition and Practice CS](https://support.ignitionapp.com/en/articles/10618716-ignition-and-practice-cs), [How payment fees are calculated](https://support.ignitionapp.com/en/articles/9489207-how-payment-fees-are-calculated), [Ignition Payments FAQ](https://support.ignitionapp.com/en/articles/12134812-ignition-payments-faq), [Ignition launches AutoCollect (press release)](https://www.ignitionapp.com/news/ignition-launches-autocollect-to-end-the-business-chase-for-late-payments).*
