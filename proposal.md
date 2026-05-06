# Carpenter Council Payments Portal: Proposal for NMRCC

**Prepared for:** Northern Midwest Regional Council of Carpenters (NMRCC)
**Prepared by:** Ashen Rayne
**Date:** May 5, 2026

---

## About this proposal

This proposal lays out what we're building for NMRCC, what it costs, and what to expect after launch. It is a companion to the scope document, which describes the system in full detail.

The platform is designed to serve NMRCC and its locals from launch, and is structured so it can expand to other UBC councils if UBC elects to adopt it more broadly. Funding works in two stages:

- **Initial period (NMRCC-only):** NMRCC funds the build, the ongoing support, and the AWS hosting costs while the platform serves only NMRCC.
- **Broader UBC adoption:** If UBC determines that the platform benefits the wider council network and elects to host it internally, AWS hosting transitions to UBC's environment and those costs shift to UBC at that point.

Outbound email uses UBC's existing internal mail relay throughout, so there is no separate email-provider line item in either phase.

Numbers below are estimates based on the scope as currently written. If scope changes meaningfully, we'll revise together before any work shifts.

---

## What we're building, in one paragraph

A multi-tenant payments portal for NMRCC and, eventually, other UBC councils and their locals. Members sign in through UBC SSO, see what they owe, and pay by card or bank debit, one-time or on a recurring schedule. Council and local admins get scoped dashboards for managing members, processing payments, running reconciliation, and (optionally) approving each payment through an internal bookkeeper queue. US councils transact in USD; Canadian councils in CAD. The platform is built on AWS, designed so hosting can move into UBC's environment if and when UBC elects to take it on. Full detail lives in the scope document.

---

## Phase 1: MVP

**Estimated cost: $35,000 – $45,000**
**Estimated timeline: 6 – 8 weeks from kickoff**

This covers everything in section 3 of the scope: multi-tenant foundation, UBC SSO, Stripe Connect onboarding, card and bank-debit payments, recurring billing, the full admin and bookkeeper experience, the internal approval workflow with its pluggable interface, role-based permissions, audit logging, transactional email with per-council branding, the standard report set, reconciliation, and pilot rollout with NMRCC.

A representative sequencing of the work, for visibility into how the build progresses:

| Milestone | What lands | Approx. share of total |
|-----------|-----------|------------------------|
| 1. Discovery & foundation | UBC SSO contract resolved, Terraform infra, multi-tenant data model, auth, role/permission framework, audit log | 20% |
| 2. Payments core | Stripe Connect, one-time payments, member portal, webhook + event log, receipts | 20% |
| 3. Recurring + bank debit | Subscriptions, ACH (US) and PAD (Canada), payment method management, dunning | 15% |
| 4. Admin & approval workflow | Dashboards, payment lists, member views, refunds, approval queue, escalation, plugin interface | 25% |
| 5. Notifications, reports, polish | Email templates, branding, 5 standard reports, reconciliation, accessibility, hardening | 10% |
| 6. Pilot launch | NMRCC onboarding, training, runbook, support setup | 10% |

What pushes us toward the **upper end** of the range: UBC SSO turning out to expose less than expected and requiring a custom integration; NMRCC needing meaningful workflow customization beyond the configurable policy options.

What keeps us toward the **lower end**: UBC providing a clean OIDC contract with dues balance API; NMRCC adopting the default approval policy and a standard branding template.

---

## Phase 2: External Approval Framework

**Estimated cost: $10,000 – $15,000 for the framework + one concrete integration**
**Each additional external system integration: $3,000 – $5,000**
**Estimated timeline: 3 – 4 weeks for framework + first integration**

This covers the external approval plugin framework, the plugin admin UI for configuring integrations (auth, field mapping, retry policy, timeout fallback), plugin-specific monitoring and dead-letter tooling, and one concrete integration with a third-party system of record (target system to be identified during MVP pilot).

Subsequent integrations build on the framework and price per the complexity of the target system. We'll quote each one individually once the target is known.

Phase 2 also includes a small budget for items deferred from MVP based on NMRCC pilot feedback (custom domains per council, additional reports, etc.). If those grow beyond a small budget, they're handled as separate change orders.

---

## Ongoing Support

After launch, NMRCC's ongoing cost has two parts: our support fee and AWS hosting costs. AWS hosting stays on NMRCC's bill while the platform serves only NMRCC; if UBC later elects to host the platform internally, those AWS costs are assumed by UBC at that point.

### Our support fee (billed to NMRCC initially)

**Base platform support: $900 / month**

Includes:
- NMRCC as a council
- Production monitoring with on-call response during business hours (Mon – Fri, 8am – 6pm Eastern)
- Up to 6 hours / month of small enhancements, bug fixes, and support questions (not cumulative)
- Dependency and security patching on a quarterly cadence
- Quarterly check-in on platform health, council adoption, and roadmap

**Per active council: $300 / month** (covers 1 – 2 Stripe accounts)

Each council on the platform after the first adds modest operational overhead: additional monitoring scope, support volume, council-specific Stripe questions, branding adjustments, role tweaks, and so on. The first council is included in the base fee. The $300 covers up to 2 Stripe accounts per council, which fits the typical case: one shared council Stripe account, optionally plus a single local-owned account.

**Additional Stripe accounts: +$25 / month per additional 2 accounts**

If a council operates more Stripe accounts (for example, several locals running their own), the per-account operational load grows: separate KYC reverification, webhooks, payouts, and reconciliation per account. The per-council fee scales in $25 steps:

| Stripe accounts on the council | Per-council monthly fee |
|--------------------------------|-------------------------|
| 1 – 2 | $300 |
| 3 – 4 | $325 |
| 5 – 6 | $350 |
| 7 – 8 | $375 |
| 9 – 10 | $400 |

So a platform with 5 active councils, each on the default 1 – 2 Stripe accounts, runs **$900 + (4 × $300) = $2,000 / month** in support. A council with, say, 6 Stripe accounts adds $50 to its line ($350 instead of $300).

**Per-council onboarding (one-time): $1,200**

Each new council added after the pilot involves Stripe Connect onboarding assistance, branding setup, email template configuration, role and permission setup, admin training (one session for council admins, one for bookkeepers), and a soak window with us watching for issues. Charged once per council at onboarding.

**Out-of-scope work: $150 / hour**

Anything beyond the included monthly hours, or work that doesn't fit a small enhancement (new report, new integration, custom workflow), is quoted hourly or as a fixed-price change order. Your choice.

### AWS hosting

While the platform serves only NMRCC, AWS hosting is billed to NMRCC. We pass these costs through directly with no markup. If UBC later elects to host the platform internally for broader council adoption, hosting transitions to UBC's AWS environment and those costs shift to UBC's bill at that time.

Approximate monthly hosting cost at NMRCC pilot scale:

| Component | Approximate monthly cost |
|-----------|--------------------------|
| ECS Fargate (application) | $30 – $60 |
| RDS Postgres (Multi-AZ, db.t4g.small) | $55 – $80 |
| Application Load Balancer | $25 – $30 |
| NAT Gateway | $35 – $45 |
| S3, Secrets Manager, Route 53, CloudWatch | $15 – $40 |
| Inngest (background jobs; free tier covers pilot volume) | $0 – $50 |
| **Estimated total at NMRCC pilot scale** | **$160 – $305 / month** |

Email delivery is handled through UBC's existing internal mail relay throughout, so there's no separate email-provider line item.

If the platform is later adopted by additional councils and hosting moves to UBC, that broader scale (stepping up to a db.t4g.medium Multi-AZ database, autoscaled application tasks, and a paid Inngest tier) would likely run **$500 – $1,000 / month** in AWS costs, billed to UBC at that point. We size everything for cost efficiency and can review as needed.

**One-time migration fee (only if hosting moves to UBC): $2,400 – $3,600**

Moving the platform from our AWS account into UBC's AWS account is a planned, runbook-driven exercise, not a re-architecture, because the platform is built from day one to support exactly this transition. The work covers standing up infrastructure in UBC's account, copying the database, syncing files, re-pointing DNS, and validating end to end. It runs 16 – 24 hours of work over 1 – 2 weeks of preparation, with a single 2 – 4 hour planned downtime window at cutover.

**Impact on members and active payments during the migration:** minimal. The councils' Stripe accounts are not part of the AWS environment, so active recurring payments continue charging on Stripe's schedule throughout the migration. Members do not need to re-enter payment methods, re-authorize, or take any action. The only member-visible impact is a short maintenance window during which new one-time payments cannot be made; any recurring renewals scheduled during that window still run, and members receive their receipts as normal.

This fee is invoiced to whichever party is taking on hosting going forward, and is only incurred if and when UBC elects to host the platform internally.

---

## What's not included

- **AWS hosting costs**: billed to NMRCC during the initial NMRCC-only period; transition to UBC if and when UBC elects to host the platform internally. See the AWS hosting section above for current ranges.
- **AWS account migration fee**: only incurred if and when hosting moves into UBC's AWS environment. Estimated at $2,400 – $3,600; see the AWS hosting section above.
- **Stripe processing fees**: paid by each council directly to Stripe. The platform takes no application fee.
- **UBC integration costs**: if UBC needs work on their side to expose SSO or dues balance APIs, that's outside this scope and is handled directly between UBC and NMRCC.
- **Legal and compliance review**: each council is responsible for their own merchant agreements, terms of service, and any state or provincial registrations.
- **Historical data migration**: if NMRCC has prior digital payment data to import, that's a separate engagement we'll quote once the source is known.

---

## Payment terms

All amounts in USD.

- **Phase 1**: 50% on signature, 50% on completion.
- **Phase 2**: 50% on signature, 50% on completion.
- **Ongoing support**: invoiced monthly in advance.
- **Onboarding**: invoiced when a new council is added.
- **Hourly / change-order work**: invoiced monthly for hours worked.

---
---

## Next steps

If this proposal looks right, the next step is a discovery kickoff. We'd walk through the open questions in the scope together, lock down anything that affects pricing materially, and sign a statement of work.

Happy to walk through any line item, adjust phasing, or restructure to fit how NMRCC prefers to budget this work.

---

*Estimates are valid for 60 days from the date above. Material changes to UBC's available APIs, NMRCC's requirements, or the regulatory environment may require revision.*
