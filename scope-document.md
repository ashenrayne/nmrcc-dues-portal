# Carpenter Council Payments Portal: Project Scope

**Document type:** Client-facing scope of work
**Prepared by:** Ashen Rayne
**Status:** Draft for client review
**Version:** 0.1

---

## 1. Executive Summary

This document describes the scope, assumptions, deliverables, and phasing for a new multi-tenanted online payments portal serving carpenter union councils and their constituent locals. The platform will allow members of the United Brotherhood of Carpenters (UBC) to view dues balances, make one-time payments, and enroll in recurring payments via Stripe, while giving councils and locals scoped administrative tools for managing members, processing payments, approving transactions, and reporting.

The platform is structured around a multi-tenant model in which each council operates an independent payments environment, with optional further isolation at the local level. Authentication is delegated to UBC's existing single sign-on, ensuring members do not need a separate account, and dues balance information flows from UBC's systems into the portal.

The MVP delivers a complete payments experience including card and ACH support, recurring billing, an internal bookkeeper approval workflow, member self-service, council and local admin dashboards, audit logging, and reconciliation reporting. Phase 2 introduces a pluggable external approval system that allows councils to delegate payment sign-off to third-party systems of record.

This is a financial system handling member dues across multiple legal entities. Reliability, auditability, and clear separation of tenant data are first-class concerns and shape several of the architectural decisions below.

---

## 2. Background and Goals

### 2.1 Background

UBC councils and locals already accept member dues through a range of channels (in-person, mailed checks, and one or another online method), with the specific mix varying by council. Online payment is part of the picture today; what is missing is a consistent online experience across councils and locals, and the operational tooling around it.

This platform is focused specifically on the **online** portion of dues collection. It is not intended to replace in-person or mail-in payment, and it does not change how councils handle those channels. Its goal is to simplify and unify how members pay dues online (one familiar experience regardless of which council or local they belong to) and to give councils and locals a stronger reporting, reconciliation, and collection workflow for the payments that flow through it.

### 2.2 Project Goals

The portal is intended to:

- Provide a consistent, modern payments experience for UBC members across all participating councils
- Reduce the manual workload of council and local bookkeepers through automation of recurring payments, receipts, and reconciliation reporting
- Give council and local administrators clear visibility into payment activity
- Maintain strict auditability of all financial actions, with logs sufficient to support both day-to-day support and formal audits
- Support each council's autonomy by allowing independent Stripe accounts, branding, settings, and approval policies
- Establish a foundation that can integrate with external systems of record in a later phase without re-architecture

### 2.3 Success Criteria

The MVP will be considered successful if:

- A pilot council can onboard, configure their Stripe account, allow members to sign in via UBC SSO, and accept live payments end to end
- Members can authenticate via UBC SSO, see their balance, pay, enroll in recurring, and self-serve common changes
- Council and local admins can view, filter, export, and reconcile payments without engineering involvement
- Bookkeepers can process the expected daily approval volume within their normal working hours
- All payment events and admin actions are captured in audit and event logs sufficient to investigate any individual member's payment history

---

## 3. In-Scope (MVP)

### 3.1 Multi-Tenant Architecture

The platform supports a hierarchy of tenants:

- **Councils** are the top-level tenant. Each council operates an independent environment with one or more Stripe Connect accounts, branding, settings, member roster, and admin users. A council can register multiple Stripe accounts to organize payments by region, fund type, or any other meaningful split.
- **Locals** are sub-tenants of a council. Each local is configured to receive its payments through one of the council's Stripe accounts, or (if the local operates with financial independence) its own Stripe account. Locals inherit certain settings from their council and override others.

Tenant isolation is enforced at the data access layer such that no admin or member can see data from another council or unrelated local. This is verified through automated tests as well as manual review.

Each council is reachable through one or both of the following surfaces:

- A council-scoped URL on a central UBC dues subdomain (e.g., `dues.carpenters.org/nmrcc` or `nmrcc.dues.carpenters.org`; final pattern to be confirmed during design)
- A per-council CNAME subdomain (e.g., `dues.nmrcc.org`) pointed at the platform's infrastructure, for councils that want a fully council-branded URL

Each council selects during onboarding which surface(s) to enable; both can be active simultaneously. The platform handles TLS certificate provisioning and renewal for either pattern. Light branding (logo, name, accent color, sender name on emails) is configurable per council and applies consistently across all delivery surfaces. Distribution, embedding, and mobile integration patterns are described in section 3.14.

### 3.2 Authentication and Member Identity

Members authenticate via the UBC single sign-on. The portal does not manage passwords for members. On first sign-in, a member record is provisioned in the relevant council and local based on UBC's response.

The portal queries UBC for member-specific data:

- **Current dues balance** including any past-due amount, retrieved at sign-in for display and as the default for one-time payments
- **Current dues rate** (the per-period amount used for recurring subscriptions), retrieved at subscription setup and refreshed via the background sync described in section 3.4
- **Member classification** (the status taxonomy used by the council for dues purposes, such as apprentice, journeyman, or retired), retrieved as part of the SSO claim or as a queryable field

The exact contract for these queries (endpoints, authentication, fields returned, refresh frequency, support for bulk or per-member access) is dependent on UBC documentation and will be confirmed in the discovery phase (open question 1, referenced in section 7). The portal treats UBC as the sole source of truth for what is owed, the dues rate, and member classification. The portal does not maintain its own rate data and does not allow rate changes to be entered or overridden in the portal admin UI. The portal is the source of truth for what has been paid through the platform.

Admins authenticate through a separate flow appropriate to their role (details in section 3.9).

### 3.3 Payment Processing: Stripe

Stripe is the payments provider for the MVP. Each council configures one or more Stripe accounts via Stripe Connect using Standard connected accounts, which preserves the council's direct relationship with Stripe and minimizes compliance burden on the platform.

Each local on the platform is configured with the Stripe account that will receive its payments. By default the council admin assigns one of the council's Stripe accounts to each local during local setup. A local that operates with financial independence may instead register its own Stripe account, in which case payments to that local route to the local's account rather than the council's. The assignment is per-local and can be changed by an authorized admin (subject to the migration considerations noted in section 8).

The platform supports both US and Canadian councils. Each Stripe account is tied to a single country and currency: US councils use US Stripe accounts settling in USD, Canadian councils use Canadian Stripe accounts settling in CAD. A member belongs to a single council and therefore transacts in a single currency; the platform does not perform currency conversion or cross-border settlement. Currency is shown explicitly in member-facing UI, receipts, admin dashboards, and reports so there is no ambiguity.

Payment methods supported at launch:

- **Card payments** via Stripe Elements / Checkout, in both USD and CAD. Card data never touches platform servers; PCI scope is limited to SAQ-A.
- **Bank debit** via Stripe ACH (US, USD) and Stripe Pre-Authorized Debit (Canada, CAD). Bank debit is included in the MVP because the per-transaction fee economics are materially better for monthly dues, and many members prefer it.

Payment types supported:

- **One-time payments** for current dues, past-due amounts, or partial payments (subject to council policy)
- **Recurring payments** with monthly, quarterly, and annual cadences, implemented via Stripe Subscriptions (cards) or Stripe-managed recurring bank debit

Stripe Smart Retries handle transient failures. Webhook events from Stripe are stored in an event log, processed idempotently, and exposed to ops for replay where needed.

### 3.4 Dues Amount Handling and Changes

Dues are a fixed amount per member per billing period and do not change on a regular basis. The amount applicable to a given member is determined from data provided by UBC SSO or council configuration at the time of payment, and the same amount is used for both one-time payments and each occurrence of a recurring payment. Recurring payments are implemented via fixed-amount Stripe Subscriptions rather than periodic invoicing.

Dues amounts change for two reasons, both originating in UBC's system:

- **Council-wide rate changes**: a council adjusts the dues rate that applies to a class of members in UBC. Examples: an annual rate increase, or a separate rate for a member status change (apprentice, journeyman, retired, etc.).
- **Per-member adjustments**: UBC's records reflect a different amount for a specific member, for example following a hardship adjustment or correction recorded in UBC's system.

The platform learns of both kinds of change through the sync described below. The portal does not provide a UI for entering or overriding rates.

**How rate changes reach the platform**

UBC is the sole source of truth for dues rates. The platform stores each active subscription's current rate (the per-period amount Stripe charges) and refreshes it from UBC; council admins do not enter rate changes directly in the portal.

A scheduled job (daily or weekly, configurable) refreshes per-member dues data from UBC for all active subscribers, regardless of whether the member has signed in recently. When UBC's data shows a different rate for a member, the platform queues a subscription rate update for the next charge on or after the effective date. This single mechanism catches all rate changes tracked in UBC's records: annual rate changes applied at the council level, classification changes (e.g., apprentice to journeyman), and per-member adjustments recorded in UBC.

The background sync requires UBC to expose a queryable per-member API or bulk feed (open question 1). If UBC's data is only available at SSO time, rate updates for an active recurring subscription can only happen when the member next signs in. This is an operational limitation rather than a workaround the portal can route around: a member with an active recurring subscription who does not sign in for a year would be charged at the prior rate for that year. UBC providing per-member data access between sign-ins is therefore a prerequisite for reliable rate-change propagation, not an optimization, and is captured in section 7 risks.

The platform does not compute rates from raw inputs (hours, status, rate tables). It stores the rate per subscription as provided by UBC.

**Effective-date semantics**

Each rate change has a configurable effective date. The new amount applies to the next regularly-scheduled charge on or after the effective date.

**Member notification is the council's responsibility**

Advance member notification of recurring-amount changes (where required by NACHA, PAD, or council policy) is handled by the council outside the platform. The platform does not send advance-notice emails for rate changes; it applies the new amount on the configured effective date and emits the standard payment receipt when the new amount is charged. The council is responsible for ensuring its own notice channel meets the applicable regulatory minimum before a recurring amount changes.

**Re-authorization rules**

Most rate changes do not require new member authorization. The platform requires re-authorization in the following cases:

- A change in funding source (member moves to a local with a different Stripe account; the existing subscription is cancelled and re-enrolled, per section 3.15)
- A change in cadence (monthly to quarterly, etc.)
- A council-policy-defined "substantial increase" threshold, configurable as a percentage or absolute amount

When re-authorization is required, the existing subscription is cancelled at end-of-cycle and the member is prompted to re-enroll on the new amount and terms. The re-enrollment flow makes the reason for the change visible to the member.

**Member recourse**

A member who wants to update their payment method, change cadence, pause, or cancel a subscription in response to a rate change can do so at any time from their payment management page in the portal. The platform does not gate or delay rate changes pending member acknowledgement.

**Failure handling**

If the new amount fails to charge (for example, insufficient funds when an increase pushes past the member's available balance), the platform applies its normal dunning process (Stripe Smart Retries plus the configured dunning notifications). The failure is also flagged in the admin dashboard with a rate-change-related indicator so council support can prioritize follow-up.

**Audit trail**

Every dues amount change is recorded in the audit log (section 3.11) with the source (council-wide rate change in UBC, classification change in UBC, per-member adjustment in UBC), the affected scope (which members and which subscriptions), the effective date, the sync run that detected the change, and the disposition of any required re-authorizations.

### 3.5 Member Experience

A signed-in member can:

- View their current balance and any past-due amount
- View their payment history with downloadable PDF receipts
- Make a one-time payment by card or ACH
- Enroll in recurring payments and choose a cadence
- Update or replace their saved payment method
- Pause, resume, or cancel a recurring payment
- Receive email notifications for receipts, failures, expiring cards, upcoming charges (especially for ACH), and past-due notices

### 3.6 Admin Experience

Council and local admins (and bookkeepers; see section 3.9) have access to a dashboard scoped to their council or local. Capabilities include:

- A payments list showing all payment records, filterable by date range, status (succeeded, failed, refunded, pending approval, approved, rejected), payment method, member, local, and amount
- Saved filter views per user
- Export to CSV and Excel
- Drill-down to a specific payment, showing the full event timeline (initiated, charged, webhook received, approved, receipt sent, etc.) and links to the underlying Stripe charge for support purposes
- Member view: search and filter members, see member detail, view a member's payment history and recurring schedule, issue a refund or credit (subject to permission)
- Council settings (council admin only): branding, notification copy, approval policy, Stripe accounts (add, configure, and assign to locals), role assignments
- Local settings (local admin only, where applicable): local branding overrides, contact information, and (if the local operates its own Stripe account) the configuration for that account

### 3.7 Refunds, Adjustments, and Credits

The MVP supports issuing refunds through Stripe directly from the admin UI, with permission required. Council policy determines whether full and partial refunds are allowed, who can issue them, and any threshold above which a higher-level approval is required. Refund issuance is logged in the audit log with required reason.

Credits (applying a balance to a future payment rather than refunding cash) are supported as a manual adjustment recorded against the member.

### 3.8 Payment Approval Workflow

Each council can enable an internal approval workflow under which every successful payment (both one-time and each occurrence of a recurring payment) must be marked as approved by a bookkeeper before the payment is recorded as a valid contribution to the member's record.

The approval workflow is built on a pluggable interface from day one. The MVP ships with an **internal approval plugin** that places payments into a queue for designated bookkeepers to review. The same interface will accommodate external approval plugins in Phase 2 without rework.

Key behaviors:

- A successful Stripe charge enters a `pending_approval` state. Approval status is internal-only; members see the payment as successful and are not exposed to the pending/approval workflow in the UI or in notifications.
- Until the payment is approved, dunning notifications and past-due status are suppressed for that payment. (A past-due member who pays does not receive a "you are still past due" message during the verification window.)
- The member receipt is sent immediately on successful Stripe charge, regardless of approval status. The member is not notified again when a bookkeeper approves the payment; approval is an internal bookkeeping step.
- Bookkeepers work the approval queue with bulk actions, filtering, sorting, keyboard shortcuts, saved views, and inline member context. Anomalous payments (unusual amount, new payment method, new member, member with prior rejected payments) are visually flagged.
- Approvals can be reversed within a defined window via an explicit "reverse approval" action that itself logs.
- Rejected payments can be refunded automatically, held for further review, or handed back to admin discretion based on council policy.
- Backlog escalation: payments pending more than a configurable number of business days surface to council-level admins.

A council that does not enable internal approval has payments auto-marked as approved on charge success, preserving the same data model and audit trail.

### 3.9 Roles and Permissions

The MVP uses a permission-based authorization model with the following preset roles:

| Role | Scope | Typical capabilities |
|------|-------|---------------------|
| Global Admin | Platform | Manage councils, platform-level configuration, support tooling |
| Council Admin | Council | Manage council settings, locals, members, refunds, role assignments within the council |
| Council Bookkeeper | Council | Approve/reject payments across all locals in the council; view payment history; cannot change settings |
| Council Auditor (read-only) | Council | View all payment data and audit logs; no write actions |
| Local Admin | Local | Manage members within the local, view payments, issue refunds (if permitted) |
| Local Bookkeeper | Local | Approve/reject payments within the local; view payment history |
| Local Auditor (read-only) | Local | View payment data within the local; no write actions |

Permissions are stored as flags so that custom role variants can be created if a council requires a non-standard combination.

### 3.10 Notifications

The platform sends transactional emails to members and to council admins/ops staff. All emails are templated with per-council branding and customizable copy. Email delivery is via UBC's existing internal mail relay; the platform sends through UBC's relay rather than a separate third-party email provider. Relay configuration (address, port, authentication, sending domain) is confirmed during discovery.

The **Configurability** column below uses the following shorthand:

- **Copy**: the email subject, body, and branding can be customized by the council
- **Timing**: the trigger interval or threshold (e.g., days before, days after) is configurable
- **On/off**: the council can disable the email entirely
- **Intervals**: multi-send schedules (e.g., 30/14/7 days) are configurable
- **Cadence**: digest schedule (daily, weekly, monthly) is configurable
- **Threshold**: alert threshold count or duration is configurable
- **Recipients**: which roles receive the email is configurable

Rows that say *cannot disable* in the configurability column are required for compliance, payment-method maintenance, or council operations and are delivered regardless of council preference; copy can still be customized.

#### Member Emails

| Event | Trigger (default) | Content summary | Configurability |
|-------|-------------------|-----------------|-----------------|
| Welcome | First successful sign-in to the portal | Account confirmation, link to manage payment methods, council contact | Copy, on/off |
| Recurring enrollment confirmation | Member enrolls in recurring | Cadence, next charge date, amount, payment method (last 4), manage/cancel link | Copy |
| Payment receipt | Successful Stripe charge | Amount, date, payment method (last 4), council and local, payment ID, downloadable PDF | Copy only (cannot disable; required for member records) |
| Upcoming ACH/PAD charge | 3 calendar days before scheduled bank-debit charge | Amount, date, payment method, manage/cancel link | Copy, timing, on/off |
| Card expiring soon | 30, 14, and 7 days before stored card expiry on an active subscription | Last 4, expiry month/year, link to update payment method | Copy, intervals, on/off |
| Payment failed (first attempt) | Stripe `charge.failed` on a recurring charge | Reason, retry schedule, link to update payment method | Copy |
| Payment failed (retries exhausted) | Stripe Smart Retries exhausted | Action required, dunning timeline, contact path | Copy |
| Past-due notice | Configurable: default 7 days after retries exhausted | Outstanding amount, payment link, council contact for hardship | Copy, timing, on/off |
| Recurring canceled (member-initiated) | Member cancels subscription | Confirmation, last/next charge, re-enroll link | Copy, on/off |
| Recurring canceled (system-initiated) | Cancellation after dunning escalation per council policy | Reason, contact council, re-enroll link | Copy |
| Recurring paused / resumed | Member pauses or resumes subscription | Confirmation, what changes, when next charge resumes | Copy, on/off |
| Payment method updated | Member updates saved payment method | Confirmation, last 4 of new method | Copy, on/off |
| Re-authorization required (lifecycle event) | Cancel-and-re-enroll triggered by a lifecycle event (section 3.15) | Reason for change, deadline, re-enroll link | Copy, timing |
| Re-authorization reminder | 7 days before re-auth deadline (configurable; multiple sends) | Reminder, deadline, re-enroll link | Copy, timing, intervals |

#### Admin and Ops Emails

| Event | Recipient | Trigger (default) | Content summary | Configurability |
|-------|-----------|-------------------|-----------------|-----------------|
| Approval queue daily digest | Council Bookkeeper, Local Bookkeeper | 8am Eastern weekdays | Pending count, oldest item age, link to queue | Copy, time, cadence, on/off |
| Approval queue threshold breach | Council Bookkeeper | Pending count exceeds configurable threshold (default 100) | Pending state plus alert framing | Copy, threshold, on/off |
| Approval queue escalation | Council Admin | Item pending more than configured business days (per section 3.8) | Specific item(s), age, member, link | Copy, threshold |
| Anomalous payment flagged | Council Bookkeeper, Local Bookkeeper | Anomaly rule fires (per section 3.8) | Payment ID, anomaly reason, link | Copy, on/off |
| Refund issued | Council Admin | Refund processed | Member, amount, reason, executor, payment link | Copy, on/off |
| Dispute / chargeback opened | Council Admin, Council Bookkeeper | Stripe `charge.dispute.created` | Member, amount, dispute reason, deadline to respond, Stripe dashboard link | Copy only (cannot disable; council must respond within Stripe's deadline) |
| Stripe payout received | Council Admin (council-owned account) or Local Admin (local-owned account) | Stripe `payout.paid` | Amount, date, account, link to reconciliation view | Copy, on/off |
| KYC reverification required | Council Admin (council-owned account) or Local Admin (local-owned account) | Stripe `account.updated` indicates verification needed | Account, requirements, deadline, Stripe link | Copy only (cannot disable; payouts blocked otherwise) |
| Webhook backlog alert | Council Admin | Backlog exceeds threshold or oldest unprocessed event exceeds duration | Backlog summary, link to event log | Copy, threshold |
| UBC sync error or rate divergence | Council Admin | Sync run fails or detects divergence above threshold (per section 3.4) | Members affected, divergence summary, link to admin tooling | Copy, threshold |
| Reconciliation report ready | Council Admin, Council Auditor | Monthly (configurable) | Period summary, payouts vs. payments, download link | Copy, cadence, recipients |
| Past-due member report | Council Admin | Weekly (configurable) | Past-due roster, totals, download link | Copy, cadence, on/off |
| Lifecycle event executed | Council Admin of source and destination | Lifecycle event record written (per section 3.15) | Event summary, member-reassignment summary, post-cutover checklist link | Copy |
| Council settings changed | Council Admin | Significant settings change (branding, approval policy, role assignments, Stripe account add/remove) | What changed, by whom, when | Copy, on/off, which settings |

SMS notifications are out of scope for the MVP.

### 3.11 Logging, Auditing, and Observability

Three distinct logging channels are established:

- **Application logs** for engineering: errors, performance, request traces. Routed to AWS CloudWatch Logs.
- **Audit log** for compliance and trust: who did what, when, with what reason. Visible to admins, immutable, exportable. Includes role assignments, refunds, credit adjustments, approvals and reversals, dues amount changes (with source, scope, effective date, and re-authorization disposition; see section 3.4), lifecycle events such as council mergers and member reassignments (see section 3.15), settings changes, and login events.
- **Payment event log** for support and ops: every Stripe webhook, every charge attempt, every state transition on a payment, with full payload preserved. Critical for investigating individual member issues.

Webhooks are processed idempotently with a manual replay tool for ops use during incident recovery.

### 3.12 Reporting and Reconciliation

Council and local admins can export payment activity by date range. A monthly reconciliation report ties payment activity to Stripe payouts, accounting for the timing offset between charge and payout, so that bookkeepers can match deposits to recorded payments with minimal manual work.

Standard reports included in the MVP:

- Monthly payment activity by local
- Reconciliation report (payouts vs. payments)
- Past-due member report
- Approval activity report (volume, time-to-approval, rejection reasons)
- Failed payment report

### 3.13 Responsive Web

The platform is delivered as a responsive web application that works on modern desktop and mobile browsers. The MVP does not include a native mobile app built by Ashen Rayne. Integration with UBC's existing React Native app is supported as described in section 3.14.

### 3.14 Delivery Surfaces, Embedding, and Mobile Integration

The platform reaches members through three coordinated surfaces. All share authentication, payment data, audit logging, and per-council branding.

**1. Hosted portal (primary surface).** The portal lives at the URL(s) configured in section 3.1: a central UBC dues subdomain, a per-council CNAME subdomain, or both. SSO and Stripe operate in their normal top-level browser context, with no third-party cookie or framing concerns.

The **central UBC dues URL** (e.g., `dues.carpenters.org`) serves as the canonical entry point for members who don't know their council's path or who arrived through a generic UBC link. Anonymous visitors see:

- A brief explainer of the program (what the platform does, what payment methods are accepted, where to get help)
- A directory of participating councils, each linking to its council surface
- A prominent "Sign in" call-to-action for members

Signing in from the central URL triggers UBC SSO. On successful return, the platform reads the member's council association from the SSO claims (the same claim referenced in section 3.2) and redirects the member to the correct council surface. If the SSO claims do not include a council association (edge case to be confirmed during discovery), the platform shows a council picker as a fallback. The central URL also hosts:

- Council admin sign-in, as a separate entry distinct from the member SSO flow (per section 3.9 roles)
- Stripe Connect OAuth callbacks for council onboarding
- A system status link and support contact information

A **council surface** (e.g., `nmrcc.dues.carpenters.org`, `dues.carpenters.org/nmrcc`, or a CNAME like `dues.nmrcc.org`) lands a member directly in the council-branded experience without going through the directory. SSO from a council surface is scoped to that council, and the member proceeds straight to balance and payment.

**2. Embedded "Pay Dues" widget for council websites.** Each council can drop a small JavaScript snippet into their own website to expose dues functionality without visually leaving the council site. The widget renders branded entry points (a "Pay Dues" button, an optional balance summary card, or both) and opens the full SSO and payment flow in a top-level navigation or popup. A `postMessage` callback notifies the embedding page when the flow completes (success, cancel, or error).

In-page iframe payment is explicitly not supported because third-party cookie restrictions in modern browsers (Safari ITP, Chrome's cookie phase-out) silently break SSO and Stripe sessions inside iframes. The widget approach gives the embedded look-and-feel without the reliability risk.

**3. UBC mobile app integration (system-browser handoff).** For UBC's React Native app, the platform supports a system-browser handoff pattern using Safari View Controller on iOS and Chrome Custom Tabs on Android:

- The UBC app exposes a "Pay Dues" entry that opens the portal in the OS browser
- The member completes SSO and payment in a real browser context, with full cookie support and Stripe SDK behavior identical to the web experience
- The portal deep-links back to the UBC app on completion, with a status payload (success, cancel, or error)

In-app WebView payment is explicitly not supported. Mobile WebViews have stricter cookie isolation than even mobile Safari, and Stripe's guidance discourages running Stripe Elements inside an app WebView for PCI scrutiny reasons. Native in-app payment via the Stripe React Native SDK (with a corresponding authenticated platform API) is a Phase 2 candidate if UBC's app team wants an experience that feels fully native.

The MVP exposes the public surface needed for these patterns: stable URLs, deep-link return, and a small `postMessage` event surface for the widget. It does not yet expose a general-purpose authenticated API for native callers.

### 3.15 Lifecycle Events: Mergers, Dissolutions, and Reassignments

Councils, locals, and members are not static. Councils occasionally merge, locals close or are reorganized, and members move between locals. The MVP supports these lifecycle events through architectural primitives built into the platform, combined with a documented operational workflow for higher-risk events.

**Architectural primitives in the MVP**

- **Soft-archive** for councils and locals: archived entities are never hard-deleted. All historical payment data, audit log entries, and reports remain attributable to the original entity indefinitely.
- **Bulk member reassignment** between councils or locals, with full audit trail. A dry-run mode shows the impact (in-flight payments, recurring subscriptions, refunds in process) before commit.
- **URL and CNAME redirects** from archived entities to their successors, configurable as part of the reassignment.
- **Subscription disposition rules** per lifecycle event, configurable to either:
  - *Continue on the original Stripe account*: existing recurring payments keep charging against the original account; reporting attributes them to the successor going forward via the merger event link. No member action required. Used when the source Stripe account is being retained by the survivor.
  - *Cancel and re-enroll on the successor's Stripe account*: required when the source Stripe account is being retired, since Stripe subscriptions cannot be moved between accounts. Members are notified and re-authorize on the new account within a configurable window.
- **Lifecycle event records**: each merger, dissolution, or significant reassignment produces an immutable record linking source and destination entities, timestamp, executor, subscription disposition applied, and member-reassignment summary. Reports use these records to keep historical numbers attributable while showing current roll-ups under the successor.

**Self-service vs. runbook**

Routine lifecycle events are self-service for council admins, with dry-run and confirmation. Higher-risk events are runbook-driven and executed by Ashen Rayne under a paid support engagement (see proposal). The split:

| Event | Mechanism |
|-------|-----------|
| Member moves between locals in the same council | Council admin self-service |
| Member moves between councils | Council admin self-service, with confirmation |
| Local closure with members reassigned within the same council | Council admin self-service, dry-run + confirm |
| Local migration between councils | Runbook (Ashen Rayne) |
| Council merger (one or more councils into a survivor) | Runbook (Ashen Rayne) |
| Council dissolution with no successor | Runbook (Ashen Rayne) |

The MVP intentionally does not include a self-service "merge council" UI. Council-level lifecycle events are rare, high-stakes, and the cost of building a self-service wizard against the cost of getting it wrong does not pay back at the frequency these events occur. Council admins request a council-level event via the support channel; Ashen Rayne executes it under the documented runbook with the council in the loop at each phase.

**Runbook phases for Ashen Rayne-executed events**

1. **Pre-cutover**: identify in-flight payments and pending approvals; inventory recurring subscriptions and their Stripe account placement; agree on the subscription disposition rule with the council; configure URL and CNAME redirects; draft member communication.
2. **Cutover**: archive source entities, reassign members to destinations, apply subscription disposition rule, apply redirects, write the lifecycle event record.
3. **Post-cutover**: reconciliation across the affected period, member support window for re-enrollment if the cancel-and-re-enroll path was used, validation that historical reports still resolve correctly.

**Out of scope for the MVP**

- A self-service merger or council-dissolution UI for council admins
- Automatic re-enrollment of members on a new Stripe account (Stripe and payment regulations require explicit member authorization for subscriptions on a new account)

### 3.16 Testing and Quality Assurance

The platform handles member dues across multiple legal entities and Stripe accounts; correctness in the payment paths is a first-class delivery commitment. The MVP includes the following testing layers:

- **Unit tests** for business logic where bugs would change financial outcomes: rate updates applied from the UBC sync, subscription update logic, approval workflow state transitions, dunning timing and suppression, lifecycle event reassignment logic, and Stripe webhook idempotency.
- **Integration tests** for the boundaries where the platform meets external systems: Stripe webhook handling end-to-end, Postgres Row-Level Security policy enforcement, the UBC sync path (against recorded fixtures if a UBC test environment is not available), and the approval plugin interface.
- **End-to-end tests** (Playwright) for the critical member and admin paths: one-time payment, recurring enrollment, payment method update, refund processing, approval workflow (approve and reject), and the council admin sign-in flow.
- **Cross-tenant isolation tests** running in CI on every commit. These verify that no admin or member can read or write data belonging to another council or unrelated local. This is the highest-stakes correctness property in the system and is treated accordingly.
- **Manual QA** for items that do not automate well: accessibility (keyboard navigation, screen reader behavior on the critical paths), per-council branding correctness across all delivery surfaces (per section 3.14), email rendering across major clients, and the bookkeeper queue UX under realistic data volumes.
- **Security review** of the codebase and configuration before pilot launch, focused on the auth and RLS surface, Stripe webhook signature verification, the embed and widget cross-origin behavior, and the absence of cross-tenant data exposure.

The platform commits to coverage of *what gets tested* (the critical paths above) rather than a percentage-of-lines-covered target. Coverage targets without a specification of what is covered tend to incentivize coverage-padding tests that do not catch real bugs; specifying the paths that must remain green is more honest about what is being delivered.

Test failures block deploys. The CI pipeline runs the unit, integration, and cross-tenant suites on every commit, with a deploy-gating policy that prevents merging or deploying code that breaks any of them. End-to-end tests run on every merge to the main branch and on a nightly schedule against a deployed environment.

---

## 4. Out of Scope (MVP)

The following are explicitly excluded from the MVP. Items marked **Phase 2** are planned for the next phase; others are deferred or excluded entirely.

- **External approval plugins**: Phase 2. The internal approval plugin ships in MVP; external plugins for third-party systems of record ship in Phase 2.
- **Additional payment providers**: Stripe is the sole provider for MVP. The data model and service boundaries are designed to allow another provider to slot in later, but no second provider is implemented.
- **Cross-currency operations**: a member transacts in a single currency (their council's currency). The platform does not convert between USD and CAD or perform cross-border settlement. Multi-currency support means each council settles in its own currency, not that any individual member or transaction is multi-currency.
- **Manual / offline payment recording**: the platform tracks digital payments only. Checks, cash, and money orders received outside the platform are not entered into the system; councils continue to handle those through their existing bookkeeping processes.
- **Native mobile applications built by Ashen Rayne**: the MVP does not include a separate iOS or Android app. The platform does support UBC's existing React Native app via the system-browser handoff pattern in section 3.14. Native in-app payment via the Stripe React Native SDK is a Phase 2 candidate.
- **In-page iframe payment** in council websites: see section 3.14 for the supported widget pattern.
- **Advance-notice emails for dues rate changes** to members. Notification of rate changes (where regulatorily required or by council policy) is handled by the council outside the platform; see section 3.4. Standard payment receipts are still sent for each successful charge.
- **Proration of mid-cycle rate changes**: rate changes apply to the next regularly-scheduled charge after the effective date. The platform does not generate one-time mid-cycle charges or credits to settle the difference at change time.
- **Council admin entry or override of dues rates** in the portal. UBC is the sole source of truth for dues rates; the portal admin UI does not include a path for entering, overriding, or adjusting rates (see sections 3.2 and 3.4). Rate changes happen in UBC and propagate to the portal via the sync.
- **In-app WebView payment** in UBC's React Native app: see section 3.14 for the supported system-browser handoff.
- **In-app messaging between members and admins**: email is the communication channel.
- **SMS notifications.**
- **Member-to-member features, forums, content**: this is a payments portal, not a member portal.
- **Full general-ledger accounting**: the platform produces reconciliation reports; it does not replace council bookkeeping software.
- **Migration of historical payment data** from prior systems.
- **White-label custom domains per council** beyond a configurable subdomain (Phase 2).
- **Custom report builder**: standard reports are included; ad-hoc report building is Phase 2.

---

## 5. Phasing

### Phase 1: MVP

All items in section 3. A representative sequencing is:

1. Discovery and design: UBC SSO contract, Stripe Connect onboarding flow, dues amount confirmation, branding requirements, role refinement
2. Foundation: multi-tenant data model, authentication, council/local provisioning, role and permission framework, audit log
3. Payments core: Stripe Connect integration, one-time payments, member dashboard, payment event log, webhooks, receipts
4. Recurring payments: subscription enrollment, payment method management, dunning, retries, ACH support
5. Admin experience: dashboards, payment list and filters, exports, member management, refunds
6. Approval workflow: internal approval plugin, approval queue UI, bulk actions, anomaly flagging, escalation, configurable approval policy
7. Notifications and templates: per-council branding, customizable copy, transactional email integration
8. Reporting and reconciliation: standard reports, payout reconciliation
9. Pilot with one council: hardening, training, support runbook, incident response process
10. Broader rollout

### Phase 2

- External approval plugin framework with one or more concrete external system integrations (target system to be identified)
- Plugin admin UI for configuring integrations, including authentication, field mapping, retry policy, and timeout behavior
- Plugin-specific monitoring, replay, and dead-letter tooling
- Additional items deferred from MVP based on pilot feedback (custom domains, additional reports, etc.)

---

## 6. Assumptions

The estimates and approach in this document depend on the following assumptions. If any prove incorrect, scope, timeline, or cost may need adjustment.

- UBC provides a documented SSO mechanism (OIDC, SAML, or equivalent) with reasonable availability, and exposes a queryable API or feed for per-member dues data (current balance, current rate, classification). Per-member access between sign-ins is a prerequisite for reliable rate-change propagation to active recurring subscriptions, since the portal does not allow rate changes to be entered or overridden in its own admin UI. If UBC can only return data at SSO time, rate updates for active recurring subscriptions will lag until the member next signs in (see section 7 risks).
- UBC's dues balance data is sufficient to determine what a member owes at the time of payment. The portal does not need to compute dues from raw inputs (hours, status, rate tables).
- Dues amounts are fixed per member per billing period and do not change on a regular basis. Annual or occasional rate changes are applied to subscriptions on the next billing cycle on or after the configured effective date.
- Advance notification of recurring-amount changes (where required by NACHA, PAD, or council policy) is performed by the council outside the platform. The platform applies the new amount on the configured effective date and does not send advance-notice emails for rate changes.
- The MVP supports both US and Canadian councils. US councils settle in USD via US Stripe accounts; Canadian councils settle in CAD via Canadian Stripe accounts. Each council operates in a single currency.
- Each council can complete Stripe Connect onboarding (including business verification) for their own account(s). The platform facilitates this but cannot complete it on the council's behalf.
- Councils that elect a per-council CNAME subdomain provide the DNS configuration (CNAME record pointing at the platform's endpoint).
- Councils that want the embedded "Pay Dues" widget on their site provide the integration point (where the snippet is placed) and any non-default styling requirements.
- UBC's React Native app team provides the deep-link or Universal Link configuration that the platform redirects to on payment completion, plus the in-app entry point that opens the system browser.
- The platform charges no per-transaction application fee in the MVP unless agreed otherwise; transaction-fee economics flow directly between councils and Stripe.
- The pilot council has identified bookkeepers willing to operate the approval queue and provide feedback during pilot.
- Hosting is on AWS, in an account managed by Ashen Rayne. The architecture supports a future move to UBC's AWS organization without code changes if UBC elects to host the platform internally.

---

## 7. Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| UBC SSO outage prevents member sign-in and therefore payment | High | Status page and member communication during outages; in-flight recurring payments continue to charge via stored payment methods independent of SSO; monitor UBC SSO availability and alert on degradation |
| Stale dues balance from UBC causes duplicate payments or incorrect past-due display | Medium-high | Define explicit handshake on payment success; show last-synced timestamp to members; idempotency on payment intent creation |
| UBC-side rate change does not propagate to active recurring Stripe subscriptions before the next charge, causing members to be charged the prior amount | Medium-high | Background sync (see section 3.4) refreshes per-member rates between sign-ins on a configurable cadence; reconciliation reports flag rate divergence between UBC and the portal. If UBC does not expose per-member data between sign-ins, propagation depends on member sign-in and is documented as an operational limitation (the portal does not provide a manual override path for rates by design) |
| Webhook delivery failures cause payments to appear missing | Medium | Webhook event log with manual replay; reconciliation report catches discrepancies; alerting on webhook backlog |
| Bookkeeper backlog delays UBC balance update, causing the member to still see past due after paying | Medium | Suppress dunning during pending state; backlog escalation to council admin; rely on the source-of-truth handshake (section 7) so balance updates do not depend on bookkeeper turnaround when avoidable |
| Cross-tenant data leak | High | Row-level security or strict scoping at ORM layer; automated cross-tenant access tests in CI; security review before pilot |
| PCI scope creep | Medium | Card data handled exclusively via Stripe Elements / Checkout; no card data on platform servers; periodic review |
| A local's Stripe account assignment changes mid-flight (between council accounts, or to/from a local-owned account) | Medium | Stripe subscriptions cannot move between accounts; document a migration procedure that cancels and recreates active recurring payments with member consent and clear notification |
| External approval system (Phase 2) outage stalls payment finalization | Medium | Manual override always available; configurable timeout fallback (auto-approve, auto-reject, or manual queue); clear shared-responsibility documentation |
| Third-party cookie restrictions in browsers and mobile WebViews break embedded SSO sessions | Medium-high if iframes/WebViews are used | The platform avoids in-frame payment entirely: top-level navigation or popup for council site embeds, system-browser handoff for the UBC mobile app (see section 3.14). Both patterns sidestep the third-party cookie surface |
| UBC mobile app deep-link configuration drifts after a UBC app update, breaking return-to-app on payment completion | Low | Deep-link target is configurable in the platform admin; UBC app and platform agree on a versioned link scheme; smoke test included in UBC's release checklist |
| Council merger or dissolution causes loss of historical attribution, member confusion, or orphaned recurring payments | Medium-high if mishandled | Soft-archive (no hard deletes); immutable lifecycle event records linking source and destination; runbook-driven cutover with member communication; subscription disposition rule decided up front per event (see section 3.15) |
| Members do not re-authorize subscriptions in time after a "cancel and re-enroll" lifecycle event, falling past-due | Medium | Configurable re-authorization window with multiple reminder emails; dunning suppression during the window; council-admin visibility into who has not yet re-enrolled; fallback to one-time invoicing for members who lapse |
| Council fails to send NACHA-compliant or Canadian-PAD-compliant advance notice for a recurring dues amount change before the new amount charges, exposing the council to compliance penalties or returned debits | Medium | Advance notification is the council's responsibility and is handled outside the platform (per section 3.4 and section 6 assumptions); the responsibility split is documented and covered in council admin onboarding/training; rate-change effective dates are configurable with sufficient lead time so the council can run its notification process before the platform applies the change |

---

## 8. Architecture Overview (high level)

A more detailed architecture document will be produced during discovery. At a high level, the system comprises:

- A web application frontend for members and admins, served from a single codebase with role-based experiences
- A backend API enforcing tenant scoping and role-based permissions on every request
- A primary relational database (PostgreSQL) with strict tenant scoping, augmented by a payment event store
- Stripe integration via Stripe Connect (Standard accounts), Stripe Elements / Checkout for card capture, Stripe webhooks for event ingestion
- An asynchronous job system for sending notifications, processing webhooks, generating reports, and running scheduled reconciliations
- An approval plugin interface implemented by an internal plugin in MVP, designed to support external plugins in Phase 2
- Outbound email via UBC's internal SMTP relay for member and admin notifications
- Logging, audit, and metrics pipelines feeding AWS CloudWatch (with optional forwarding to a managed log aggregator)
- AWS-hosted infrastructure with separate development and production environments (see the technology stack document for specifics)

---

## 9. Deliverables

The MVP engagement will produce:

- A working production deployment of the platform with one pilot council fully onboarded
- Complete source code in a repository transferred to or accessible by the client
- Architecture and data model documentation
- API documentation for the approval plugin interface (preparing for Phase 2)
- Admin user guide and bookkeeper user guide

---

## 10. Acceptance Criteria

The MVP is accepted when:

- All in-scope items in section 3 are implemented, tested, and demonstrated to the client
- The pilot council has processed live payments in production for an agreed period without critical issues
- Cross-tenant isolation tests pass in CI on every build
- All test suites described in section 3.16 (unit, integration, end-to-end) pass in CI for the version delivered for sign-off
- Security review (per section 3.16) is complete and any findings are resolved or accepted with a documented mitigation
- The audit log captures all admin and bookkeeper actions described in section 3.11
- Open production issues are categorized; no severity-1 issues remain open at sign-off

---

## 11. Out-of-Scope Restated and Change Management

Items not enumerated in section 3 are not part of the MVP. Requests to add to scope during build are welcomed and handled through a lightweight change request process: a written description of the change, an estimate of impact to scope, schedule, and cost, and explicit client approval before work begins.

---

## Appendix A: Glossary

- **Council**: top-level UBC organizational unit and tenant on the platform
- **Local**: sub-unit of a council; sub-tenant on the platform
- **UBC SSO**: United Brotherhood of Carpenters single sign-on, providing member identity and dues balance information
- **Stripe Connect**: Stripe's platform offering supporting multiple connected merchant accounts under one platform
- **Standard connected account**: a Stripe Connect account model where the council retains direct ownership of their Stripe relationship
- **Smart Retries**: Stripe-managed retry logic for failed payments
- **SAQ-A**: the lowest-burden PCI compliance level, applicable when card data does not touch platform servers
- **Approval plugin**: a pluggable component implementing a payment approval workflow; the MVP ships an internal plugin and Phase 2 adds external plugins
- **Pending approval**: payment state after Stripe success but before bookkeeper sign-off; not yet credited as a valid contribution
