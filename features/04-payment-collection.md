# Feature Research: Payment Collection
> Source: ignitionapp.com — official product, help center (support.ignitionapp.com), learning center, Stripe case study (stripe.com/customers/ignition), and Ignition security page.
> Purpose: Reference document for a planner agent writing software development epics.

---

## Table of Contents

1. [Feature Overview](#1-feature-overview)
2. [Payment Infrastructure — Stripe Custom Connect Architecture](#2-payment-infrastructure--stripe-custom-connect-architecture)
3. [Payment Methods by Region](#3-payment-methods-by-region)
4. [Transaction Fees](#4-transaction-fees)
   - 4.1 [Card Fees](#41-card-fees)
   - 4.2 [Bank / Direct Debit Fees](#42-bank--direct-debit-fees)
   - 4.3 [High-Value Payment Surcharge](#43-high-value-payment-surcharge)
   - 4.4 [Surcharging (Passing Fees to Clients)](#44-surcharging-passing-fees-to-clients)
   - 4.5 [Fee Billing Mechanics](#45-fee-billing-mechanics)
5. [Client-Facing Payment Capture](#5-client-facing-payment-capture)
   - 5.1 [Where Payment Details Are Collected](#51-where-payment-details-are-collected)
   - 5.2 [Stripe Elements and PCI Scope](#52-stripe-elements-and-pci-scope)
   - 5.3 [Data Flow: What Goes Where](#53-data-flow-what-goes-where)
6. [Payment Timing and Billing Schedules](#6-payment-timing-and-billing-schedules)
   - 6.1 [Billing Types and Charge Initiation](#61-billing-types-and-charge-initiation)
   - 6.2 [Collection Schedule Configuration](#62-collection-schedule-configuration)
   - 6.3 [Catch-Up Billing for Backdated Starts](#63-catch-up-billing-for-backdated-starts)
7. [Payment Method Management](#7-payment-method-management)
   - 7.1 [Firm-Side Management](#71-firm-side-management)
   - 7.2 [Client Self-Service Portal](#72-client-self-service-portal)
8. [ACH Verification Flow (US)](#8-ach-verification-flow-us)
9. [Failed Payment Handling — Retry and Dunning](#9-failed-payment-handling--retry-and-dunning)
10. [Payouts to the Firm](#10-payouts-to-the-firm)
    - 10.1 [Payout Model](#101-payout-model)
    - 10.2 [KYC Onboarding and Bank Verification](#102-kyc-onboarding-and-bank-verification)
    - 10.3 [Processing Times by Payment Method](#103-processing-times-by-payment-method)
    - 10.4 [Changing Payout Account](#104-changing-payout-account)
11. [Invoices, Receipts, and Accounting Sync](#11-invoices-receipts-and-accounting-sync)
    - 11.1 [Documents Generated After Payment](#111-documents-generated-after-payment)
    - 11.2 [Xero Reconciliation](#112-xero-reconciliation)
    - 11.3 [QuickBooks Reconciliation](#113-quickbooks-reconciliation)
    - 11.4 [Statement Descriptor](#114-statement-descriptor)
12. [Refunds](#12-refunds)
13. [Disputes and Chargebacks by Region](#13-disputes-and-chargebacks-by-region)
14. [Security and PCI Compliance](#14-security-and-pci-compliance)
15. [Data Model: Payment-Related Objects](#15-data-model-payment-related-objects)

---

## 1. Feature Overview

Payment Collection is Ignition's embedded payment infrastructure — the engine that charges clients and deposits funds into the firm's bank account, automatically, as part of the proposal-to-payment lifecycle. It is the core differentiator that transforms Ignition from a proposal tool into a revenue automation platform.

**Key capabilities:**
- Charges clients automatically when a proposal is accepted (or on a recurring schedule)
- Supports credit/debit cards and bank account direct debit in all five major markets (US, AU, UK, CA, NZ)
- Gross daily payouts deposited to the firm's bank account
- Automated retry and client dunning on payment failure
- Bidirectional reconciliation with Xero and QuickBooks
- 91% of payments collected automatically in 2025 (platform-wide stat)

**Architectural principle:** Ignition never stores raw payment credentials. All sensitive data is held by Stripe. Ignition holds only tokens, mandate records, and invoice metadata.

---

## 2. Payment Infrastructure — Stripe Custom Connect Architecture

Ignition's entire payment infrastructure is built on **Stripe**, using **Stripe Custom Connect** accounts. This is confirmed directly by the [Stripe case study on Ignition](https://stripe.com/customers/ignition), which states the integration began in 2015 with Stripe Payments and Custom Connect, expanding globally thereafter.

### What "Custom Connect" Means

Custom Connect is the most deeply embedded of the three Stripe Connect account types (Standard, Express, Custom). Under this model:

| Dimension | Ignition / Custom Connect Behavior |
|---|---|
| **Who is the connected account** | Each accounting/bookkeeping firm (Ignition customer) |
| **Who controls the UX** | Ignition entirely — the firm never interacts with Stripe directly |
| **Stripe Dashboard access** | Connected firm accounts have **no access** to the Stripe Dashboard |
| **KYC / onboarding** | Ignition collects and submits all identity and bank data to Stripe via Ignition's UI |
| **Fund flow control** | Ignition platform controls the entire flow from client charge to firm payout |
| **Liability** | Ignition (as platform) carries compliance and regulatory responsibility |

This architecture enables the white-labeled, integrated experience where clients interact with the accounting firm's brand and never see Stripe branding.

### Stripe Products in Use

| Stripe Product | Role in Ignition |
|---|---|
| **Stripe Payments** | Core card and bank payment processing |
| **Stripe Custom Connect** | Multi-party connected account architecture for firms |
| **Stripe Connect Onboarding & Identity** | KYC and ID verification during firm onboarding |
| **Stripe Elements / Stripe.js** | Hosted payment field iframes in proposals (PCI scope reduction) |
| **Stripe Financial Connections** | Instant US bank account verification for ACH |
| **Stripe Radar** | Fraud detection and prevention (ML-based, cross-network) |

---

## 3. Payment Methods by Region

### Credit / Debit Cards (All Regions)

Cards are available in all five markets: US, AU, UK, CA, NZ.

| Card Network | US | AU | UK | CA | NZ |
|---|---|---|---|---|---|
| Visa | Yes | Yes | Yes | Yes | Yes |
| Mastercard | Yes | Yes | Yes | Yes | Yes |
| American Express | Yes | Yes | Yes | Yes | Yes |
| Discover | Yes | Yes | Yes | Yes | Yes |
| JCB | Yes | Yes | No | Yes | Yes |
| UnionPay | Yes | Yes | No | Yes | No |

**Apple Pay and Google Pay:** Available as card-equivalent payment methods in **US, AU, CA, and NZ**. Not documented as available in UK.

### Bank Account / Direct Debit Methods

| Market | Method | Payment Rails |
|---|---|---|
| **United States** | ACH Direct Debit | ACH network; Stripe Financial Connections (instant) or micro-deposits (fallback) |
| **Australia** | Direct Debit | BECS (Bulk Electronic Clearing System) |
| **United Kingdom** | Direct Debit | BACS |
| **Canada** | Pre-Authorized Debit (PAD) | ACSS (Automated Clearing Settlement System via Payments Canada) |
| **New Zealand** | Direct Debit | Local NZ direct debit scheme via Stripe |

**Firm requirement:** To use Ignition Payments at all, the firm must have a **company domain email address** — free mail providers (Gmail, Outlook, Hotmail, Protonmail, Yahoo) are not accepted.

---

## 4. Transaction Fees

### 4.1 Card Fees

Ignition uses a **two-tier card fee structure** based on card type:

| Card Category | Examples | Fee Range |
|---|---|---|
| **Standard cards** | Consumer Visa/Mastercard/Discover without rewards programs | Lower end of range |
| **Premium cards** | Business/corporate cards (all networks), any rewards-linked consumer card (Qantas, Velocity, bank points) | Higher end of range |

**US-confirmed published rates:**
- Premium credit card: **3.6% + $0.30** per transaction
- Overall card range: **1.3% to 3.6% + $0.30** per transaction

**Limitation:** Ignition (and Stripe) cannot determine whether a card is Standard or Premium at the time of capture. The card network only discloses this after processing. The firm therefore cannot predict which fee tier will apply for a given client's card.

**Free trial:** Zero transaction fees on the first **$10,000 (US) / £6,000 (UK)** collected during the 14-day trial.

### 4.2 Bank / Direct Debit Fees

| Market | Method | Base Fee | Cap |
|---|---|---|---|
| US | ACH | ~1% + $0.30 | $5.00 |
| UK | BACS | ~1% + £0.30 | £4.00 |
| AU | BECS | Per current pricing page | — |
| CA | PAD/ACSS | Per current pricing page | — |
| NZ | Direct Debit | Per current pricing page | — |

Bank account payments have significantly lower fees than cards and are capped — making them the preferred collection method for higher-value invoices.

### 4.3 High-Value Payment Surcharge

An additional **0.3% fee** applies to the portion of any **bank account payment exceeding $3,000 (US/AU/CA/NZ) or £2,000 (UK)**. This is additive on top of the base fee.

**Does not apply to card payments.**

**Example (US ACH, $10,000 invoice):**
- Base fee: $5.00 (capped)
- High-value surcharge: 0.3% × ($10,000 − $3,000) = 0.3% × $7,000 = $21.00
- **Total: $26.00**

Ignition reports that over 95% of bank account payments fall below the threshold.

### 4.4 Surcharging (Passing Fees to Clients)

Firms can optionally pass card processing fees to clients as a surcharge in **US, AU, and CA only**.

| Market | Maximum Surcharge | Key Constraints |
|---|---|---|
| **US** | 3% (Visa cap) / 4% (Mastercard cap) | Debit cards must be excluded (Durbin Amendment); California banned surcharging from July 1, 2024 |
| **AU** | Cost of acceptance (~1.95% effective ceiling) | Cannot exceed merchant's actual cost; ACCC regulates excessive surcharges; GST included in rate |
| **CA** | 2.4% maximum | |
| **UK** | Not available | — |
| **NZ** | Not available | — |

**Bank/direct debit payments cannot be surcharged in any region.**

Credits from client-paid surcharges are applied against the firm's Ignition processing fee invoices.

### 4.5 Fee Billing Mechanics

- Transaction fees are billed **monthly in arrears** to the firm's Ignition subscription card.
- This is separate from payout timing — the firm receives 100% gross payouts (see Section 10), and fees are charged separately as a monthly invoice.
- Per-plan fee differentiation is not explicitly documented in available sources; the Ignition pricing page should be consulted for current per-plan rates.

---

## 5. Client-Facing Payment Capture

### 5.1 Where Payment Details Are Collected

Payment credentials are collected at one of two touchpoints:

1. **Within the proposal acceptance flow** — hosted at `go.ignitionapp.com/`. If "Require Payments" is enabled on the proposal, the client cannot complete acceptance without providing a payment method. This is Ignition's most-used feature.
2. **Via the client payment portal** — a standalone branded page at `go.ignitionapp.com/` linked from a payment method request email the firm can send at any time (e.g., to add a method to an existing client without a new proposal).

### 5.2 Stripe Elements and PCI Scope

The payment form is rendered using **Stripe Elements / Stripe.js** — hosted iframe fields served directly from Stripe's servers. This is critical for PCI scope:

- Card number, expiry, and CVV fields are **iframes from stripe.com**, not from Ignition's domain.
- Raw cardholder data **never touches Ignition's servers or code paths**.
- This qualifies Ignition for **PCI SAQ-A** (the least-burdensome self-assessment type), rather than a full QSA audit.
- Ignition has completed annual SAQ-A attestations. Stripe operates at **PCI Service Provider Level 1** (highest tier).

A **$0 or $1 authorization hold** may appear on the client's card statement at capture time (to validate the card is real and has sufficient authorization). This hold disappears and is never settled.

### 5.3 Data Flow: What Goes Where

| Data | Stored At | Notes |
|---|---|---|
| Raw card number, CVV | **Stripe only** | Never transmitted to or stored by Ignition |
| Card expiry date | **Stripe only** | |
| Bank account number, routing number, BSB | **Stripe only** | |
| Stripe payment method token / ID | **Ignition** | Used to initiate future charges; useless without Stripe |
| Direct debit mandate / authorization record | **Ignition + Stripe** | Legal record of client's payment authorization |
| Client identity (name, email, address) | **Ignition** | For invoice and mandate records |
| Invoice amounts, service descriptions | **Ignition** | Core proposal and billing data |
| KYC identity data (firm representatives) | **Stripe** (via Ignition collection UI) | Submitted during firm onboarding |

---

## 6. Payment Timing and Billing Schedules

### 6.1 Billing Types and Charge Initiation

| Billing Type | Invoice Generated | Charge Initiated |
|---|---|---|
| **On Acceptance** | Immediately when client signs proposal | Per configured collection schedule (same day or N days after invoice date) |
| **Recurring** | Automatically on each configured period (daily / weekly / monthly / annually) | Per collection schedule after invoice generation |
| **On Completion / Estimate** | Manually by the firm when work is complete | Per manual payment terms (N days after invoice date) |
| **Deposit** | Two events: deposit invoice on acceptance; balance invoice manually | Per respective collection schedule for each event |

### 6.2 Collection Schedule Configuration

Configured under **Settings → Payments**:

- **Automatic Payment Terms**: applies to On Acceptance and Recurring billing. Set to "Same Day" for immediate collection.
- **Manual Payment Terms**: applies to On Completion and Estimate billing. Configures net-N days.
- Individual proposals can override the default.
- **Default Recurring Invoice Day** in Settings → General does **not** override individual billing rule configurations on existing proposals.

### 6.3 Catch-Up Billing for Backdated Starts

If a proposal's billing start date is set in the past, Ignition automatically calculates and generates **catch-up period invoices** for all missed billing events between the backdated start and today.

---

## 7. Payment Method Management

### 7.1 Firm-Side Management

- Firms manage client payment methods from the **client record** in Ignition.
- Expired payment methods are flagged with a **red badge** and cannot be used for collection; they can only be deleted (not edited).
- **Existing payment methods cannot be edited** (e.g., updating only an expiry date is not supported) — the entire payment method must be deleted and re-added. This is a deliberate security and compliance design decision.
- A **payment method activity log** records all changes: additions, replacements, deletions, and micro-deposit verification events, with timestamps.

### 7.2 Client Self-Service Portal

- Firms send a **payment method request** email (to one or multiple clients) from within Ignition.
- The email links to the client payment portal at `go.ignitionapp.com/` where clients update their own details.
- **3 automated reminder emails** are sent (spaced 3 days apart) until the client responds.
- During a failed payment, the client receives an email with options to retry with the existing method or add a new one.
- Client self-retry resolves ~50% of failed payment cases (Ignition early access program data).

---

## 8. ACH Verification Flow (US)

US bank account payments via ACH require account verification before charges can be processed. Two paths:

### Instant Verification (Preferred)
1. Client selects "US bank account" in the proposal payment step.
2. The **Stripe Financial Connections** widget opens (bank OAuth login).
3. Client selects their institution and logs in with their banking credentials.
4. Account is **instantly verified**; scheduled payments proceed immediately.

### Manual / Micro-Deposit Verification (Fallback)
1. Client (or firm on their behalf) enters routing number, account number, and account name.
2. Stripe sends **two small micro-deposits** with the descriptor "ACCTVERIFY" (amount-based method) **or** a **single micro-deposit with a unique 6-digit code** in the bank statement descriptor (code-based method).
3. Client receives a **verification email ~48 hours later** (timed to when deposits appear on statements).
4. Client enters the amounts or code to complete verification.
5. No payments are processed until verification is complete.

**Important:** If the firm enters the bank details on the client's behalf (via the firm-side payment management UI), the verification email goes to the **company's email address** in Ignition Settings — not the client's email.

---

## 9. Failed Payment Handling — Retry and Dunning

### Automatic Retry

- Auto-retry is **enabled by default** on all Ignition accounts.
- A failed payment is automatically retried **once, 3 business days after failure**.
- Toggle under **Settings → Payments → Auto-retry payments**.
- Retry is logged in the invoice's Activity section and visible in **Payments → Collections → Scheduled**.

### Manual Retry

- Firms can manually reschedule a failed payment from **Payments → Collections → Failed**.
- Options: retry today, retry on a specific future date, or retry with a different payment method (requires updating the payment method on the billing item first).

### Client Notifications on Failure

- Client receives an **automated email** on payment failure.
- Email includes: reason for failure, option to retry immediately, option to add a new payment method, and notice that auto-retry will occur in 3 business days.
- After a successful client-initiated retry, the **firm receives a notification email**.

### Firm Notifications

- Configurable under **Settings → Payments → Notifications Email**.
- Multiple email addresses can receive failure notifications.

### Recurring Services and Failed Payments

- A past failed payment does **not block future recurring collections**. Future invoices continue to generate and collect on schedule.
- The failed invoice remains in "Failed" status indefinitely until the firm manually retries or writes it off.

### Common Failure Reasons

Card expired, insufficient funds, declined by issuing bank, invalid account number, closed bank account.

---

## 10. Payouts to the Firm

### 10.1 Payout Model

- **Daily lump-sum payouts**: each business day, all payments cleared overnight are aggregated into a single transfer ("Practice Payment") to the firm's bank account.
- **Gross payouts (not net-of-fees)**: the firm receives 100% of each invoice total. Transaction fees are billed separately in arrears on a monthly basis. This ensures clean reconciliation in accounting software.
- No payouts on weekends; the displayed payout date defaults to the nearest business day.

**Payout statuses:**

| Status | Meaning |
|---|---|
| In Transit | Funds sent from Ignition/Stripe to firm's bank |
| Complete | Funds confirmed received by firm's bank |
| Failed | Transfer failed (invalid account, closed account, etc.) |

### 10.2 KYC Onboarding and Bank Verification

Before receiving any payouts, the firm completes a one-time onboarding process in Ignition (powered by Stripe Connect Onboarding):

1. **Enter bank account details** — account number, BSB/routing number, and a statement descriptor (5–16 alphanumeric characters, no special characters).
2. **Enter business details** — legal entity name, address, registration details.
3. **Enter representative details** — the principal user's personal information for KYC.
4. **Upload government-issued ID** — photo ID with a selfie; for driver's license, front and back uploaded separately.

**Verification timelines:**
- Standard: **1–2 business days**
- Busy periods: up to **3 days**
- First-time business verification: up to **3–7 business days** (typically faster)
- Payouts are **held during verification**, but client payment collection proceeds normally
- After verification, the **first payout** may take up to **7 business days** to reach the firm's bank

### 10.3 Processing Times by Payment Method

| Method | Typical Time to Payout | Notes |
|---|---|---|
| Credit / debit card | 1–2 business days | After client charge |
| ACH (US) | 3–7 business days | First payment longer if micro-deposit verification pending |
| BACS (UK) | 10 business days (first) / 6 business days (subsequent) | Based on 5 PM GMT cutoff |
| BECS (AU) | 3–7 business days | Add ~3 days for first mandate confirmation |
| PAD (CA) | Up to 5 business days | ACSS clearing |
| NZ Direct Debit | 3–7 business days | |

### 10.4 Changing Payout Account

- Navigate to **Settings → Payments → Edit (Pay account)**.
- Principal user must complete **2FA**.
- Change requires **Ignition support team approval** (fraud and risk control measure).
- Account re-enters the verification process before new payouts resume.

---

## 11. Invoices, Receipts, and Accounting Sync

### 11.1 Documents Generated After Payment

| Document | When Generated | Notes |
|---|---|---|
| **Invoice** | On billing trigger event (acceptance, recurring date, etc.) | Generated in Xero/QBO as "Awaiting Payment"; Ignition-only accounts generate an internal record |
| **Invoice PDF** | When accounting integration is active | Sent to client as an attachment to the invoice notification email |
| **Payment receipt email** | After successful payment collection | Sent automatically; the invoice PDF is **not attached** to the receipt email (deliberate design choice to reduce confusion) |

**No accounting integration:** If Xero or QBO is not connected, Ignition creates internal invoice records but does not generate a standalone PDF invoice for the client.

### 11.2 Xero Reconciliation

Ignition creates a dedicated clearing account in Xero:
- **Account name:** "IGNPayments - Ignition Clearing Account"
- **Account type:** Current Asset

**Reconciliation flow:**
1. Payment collected → Ignition creates a debit transaction in the clearing account, linked to the Xero invoice → invoice marked "Paid".
2. Daily lump-sum payout hits the firm's bank account.
3. Xero **bank rule** matches the daily bank deposit to the clearing account transactions.
4. Auto-reconciliation completes within **~24 hours** of funds arriving.

### 11.3 QuickBooks Reconciliation

- Ignition uses QuickBooks' **Undeposited Funds** account as the clearing mechanism.
- Payments are linked to invoices automatically on collection.
- Auto-reconciliation within **~24 hours** of funds arriving.

### 11.4 Statement Descriptor

The descriptor that appears on the client's bank/card statement:
- **US, AU, CA, NZ:** Configurable by the firm (5–16 alphanumeric characters, no special characters). Set during onboarding.
- **UK:** Locked to **"IGNITIONPAY"** — cannot be customized. Ignition advises firms to inform clients in their proposal text that the charge will appear as this.

Ignition recommends noting in proposals: *"Payments will appear on your statement as [descriptor] or IGNITIONPAY depending on your bank."*

---

## 12. Refunds

- Refunds can be issued through Ignition for any payment originally collected via Ignition.
- **Partial refunds are not supported** — must refund the full invoice amount.
- **Timeline:** 7–10 business days for funds to appear in the client's account.
- **Time limit:** Refunds may be blocked for payments older than 60–90 days (varies by financial institution).

### Clawback Mechanism

Ignition offsets refunds against the firm's next payout rather than processing a separate transfer:

- Example: $500 refund due, $200 upcoming payout → payout is withheld ($200), and $300 is additionally debited from the firm's account.
- Refund status: "Refund approved" → "Complete" when clawback is processed.

### Accounting Sync for Refunds

Refunds are **not automatically synced** to Xero or QuickBooks:
- **Xero:** Firm must manually create a **Credit Note** in Xero and reconcile it against the clawback transaction.
- **QuickBooks:** Same — must be manually recorded.

---

## 13. Disputes and Chargebacks by Region

### United States (Card)

- Firm is notified by email and has **2 business days** to submit evidence before the window closes.
- Evidence submission is managed through Ignition (or directly in practice, via Stripe's infrastructure).
- Best practices: clear statement descriptor, up-to-date signed engagement letter on file, never use "accept on behalf" without written client consent.

### United Kingdom (BACS Direct Debit)

- Disputes handled under the **Direct Debit Guarantee** scheme.
- Statement descriptor is locked to "IGNITIONPAY" — important for dispute context.
- Firm must respond promptly when notified; failure to respond can result in an automatic legitimate chargeback.

### Australia (BECS Direct Debit)

- BECS disputes are automatically resolved **in favor of the client** — they cannot be contested by the firm.
- The BECS network allows consumers to dispute a debit on a "no questions asked" basis for up to **7 years**.
- A successful dispute **automatically cancels the BECS mandate** — the client must re-enter payment details to establish a new one.
- Refunds for BECS payments must be issued within **90 days** of the original payment.
- Firms must resolve the underlying commercial dispute directly with the client.

### Canada and New Zealand

- Disputes handled per applicable regional rules (PAD and NZ direct debit schemes respectively).
- Similar framework to other direct debit regions.

---

## 14. Security and PCI Compliance

### PCI DSS

| Entity | Certification Level | Method |
|---|---|---|
| **Stripe** | PCI Service Provider Level 1 (highest) | Annual QSA audit |
| **Ignition** | SAQ-A (least-burdensome) | Annual self-assessment attestation |

Ignition qualifies for SAQ-A because Stripe Elements iframes handle all card field rendering — cardholder data never passes through or is stored on Ignition's infrastructure.

### Tokenization

- When a client submits payment details, Stripe immediately tokenizes the data and returns a **payment method ID** (opaque token) to Ignition.
- Ignition stores only the token — not the underlying card number, expiry, CVV, or account number.
- The token cannot be reverse-engineered to recover payment credentials.
- A compromise of Ignition's systems would expose billing metadata (names, amounts, invoice records) but **not payment credentials**.

### Fraud Protection

- **Stripe Radar** is active on Ignition's Stripe integration, providing ML-based fraud detection trained on data across the entire Stripe network.
- Stripe has partnerships with Visa, Mastercard, Amex, and major banks to access TC40 dispute data, SAFE reports, and early fraud signals.
- Any payout account change triggers additional verification checks by Ignition's internal Fraud and Controls team.
- 2FA required for all sensitive account changes.
- Company domain email required for all payment-enabled accounts.
- All payment data transmitted over TLS/SSL.

---

## 15. Data Model: Payment-Related Objects

The following is a reconstruction of core payment-related entities based on observed behavior and help center documentation. Not all field names are official API names.

### PaymentMethod

| Field | Type | Stored At | Notes |
|---|---|---|---|
| `stripe_payment_method_id` | String | Ignition | Token referencing Stripe PM object |
| `type` | Enum | Ignition | card, ach_debit, becs_debit, bacs_debit, acss_debit, nz_bank_account |
| `card_brand` | String | Ignition | visa, mastercard, amex, discover, etc. |
| `card_last4` | String | Ignition | Display only; not sensitive |
| `card_expiry` | MM/YY | Ignition | Display only |
| `bank_name` | String | Ignition | For display |
| `account_last4` | String | Ignition | For display |
| `status` | Enum | Ignition | active, expired |
| `mandate_id` | String | Ignition + Stripe | For direct debit methods; legal authorization record |
| `verified` | Boolean | Ignition | For ACH; false until micro-deposit or Financial Connections complete |
| `client_id` | FK | Ignition | Links to client record |
| `activity_log` | Array | Ignition | Timestamped record of all changes |

### Invoice (Billing Item)

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | |
| `proposal_id` | FK | Parent proposal |
| `service_id` | FK | Service being billed |
| `client_id` | FK | |
| `amount` | Decimal | Invoice total (gross) |
| `billing_type` | Enum | on_acceptance, recurring, on_completion, deposit |
| `status` | Enum | scheduled, collecting, collected, failed, refunded, written_off |
| `due_date` | Date | When collection is triggered |
| `xero_invoice_id` | String | External reference; null if no Xero integration |
| `qbo_invoice_id` | String | External reference; null if no QBO integration |
| `stripe_payment_intent_id` | String | Stripe reference for the charge |
| `collected_at` | Timestamp | |
| `failed_at` | Timestamp | |
| `retry_scheduled_at` | Timestamp | Set 3 business days after failure if auto-retry is on |
| `surcharge_amount` | Decimal | If surcharging enabled |

### Payout

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | |
| `stripe_payout_id` | String | |
| `amount` | Decimal | Gross daily lump sum |
| `status` | Enum | in_transit, complete, failed |
| `initiated_at` | Timestamp | |
| `expected_arrival` | Date | |
| `invoice_ids` | Array | All invoices included in this payout |

### PaymentFeeInvoice

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | |
| `billing_period` | Month | Month the fees relate to |
| `total_fees` | Decimal | Sum of all transaction fees for the period |
| `surcharge_credits` | Decimal | Credits from client-paid surcharges |
| `net_fee_charged` | Decimal | `total_fees` − `surcharge_credits` |
| `charged_at` | Timestamp | Monthly, in arrears, to firm's subscription card |

---

*Sources: [Ignition Help Center](https://support.ignitionapp.com), [Ignition Payments Overview](https://support.ignitionapp.com/en/articles/600819-payments-overview), [Stripe Case Study: Ignition](https://stripe.com/customers/ignition), [Ignition Security](https://www.ignitionapp.com/security), [How Payment Fees Are Calculated](https://support.ignitionapp.com/en/articles/9489207-how-payment-fees-are-calculated), [High-Value Bank Payment Fee](https://support.ignitionapp.com/en/articles/8534326-high-value-bank-account-payment-fee), [Surcharges Guide (USA)](https://support.ignitionapp.com/en/articles/8265256-compliance-and-card-surcharges-guide-usa), [ACH Verification Process](https://support.ignitionapp.com/en/articles/601452-usa-only-the-ach-payment-verification-process), [Failed Payments](https://support.ignitionapp.com/en/articles/1091256-how-to-handle-failed-payments), [Automatic Retries](https://support.ignitionapp.com/en/articles/9625741-automatic-payment-collection-retries), [Payouts Tab](https://support.ignitionapp.com/en/articles/5534271-the-payouts-tab), [Payment Processing Times](https://support.ignitionapp.com/en/articles/3111334-payment-processing-times), [Refunding Client Payments](https://support.ignitionapp.com/en/articles/2891123-refunding-client-payments), [Disputes - USA](https://support.ignitionapp.com/en/articles/2874050-payment-disputes-usa), [Disputes - Australia](https://support.ignitionapp.com/en/articles/2903754-payment-disputes-australia).*
