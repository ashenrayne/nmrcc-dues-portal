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

Each council receives a distinct subdomain or path-based URL (final pattern to be confirmed during design). Light branding (logo, name, accent color, sender name on emails) is configurable per council.

### 3.2 Authentication and Member Identity

Members authenticate via the UBC single sign-on. The portal does not manage passwords for members. On first sign-in, a member record is provisioned in the relevant council and local based on UBC's response.

The portal queries UBC to retrieve the member's current dues balance, including any past-due amount. The exact contract for this query (endpoint, authentication, fields returned, refresh frequency) is dependent on UBC documentation and will be confirmed in the discovery phase. The portal treats UBC as the source of truth for what is owed; the portal is the source of truth for what has been paid through the platform.

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

### 3.4 Dues Amount Handling

Dues are a fixed amount per member per billing period and do not change on a regular basis. The amount applicable to a given member is determined from data provided by UBC SSO or council configuration at the time of payment, and the same amount is used for both one-time payments and each occurrence of a recurring payment.

Because dues are fixed per member, recurring payments are implemented using fixed-amount Stripe Subscriptions rather than periodic invoicing. When a member's dues amount is updated by the council (for example, on an annual rate change), the affected subscriptions are updated to the new amount on the next billing cycle.

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

Transactional emails sent by the platform include payment receipts (sent immediately on successful charge), payment failures, upcoming recurring charges (especially for ACH), expiring card warnings, past-due notices, and recurring canceled or completed. Each email is templated with per-council branding and customizable copy. Email delivery is handled by a transactional email provider (final selection during design, likely Postmark or similar).

SMS notifications are out of scope for the MVP.

### 3.11 Logging, Auditing, and Observability

Three distinct logging channels are established:

- **Application logs** for engineering: errors, performance, request traces. Routed to a managed logging service.
- **Audit log** for compliance and trust: who did what, when, with what reason. Visible to admins, immutable, exportable. Includes role assignments, refunds, credit adjustments, approvals and reversals, settings changes, and login events.
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

The platform is delivered as a responsive web application that works on modern desktop and mobile browsers. A native mobile app is out of scope for the MVP.

---

## 4. Out of Scope (MVP)

The following are explicitly excluded from the MVP. Items marked **Phase 2** are planned for the next phase; others are deferred or excluded entirely.

- **External approval plugins**: Phase 2. The internal approval plugin ships in MVP; external plugins for third-party systems of record ship in Phase 2.
- **Additional payment providers**: Stripe is the sole provider for MVP. The data model and service boundaries are designed to allow another provider to slot in later, but no second provider is implemented.
- **Cross-currency operations**: a member transacts in a single currency (their council's currency). The platform does not convert between USD and CAD or perform cross-border settlement. Multi-currency support means each council settles in its own currency, not that any individual member or transaction is multi-currency.
- **Manual / offline payment recording**: the platform tracks digital payments only. Checks, cash, and money orders received outside the platform are not entered into the system; councils continue to handle those through their existing bookkeeping processes.
- **Native mobile applications**: responsive web only.
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

- UBC provides a documented SSO mechanism (OIDC, SAML, or equivalent) with reasonable availability, and exposes a queryable API or feed for member dues balances.
- UBC's dues balance data is sufficient to determine what a member owes at the time of payment. The portal does not need to compute dues from raw inputs (hours, status, rate tables).
- Dues amounts are fixed per member per billing period and do not change on a regular basis. Annual or occasional rate changes are applied to subscriptions on the next billing cycle.
- The MVP supports both US and Canadian councils. US councils settle in USD via US Stripe accounts; Canadian councils settle in CAD via Canadian Stripe accounts. Each council operates in a single currency.
- Each council can complete Stripe Connect onboarding (including business verification) for their own account(s). The platform facilitates this but cannot complete it on the council's behalf.
- The platform charges no per-transaction application fee in the MVP unless agreed otherwise; transaction-fee economics flow directly between councils and Stripe.
- The pilot council has identified bookkeepers willing to operate the approval queue and provide feedback during pilot.
- Hosting will be on a major cloud provider (AWS, GCP, or Azure) selected during design. Likely provided by the UBC.

---

## 7. Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| UBC SSO outage prevents member sign-in and therefore payment | High | Status page and member communication during outages; in-flight recurring payments continue to charge via stored payment methods independent of SSO; monitor UBC SSO availability and alert on degradation |
| Stale dues balance from UBC causes duplicate payments or incorrect past-due display | Medium-high | Define explicit handshake on payment success; show last-synced timestamp to members; idempotency on payment intent creation |
| Webhook delivery failures cause payments to appear missing | Medium | Webhook event log with manual replay; reconciliation report catches discrepancies; alerting on webhook backlog |
| Bookkeeper backlog delays UBC balance update, causing the member to still see past due after paying | Medium | Suppress dunning during pending state; backlog escalation to council admin; rely on the source-of-truth handshake (section 7) so balance updates do not depend on bookkeeper turnaround when avoidable |
| Cross-tenant data leak | High | Row-level security or strict scoping at ORM layer; automated cross-tenant access tests in CI; security review before pilot |
| PCI scope creep | Medium | Card data handled exclusively via Stripe Elements / Checkout; no card data on platform servers; periodic review |
| A local's Stripe account assignment changes mid-flight (between council accounts, or to/from a local-owned account) | Medium | Stripe subscriptions cannot move between accounts; document a migration procedure that cancels and recreates active recurring payments with member consent and clear notification |
| External approval system (Phase 2) outage stalls payment finalization | Medium | Manual override always available; configurable timeout fallback (auto-approve, auto-reject, or manual queue); clear shared-responsibility documentation |

---

## 8. Architecture Overview (high level)

A more detailed architecture document will be produced during discovery. At a high level, the system comprises:

- A web application frontend for members and admins, served from a single codebase with role-based experiences
- A backend API enforcing tenant scoping and role-based permissions on every request
- A primary relational database (PostgreSQL) with strict tenant scoping, augmented by a payment event store
- Stripe integration via Stripe Connect (Standard accounts), Stripe Elements / Checkout for card capture, Stripe webhooks for event ingestion
- An asynchronous job system for sending notifications, processing webhooks, generating reports, and running scheduled reconciliations
- An approval plugin interface implemented by an internal plugin in MVP, designed to support external plugins in Phase 2
- A transactional email service for member and admin notifications
- Logging, audit, and metrics pipelines feeding a managed observability stack
- A managed cloud hosting environment with separate development, and production environments

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
