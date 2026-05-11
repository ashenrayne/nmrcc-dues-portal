# Carpenter Council Payments Portal: Technology Stack

**Document type:** Internal technology stack and infrastructure plan
**Companion to:** [scope-document.md](scope-document.md)
**Prepared by:** Ashen Rayne
**Status:** Draft
**Version:** 0.1

---

## 1. Overview

This document records the technology choices for building the Carpenter Council Payments Portal described in the scope document. It complements section 8 of the scope (Architecture Overview) by naming concrete frameworks, services, and infrastructure components, and by recording the rationale and the constraints those choices honor.

Two constraints shape the entire stack:

- **AWS-native from day one.** The platform launches on an AWS account we manage, with NMRCC funding the AWS hosting costs while the platform serves only NMRCC. If UBC later elects to host the platform internally for broader council adoption, hosting transitions into UBC's AWS organization at that time.
- **Designed to migrate between AWS accounts without code changes.** Everything is Infrastructure-as-Code, no account IDs are hardcoded, secrets live in Secrets Manager, and storage / DB names are parameterized, so the eventual move into UBC's account (if it happens) is a configuration change, not a re-architecture.

---

## 2. Application Stack

### 2.1 Frontend and Backend (one codebase)

- **Next.js 15 (App Router) + TypeScript**: single codebase for both the member portal and the admin/bookkeeper experience. Route groups separate the two surfaces.
- **shadcn/ui + Tailwind CSS**: composable component primitives for the design system. Per-council branding is applied via CSS variables.
- **TanStack Table**: powers the payments list, approval queue, member list, and any other dense tabular admin view. Required for the bulk actions, saved filters, and keyboard navigation called for in scope sections 3.6 and 3.8.
- **React Hook Form + Zod**: forms and end-to-end schema validation; the same Zod schemas are reused in server actions to enforce invariants at the trust boundary.

### 2.2 Server Logic

- **Next.js route handlers** for Stripe webhooks and any third-party-callable endpoints.
- **Next.js server actions** for admin and member mutations (refunds, approvals, payment method updates, settings).
- **Node runtime only**: no Edge runtime. Keeps deployment portable to ECS/Fargate without a runtime rewrite.

---

## 3. Data Layer

- **PostgreSQL** as the system of record. Hosted on **Amazon RDS** (Postgres 16+).
- **Drizzle ORM** for typed queries and schema migrations. Drizzle gives explicit control over query composition, which matters for the tenant-scoping patterns this app needs.
- **Postgres Row-Level Security (RLS)** on every tenant-scoped table as a defense-in-depth layer beneath the application's tenant scoping. Cross-tenant access is blocked at the database even if application code has a bug.
- **Append-only audit log table** with a database trigger that blocks `UPDATE` and `DELETE`. The immutability property in scope section 3.11 is enforced by Postgres, not by convention.
- **Payment event log table** stores every Stripe webhook payload verbatim (JSONB) keyed by Stripe event ID for idempotent processing and replay.
- **RDS Proxy** in front of RDS for connection pooling once we move to ECS/Fargate or Lambda.

---

## 4. Authentication

- **Auth.js (NextAuth)** for both member and admin authentication.
  - Members authenticate through UBC SSO via OIDC or SAML, depending on what UBC exposes (open question 1 in scope section 7).
  - Admins authenticate through a separate provider configuration, likely email + password with TOTP.
- **Sessions stored in Postgres** via the Drizzle adapter. No reliance on a hosted auth service.
- **JWT claims set the Postgres RLS context** on each request via a `set_config('request.jwt.claims', ..., true)` call inside a transaction wrapper. The wrapper is the only correct path to query the database; reviews enforce this.

We deliberately do not use Supabase Auth, Cognito, or any other hosted identity service. Auth lives in our own Postgres so it travels with the data and is not coupled to a vendor.

---

## 5. Payments Integration

- **Stripe Connect (Standard accounts)**: each council registers one or more Stripe Connect Standard accounts. Each local on the platform is configured to receive its payments through one of the council's accounts (the default) or, if the local operates with financial independence, through its own Stripe account. The data model treats the Stripe-account assignment as a property of the local, with the council holding the catalog of available accounts. The platform is the Connect platform but does not take custody of funds.
- **Stripe Node SDK** for all server-side calls.
- **Stripe Elements / Checkout** for card capture in the browser. Card data never touches our servers; PCI scope stays at SAQ-A.
- **Stripe Subscriptions** for recurring payments. Fixed-amount because dues are fixed per member per period (scope section 3.4).
- **Stripe ACH** (US, USD) and **Stripe Pre-Authorized Debit** (Canada, CAD) for bank debit.
- **Webhook handler** verifies the Stripe signature, persists the raw event to the payment event log, enqueues an Inngest function, and returns 200 quickly. All business logic happens asynchronously and idempotently against the persisted event.

---

## 6. Background Jobs and Async Work

- **Inngest** for background work. Cloud-agnostic SaaS that handles retries, durable execution, scheduled jobs, and a replay UI for ops.
- Used for:
  - Stripe webhook processing (outside the synchronous handler)
  - Recurring charge orchestration where Stripe Subscriptions does not suffice
  - Dunning notifications
  - Approval-workflow side effects (writing back to UBC, sending receipts, escalations)
  - Scheduled reconciliation reports
  - Email send-outs

We choose Inngest over Vercel Cron + Lambda or self-hosted BullMQ because: (a) it follows us across hosting providers, (b) the durable-execution model maps cleanly onto the multi-step webhook → approval → write-back flow, and (c) the manual replay tool requirement in scope section 3.11 is built in.

If UBC has a policy concern about a third-party SaaS handling job orchestration, the fallback is **EventBridge + Lambda** for scheduled jobs and a self-hosted **BullMQ + Redis (ElastiCache)** worker on ECS for queued work. We confirm the policy posture before committing.

---

## 7. Infrastructure (AWS)

### 7.1 Compute

- **Next.js app on Amazon ECS / Fargate** behind an Application Load Balancer.
  - Single Docker image, `next start`, two or more tasks for availability.
  - Autoscaling on CPU and request count.
  - Long-running container preferred over Lambda to avoid cold starts on the bookkeeper queue UI and to simplify connection pooling.
- **OpenNext + Lambda** is a viable alternative we revisit if UBC's procurement strongly prefers serverless. Decision not blocking for MVP.

### 7.2 Data and Storage

- **RDS Postgres** (Multi-AZ in production) for the application database.
- **S3** for receipt PDFs, exports, branding assets, and any other object storage. Bucket names are parameterized by environment and account alias so they can be re-created in UBC's account.
- **ElastiCache Redis** if we need a session/cache layer or self-hosted job queue (deferred until needed).

### 7.3 Networking

- **VPC** with public ALB subnets and private application/data subnets.
- **NAT Gateway** for outbound calls (Stripe API, Inngest, UBC SSO). Single NAT in development; HA pair in production.
- **VPC endpoints** for S3 and Secrets Manager to keep traffic off the public internet.
- **WAF** in front of the ALB for basic rate limiting and common OWASP rules.

### 7.4 Secrets and Config

- **AWS Secrets Manager** for Stripe API keys, database credentials, UBC SSO client secrets, and any other secret material. Never in env files committed to git, never in code.
- **SSM Parameter Store** for non-secret config (feature flags, environment names, etc.).
- All secrets accessed via IAM role at runtime; no long-lived credentials in containers.

### 7.5 Email and Notifications

- **Outbound SMTP via UBC's internal mail relay**, using Nodemailer in the application. UBC's existing relay handles delivery, DKIM, SPF, DMARC, and IP reputation, the same infrastructure that already serves UBC's Drupal sites.
- Templated per council with custom branding via the application's email templates.
- No third-party email provider (Postmark, SES) required for the MVP. SMTP relay address, sending domain, and authentication method to be confirmed with UBC during discovery.

### 7.6 DNS

- **Route 53** for DNS, with low TTLs on records that will move at cutover.
- ACM certificates for HTTPS, auto-renewed.

---

## 8. Observability

The MVP commits to the minimum observability needed to run the service reliably and investigate incidents. The bias is toward surfacing operationally-relevant information inside the application itself, so that council admins, bookkeepers, and Ashen Rayne support can answer most questions without infrastructure access.

**In-application surfaces** (visible to admins per their role; see scope section 3.11 and related):

- **Audit log**: who did what, when, immutable, exportable
- **Payment event log**: every Stripe webhook, every charge attempt, every state transition on a payment, with full payload preserved and queryable from the admin dashboard
- **UBC sync log**: each background sync run (started, completed, failed), per-member changes detected, and any errors (per scope section 3.4)
- **Email delivery log**: per-member record of platform-sent emails (event, timestamp, recipient, result returned by the SMTP relay)
- **Lifecycle event records**: merger and reassignment history with linked source and destination entities (per scope section 3.15)

These cover most day-to-day investigations without requiring infrastructure access.

**Infrastructure observability** (for Ashen Rayne's operations):

- **CloudWatch Logs** for application logs, with structured JSON logging
- **CloudWatch built-in metrics** for ALB, ECS, and RDS (request rate, error rate, latency, CPU, memory, connections). No custom application metrics in MVP.
- **A small set of CloudWatch alarms** for issues that warrant immediate attention: elevated 5xx error rate, ECS task failure, RDS CPU or storage pressure, and webhook event log not advancing. Alarms route to email and/or Slack.

Custom business metrics, log aggregator overlays, dedicated incident management tooling, and distributed tracing are out of scope for the MVP. They are candidates for later phases as platform volume grows.

---

## 9. Development Practices

### 9.1 Infrastructure-as-Code

- **Terraform** for all infrastructure. Nothing clicked in the console persists.
- One Terraform configuration, multiple workspaces or backends per environment. The same configuration deploys into UBC's account at cutover by changing the AWS provider configuration.
- Account ID is read from `data.aws_caller_identity.current.account_id`, never hardcoded.

### 9.2 Environments

- **Development**: shared, hits Stripe test mode.
- **Production**: live, hits Stripe live mode, Multi-AZ RDS, autoscaling enabled.
- A staging environment may be added if pilot feedback indicates value; not in MVP.
- Per-PR preview environments are out of scope for MVP given the data-isolation considerations of a financial system.

### 9.3 CI/CD

- **GitLab CI/CD** for build, test, and deploy. UBC standardizes on GitLab; the production repository lives there.
- Build pushes a Docker image to **ECR** (or GitLab Container Registry if UBC prefers); deploy updates the ECS service with rolling deployment.
- Database migrations run as a one-off ECS task before the new service version is promoted.
- **Cross-tenant isolation tests** run in CI on every commit (scope acceptance criterion).
- AWS credentials in CI are obtained via **OIDC federation** from GitLab to AWS (no long-lived access keys stored in CI variables).

### 9.4 Code Quality

- **TypeScript strict mode** across the codebase.
- **ESLint + Prettier** with shared configs.
- **Vitest** for unit tests; **Playwright** for end-to-end tests of critical paths (member payment, recurring enrollment, refund, approval).
- **Stripe CLI** for local webhook testing.

---

## 10. Migration Plan (Our Account → UBC's Account)

The application is built so that the AWS account migration is a runbook, not a project. The mechanics:

1. UBC provisions an AWS account (or sub-account in their Organization) and grants us bootstrap access.
2. We point the Terraform configuration at the new account and apply. The full stack is created: VPC, RDS, ECS, ALB, S3 buckets, Secrets Manager entries, Route 53 records.
3. Recreate Secrets Manager entries with production values from UBC.
4. Migrate the database via RDS snapshot sharing across accounts, or `pg_dump` / `pg_restore` over a temporary peering connection.
5. Sync S3 objects with `aws s3 sync` (or set up cross-account replication for a longer cutover window).
6. Reconfigure Stripe webhook URL to point at the new ALB.
7. Update DNS to point at the new ALB.
8. Monitor; cut traffic over.
9. Decommission resources in our account once UBC has soaked production for an agreed window.

Estimated cutover window with the above setup: a single planned-downtime window measured in hours, most of which is database migration time. Read-only mode during the cutover is the only member-visible impact.

---

## 11. Choices We Are Deliberately Not Making (and Why)

- **No Vercel.** Adds a hosting platform we'd rip out at UBC cutover.
- **No Supabase.** Same reason; if we want managed Postgres we use RDS.
- **No Supabase Auth, Cognito, or Auth0.** Auth lives in our own Postgres via Auth.js so it travels with the data and is not coupled to a vendor.
- **No `supabase-js` or other auto-generated database APIs.** Database access goes through Drizzle in server-side code only.
- **No Vercel KV / Vercel Blob / Vercel Postgres / Edge Config.** Each is a thing we'd rewrite at migration.
- **No Edge runtime.** Node runtime only, so the deploy target is "any container host."
- **No native mobile SDKs, no GraphQL gateway, no event-sourcing framework.** Out of scope and unjustified by current requirements.
- **No microservices.** A single Next.js application is appropriate for the MVP scope; service splits would be premature and would multiply the operational burden.

---

## 12. Open Decisions

These remain open and will be resolved during discovery or before the relevant work begins:

- **Job orchestration**: Inngest (preferred) vs. EventBridge + Lambda + self-hosted BullMQ. Decision driven by UBC's policy on third-party SaaS in the data path.
- **SMTP relay details**: address, port, authentication mechanism, allowed sending domain, and any rate limits on UBC's internal mail relay. Confirmed during discovery before email-dependent flows are built.
- **Observability stack**: CloudWatch-only vs. Datadog (or similar) overlay. Driven by UBC's existing observability tooling.
- **AWS Region**: needs confirmation per council geography. Likely `us-east-1` or `us-east-2` for US councils and `ca-central-1` for Canadian councils, with a single multi-region deployment if data residency rules require it. Open question for UBC.
- **Compliance attestations** (SOC2, etc.): UBC may require us to operate within their existing attestation boundary, which can constrain tooling choices.
