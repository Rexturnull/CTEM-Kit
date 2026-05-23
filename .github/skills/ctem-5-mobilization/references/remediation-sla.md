# Remediation SLA — Default Timeline & Override Rules

This reference defines the Service Level Agreement (SLA) framework for CTEM Phase 5 (Mobilization). It includes default remediation timelines, override mechanisms, deadline calculation logic, and regulatory context notes.

## Framework Basis

The SLA defaults are informed by industry-standard remediation timelines commonly adopted in vulnerability management programs. The tiered approach aligns severity with urgency — critical findings demand immediate action while lower-severity findings allow more planning time.

| Source | Role |
|--------|------|
| **CVSS v3.1 Severity Levels** (FIRST, 2019) | Defines the severity tiers that map to SLA durations |
| **Industry Practice** | SLA durations reflect commonly adopted timelines across enterprise vulnerability management programs (e.g., PCI-DSS 30-day requirement for high-risk vulnerabilities) |
| **Gartner CTEM** (2022, G00766755) | Mobilization must produce actionable, time-bound remediation plans — not open-ended recommendations |

---

## Part 1 — Default SLA Table

The SLA duration is based on the exposure's **Adjusted Severity** (from Prioritization), not Raw Severity. This ensures business context is factored into remediation urgency.

| Adjusted Severity | Default SLA | Description | Interim Measure |
|-------------------|-------------|-------------|-----------------|
| `critical` | **24 hours** | Immediate remediation required. If full fix cannot be completed within 24h, an interim mitigation measure must be in place. | Required — e.g., WAF rule, service isolation, access restriction |
| `high` | **7 days** | Remediate within one week. Plan and execute during normal operations. | Recommended if fix requires maintenance window |
| `medium` | **30 days** | Remediate within one month. Can be scheduled with regular maintenance cycles. | Optional |
| `low` | **90 days** | Remediate within one quarter. Low urgency; address during planned maintenance or upgrade cycles. | Not required |
| `info` | **N/A** | Informational finding. No SLA — address at discretion during routine maintenance. | Not applicable |

### Interim Mitigation Measures

For `critical` exposures where the full remediation cannot be completed within 24 hours, an interim mitigation measure should be documented in the remediation plan. Common interim measures include:

| Measure | Example |
|---------|---------|
| Network isolation | Block access to the vulnerable service from untrusted networks |
| WAF rule | Add a specific WAF rule to block the known attack pattern |
| Service disable | Temporarily disable the vulnerable service if non-critical |
| Access restriction | Restrict access to the service via ACL / firewall rules |
| Monitoring escalation | Increase monitoring/alerting on the vulnerable service |

The interim measure is documented in the Quick Fix or Notes field of the remediation action.

---

## Part 2 — Deadline Calculation

### Formula

```
Deadline = Plan Date + SLA Duration
```

- **Plan Date**: the date the Mobilization plan is created (ISO 8601 format: YYYY-MM-DD).
- **SLA Duration**: from the Default SLA Table above, based on Adjusted Severity.
- **Deadline**: ISO 8601 date (YYYY-MM-DD).

### Duration Mapping

| SLA Duration | Calendar Days |
|-------------|---------------|
| 24 hours | +1 day |
| 7 days | +7 days |
| 30 days | +30 days |
| 90 days | +90 days |
| N/A | No deadline |

### Examples

If Plan Date = 2026-04-05:

| Adjusted Severity | SLA | Deadline |
|-------------------|-----|----------|
| critical | 24h | 2026-04-06 |
| high | 7d | 2026-04-12 |
| medium | 30d | 2026-05-05 |
| low | 90d | 2026-07-04 |
| info | N/A | — |

---

## Part 3 — SLA Override Rules

Users can override SLA defaults at two levels:

### Global Override

Changes the entire SLA table for the session. The user provides new durations for one or more severity levels.

**Trigger**: user says something like "Our critical SLA is 48 hours" or "Change all SLAs to 2x the default".

**Action**:
1. Update the SLA table for the affected severity levels.
2. Recalculate all deadlines using the new durations.
3. Present the updated assignment table for user confirmation.

**Example**:
> User: "Our policy requires critical within 48 hours, high within 14 days."
>
> Updated SLA:
> | Severity | Original | Override |
> |----------|----------|---------|
> | critical | 24h | 48h |
> | high | 7d | 14d |
> | medium | 30d | 30d (unchanged) |
> | low | 90d | 90d (unchanged) |

### Per-Item Override

Changes the deadline for a specific action only.

**Trigger**: user says something like "EXP-003 extend to 14 days — needs a maintenance window".

**Action**:
1. Update only the specified action's deadline.
2. Record the override reason in the action's Notes/Status field.
3. Other actions remain on their original deadlines.

**Recording**: When reporting, mark overridden deadlines with `(override)` annotation so the deviation from default SLA is visible.

---

## Part 4 — Effort Level Definitions

Effort Level estimates the engineering cost of implementing a remediation action. These are used in Step 1 (Remediation Plan Generation) and Step 2 (Chain-Breaker identification).

| Effort | Definition | Typical Time | Examples |
|--------|-----------|--------------|---------|
| `trivial` | Single command or configuration change. No service interruption. Minimal risk of side effects. | < 30 minutes | Change a config parameter, add a firewall rule, disable a feature flag |
| `low` | A few sequential steps. May require basic testing afterward. No planned downtime. | 30 min – 2 hours | Software package upgrade, certificate renewal, add security headers |
| `medium` | Multiple coordinated steps. May require service restart or brief downtime. Should be tested in staging first. | 2 – 8 hours | Major version upgrade with config migration, enable TLS across services, restructure file permissions |
| `high` | Significant change requiring planning, a maintenance window, backup, and rollback procedures. Affects multiple components or workflows. | 1 – 3 days | Database migration, network architecture changes, multi-service authentication overhaul |
| `critical` | Architecture-level change or complete component replacement. Requires extensive planning, testing, and staged rollout. | > 3 days | Replace legacy system, migrate to new service platform, redesign authentication infrastructure |

### Effort and Chain-Breaker Analysis

In chain-breaker selection (Step 2), Effort Level determines the "cost" side of the ROI equation:
- A `trivial` effort fix that breaks an attack chain = highest ROI
- A `critical` effort fix that breaks the same chain = low ROI compared to alternatives

When multiple exposures in a chain have the same Effort Level, the one earliest in the chain is preferred (breaking the chain earlier prevents more downstream exploitation steps).

---

## Part 5 — Regulatory Context Reference

When the Scoping Summary includes a Regulatory Context, the following reference points help contextualize SLA compliance. These do **not** override the default SLA — they provide information for user decision-making.

| Framework | Relevant Remediation Requirement | Notes |
|----------|----------------------------------|-------|
| **PCI-DSS v4.0** | Requirement 6.3.3: Critical and high-risk vulnerabilities must be addressed with patches or mitigations within defined timeframes. Typically interpreted as 30 days for high-risk, with critical requiring immediate action. | Addresses payment card data environments specifically |
| **GDPR** | Article 32: Implement appropriate technical measures. No specific SLA, but breach notification within 72 hours implies rapid remediation expectations. | Focuses on personal data protection |
| **ISO 27001:2022** | Clause 8.1 & Annex A.8.8: Manage technical vulnerabilities in a timely manner with defined processes. | Requires documented vulnerability management process |
| **NIST CSF 2.0** | RS.MI: Incidents are contained and mitigated. ID.RA: Risk assessments inform prioritization. | Framework-level guidance, no specific SLA numbers |
| **SOC 2 Type II** | CC7.1: Detect and respond to security events. Remediation timelines must align with defined procedures. | Auditor expects documented process and adherence |

### Usage in Mobilization

When presenting the remediation timeline (Step 4), if a Regulatory Context is defined:
1. Check if any action's deadline exceeds the regulatory framework's typical expectation.
2. If so, add a compliance note to that action's presentation — not as a hard override, but as an advisory.
3. Record the regulatory reference in the Mobilization Summary notes.

**Example advisory**:
> *"Note: Regulatory Context is PCI-DSS. ACT-001 (critical, deadline: 2026-04-06) aligns with PCI-DSS immediate remediation expectations. ACT-003 (medium, deadline: 2026-05-05) is within PCI-DSS's 30-day high-risk window."*
