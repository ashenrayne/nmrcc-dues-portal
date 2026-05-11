# Platform Integration Brief: UBC IT

**Prepared by:** Ashen Rayne
**For:** UBC IT
**Subject:** New dues payment platform, hosting and integration on UBC infrastructure
**Status:** Draft for UBC IT review

---

## 1. Purpose of this document

Ashen Rayne is building a new online dues payment platform for the Northern Midwest Regional Council of Carpenters (NMRCC), designed from day one to expand to other UBC councils if UBC elects to adopt it more broadly. This platform is a separate application from the existing PHP applications Ashen Rayne maintains on UBC's DMA server.

The platform will be hosted on UBC-provided infrastructure. This document describes what UBC IT needs to provision and configure for the platform to run, the integration surface with UBC's existing systems (SSO, partner APIs, SMTP relay), and the responsibility split between Ashen Rayne and UBC IT.

We are looking for UBC IT's guidance on which of the infrastructure options in section 4 are available, and on the authentication and access patterns we should use for the integrations in section 3.

---

## 2. What the platform does

Members sign in via UBC SSO, see their current dues balance, and pay by card or bank debit (ACH for US councils, PAD for Canadian councils), either one-time or on a recurring schedule. Council and local administrators get scoped dashboards for managing payments, reconciliation, refunds, and an optional internal approval workflow. The platform uses Stripe Connect Standard accounts for payment processing; each council holds its own Stripe relationship, and Stripe (not the platform) takes custody of funds.

---

## 3. Integration with UBC systems

The platform consumes UBC APIs to authenticate members and read dues data. It does not write to UBC's systems in the MVP except where explicitly called out below.

### 3.1 SSO (member authentication)

Members authenticate via UBC's existing SSO mechanism. The platform supports OIDC and SAML; the specific protocol and identity claims are open for confirmation during integration design.

Claims the platform expects to receive (either at SSO time or via subsequent API calls):

- UBC member identifier (`ubcid`)
- Affiliated council and local
- Current member status (good standing, arrears, suspended, withdrawn, etc.), used to gate eligibility for online payment
- Member classification (trade taxonomy, e.g., apprentice, journeyman, retired)

If UBC's SSO does not expose all of these as claims, the platform retrieves the missing fields via the APIs in section 3.2.

### 3.2 UBC partner API consumption

The platform calls UBC's partner APIs (per the documentation UBC provided) for:

- Member identity verification on first sign-in (`POST /pfy/member` and related)
- Current dues balance (`POST /financialsnapshot`)
- Current dues rate and classification (per available endpoints)
- Member status (`GET /member/status/{ubcid}` and the bulk variant)
- Member record updates if applicable (`PATCH /pfy/member`), read-mostly in MVP

Expected request patterns:

- **At member sign-in**: balance and status fetch (1 to 3 API calls per sign-in)
- **Scheduled background sync**: daily or weekly batch refresh for all active recurring subscribers, used to detect dues rate changes and status changes between sign-ins. Volume scales with active subscriber count; for NMRCC pilot we expect this to be in the low thousands of records per run, executed via the bulk endpoint where available.
- **On-demand**: ad-hoc reads triggered by admin or support actions

Estimated volume at NMRCC pilot scale: a few thousand API calls per day, dominated by the background sync.

Since the platform runs inside UBC's infrastructure, calls to UBC's partner APIs are internal-network traffic. We'd like UBC IT to confirm the authentication model used by other applications inside UBC's network (service-account credential, mTLS, signed token, etc.) so the new platform follows the same pattern.

### 3.3 SMTP relay (outbound email)

Outbound transactional email (payment receipts, payment failure notices, expiring card reminders, etc.) goes through **UBC's existing internal SMTP relay** using Nodemailer. The platform does not use a third-party email provider (Postmark, SES, SendGrid, etc.).

Configuration we need from UBC IT: relay address, port, authentication mechanism, allowed sending domain, and any rate limits.

---

## 4. Infrastructure we need UBC to provide

This section is the load-bearing one for UBC IT. The platform is a Node.js application backed by PostgreSQL. It requires the following from UBC's infrastructure.

### 4.1 Compute

A runtime environment capable of running a Node.js 20+ application as a long-running process. Acceptable forms (in order of preference):

- **Container platform** (ECS, EKS/Kubernetes, Docker host, etc.). Preferred because it simplifies deployment, rollback, and horizontal scaling.
- **Virtual machines** (Linux, Ubuntu 22.04+ or RHEL 8+) with Node.js installed. Workable; deployment is a bit more involved.
- **Serverless** (Lambda, Cloud Run, etc.). Not preferred for this application because long-running webhook handlers, scheduled jobs, and connection pooling are awkward in serverless. Possible if it is UBC's only option, but it changes some implementation choices.

Sizing for the NMRCC pilot: two or more instances of the application running concurrently for availability, with rolling deployments (no downtime during deploys). Each instance is modest (1-2 vCPU, 2-4 GB RAM is sufficient).

### 4.2 Database

PostgreSQL 16 or higher. Acceptable forms:

- **Managed PostgreSQL service** (RDS, Cloud SQL, equivalent). Preferred.
- **Self-hosted PostgreSQL** on a VM is workable if backups, replication, and high-availability are operated by UBC IT.

Requirements:

- Multi-AZ or equivalent high-availability in production
- Automated daily backups with point-in-time recovery
- Encryption at rest
- Network-level access control so only the application instances can connect
- A way to run database migrations as part of deployment (one-off migration task before the new application version is promoted)

### 4.3 Object storage

Storage for binary application data:

- Receipt PDFs (one per payment)
- Exports (CSV, Excel)
- Per-council branding assets (logos, accent images, etc.)

Approximately 10 GB initial capacity, growing slowly with payment volume.

Acceptable forms (in order of preference):

- **S3 or S3-compatible managed service** (AWS S3, Azure Blob Storage, Google Cloud Storage, etc.). Preferred. If UBC's compute platform (4.1) is in a major cloud, the corresponding object storage service is usually available alongside.
- **Self-hosted MinIO** (S3-compatible) deployed on a UBC-provided VM. Workable if UBC's compute is VM-based and no managed object storage is available; we would operate the MinIO instance.
- **Network-attached storage** (NFS or SMB) mounted to the application VMs. Workable but less convenient: signed-URL access is not native, so file delivery would proxy through the application. Last resort.

Requirements regardless of form:

- Encryption at rest
- Application-mediated access only (no public bucket or direct file URLs for member data)
- Backup and retention policy consistent with the database (see section 4.2)

### 4.4 Secrets management

A secure secrets store for:

- UBC partner API credentials
- Stripe API keys (platform-level: one secret key and one webhook signing secret for the entire platform; council Stripe accounts are referenced by non-secret `account_id` values stored in the application database, not in the secrets store)
- Database connection credentials
- SSO client secrets

Acceptable forms (in order of preference):

- **Cloud-managed secrets service** (AWS Secrets Manager, Azure Key Vault, GCP Secret Manager, etc.). Preferred. Provides rotation, fine-grained access control, and audit logging out of the box, and integrates with cloud IAM for service-identity-based access.
- **Self-hosted HashiCorp Vault** on UBC infrastructure. Workable for VM-only environments. UBC IT operates Vault; the application reads secrets via a Vault service identity (AppRole, JWT auth, etc.).
- **Encrypted secrets file** managed by UBC's configuration management (Ansible Vault, sops with KMS, etc.) and delivered to the application's runtime via the deployment pipeline. Last resort; rotation and access audit are more manual.

Requirements regardless of form:

- Application reads secrets via service identity (IAM role, service account, AppRole, etc.); no long-lived credentials in environment variables, config files, or source control
- Secret rotation supported (manual or automatic)
- Access to secrets is auditable

### 4.5 Background job runtime

The platform needs to run scheduled and event-driven background work:

- Stripe webhook processing (after the synchronous handler returns 200 to Stripe)
- Daily or weekly UBC sync (refresh per-member dues data for active subscribers)
- Recurring renewal orchestration where Stripe's built-in retry behavior does not suffice
- Email send-outs
- Reconciliation report generation

**Preferred implementation: Inngest.** Inngest is a managed durable-execution service (SaaS) that handles retries, scheduled jobs, multi-step workflow orchestration, and exposes a replay UI for ops investigations. It is not infrastructure UBC needs to provision; it is a service the platform consumes, similar to Stripe. The dependency requires:

- Outbound HTTPS from the platform to `api.inngest.com`
- Inbound HTTPS from Inngest's published IP ranges to the platform's public endpoint (Inngest invokes the platform's functions over HTTPS to execute background work)
- No PII or financial data is sent to Inngest beyond the event identifiers needed to drive workflow execution (member identifiers, payment IDs, internal event names); detailed payload content stays in the platform's database

**Fallback if Inngest is not acceptable under UBC policy:** background workers running inside the application processes, combined with a scheduler (Kubernetes CronJob, EventBridge + Lambda, or equivalent) and self-hosted Redis on UBC infrastructure for queue durability. This path is materially more code to build and maintain, since we would be implementing the durable-retry, replay, and observability features Inngest provides off the shelf. We recommend Inngest if UBC's third-party SaaS policy allows it.

### 4.6 Networking and DNS

- **Public HTTPS endpoint** for the application. Required for member access (members reach the portal from home, phone, or work) and for Stripe webhook delivery (Stripe pushes events from public IP ranges to the platform).
- **Load balancer / reverse proxy** in front of the application instances (ALB, NGINX, etc.).
- **TLS certificates** with automated provisioning and renewal (ACM, Let's Encrypt, or UBC's standard).
- **DNS**: ability to register a central UBC dues subdomain (e.g., `dues.carpenters.org`) and per-council subdomains or CNAMEs (e.g., `nmrcc.dues.carpenters.org` or `dues.nmrcc.org`). UBC IT manages the central subdomain; each council manages its own CNAME if used.
- **WAF or equivalent** in front of the public endpoint, for basic rate limiting and common OWASP rules.
- **Outbound HTTPS** from the application to public endpoints (Stripe API).

### 4.7 Observability

- **Application logs** collected to a queryable store (CloudWatch Logs, Elasticsearch, Splunk, whatever UBC uses), with at least 30-day retention.
- **Basic infrastructure metrics**: request rate, error rate, latency, CPU, memory, database connections.
- **Alarms** for issues that warrant immediate attention: elevated 5xx error rate, database health, webhook processing backlog. Alarms route to email or UBC's preferred chat (Slack, Teams).

### 4.8 CI/CD

The platform is built and deployed via **GitLab CI/CD** (UBC's standard). The application's source repository lives in UBC's GitLab instance; Ashen Rayne maintains the pipeline configuration in the repository and deploys via standard merge and tag workflows. We need from UBC:

- A project in UBC's GitLab where Ashen Rayne can push commits, open merge requests, and run pipelines
- **GitLab Container Registry** access in that project for application images (or VM deployment hooks, depending on 4.1)
- **GitLab Runner** availability (UBC-hosted runners preferred; SaaS runners workable if allowed by policy)
- **OIDC federation between GitLab and UBC's cloud** for deployment authentication, so no long-lived deployment credentials are stored in GitLab CI variables
- A path to run database migrations as part of deployment (one-off task before the new application version is promoted)
- Standard GitLab deployment audit trail (built-in)

### 4.9 Environments

Two environments at MVP:

- **Development**: hits Stripe test mode and UBC's DEV API. Shared environment used by Ashen Rayne for ongoing development and pilot validation.
- **Production**: hits Stripe live mode and UBC's PROD API. Multi-AZ, autoscaling-enabled, deployed only via the CI pipeline.

Both environments have the same infrastructure shape so that promotion is reliable.

---

## 5. Security posture

Within UBC's infrastructure, Ashen Rayne commits to the following security properties:

- **Card data**: never touches the application. Captured directly by Stripe Elements or Stripe Checkout in the browser. PCI scope is SAQ-A.
- **Tenant isolation**: enforced both in application code and at the database layer using PostgreSQL row-level security. Automated cross-tenant access tests in CI on every commit.
- **Audit logging**: immutable audit log in the database for all admin actions (refunds, approvals, role assignments, settings changes, lifecycle events). Payment event log captures every Stripe webhook and payment state transition.
- **Secrets**: never in code or in source control. Read from UBC's secrets store at runtime via service identity.
- **No SSNs or government IDs**: stored or transmitted by the platform. The platform persists only the minimum member fields needed for its function (identifier, council and local affiliation, classification, status, balance snapshot).
- **Encryption**: TLS 1.2 minimum (TLS 1.3 preferred) in transit; encryption at rest for database and object storage.

---

## 6. Data residency, backup, and retention

These are UBC's call given hosting is on UBC infrastructure. Our defaults, configurable to UBC's policy:

- Region: wherever UBC hosts the infrastructure
- Database backup retention: 30 days with point-in-time recovery
- Payment records and audit log: retained indefinitely while the platform serves the council
- Deletion: per council policy on account closure

---

## 7. What Ashen Rayne handles vs. what UBC IT handles

| Area | Ashen Rayne | UBC IT |
|------|-------------|--------|
| Application code | Yes |  |
| Database schema and migrations | Yes |  |
| Application dependency updates and security patching | Yes |  |
| Stripe Connect onboarding (with each council) | Yes |  |
| Council onboarding (branding, admin training) | Yes |  |
| Application-level monitoring (defines what to alert on) | Yes |  |
| Application incident response | Yes |  |
| Infrastructure provisioning (compute, DB, storage, networking) |  | Yes |
| OS patching and base image security updates |  | Yes |
| Database backup operation and retention |  | Yes |
| TLS certificate management |  | Yes |
| DNS for the central UBC dues subdomain |  | Yes |
| Secrets store operation (Ashen Rayne manages content, UBC IT operates the store) | Content | Operation |
| Infrastructure incident response |  | Yes |
| CI/CD pipeline operation | Defines pipeline | Provides platform |
| Deployment authentication credentials | Configures CI | Provides federation |

---

## 8. Open items we would like to confirm with UBC IT

1. **Compute platform**: which of the options in section 4.1 is available? (Container platform preferred; VMs workable.)
2. **Database**: managed PostgreSQL service available, or self-hosted PostgreSQL on a VM?
3. **Object storage**: is an S3-compatible managed service available alongside the chosen compute platform? If compute is VM-based and no managed object storage is available, is self-hosted MinIO acceptable, or do you prefer NAS?
4. **Secrets management**: which secrets store does UBC standardize on (cloud-managed service preferred; HashiCorp Vault or encrypted config-managed files acceptable for VM-only environments)?
5. **Background jobs / third-party SaaS approval**: the platform's preferred background job runner is Inngest, a managed durable-execution SaaS (see section 4.5). Please confirm whether Inngest is acceptable under UBC's third-party SaaS policy, or whether we should plan for the self-hosted fallback.
6. **GitLab access and runners**: which GitLab group should host the application repository, what access do Ashen Rayne team members need, and is UBC providing self-hosted GitLab Runners (preferred) or are SaaS runners acceptable? What is UBC's pattern for OIDC federation between GitLab and the cloud environment for deployment auth?
7. **SSO protocol** (OIDC vs. SAML) and the specific claims included in the identity assertion.
8. **Authentication model** for the platform-to-UBC partner API channel within UBC's network (section 3.2).
9. **API rate limits** on UBC's partner endpoints and any throttling expectations for the background sync.
10. **Bulk endpoints** vs. per-member calls for the background sync (the documentation includes both individual and bulk member status endpoints).
11. **SMTP relay configuration** for outbound email.
12. **DNS** for the central UBC dues subdomain (who manages it; what's the request path for adding subdomains).
13. **Compliance attestations** required from Ashen Rayne (SOC 2, etc.).

---

*Document prepared by Ashen Rayne for review by UBC IT. References to specific endpoint paths follow the API documentation UBC provided.*
