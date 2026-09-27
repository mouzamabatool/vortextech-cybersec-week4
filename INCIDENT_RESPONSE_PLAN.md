# Incident Response Plan: Customer Database Exposure via Unsecured API Endpoint

**Company (hypothetical):** Meridian Retail Co. — mid-size e-commerce platform (~2.5M registered customers)
**Incident ID:** IR-2026-0417
**Classification:** Data Breach — Unauthorized Access to Customer PII
**Framework Applied:** NIST SP 800-61 Incident Response Lifecycle (Preparation, Detection & Analysis, Containment, Eradication, Recovery, Post-Incident Activity)

---

## 1. Scenario Summary

Meridian Retail Co. migrated its order-history feature to a new microservice in Q1 2026. The new `/api/v2/orders/{customer_id}` endpoint was deployed without an authentication middleware check — a configuration step that was accidentally skipped during the CI/CD pipeline migration. As a result, any user (authenticated or not) could query the endpoint by incrementing `customer_id` values and retrieve full order histories, including names, shipping addresses, email addresses, and the last four digits of payment cards for **any** customer in the database.

The flaw was exploited by an unknown third party who scraped an estimated 180,000 customer records over a 36-hour window before detection. The exposure was first flagged by an external security researcher who found the endpoint via routine scanning and responsibly disclosed it through Meridian's bug bounty inbox.

---

## 2. Preparation (Pre-Incident State)

Controls that were already in place and that enabled a fast, orderly response:

- **Incident Response Team (IRT)** with defined roles: IR Lead (CISO delegate), Security Engineer on-call, Engineering Lead for the affected service, Legal/Compliance liaison, and Communications/PR liaison.
- **Bug bounty and responsible-disclosure program** with a monitored inbox (security@meridianretail.example) and a documented SLA for triage.
- **Centralized logging** (API gateway logs, WAF logs, database query logs) retained for 90 days and searchable via a SIEM.
- **Automated backups** of the customer database, taken every 6 hours, with integrity verification.
- **Runbook templates** for common incident types, including "unauthorized data access," pre-approved by Legal so the team isn't drafting process from scratch mid-crisis.
- **Pre-drafted regulatory notification templates** (GDPR/CCPA-style) reviewed by outside counsel in advance.

---

## 3. Detection & Analysis

**How it was discovered:** An external researcher submitted a disclosure report showing they could retrieve arbitrary customer order records by manipulating the `customer_id` parameter, with no authentication token required.

**Initial investigation steps taken by the IRT:**

1. IR Lead validated the report within 1 hour by reproducing the request in a staging-mirrored environment (never against production with real customer IDs).
2. Security Engineer pulled API gateway logs for the endpoint and identified a sustained pattern of sequential `customer_id` requests originating from a small set of IPs over the prior 36 hours — consistent with automated scraping, not the researcher's own testing.
3. Cross-referenced WAF logs to confirm no other endpoints were affected by the same misconfiguration.
4. Estimated scope: ~180,000 unique customer records accessed, based on the range of `customer_id` values queried.
5. Classified severity as **High** (confirmed unauthorized access to PII at scale) and formally opened Incident IR-2026-0417.

---

## 4. Containment

**Short-term containment (within the first 2 hours):**

- Took the `/api/v2/orders/{customer_id}` endpoint offline immediately via the API gateway (feature flag kill switch), rather than a full service outage.
- Rate-limited and then blocked the source IP ranges identified in the log analysis at the WAF/CDN layer.
- Rotated any internal service tokens and API keys associated with the affected microservice, in case the misconfiguration extended to credential exposure.

**Longer-term containment (within 24 hours):**

- Deployed the authentication middleware fix behind a feature flag to a staging environment for validation before re-enabling the endpoint.
- Placed a temporary WAF rule requiring a valid session token on all `/api/v2/orders/*` routes as a stopgap ahead of the permanent code fix.

---

## 5. Eradication

- Root-caused the issue to a missing `@RequiresAuth` decorator that had been present on the legacy `/api/v1/orders` route but was not carried over during the microservice migration, combined with a gap in the CI/CD pipeline's pre-deployment security checklist that should have caught it.
- Engineering patched the endpoint to enforce authentication and ownership validation (a user may only request their **own** `customer_id`, not an arbitrary one).
- Added an automated integration test asserting that all `/api/v2/*` routes reject unauthenticated requests, to prevent regression.
- Ran a full audit of every other `/api/v2/*` endpoint deployed in the same migration to confirm no sibling misconfigurations existed.
- Confirmed no malware, backdoor, or persistent unauthorized access existed — this was a logic/configuration flaw, not a network intrusion, so no host remediation was required.

---

## 6. Recovery

- Re-enabled the patched endpoint behind the feature flag, first to 5% of production traffic, then ramped to 100% over 4 hours while monitoring error rates and query patterns.
- Enhanced monitoring: added an alert that fires if any single IP queries more than 20 distinct `customer_id` values within 5 minutes.
- Verified database integrity against the most recent clean backup to confirm no records were altered or deleted, only read.
- Kept the temporary WAF rule active for two additional weeks as a defense-in-depth measure while confidence in the permanent fix was established.

---

## 7. Post-Incident Activity (Lessons Learned)

A post-incident review was held within 5 business days with the full IRT plus the engineering team responsible for the migration. Key conclusions:

- The CI/CD security checklist needs an automated gate, not a manual checkbox — the missing auth decorator should have failed the build, not just a code review step.
- Bug bounty disclosure handling worked well and should be used as the model for future "how did we find out" retrospectives.
- Time-to-containment (2 hours) was acceptable, but time between deployment and detection (an estimated 3 weeks the endpoint was live before scraping began) was too long — indicates a gap in proactive API endpoint auditing.
- Updated the incident response runbook with the specific timeline from this incident as a reference case.

---

## 8. Internal Communication Plan

| Timeframe | Who is notified | By whom | Purpose |
|---|---|---|---|
| Immediately (within 30 min of confirmation) | CISO, IR Lead, Engineering Lead for affected service | Security Engineer on-call | Activate IRT, begin containment |
| Within 1 hour | CTO, VP of Engineering | IR Lead | Executive awareness, resourcing decisions |
| Within 2–4 hours | Legal/Compliance team | IR Lead | Assess regulatory notification obligations and timelines |
| Within 4–6 hours | Communications/PR lead, Customer Support lead | Legal/Compliance | Prepare customer-facing messaging and support scripts |
| Within 24–72 hours (per applicable law) | Affected customers, relevant data protection regulators | Legal/Compliance + Comms, with executive sign-off | Regulatory compliance (e.g., breach notification laws); conceptually, most frameworks require notifying regulators and affected individuals within a defined window once a breach involving PII is confirmed — exact deadlines depend on jurisdiction and were confirmed with outside counsel rather than assumed |
| Ongoing | All staff (internal memo) | HR/Internal Comms | Awareness, avoid rumor/misinformation, remind staff of communication protocol (no public statements without Comms approval) |

---

## 9. Preventative Measures Going Forward

1. **Automated security gates in CI/CD:** Require every new or modified API route to pass an automated authentication/authorization test before it can merge or deploy — closing the exact gap that caused this incident.
2. **Scheduled API endpoint audits:** Introduce a recurring (at minimum monthly) automated scan of all live production endpoints to confirm authentication is enforced, rather than relying solely on point-in-time code review during migrations.
3. **Object-level authorization by default:** Adopt a "deny by default" pattern for any endpoint accepting an ID parameter, requiring explicit ownership validation rather than trusting that authentication alone is sufficient (this addresses the broader class of vulnerability known as Broken Object Level Authorization, one of the most common API security issues).

---

*This document is a fictional training exercise prepared for the Vortex Tech Cyber Security Internship Track, Week 4. All company names, individuals, and data described are hypothetical.*
