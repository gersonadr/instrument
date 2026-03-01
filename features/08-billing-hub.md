# Feature Research: Billing Hub
> Source: ignitionapp.com — official product, help center (support.ignitionapp.com), quarterly product update blog posts, press releases, and the Ignition "What's New" changelog.
> Purpose: Reference document for a planner agent writing software development epics.

---

## Table of Contents

1. [Feature Overview and Purpose](#1-feature-overview-and-purpose)
2. [UI Location and Navigation](#2-ui-location-and-navigation)
3. [What the Billing Hub Displays](#3-what-the-billing-hub-displays)
4. [Views, Tabs, and Sort Behavior](#4-views-tabs-and-sort-behavior)
5. [Actions Available](#5-actions-available)
6. [Relationship to Proposals and Services](#6-relationship-to-proposals-and-services)
7. [Relationship to AutoCollect and Ledger-Imported Invoices](#7-relationship-to-autocollect-and-ledger-imported-invoices)
8. [Relationship to the Collections Tab](#8-relationship-to-the-collections-tab)
9. [Invoice Management Flows](#9-invoice-management-flows)
10. [Filters and Search](#10-filters-and-search)
11. [Export](#11-export)
12. [Notifications and Alerts](#12-notifications-and-alerts)
13. [Plan Tier Restrictions](#13-plan-tier-restrictions)
14. [Launch Date and History](#14-launch-date-and-history)
15. [Edge Cases and Limitations](#15-edge-cases-and-limitations)
16. [Data Model: Billing Hub–Relevant Objects](#16-data-model-billing-hubrelevant-objects)
17. [Research Gaps](#17-research-gaps)

---

## 1. Feature Overview and Purpose

**Official names:** "Billing Hub" (marketing and product communications) / "Billing Tab" (UI label and help center article title — used interchangeably)
**Help center article:** [The Billing Tab](https://support.ignitionapp.com/en/articles/9188062-the-billing-tab) (article ID 9188062)
**Introduced:** July 2024 quarterly product update

### Problem Statement

**Confidence: Confirmed**

Before the Billing Hub, practitioners had to navigate into each individual client's record — specifically the per-client Billing Schedule Tab — to identify and action any manual billing items (On Completion, Variable-Unit, or Estimate services). There was no single cross-client view of all pending manual billings.

This created two problems:
1. **Missed billings** — manual items sitting uninvoiced across many client files were easy to overlook, impacting cash flow.
2. **Operational overhead** — practitioners had to context-switch between many client records to manage billing for a full client list.

### What Billing Hub Solves

The Billing Hub aggregates all pending manual billing items across the entire practice into a single table, enabling practitioners to:
- See everything that needs to be invoiced in one place
- Action items (Invoice Now, Schedule Invoice, Delete) without entering individual client files
- Identify overdue unbilled items at a glance via a red "Overdue" badge that surfaces items past their due date

### Stated Benefits (from Ignition documentation)

- Faster payment collection — ensures manual billings are sent on time
- Better cash flow visibility — shows upcoming manual billings across all clients
- Improved operational efficiency — eliminates per-client navigation for manual invoicing tasks

### What Billing Hub Is NOT

- It is not a replacement for the per-client Billing Schedule Tab (which still exists and shows both manual and automatic items for a single client)
- It is not a post-invoice tracking tool (that is the Collections Tab)
- It is not related to AutoCollect / ledger-imported invoices (those appear in Collections → Outstanding)
- It does not show already-issued invoices (those are in the per-client Invoices Tab and Collections Tab)

---

## 2. UI Location and Navigation

**Confidence: Confirmed**

**Primary navigation path:** Sidebar → **Payments** → **Billing**

The Billing Hub is a sub-section of the top-level "Payments" area. Within Payments, the two main sections are:

```
Payments
├── Billing          ← Billing Hub lives here (pre-invoice, manual items)
└── Collections      ← Post-invoice payment tracking
     ├── Scheduled
     ├── Paid
     ├── Rejected
     └── Outstanding  (ledger-imported invoices via AutoCollect)
```

**Entry points:**

| Entry Point | Path | Notes |
|---|---|---|
| Direct navigation | Sidebar → Payments → Billing | Primary access |
| Home tab Billing Panel | Click "Unbilled" metric | Links directly to Billing Hub |
| Home tab Payments Panel | References "Payments → Billing" for un-billed items | Link-style reference; no one-click navigation |

The Home tab's **Billing Panel** displays a real-time count of "Unbilled" items (the total of On Completion and Variable-Unit services pending invoicing). Clicking this count is the fastest dashboard entry point into the Billing Hub.

---

## 3. What the Billing Hub Displays

**Confidence: Confirmed for documented fields; Unknown for exhaustive column list**

The Billing Hub displays a table where each row represents a single **manual billing item** — a service that requires the practitioner to actively trigger invoicing rather than being billed automatically.

### Confirmed Per-Row Data Points

| Field | Detail |
|---|---|
| Client name | Identifies which client the billing item belongs to |
| Service / billing item description | Name or description of the service to be invoiced |
| Amount | Expected invoice amount for the billing item |
| Due date | Date by which the item should have been invoiced; drives overdue calculation |
| Overdue badge (red) | Displayed on items past their due date; these rows float to the top of the list |
| Row actions (⋮) | Three-dot menu at end of each row for quick single-item actions |
| Multi-select checkbox | Enables bulk operations on selected items |

### What Does NOT Appear in the Billing Hub

**Confirmed exclusions:**

| Excluded item | Where it lives instead |
|---|---|
| Automatically billed recurring services | Per-client Billing Schedule Tab → "Billed Automatically" section |
| Already-issued invoices | Per-client Invoices Tab; Payments → Collections |
| Ledger-imported unpaid invoices (AutoCollect) | Payments → Collections → Outstanding sub-tab |
| Voided invoices | Per-client Invoices Tab (toggle required to view) |
| Variable Range and Variable Minimum price type services | Not forecasted; excluded from Billing Panel "Unbilled" count; relationship to Billing Hub rows not confirmed (see Research Gaps) |

### Note on Column Schema

The confirmed columns (Client, Service, Amount, Due Date) are inferred from multiple help articles. Additional columns that may exist (Proposal reference, Service Type, Billing Rule, Partner, Manager, Tags) are not explicitly documented. Full column schema requires in-app verification.

---

## 4. Views, Tabs, and Sort Behavior

**Confidence: Confirmed for overall structure; Inferred for sub-tab details**

### Structure

The Billing Hub is a single unified table view — there are no documented internal sub-tabs within the Billing Hub itself (unlike the Collections Tab, which has Scheduled / Paid / Rejected / Outstanding sub-tabs).

### Sort Behavior

- **Overdue items automatically float to the top** of the list with a red "Overdue" badge.
- Whether column headers are sortable (e.g., sort by Amount, by Client Name, by Due Date) is not explicitly documented.

### Multi-Select and Bulk Operations

- Users can select multiple rows via checkboxes.
- Bulk actions available: **Invoice Now**, **Schedule Invoice**, **Delete**.
- **Client-scoping constraint on bulk invoicing:** Selecting any billing item causes all items belonging to other clients to be greyed out. Bulk Invoice Now and bulk Schedule Invoice are restricted to one client at a time.
- **Bulk Delete has no client restriction:** Items from any combination of clients can be deleted in a single operation.

### Filters

A **"More Filters"** button provides additional attribute-based filtering beyond the default view. Specific filter attributes are documented in the context of the Proposals Tab (partner, manager, service, tags, dates) and are inferred to be similarly available in the Billing Hub. See Section 10 for detail.

---

## 5. Actions Available

**Confidence: Confirmed**

### 5.1 Invoice Now (Immediate Invoicing)

**Flow:**
1. Select one or more billing items for a **single client** (items from other clients become greyed out).
2. Click **"Invoice Now"**.
3. Review the amount and quantities; edit if needed.
4. **If a connected accounting ledger (Xero or QBO) is present:** A "Send Invoice" screen appears. The practitioner can:
   - Enable online payments (adds a "Pay Now" link for the client)
   - Send an invoice notification email to the client, or skip the email
5. Click **"Send Invoice"** to finalise.
6. Invoice is created in Ignition (and in the connected ledger if applicable).
7. The billing item is removed from the Billing Hub.

### 5.2 Schedule Invoice (Future Date)

**Flow:**
1. Select one or more billing items for a **single client**.
2. Click **"Schedule Invoice"**.
3. Review amount and quantities.
4. Pick a date using the date picker.
5. Confirm.
6. The billing item transitions from "Billed Manually" to "Billed Automatically" on the client's Billing Schedule Tab.
7. Ignition automatically raises the invoice on the selected date; no further practitioner action required.
8. The item moves out of the Billing Hub.

### 5.3 Delete

**Flow:**
1. Select one or more billing items (can span multiple clients).
2. Click **"Delete"**.
3. Confirm.

**Constraints:**
- **Permanent — no undelete function exists.**
- Deleting a billing item in Ignition does NOT affect any invoice already created in a connected Xero or QuickBooks account.
- The corresponding service remains active on the client record; only the pending billing item for that cycle is removed.

### 5.4 Quick Actions via (⋮) Menu (Per Row)

Each row has a three-dot (⋮) context menu for single-item quick actions. The documentation confirms this menu exists and enables "quick actions" but does not enumerate every available option. Likely options (inferred): Delete, and possibly Invoice Now / Schedule Invoice for the individual item without requiring multi-select.

### 5.5 Enable Online Payments (During Invoice Now Flow)

When creating an invoice from Billing Hub and a connected ledger is present, the practitioner can toggle on online payments. This adds a client-facing "Pay Now" link to the invoice, directing the client to the Ignition portal (`go.ignitionapp.com/`) to add a payment method and pay.

### 5.6 Actions NOT Available from Billing Hub

**Inferred:**
- **Void** — voiding an already-issued invoice is done from the per-client Invoices Tab or in bulk from the Collections Tab, not from the Billing Hub (the Billing Hub shows pre-invoice items only).
- **Edit service terms** — service editing is in the per-client Billing Schedule Tab or the proposal itself.
- **Mark as Paid** — available in the Collections Tab for issued invoices; not applicable to pre-invoice items in the Billing Hub.

---

## 6. Relationship to Proposals and Services

**Confidence: Confirmed**

The Billing Hub is the **downstream, cross-client aggregate** of the "Billed Manually" section on each client's individual Billing Schedule Tab. Whether a service generates a billing item in the Billing Hub depends on the service's billing rule, set at the proposal level.

### Billing Rule → Billing Hub Presence

| Billing Rule | Appears in Billing Hub? | Notes |
|---|---|---|
| **On Acceptance** | No | Invoice auto-generated immediately when client accepts proposal |
| **Deposit (first payment)** | No | Auto-generated immediately on acceptance |
| **Deposit (balance payment)** | Yes | Balance appears as a manual billing item after deposit is collected |
| **On Completion** | Yes | Primary Billing Hub use case |
| **Estimate** | Yes | Requires practitioner to confirm actual amount before invoicing |
| **Recurring (Fixed date, Monthly, etc.)** | No | Handled by Ignition's automatic billing engine; lives in "Billed Automatically" |
| **Variable Unit** | Yes | Practitioner enters quantity at billing time |
| **Variable Range** | Uncertain | Documented as "not forecasted"; Billing Hub inclusion not confirmed |
| **Variable Minimum** | Uncertain | Documented as "not forecasted"; Billing Hub inclusion not confirmed |

### Proposal Lifecycle Dependency

**Classic Proposals (legacy proposal type):**
- Billing items from a Classic Proposal only exist in the Billing Hub if the proposal is in **"Active"** status.
- If a Classic Proposal is set to **"Complete,"** its billing items are removed from the Billing Hub.
- Workaround: Temporarily move the proposal back to Active → Invoice from Billing Hub → re-set to Complete.

**New Proposals (current proposal type):**
- No such status dependency documented. Billing items persist in the Billing Hub regardless of the proposal's lifecycle state.

### Billing Item Consolidation Rules

When invoicing multiple billing items onto a single invoice from the Billing Hub, the items must share:
- The **same proposal**
- The **same payment method**
- The **same schedule / date**

Items that don't meet all three criteria cannot be consolidated and must be invoiced separately.

### Instant Bills

Instant Bills (created via "Create new → Instant Bill") allow firms to bill clients for ad hoc work without a preceding proposal. Whether a pending Instant Bill item appears as a row in the Billing Hub before invoicing is not explicitly confirmed in documentation. The Billing Panel's "Unbilled" count is documented to include Instant Bill services, which suggests pre-invoice Instant Bill items may appear in the Billing Hub. **This is an open research gap (see Section 17).**

---

## 7. Relationship to AutoCollect and Ledger-Imported Invoices

**Confidence: Confirmed**

**AutoCollect** (launched May 8, 2025) is a separate feature set that imports unpaid invoices from connected accounting software (Xero, QuickBooks Online) and enables automated payment collection on them.

**Ledger-imported invoices do NOT appear in the Billing Hub.** They appear exclusively in:

```
Payments → Collections → Outstanding
```

The Billing Hub and AutoCollect operate on completely separate tracks:

| Dimension | Billing Hub | AutoCollect / Outstanding Tab |
|---|---|---|
| Invoice origin | Ignition proposals and Instant Bills | External: Xero or QuickBooks Online |
| Stage | Pre-invoice (billing items not yet issued) | Post-invoice (invoices already issued in ledger) |
| Client requirement | Must have an accepted Ignition proposal or Instant Bill | No proposal required |
| Primary action | Invoice Now / Schedule Invoice | Request Payment / Schedule Collection |
| Navigation | Payments → Billing | Payments → Collections → Outstanding |

There is no documented workflow or UI interaction between the Billing Hub and AutoCollect.

### AutoCollect Import Rules (for context)

- Sync frequency: Xero every 1–2 hours; QBO daily
- Eligible invoices: Issued/Approved + Awaiting Payment + billed within last 90 days
- Ineligible: draft, voided, paid, multi-currency, negative line items, >90 days old
- Rate limit: 2,000 ledger events per account per 24-hour window
- Source filter in Outstanding tab: set to "Ledger" to isolate AutoCollect invoices

---

## 8. Relationship to the Collections Tab

**Confidence: Confirmed**

The Billing Hub and Collections Tab have clearly non-overlapping scopes at different stages of the billing lifecycle:

| Dimension | Billing Hub | Collections Tab |
|---|---|---|
| **Stage** | Pre-invoice | Post-invoice |
| **Content** | Billing items not yet invoiced | Invoices already issued |
| **Primary user action** | Invoice Now / Schedule Invoice / Delete | Track payments, retry failures, mark paid |
| **Ledger invoices** | No | Yes (Outstanding sub-tab) |
| **Navigation** | Payments → Billing | Payments → Collections |

### Collections Sub-Tabs Reference

| Sub-tab | Contents |
|---|---|
| **Scheduled** | Invoices issued with a payment method attached; not yet collecting |
| **Paid** | Invoices where payment collection has completed; also shows "Paid out" (funds disbursed to bank) |
| **Rejected** | Invoices where payment collection failed due to bank/card rejection |
| **Outstanding** | Unpaid invoices imported from Xero or QuickBooks via AutoCollect |

### Collections Export

The Collections Tab has a documented CSV export (triggered via Export button; sent to the user's email). It includes a "Surcharge Amount" column. This export covers post-invoice collections data only — not the pre-invoice billing items visible in the Billing Hub.

### Payment Progress Status Bar

The Collections Tab shows a visual "grey and green" progress bar per row indicating which step of the collection process a payment is at. This UI element is specific to the Collections Tab and does not appear in the Billing Hub.

---

## 9. Invoice Management Flows

**Confidence: Confirmed**

### 9.1 On Completion / Estimate Invoicing (Primary Billing Hub Use Case)

1. Client accepts a proposal containing an On Completion or Estimate service.
2. Billing item appears in the client's Billing Schedule Tab ("Billed Manually" section) and simultaneously in the cross-client **Billing Hub**.
3. When the work is done, the practitioner opens the Billing Hub, locates the item, edits the quantity or amount if needed (e.g., for Estimate services), and clicks **Invoice Now**.
4. If ledger is connected: "Send Invoice" screen → option to enable online payments → Send.
5. Invoice is created; item is removed from the Billing Hub.
6. Invoice appears in the Collections Tab (Scheduled or Outstanding depending on payment method status).

### 9.2 Deposit Balance Invoicing

1. Client accepts a proposal with a Deposit billing rule.
2. First invoice (the deposit) is auto-generated immediately on acceptance.
3. The balance portion appears as a manual billing item in the Billing Hub.
4. Practitioner invoices the balance via Invoice Now when appropriate.

### 9.3 Recurring Automatic Billing (Not in Billing Hub)

Recurring services from proposals (Fixed date, Monthly, etc.) are managed entirely by Ignition's billing engine:
- Invoices raised automatically on schedule.
- No practitioner action required in the Billing Hub.
- These items appear in the "Billed Automatically" section of the per-client Billing Schedule Tab only.

### 9.4 Scheduling a Future Invoice Date

1. Practitioner selects a billing item in the Billing Hub.
2. Clicks **Schedule Invoice** → picks a date.
3. The item moves from "Billed Manually" to "Billed Automatically" on the client's Billing Schedule.
4. Ignition automatically creates and sends the invoice on the scheduled date.
5. The item disappears from the Billing Hub once scheduled (moves to automated system).

### 9.5 Instant Bills

Instant Bills are one-off or ad hoc bills created without a proposal. Created via "Create new → Instant Bill" from any page in Ignition:
- Can bill multiple services at once (capability enhanced in July 2024 update).
- Practitioner has full control over billing timing (immediately or specific date).
- Can attach a payment method or prompt the client to provide one via the portal.
- Revenue tracked in the Billing Panel and the Services Revenue export.
- Whether a pending (pre-invoice) Instant Bill item appears in the Billing Hub is not explicitly confirmed (see Research Gaps).

---

## 10. Filters and Search

**Confidence: Partially confirmed; specific filter attributes are Inferred**

### Confirmed

- A **"More Filters"** button exists in the Billing Hub and provides attribute-based filtering.
- Overdue items automatically sort to the top (built-in sort behavior, not a user-applied filter).
- Multi-select mechanism exists for bulk operations.

### Inferred Filter Attributes

The Proposals Tab's "More Filters" is documented to include: Partner, Manager, Service, Continuous billing status, and date-based filters. Given the Billing Hub references the same "More Filters" UI pattern, analogous attributes likely exist for the Billing Hub:

| Filter | Availability | Confidence |
|---|---|---|
| Client name / search | Likely | Inferred |
| Partner (assigned user) | Likely | Inferred |
| Manager (assigned user) | Likely | Inferred |
| Client tags | Likely | Inferred |
| Due date range | Likely | Inferred |
| Service type | Likely | Inferred |
| Billing rule type (On Completion, Estimate, etc.) | Possible | Unknown |

### Search

Not explicitly documented for the Billing Hub. The Collections Tab is confirmed to have a "search function." Whether an equivalent text search bar exists in the Billing Hub is unknown.

### Sorting

Only overdue-to-top sort behavior is confirmed. Column-based sort (e.g., sort by Amount, Client Name, Due Date) is not documented.

---

## 11. Export

**Confidence: Unknown for direct Billing Hub export; Confirmed for related exports**

### Direct Export from Billing Hub

No documentation confirms a CSV export button within the Billing Hub. Unlike the Collections Tab (which has an explicit Export action), the Billing Hub's export capability is not described in any indexed source.

### Related Exports (Confirmed Alternatives)

| Export | Location | Contents | Format |
|---|---|---|---|
| **Services Revenue Export** | Home Tab → Services Panel → Export | Revenue by service or client for a date range; includes Instant Bill services. Fields: Service Long Description, Proposal Project, Client ID, Client Contact, Client Tags, Partner email, Manager email, Billing Group (XPM) | CSV (emailed) |
| **Active Services Export** | Clients Tab → Export | All currently active services (not inactive/expired) | CSV |
| **Collections Export** | Payments → Collections → Export | Post-invoice collections data; includes Surcharge Amount column | CSV (emailed) |
| **Proposals Export** | Proposals Tab → Export | Proposal metadata | CSV |
| **Weekly Summary Email** | Automated (Monday delivery) | Includes CSV download links for proposals accepted, awaiting acceptance, rejected payments, expiring credit cards | Email with CSV attachments |

**Developer note:** For extracting data equivalent to what the Billing Hub displays (pending manual billing items), the **Services Revenue Export** from the Home tab is the closest available mechanism. There is no documented standalone "Billing Hub export."

---

## 12. Notifications and Alerts

**Confidence: Confirmed for Billing Hub–adjacent alerts; Unknown for in-Hub notifications**

### Within the Billing Hub UI

| Alert | Type | Confirmed? |
|---|---|---|
| Red "Overdue" badge on items past due date | Inline row indicator; items float to top | Confirmed |
| In-app notification banners or alert panels specific to Billing Hub | Not documented | Unknown |

**Important distinction:** The per-client Billing Schedule Tab uses a **yellow** "Overdue" badge for overdue billing items. The Billing Hub uses a **red** "Overdue" badge for the same concept. These are two separate UI contexts with different badge colours confirmed by separate documentation sources.

### Adjacent Billing Notifications

| Notification | Trigger | Channel | Confirmed? |
|---|---|---|---|
| Weekly Summary Email | Every Monday (opt-in) | Email | Confirmed |
| Clients Page Overdue Filter | Clients with overdue billing items | In-app filter | Confirmed |
| Rejected payment notification | Payment rejection event | Email (firm + client) | Confirmed |
| Credit card expiry warning (in Weekly Summary) | Card expiring within 30 days or already expired | Email | Confirmed |
| Failed payment notification | Payment failure event | Email (firm + client) | Confirmed |

**Weekly Summary Email contents relevant to billing:**
- Rejected payments not yet resolved
- Credit cards expiring within the next month or already expired
- Does NOT explicitly include a "manual billing items overdue" section — though this is not explicitly ruled out in documentation.

### Notifications Settings

Ignition has a Notifications settings page (`/notifications`) where users can configure notification preferences. The specific billing-item-related triggers available are not fully documented in indexed sources.

---

## 13. Plan Tier Restrictions

**Confidence: Inferred**

No source explicitly states that the Billing Hub is restricted to specific plan tiers.

### Likely Available on All Paid Plans

- The Billing Hub is part of the core Payments experience, not referenced as a Pro-only or Pro+-only feature.
- Plan tiers (Solo, Core, Professional, Professional+) all include Ignition's billing workflow features as a core component.

### Prerequisites for Full Billing Hub Functionality

| Prerequisite | Required for |
|---|---|
| Ignition Payments enabled (KYC onboarding) | Online payment link on invoiced items; automatic collection after invoicing |
| Connected Xero or QBO | "Send Invoice" screen during Invoice Now flow; ledger invoice creation |
| Active proposals with accepted services | Billing items to appear in the Billing Hub |

Without Ignition Payments configured, the Billing Hub's core invoice-scheduling functions (Invoice Now, Schedule Invoice, Delete) are still accessible, but automated payment collection on those invoices will not occur.

### Known Plan-Gated Features (adjacent, not Billing Hub–specific)

- **Concatenating service name + description on Xero/QBO invoices:** Pro/Pro+ only
- **Multiple email templates:** Pro/Pro+ only
- **Some AutoCollect capabilities:** Time-limited free trial until May 5, 2025; plan-gated after that (specific plan restriction not fully documented)

The Billing Hub itself is not mentioned in the context of any plan-tier restriction in available documentation.

---

## 14. Launch Date and History

**Confidence: Confirmed (launch window); Inferred (exact date)**

### Timeline

| Date | Event |
|---|---|
| ~January 2024 | Ignition "What's New" page indexed with a Billing Hub reference (possibly pre-release or early rollout) |
| **July 19, 2024** | **Public announcement in quarterly product update** blog post ("Bringing E-Commerce Innovation to Professional Services") — earliest clearly dated public reference |
| July 2024 | Instant Bill multi-service capability also announced in the same update |
| May 8, 2025 | AutoCollect launched (separate feature; adds ledger invoice import to Collections → Outstanding) |
| May 15, 2025 | Ignition "Redefines Revenue Automation" press release — references AutoCollect and broader billing deepening |

### Prior to Billing Hub

Manual billing was done entirely per-client via the Billing Schedule Tab. Each client record had to be opened individually to identify and action pending On Completion or Variable-Unit billing items. The Billing Hub centralised these into a single cross-client table.

### Help Article ID Context

The Billing Hub help article ID `9188062` is consistent with a mid-2024 creation date. Ignition's older articles use IDs in the `600xxx` range; new articles use IDs in the millions, indicating this feature is relatively recent.

---

## 15. Edge Cases and Limitations

**Confidence: Mix of Confirmed and Inferred**

### Confirmed Limitations

| # | Limitation | Detail |
|---|---|---|
| 1 | **No undelete** | Billing items deleted from the Billing Hub cannot be restored; permanent action |
| 2 | **Bulk invoicing restricted to one client at a time** | Selecting any item greys out all items from other clients; applies to Invoice Now and Schedule Invoice only |
| 3 | **Bulk delete has no client restriction** | Can span all clients in a single delete operation |
| 4 | **Only manual billing items displayed** | Automatically billed (recurring) services are fully excluded |
| 5 | **Classic Proposal status dependency** | Items disappear from Billing Hub when a Classic Proposal is set to "Complete"; workaround: temporarily reactivate → invoice → re-complete |
| 6 | **Delete does not affect connected ledger** | Deleting a billing item in Ignition has no effect on any invoice already in Xero or QBO |
| 7 | **Billing item consolidation constraints** | Multi-item invoicing requires same proposal + same payment method + same schedule |
| 8 | **Single currency only** | Ignition does not support multi-currency accounts; all billing is in the account's default currency |
| 9 | **Negative line items not supported** | Services with negative pricing cannot be processed through any Ignition billing workflow |
| 10 | **Variable Range and Variable Minimum** | Not forecasted; excluded from Billing Panel "Unbilled" count; Billing Hub inclusion unconfirmed |
| 11 | **Voiding constraints** | Voiding an invoice is only possible before payment has been collected; cannot be done from the Billing Hub (pre-invoice only) |
| 12 | **Voided invoices hidden by default** | Voided invoices are hidden from client Invoices Tab (toggle required), excluded from CSV exports and Home page, and visible to Admin users only |
| 13 | **Ledger void sync is one-directional** | Voiding in Xero/QBO cascades to Ignition; voiding in Ignition does NOT cascade to Xero/QBO |

### Inferred / Likely Limitations

| # | Limitation | Basis |
|---|---|---|
| 14 | **No direct CSV export from Billing Hub** | No evidence of an Export button within the Billing Hub |
| 15 | **No in-app push notifications for overdue billing items** | Red overdue badge is the only documented visual alert mechanism |
| 16 | **No pagination or item limit documented** | Unknown; may be relevant for large client lists |
| 17 | **No real-time update documentation** | Unknown whether the list refreshes without page reload after an item is actioned |

---

## 16. Data Model: Billing Hub–Relevant Objects

Reconstructed from documentation. Field names are descriptive, not necessarily API-exact.

### BillingItem (Manual)

Represents a single pending manual billing item — one row in the Billing Hub.

| Field | Type | Notes |
|---|---|---|
| `id` | UUID | Ignition-internal identifier |
| `client_id` | FK → Client | The client this billing item belongs to |
| `proposal_id` | FK → Proposal | Source proposal; null for Instant Bills |
| `service_id` | FK → Service | The service being billed |
| `billing_rule` | Enum | on_completion, estimate, deposit_balance, variable_unit |
| `description` | String | Service description used on the invoice line item |
| `amount` | Decimal | Expected amount; editable before invoicing (especially for Estimate type) |
| `quantity` | Decimal | For Variable Unit services; entered/confirmed at billing time |
| `due_date` | Date | Date by which this item should have been invoiced |
| `overdue` | Boolean | Derived: `due_date < today` |
| `status` | Enum | pending (in Billing Hub), scheduled (moved to auto), invoiced (removed from Hub), deleted |
| `scheduled_date` | Date | Set when Schedule Invoice is used; null otherwise |
| `payment_method_id` | FK → PaymentMethod | Attached payment method if known at billing time |
| `proposal_type` | Enum | classic, new — affects lifecycle dependency rules |
| `created_at` | Timestamp | |
| `invoiced_at` | Timestamp | Set when Invoice Now is completed |
| `deleted_at` | Timestamp | Set on deletion (soft-delete if applicable) |

### BillingHubView (Aggregate Read Model)

| Field | Source | Notes |
|---|---|---|
| `items[]` | BillingItem WHERE status = pending | All manual billing items across all clients |
| `overdue_count` | COUNT WHERE overdue = true | Surfaced in Home tab Billing Panel |
| `unbilled_total` | SUM(amount) WHERE status = pending | Surfaced in Home tab Billing Panel "Unbilled" metric |
| `sort_order` | overdue first, then by due_date asc | Server-side sort; overdue items floated to top |

### Billing Panel Widget (Home Tab)

| Field | Detail |
|---|---|
| `unbilled_amount` | Total value of pending manual billing items; excludes Variable Range and Variable Minimum |
| `link_target` | Payments → Billing (Billing Hub) |
| `excluded_types` | Variable Range, Variable Minimum (not forecasted) |

---

## 17. Research Gaps

The following items could not be confirmed from available public documentation and represent engineering unknowns:

| # | Gap | Risk Level | Recommended Action |
|---|---|---|---|
| 1 | **Instant Bill items in Billing Hub** | High — product scope | Billing Panel "Unbilled" count includes Instant Bills but per-Hub-row inclusion is unconfirmed. Test with an Instant Bill set to a future billing date to see if it appears as a Billing Hub row. |
| 2 | **Exact column schema** | Medium — UI spec | Confirmed: Client, Service, Amount, Due Date. Additional columns (Proposal, Partner, Manager, Service Type) not confirmed. Verify in-app or via help article direct access. |
| 3 | **Exact "More Filters" attribute list** | Medium — UI spec | Filter options not enumerated for Billing Hub specifically. Analogous attributes documented for Proposals Tab; direct Billing Hub filter list requires in-app verification. |
| 4 | **Variable Range / Variable Minimum in Billing Hub** | Medium — data integrity | Both are excluded from the Billing Panel "Unbilled" metric as "not forecasted." Whether they appear as rows in the Billing Hub (without an amount) or are excluded entirely is undocumented. |
| 5 | **Search / text search bar** | Low–Medium — UX spec | Collections Tab has a confirmed search function; Billing Hub equivalent is unknown. |
| 6 | **Column sort controls** | Low — UX spec | Only overdue-to-top sort is confirmed. Whether columns are sortable by header click is unknown. |
| 7 | **Exact (⋮) quick actions menu options** | Low–Medium — UI spec | Documented as existing; specific per-row actions not enumerated. |
| 8 | **Plan-tier gating** | Low — product scope | No plan restriction documented; confirm against live pricing page at ignitionapp.com/pricing. |
| 9 | **Real-time update behavior** | Low — engineering | Whether Billing Hub refreshes live after a billing item is actioned or requires a page reload is undocumented. |
| 10 | **Maximum items / pagination** | Low — scaling | No documented limit or pagination for the Billing Hub item list. |
| 11 | **XPM-connected account behavior** | Low — integration | Whether the Billing Hub exposes XPM-specific fields (e.g., Billing Group) as columns or filters is not documented. |
| 12 | **Exact launch date** | Informational | July 2024 quarterly blog post is the earliest confirmed public reference. A What's New page snippet indexed with January 2024 may indicate an earlier rollout. Not blocking for engineering. |

---

*Sources: [The Billing Tab](https://support.ignitionapp.com/en/articles/9188062-the-billing-tab), [The Billing Schedule Tab](https://support.ignitionapp.com/en/articles/4373185-the-billing-schedule-tab), [Billing Panel](https://support.ignitionapp.com/en/articles/600892-billing-panel), [Payments Panel](https://support.ignitionapp.com/en/articles/600778-payments-panel), [Managing and Billing Your Clients](https://support.ignitionapp.com/en/articles/6997890-managing-and-billing-your-clients), [How to Invoice a Client Manually](https://support.ignitionapp.com/en/articles/4373236-how-to-invoice-a-client-manually), [Editing a Client's Billing Schedule](https://support.ignitionapp.com/en/articles/4569029-editing-a-client-s-billing-schedule), [Create an Instant Bill](https://support.ignitionapp.com/en/articles/8168559-create-an-instant-bill), [Rollout: Send Instant Bills from Ignition](https://support.ignitionapp.com/en/articles/8466821-rollout-send-instant-bills-from-ignition), [The Invoices Tab](https://support.ignitionapp.com/en/articles/4373249-the-invoices-tab), [Void or Delete Invoices in Ignition](https://support.ignitionapp.com/en/articles/13134076-void-or-delete-invoices-in-ignition), [The Collections Tab](https://support.ignitionapp.com/en/articles/9458405-the-collections-tab), [Payment Progress Statuses](https://support.ignitionapp.com/en/articles/9462734-payment-progress-statuses), [AutoCollect: Automate Invoice Payments](https://support.ignitionapp.com/en/articles/11362280-autocollect-automate-invoice-payments), [Import Unpaid Invoices from Connected Apps](https://support.ignitionapp.com/en/articles/10553614-import-unpaid-invoices-from-your-connected-apps), [Send Invoice Payment Requests](https://support.ignitionapp.com/en/articles/10968746-send-invoice-payment-requests), [Ignition and Xero](https://support.ignitionapp.com/en/articles/600784-ignition-and-xero), [Ignition and QuickBooks](https://support.ignitionapp.com/en/articles/600825-ignition-and-quickbooks), [Payments Overview](https://support.ignitionapp.com/en/articles/600819-payments-overview), [Mark Invoices as Paid](https://support.ignitionapp.com/en/articles/9824402-mark-invoices-as-paid), [Exporting Payment Collections Data](https://support.ignitionapp.com/en/articles/9716527-exporting-payment-collections-data), [Exporting Services Revenue from the Home Tab](https://support.ignitionapp.com/en/articles/2643778-exporting-your-services-revenue-from-the-home-tab), [How to Export Active Services](https://support.ignitionapp.com/en/articles/5722998-how-to-export-active-services), [Deposits](https://support.ignitionapp.com/en/articles/6024482-deposits), [The Weekly Summary Email](https://support.ignitionapp.com/en/articles/1765860-the-weekly-summary-email), [Notifications](https://support.ignitionapp.com/en/articles/9114609-notifications), [Quarterly Product Update — Bringing E-Commerce Innovation to Professional Services (July 2024)](https://www.ignitionapp.com/blog/quarterly-product-update-bringing-e-commerce-innovation-to-professional-services), [Ignition What's New](https://www.ignitionapp.com/whats-new), [AutoCollect press release (May 2025)](https://www.ignitionapp.com/news/ignition-launches-autocollect-to-end-the-business-chase-for-late-payments), [Ignition Redefines Revenue Automation (May 2025)](https://www.ignitionapp.com/news/ignition-redefines-revenue-automation-for-professional-services-businesses-across-the-entire-client-lifecycle), [Ignition's Instant Bill press release](https://www.ignitionapp.com/news/ignitions-instant-bill-new-feature-turns-scope-creep-into-profits-for-professional-services), [Home Overview](https://support.ignitionapp.com/en/articles/600762-home-overview).*
