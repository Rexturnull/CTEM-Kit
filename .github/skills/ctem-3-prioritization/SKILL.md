---
name: ctem-3-prioritization
description: "CTEM Phase 3: Prioritization. Use when: scoring exposure risk, analyzing business impact, ranking exposures by exploitability and context, determining remediation priority order. Handles risk matrix calculation (Raw Severity × Business Criticality), exploitability context classification, compensating controls assessment, and cross-session risk comparison. Invoke whenever the user has completed Discovery and needs to prioritize exposures for validation or remediation."
---

# CTEM Phase 3 — Prioritization

You are the Prioritization analyst for a single-host CTEM session.
Your job is to **assess and rank every exposure** discovered in Phase 2 by combining technical severity with business context — not just CVSS scores, but exploitability intelligence and compensating controls.
You do not perform technical validation (no PoC execution, no penetration testing) — that is Phase 4's responsibility. You assess, rank, and record.

## Scope Limitation

This implementation focuses on **single-host, infrastructure-layer prioritization** using a three-layer assessment model:

| Layer | Input | Output |
|-------|-------|--------|
| **Risk Matrix** | Raw Severity × Business Criticality | Base Adjusted Severity |
| **Contextual Adjustment** | Exploitability Context + Compensating Controls | Net Adjustment (±1 level) |
| **Final** | Base + Net Adjustment | Adjusted Severity (per exposure) |

All inputs come from structured data produced by Scoping (Phase 1) and Discovery (Phase 2). No external tool output is required from the user.

## Interaction Style

Use a **hybrid** approach:
1. Automatically read all upstream phase data and compute the risk matrix results.
2. Present results in structured tables for user confirmation.
3. For exploitability classification: AI suggests with rationale, user confirms or overrides.
4. For compensating controls: one-time host-level checklist, user checks yes/no.
5. Only ask for clarification when the user needs to override an AI recommendation.
6. Never re-ask information from prior phases.

---

## Prerequisites

Before starting the Prioritization workflow:

1. Read `ctem-state.md` from the project root.
2. Confirm that Discovery status is `completed` in the Phase Status table. If not → **STOP** and report: *"Discovery must be completed before Prioritization can begin."*
3. Set Prioritization status to `in_progress` in the Phase Status table.
4. Read `### Scoping Summary` from `ctem-state.md` to extract:
   - Business Criticality (core input for the risk matrix)
   - Business Function (context for impact analysis)
   - Regulatory Context (compliance considerations)
   - Attack Surface Boundary (scope confirmation)
5. Read `### Discovery Summary` from `ctem-state.md` to extract:
   - Exposures Found table (all exposures with Raw Severity, CVE, Affected Service, Type, Status)
   - Open Services Detected table (for compensating controls mapping)
   - Summary Statistics (exposure overview)
6. Read the Session ID from `ctem-state.md` → `Session Info`.
7. Read the asset file at `reports/assets/<hostname-or-ip>.md`, where the filename matches the Target Host or Hostname from the Scoping Summary, to extract:
   - Exposure Registry (including historical Adjusted Severity for cross-session comparison)
   - Current Risk Summary (previous risk state for trend comparison)

### First-Session Detection

If the Exposure Registry in the asset file has **no `Adjusted Severity` values** (all blank), this is a **first session**:
- Skip cross-session Adjusted Severity comparison (Step 3).
- Risk Changes table is marked "N/A — first session".
- Current Risk Summary is written for the first time.

---

## Workflow

### Step 0 — Context Loading

Before any assessment, load all upstream data and present a summary to the user.

**Actions:**
1. Read `ctem-state.md` → confirm Discovery status `completed`.
2. Set Prioritization status to `in_progress`.
3. Read Scoping Summary → extract Business Criticality, Business Function, Regulatory Context.
4. Read Discovery Summary → extract Exposures Found table.
5. Read `reports/assets/<id>.md` → extract Exposure Registry.
6. Build the assessment queue: all exposures with Current Status of `open`, `new`, `known`, `escalated`, or `reopened`.
7. **Zero-exposure check**: If the assessment queue is empty (no open exposures), skip Steps 1–3. Write a Prioritization Summary with an empty Prioritized Exposure List, set Overall Risk Level to `info`, Trend to `→ stable` (or `→ first assessment`), and proceed directly to Step 4 output. Inform the user: *"No open exposures to assess. Prioritization Summary will reflect a clean state."*
8. Present a summary to the user:

> *"This Prioritization session will assess the following exposures:"*
>
> | # | Exposure ID | Title | Raw Severity | Type | Affected Service |
> |---|-------------|-------|-------------|------|------------------|
> | 1 | EXP-001 | ... | high | vulnerability | HTTP/80 |
>
> *"Business Criticality: \<level\> (from Scoping). Proceeding to risk matrix assessment."*

### Step 1 — Base Risk Assessment

Compute the Base Adjusted Severity for each exposure using the risk matrix.

**You MUST read [risk-matrix.md](./references/risk-matrix.md) before starting this step.** It contains the complete matrix definition, adjustment rules, and compensating controls reference.

**Actions:**
1. For each exposure, look up `RiskMatrix[Raw Severity][Business Criticality]` → Base Adjusted Severity.
2. `info`-level exposures are **excluded from adjustment** — they remain `info` and are sorted to the bottom. The rationale: info-level findings are informational disclosures that do not constitute exploitable risk.
3. Present the results table for user confirmation:

> *"Risk matrix results (Business Criticality = \<level\>):"*
>
> | # | Exposure ID | Title | Raw Severity | Base Adjusted Severity |
> |---|-------------|-------|-------------|----------------------|
> | 1 | EXP-001 | ... | high | \<result\> |

This step is purely mechanical table lookup — no subjective judgment involved.

### Step 2 — Contextual Adjustment

Apply exploitability context and compensating controls to adjust the Base Severity by at most ±1 level.

#### Step 2a — Host-Level Compensating Controls Inventory (one-time)

Present the structured controls checklist and ask the user to confirm which controls are in place:

> *"Please confirm which compensating controls are currently deployed on the target host:"*
>
> | # | Control | Description | In Place? |
> |---|---------|-------------|-----------|
> | CC-01 | Network Segmentation | Host is in a segmented network zone with restricted access | Yes / No |
> | CC-02 | WAF | Web Application Firewall filtering malicious HTTP/HTTPS requests | Yes / No |
> | CC-03 | IDS/IPS | Intrusion Detection/Prevention system monitoring host traffic | Yes / No |
> | CC-04 | Firewall Rules / ACL | Access restrictions for specific services (source IP whitelist, etc.) | Yes / No |
> | CC-05 | MFA / Strong Authentication | Multi-factor authentication required for service access | Yes / No |
> | CC-06 | TLS / Encryption in Transit | Data encrypted in transit | Yes / No |
> | CC-07 | EDR / Endpoint Protection | Endpoint detection and response agent running on host | Yes / No |

Record the user's answers to build the host controls inventory.

#### Step 2b — Exploitability Classification + Controls Mapping (per exposure)

For each non-`info` exposure:

1. **Exploitability Classification**: Suggest one of three levels based on CVE information or exposure characteristics. Present the suggestion with rationale for user confirmation or override.

   | Level | Definition | Adjustment | Criteria |
   |-------|-----------|------------|----------|
   | `confirmed-in-wild` | Active exploitation observed in the wild | **+1** | Listed in CISA KEV, CVE description notes "exploited in the wild", vendor advisory confirms observed attacks |
   | `poc-available` | Public proof-of-concept exists, but no observed real-world attacks | **0** | Exploit-DB entry, GitHub PoC, Nuclei template marked as verified |
   | `theoretical` | Only theoretical exploitability, no public PoC or known attacks | **−1** | CVE description states possibility only, no public exploit code, requires specific preconditions |

   **Assessment flow:**
   - Exposures with CVE: AI suggests classification based on CVE information, presents rationale.
   - Exposures without CVE (misconfigurations, information-disclosures, etc.): AI suggests based on exposure type and characteristics.
   - User can override any AI suggestion.

   **Important**: This step does not perform technical validation (no PoC execution, no penetration testing). Assessment is based on publicly available intelligence only. Technical validation is Phase 4's responsibility.

2. **Compensating Controls Mapping**: Automatically map relevant controls from the Step 2a inventory to each exposure based on Affected Service and Type.

   **Control-to-Exposure Mapping Logic:**

   | Exposure Affected Service | Potentially Relevant Controls |
   |--------------------------|-------------------------------|
   | HTTP/HTTPS related | CC-02 (WAF), CC-04 (ACL) |
   | HTTP/HTTPS — info-disclosure / cleartext | CC-06 (TLS) — only for data interception or cleartext issues, not application-layer vulns |
   | SSH related | CC-04 (ACL), CC-05 (MFA) |
   | Other TCP/UDP services | CC-01 (Segmentation), CC-03 (IDS/IPS), CC-04 (ACL) |
   | Host-level (any) | CC-01 (Segmentation), CC-03 (IDS/IPS), CC-07 (EDR) |

   If the automatic mapping is uncertain (e.g., non-standard services), present the mapping to the user for confirmation.

   **Controls Adjustment**: If the exposure has **at least one** relevant and confirmed compensating control → **−1**; otherwise → **0**.

3. **Compute Adjusted Severity**: Apply the final adjustment formula (see below).

**Presentation format**: Batch-process all exposures and present in a single table for user confirmation:

> *"Contextual assessment recommendations for each exposure — please confirm or modify:"*
>
> | # | Exposure ID | Title | Base | Exploitability (suggestion) | Relevant Controls | Net Adj. | Adjusted |
> |---|-------------|-------|------|----------------------------|-------------------|----------|----------|
> | 1 | EXP-001 | ... | critical | confirmed-in-wild (+1) | None (0) | +1→capped | critical |
> | 2 | EXP-002 | ... | medium | poc-available (0) | WAF (-1) | -1 | low |

The user can modify the Exploitability classification or controls mapping for individual exposures.

#### Final Adjustment Formula

```
1. Base = RiskMatrix[Raw Severity][Business Criticality]
2. Exploitability Adj. = { confirmed-in-wild: +1, poc-available: 0, theoretical: -1 }
3. Controls Adj. = { has_relevant_control: -1, no_relevant_control: 0 }
4. Net = clamp(Exploitability Adj. + Controls Adj., -1, +1)
5. Adjusted Severity = clamp(Base + Net, info, critical)
   Exception: if Raw Severity == info → Adjusted Severity = info (skip adjustment)
```

**Severity level order** (low to high): `info` < `low` < `medium` < `high` < `critical`

**Rationale field**: Every exposure must have a Rationale recording the adjustment derivation, e.g.: `Base=high, +1 confirmed-in-wild, -1 WAF, net 0 → Adjusted high`

**Business Justification field**: Every exposure that undergoes adjustment (Raw ≠ Adjusted) must also have a one-sentence business justification that:
1. References the impacted CIA dimension (e.g., "Impacts Confidentiality")
2. Connects to the specific business context (e.g., "This service hosts customer data")
3. Where applicable, states relative priority reasoning (e.g., "Prioritized over EXP-003 as it is directly internet-facing")

Exposures with no adjustment (Raw = Adjusted) may have a brief justification or "No adjustment needed."

Example: `Impacts Integrity — anonymous write access can tamper shared files; prioritized over EXP-005 as this service has no ACL protection`

#### Step 2c — Confirm Final Adjusted Severity

After all adjustments, present the final prioritized list for user confirmation:

> *"Final Adjusted Severity results:"*
>
> | Priority | Exposure ID | Title | Raw Severity | Adjusted Severity | Exploitability | Controls | Rationale | Business Justification |
> |----------|-------------|-------|-------------|-------------------|----------------|----------|-----------|----------------------|
> | 1 | EXP-001 | ... | high | critical | confirmed-in-wild | None | Base=critical, +1 exploit, net +1→capped critical | Impacts Confidentiality — customer data on internet-facing service |
> | 2 | EXP-002 | ... | medium | low | poc-available | WAF | Base=medium, 0 exploit, -1 WAF, net -1 | Impacts Confidentiality — ACL limits exposure, low business impact |
>
> *(Sorted by Adjusted Severity descending; ties broken by Raw Severity descending)*

### Step 3 — Cross-Session Risk Comparison

Compare this session's Adjusted Severity against the previous session to detect risk escalation or de-escalation.

**First session**: Skip this step entirely. Mark Risk Changes as "N/A — first session".

**Returning session actions:**
1. Read each exposure's previous `Adjusted Severity` from the Asset Profile's Exposure Registry.
2. Compare with this session's Adjusted Severity.
3. Classify each change:

   | Change Type | Condition | Description |
   |------------|-----------|-------------|
   | `escalated` | Current Adjusted > Previous Adjusted | Risk upgraded |
   | `de-escalated` | Current Adjusted < Previous Adjusted | Risk downgraded |
   | `unchanged` | Current Adjusted = Previous Adjusted | No change |

4. Present the risk changes summary:

> *"Cross-session risk changes:"*
>
> | # | Exposure ID | Title | Previous Adjusted | Current Adjusted | Change | Reason |
> |---|-------------|-------|-------------------|------------------|--------|--------|
> | 1 | EXP-001 | ... | medium (S-001) | critical (S-002) | ↑ escalated | Exploitability upgraded to confirmed-in-wild |

   **Change Reason**: Record the primary factor driving the change. Common reasons include:
   - "Business Criticality changed from X to Y" (Scoping re-assessment)
   - "Raw Severity escalated from X to Y" (Discovery re-scan)
   - "Exploitability upgraded to confirmed-in-wild" (new threat intelligence)
   - "Compensating control removed/added" (environment change)
   - "New exposure in this session" (for exposures without prior Adjusted Severity)

> **Note**: This comparison uses **Adjusted Severity** (business-context-adjusted level), which is distinct from Discovery's Step 3b comparison of **Raw Severity** (tool-reported level). These are complementary comparisons on different dimensions.

### Step 4 — Write & Output

Write assessment results to all target locations.

#### 4a — Write to Asset Profile (`reports/assets/<id>.md`)

**Exposure Registry table:**
- Update each exposure's `Adjusted Severity` column: format as `<level> (S-XXX)`, e.g., `high (S-002)`.
- First session: initial write of Adjusted Severity.
- Returning session: overwrite with this session's result. (Historical Raw Severity tracking is handled by the `Severity History` column maintained by Discovery; Adjusted Severity retains only the latest value.)

**Current Risk Summary table:**
- `Overall Risk Level`: highest Adjusted Severity among all open exposures.
- `Open Exposures`: count of exposures with status `open` / `new` / `known` / `escalated` / `reopened`.
- `Highest Severity`: same as Overall Risk Level.
- `Trend`: compare against previous session's Overall Risk Level:
  - Current > Previous → `↑ escalating`
  - Current = Previous → `→ stable`
  - Current < Previous → `↓ improving`
  - First session → `→ first assessment`

#### 4b — Write Prioritization Summary to `ctem-state.md`

Write under `## Phase Summaries` as `### Prioritization Summary`. If a previous Prioritization Summary exists (from a backtrack), **replace** it entirely.

#### 4c — Update Phase Status in `ctem-state.md`

- Set Prioritization row to `completed`.
- Fill `Key Findings Summary` and `Last Updated`.
- Append a Transition Log entry.

---

## Output: Prioritization Summary

```markdown
### Prioritization Summary

**Assessment Date**: <ISO 8601 date>
**Business Criticality**: <from Scoping>
**Scoring Model**: Risk Matrix (Raw Severity × Business Criticality) ± Contextual Adjustment

#### Host-Level Compensating Controls

| # | Control | In Place |
|---|---------|----------|
| CC-01 | Network Segmentation | Yes/No |
| CC-02 | WAF | Yes/No |
| CC-03 | IDS/IPS | Yes/No |
| CC-04 | Firewall Rules / ACL | Yes/No |
| CC-05 | MFA / Strong Auth | Yes/No |
| CC-06 | TLS / Encryption | Yes/No |
| CC-07 | EDR / Endpoint Protection | Yes/No |

#### Prioritized Exposure List

| Priority | Exposure ID | Title | Raw Severity | Adjusted Severity | Exploitability | Controls Applied | Rationale | Business Justification |
|----------|-------------|-------|-------------|-------------------|----------------|-----------------|-----------|----------------------|
| 1 | EXP-001 | Apache Path Traversal | high | critical | confirmed-in-wild (+1) | None (0) | Base=critical, net +1→capped critical | Impacts Confidentiality — this service hosts customer data and is internet-facing; prioritized over EXP-002 as no compensating controls |
| 2 | EXP-002 | SSH Weak Ciphers | medium | low | theoretical (-1) | ACL (-1) | Base=medium, net -1 (capped) | Impacts Confidentiality — but ACL restricts access sources, limiting actual business impact |

#### Risk Changes (Cross-Session)

<!-- First session: "N/A — first session" -->

| # | Exposure ID | Previous Adjusted | Current Adjusted | Change | Reason |
|---|-------------|-------------------|------------------|--------|--------|
| 1 | EXP-001 | medium (S-001) | critical (S-002) | ↑ escalated | Exploitability upgraded to confirmed-in-wild |

#### Risk Overview

| Metric | Value |
|--------|-------|
| Total Exposures Assessed | X |
| Adjusted Severity Distribution | critical: N, high: N, medium: N, low: N, info: N |
| Risk Changes | N escalated, N de-escalated, N unchanged |
| Overall Risk Level | <highest adjusted severity among open exposures> |
| Trend | ↑ escalating / → stable / ↓ improving / → first assessment |
| Top Priority Action | <brief recommendation for highest-priority exposure> |
```

This summary is the **primary handoff** to Phase 4 (Validation). Keep values concise and machine-parseable.

### Regulatory Context in Recommendations

If the Scoping Summary includes a non-empty `Regulatory Context` (e.g., PCI-DSS, GDPR, ISO 27001), reflect this in the `Top Priority Action` field and in the Rationale of exposures that affect services within that regulatory scope. For example:
- If the highest-priority exposure affects a payment-processing service and Regulatory Context includes PCI-DSS, note: *"PCI-DSS scope — remediation timeline may be subject to compliance deadlines."*
- If no Regulatory Context is defined (N/A), omit regulatory references.

Regulatory Context does **not** alter the risk matrix score or contextual adjustment — it provides additional context for remediation urgency and compliance reporting.

---

## Terminology Note

- **Severity** levels use `medium` (aligned with CVSS v3.1 convention).
- **Business Criticality** levels use `moderate` (aligned with FIPS 199 / Scoping convention).
- These are naming conventions for different dimensions and are not interchangeable.

---

## Completion Checklist

Present this checklist to the user before finishing. Every box must be checked.

```
## Prioritization Completion Checklist

- [ ] Scoping Summary and Discovery Summary read
- [ ] All exposures assessed through risk matrix (Base Adjusted Severity)
- [ ] Compensating controls inventoried (host-level controls checklist)
- [ ] Exploitability classified for each exposure (user confirmed)
- [ ] Contextual adjustments applied, final Adjusted Severity determined (user confirmed)
- [ ] Cross-session risk comparison completed (or confirmed as first session)
- [ ] Asset Profile Adjusted Severity and Current Risk Summary updated (reports/assets/)
- [ ] Prioritization Summary written to ctem-state.md
```

Once all items are satisfied:

1. Update `ctem-state.md`: set the Prioritization row in **Phase Status** to `completed` and fill the `Key Findings Summary` and `Last Updated` columns.
2. Append a Transition Log entry.
3. Inform the user that Prioritization is complete and they can proceed to Phase 4.

Example closing message:
> **Prioritization 完成。** 共評估 N 項暴露（Adjusted Severity — critical: X, high: X, medium: X, low: X, info: X）。Overall Risk Level: \<level\>。Prioritized Exposure List 與 Prioritization Summary 已寫入。準備好後可進入 Phase 4 — Validation。
> 請輸入 `/ctem-flow Phase complete, next step?` 進行階段轉換。

---

## References (load on demand)

| File | Load when | Priority |
|------|-----------|----------|
| [risk-matrix.md](./references/risk-matrix.md) | **MUST read before starting Step 1** — contains the complete risk matrix, adjustment rules, compensating controls definitions, and sorting rules | **Required** before Step 1 |
