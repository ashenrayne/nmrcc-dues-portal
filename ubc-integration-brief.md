# Platform Integration Brief: UBC IT

**Prepared by:** Ashen Rayne
**For:** UBC IT
**Status:** Draft for UBC IT review

---

## 1. Purpose

Ashen Rayne is building a new online dues payment platform for NMRCC, designed to extend to other UBC councils if UBC elects to adopt it more broadly. The preferred path is a Node.js application backed by PostgreSQL, with a Laravel (PHP) fallback if coexisting with the existing Drupal sites on the DMA server is a UBC priority. Either way, the new platform is a separate application from the PHP/Drupal work Ashen Rayne maintains on the DMA server, and will be hosted on UBC-provided infrastructure.

This brief covers the integration surface with UBC's systems, the infrastructure UBC IT needs to provision, the responsibility split, and a short list of open questions.

---

## 2. What the platform does

Members sign in via UBC SSO, view their dues balance, and pay by card or bank debit (one-time or recurring). Council and local admins get scoped dashboards for payments, reconciliation, refunds, and an optional internal approval workflow. The platform uses Stripe Connect Standard accounts; each council holds its own Stripe relationship, and Stripe (not the platform) takes custody of funds.

---

## 3. Integration with UBC systems

### 3.1 SSO

OIDC or SAML, depending on what UBC provides. Claims the platform expects (at SSO time or via the partner API):

- UBC member identifier (`ubcid`)
- Affiliated council and local
- Member status (good standing, arrears, suspended, etc.), used to gate eligibility for online payment
- Member classification (apprentice, journeyman, retired, etc.)

### 3.2 UBC partner API

The platform calls UBC's partner APIs (per the documentation you provided) for member identity, balance (`POST /financialsnapshot`), rate, classification, and status (`GET /member/status/{ubcid}` and the bulk variant).

Expected volume at NMRCC pilot: a few thousand calls per day, dominated by a daily or weekly background sync that refreshes per-member dues data for active recurring subscribers.

Since the platform runs inside UBC's network, calls are internal-network traffic. Please confirm the auth model UBC uses for other internal applications (service account, mTLS, signed token, etc.) and we will follow the same pattern.

### 3.3 Outbound email

Transactional emails (receipts, payment failures, expiring card reminders, etc.) via UBC's existing internal SMTP relay using Nodemailer. No third-party email provider. We need relay address, port, auth mechanism, allowed sending domain, and rate limits.

---

## 4. Infrastructure we need UBC to provide

The platform is a Node.js 20+ application backed by PostgreSQL 16+. If UBC chooses the Laravel fallback (see section 1), runtime substitutes to PHP 8.2+ with PHP-FPM and nginx; the database, file storage, secrets, networking, observability, CI/CD, and environments sections below apply to both paths.

**4.1 Compute.** Container platform preferred (ECS, EKS, Docker host, etc.); VMs (Ubuntu 22.04+ or RHEL 8+) workable. Two or more instances behind a load balancer for HA. Modest sizing per instance (1-2 vCPU, 2-4 GB RAM).

**4.2 Database.** Managed PostgreSQL 16+ preferred; self-hosted on a VM workable if UBC operates backups and HA. Multi-AZ in production, daily backups with point-in-time recovery, encryption at rest, network-restricted access.

**4.3 File storage.** For receipt PDFs, exports, and branding assets (~10 GB initial, slow growth). Local filesystem on the VM is the simplest fit at NMRCC pilot scale and matches the Drupal pattern. NAS or managed object storage become useful as the platform scales to multiple application instances or many councils.

**4.4 Secrets.** Cloud-managed secrets service preferred (AWS Secrets Manager, Azure Key Vault, etc.); HashiCorp Vault workable on VMs. Stores UBC API credentials, platform-level Stripe keys (one set for the whole platform; council accounts are referenced by non-secret `account_id` values in the database), DB credentials, and the SSO client secret. Application reads secrets via service identity; rotation and access auditing required.

**4.5 Background jobs.** On the Node.js path, preferred is **Inngest** (managed durable-execution SaaS for retries, scheduled jobs, replay UI). Requires outbound HTTPS to `api.inngest.com` and inbound HTTPS from Inngest's IP ranges. No PII or financial payload crosses the boundary; only event identifiers. This is a third-party SaaS approval question for UBC's vendor policy. Fallback if not acceptable: in-process workers + scheduler + self-hosted Redis on UBC infrastructure (more code to build and operate). On the Laravel path, Laravel Horizon (built into the framework) on self-hosted Redis replaces this section entirely, with no third-party SaaS approval needed.

**4.6 Networking and DNS.** Public HTTPS endpoint for member access and Stripe webhook delivery. Load balancer in front, TLS 1.2+ with automated cert provisioning, WAF or equivalent for basic rate limiting and OWASP rules. DNS for a central UBC dues subdomain (e.g., `dues.carpenters.org`) and per-council subdomains or CNAMEs (e.g., `dues.nmrcc.org`).

**4.7 Observability.** Application log aggregation (whatever UBC uses) with ~30-day retention. Basic infrastructure metrics. Alarms for elevated 5xx, database health, and webhook processing backlog, routed to email or UBC's preferred chat.

**4.8 CI/CD.** Built and deployed via GitLab CI/CD in UBC's GitLab. We need a project for the application repository, GitLab Container Registry access, GitLab Runner availability (self-hosted preferred), and OIDC federation between GitLab and UBC's cloud for deployment auth so no long-lived deployment credentials are stored.

**4.9 Environments.** Development (Stripe test mode, UBC DEV API) and Production (Stripe live mode, UBC PROD API, Multi-AZ).

---

## 5. Security posture

- **PCI scope: SAQ-A.** Card data captured by Stripe Elements in the browser; never touches the application.
- **Tenant isolation** enforced at the database layer via PostgreSQL row-level security; verified by automated cross-tenant tests in CI.
- **Audit log** in the database, immutable, captures all admin actions and payment state transitions. Queryable from the admin dashboard.
- **No SSNs or government IDs** stored or transmitted by the platform.
- **TLS 1.2+ in transit, encryption at rest** for the database and file storage.

---

## 6. Responsibility split

| Area | Ashen Rayne | UBC IT |
|------|-------------|--------|
| Application code, schema, migrations, dependency updates and patching | Yes |  |
| Stripe Connect onboarding (with each council), council onboarding, admin training | Yes |  |
| Application-level monitoring (defines what to alert on), incident response | Yes |  |
| Infrastructure provisioning, OS patching, DB backups |  | Yes |
| TLS, DNS, secrets store operation, CI/CD platform |  | Yes |
| Infrastructure incident response |  | Yes |
| Secrets content (Ashen Rayne manages); secrets store (UBC IT operates) | Content | Operation |

---

## 7. Open items

1. **Compute**: container platform or VMs?
2. **Database**: managed PostgreSQL or self-hosted on a VM?
3. **File storage**: local filesystem, NAS, or managed object storage?
4. **Secrets store**: which service does UBC standardize on?
5. **Inngest approval** under UBC's third-party SaaS policy, or plan for the self-hosted fallback?
6. **SSO protocol** (OIDC vs. SAML) and identity claims included?
7. **Auth model** for the platform-to-UBC partner API channel within UBC's network?
8. **API rate limits and bulk endpoint patterns** for the background sync?
9. **SMTP relay configuration**?
10. **GitLab access**: which group hosts the repo, runner availability, OIDC federation pattern?

---

*References to endpoint paths follow the API documentation UBC provided.*
