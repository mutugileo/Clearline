# ADR 0001: Platform architecture for Clearline

- **Status:** Proposed
- **Date:** 2026-10-07
- **Deciders:** Clearline founding team (to confirm before the pilot)

## Context

Clearline audits a bank's APIs for security vulnerabilities. It sends test traffic to systems that move money, so a bug in Clearline can cause a real incident, and a breach of Clearline can expose bank credentials and API designs.

The first customers are banks. Their security, risk and compliance teams will judge the product on how it is deployed, secured, audited and supported, not only on how well it finds bugs. The product must run inside the bank's network, must be auditable, and must fit the bank's existing operations.

This ADR sets the platform architecture. The front-end stack is in ADR 0002.

## Decision drivers

1. Runs fully inside the bank's network, including air-gapped sites.
2. Every action is attributable to a person and recorded in a way that can be proven later.
3. Credentials and bank data are never exposed in plain text, and never leave the bank's control.
4. Small team can build and operate it.
5. Standard, well-understood components that the bank's operations team can run.

## Options considered

| Option | Summary | Verdict |
|---|---|---|
| A. Multi-tenant SaaS | One hosted product for many banks | Rejected for v1. Banks' risk teams will not accept their API traffic and credentials in a shared, vendor-hosted system. |
| B. Single-tenant, bank-hosted deployment | One installation per bank, inside the bank's network | **Chosen.** |
| C. Single-tenant, vendor-hosted (dedicated cloud) | One installation per bank, in our cloud account | Possible later for banks that allow it. Not the default. |
| D. Microservices | Separate services for scanning, reporting, auth and so on | Rejected. More operational surface for a small team and more places for failure. |
| E. Modular monolith plus worker pool | One API service with clear modules, plus worker processes that run scans | **Chosen.** |

## Decision

1. **Deployment:** single-tenant, deployed to Kubernetes (or OpenShift where the bank runs it) with Helm charts. Releases are applied through GitOps (Argo CD or Flux). The same chart runs in the bank's network and in our test environment.
2. **Air-gap support:** every release ships as an offline bundle containing container images, Nuclei templates, language packages and documentation, with no runtime calls to the internet. Updates are imported from that bundle.
3. **Application:** Python with FastAPI. A modular monolith with these modules: `scope`, `specs`, `scans`, `checks`, `findings`, `reports`, `audit`, `auth`. Module boundaries are enforced by import rules in CI.
4. **Scanning:** scan jobs run in a separate worker deployment. Known-issue checks call Nuclei as a subprocess. Authorisation checks use Clearline's own Python code. Workers hold no long-lived credentials; they receive per-scan credentials for the duration of the scan only.
5. **Scope enforcement:** every outbound request passes through one egress component that checks the target against the approved scope and rate limits. Nothing else in the code base can send traffic to a target. Out-of-scope requests fail closed and are logged.
6. **Data:** PostgreSQL in a highly available setup (CloudNativePG or Patroni) with synchronous or semi-synchronous replication, automated failover, and point-in-time recovery backups. Backups are encrypted and restore-tested on a schedule.
7. **Keys and secrets:** encryption keys come from the bank's KMS or HSM. Test credentials are encrypted with a data key and never written to logs. The bank's secret manager (or Vault) holds the root material.
8. **Identity:** OpenID Connect or SAML with the bank's identity provider. MFA is enforced by the identity provider. Clearline applies role-based access control with four roles: Viewer, Analyst, Approver and Administrator. Destructive or production-facing actions require an Approver.
9. **Audit log:** an append-only table in which each row stores a hash of the previous row. Every write, scan start, scan stop, scope change and export is logged with user, time, and request ID. Logs are forwarded to the bank's SIEM over syslog (CEF format).
10. **Transport security:** TLS 1.2 minimum, TLS 1.3 preferred. Mutual TLS between internal services. Certificates come from the bank's internal CA.
11. **Supply chain:** each release has a software bill of materials (SBOM, CycloneDX), signed images (cosign), and a pinned, scanned base image. Admission policy (Kyverno or OPA Gatekeeper) allows only signed images from the approved registry.
12. **Observability:** OpenTelemetry for traces and metrics, exported to the bank's Prometheus or equivalent. Structured JSON logs with secrets redacted.
13. **Quality gates in CI:** unit and integration tests, static analysis (Semgrep), dependency and container scanning, secret scanning, SBOM generation, and a test run against OWASP crAPI.

## Consequences

**Positive**
- The product fits how banks already buy and run software, which shortens procurement.
- Credentials and traffic stay inside the bank.
- Auditors can trace every action.

**Negative**
- Higher cost to run per customer, because each bank has its own database, cluster footprint and disaster recovery setup. This must be reflected in pricing.
- Releases are slower: each customer upgrades on its own schedule, so the team must support more than one version.
- Some features that are easy in SaaS (telemetry, instant fixes) need a bank-approved process.

**Risks to watch**
- The bank's Kubernetes platform may differ from ours. Test the chart on OpenShift before the pilot.
- Central Bank of Kenya outsourcing rules may restrict where the software can run. Confirm this with the bank's compliance team before design freeze.

## Requirements this ADR must meet

These are non-negotiable for any change to the platform:

- [ ] Works fully offline after install
- [ ] No plain-text credentials in the database, logs, or images
- [ ] Every action is in the tamper-evident audit log
- [ ] Out-of-scope traffic is blocked in code, not only in configuration
- [ ] Restore from backup is tested before each pilot milestone
- [ ] Every release has an SBOM and signed images
- [ ] Access is by SSO with MFA, and role checks run on the server

## Open questions

1. Which Kubernetes distribution does the first pilot bank run?
2. Does the first pilot bank require a specific HSM or KMS product?
3. What RPO and RTO does the bank require?
4. Does the bank's SIEM accept CEF over syslog, or does it need another format?
5. Does Clearline handle cardholder data at any point, which would bring PCI DSS into scope?

## Related

- ADR 0002: Front-end architecture
