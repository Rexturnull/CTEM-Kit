# Risk Assessment Matrix

This reference defines the complete risk assessment framework for CTEM Prioritization (Phase 3). It includes the risk matrix, contextual adjustment rules, compensating controls definitions, and sorting logic.

## Framework Basis

This matrix combines exposure-level technical severity with asset-level business criticality to produce a risk-informed priority:

| Layer | Framework | Role |
|-------|-----------|------|
| **X-axis** — Raw Severity | **CVSS v3.1** (FIRST, 2019) | Technical severity from Discovery scan tools |
| **Y-axis** — Business Criticality | **FIPS 199** / **NIST SP 800-30 Rev. 1** | Business impact from Scoping CIA assessment |
| **Output** — Base Adjusted Severity | Combined | Risk-informed severity for prioritization |

### Primary Sources

1. **FIRST — CVSS v3.1 Specification Document** (2019)
   - URL: https://www.first.org/cvss/v3.1/specification-document
   - Role: Defines the severity rating scale (critical/high/medium/low) used for Raw Severity.

2. **NIST SP 800-30 Rev. 1 — Guide for Conducting Risk Assessments** (2012)
   - URL: https://csrc.nist.gov/pubs/sp/800-30/r1/final
   - Cited sections: **Appendix H, Table H-3** — Assessment Scale: Level of Impact; **Table G-3** — Assessment Scale: Likelihood of Threat Event Occurrence
   - Role: Informs the matrix design — combining likelihood (approximated by exploitability) with impact (business criticality) to determine risk level.

3. **FIPS 199 — Standards for Security Categorization** (2004)
   - URL: https://csrc.nist.gov/pubs/fips/199/final
   - Role: Defines the Business Criticality levels used in Scoping (critical/high/moderate/low).

### CTEM Alignment

Gartner's CTEM framework requires Prioritization to go **beyond traditional vulnerability severity** and incorporate business impact, exploitability context, and compensating controls (Gartner, 2022, G00766755). This matrix operationalizes that principle: Raw Severity alone does not determine priority — it is combined with business context and adjusted for real-world factors.

---

## Part 1 — Risk Matrix (Base Adjusted Severity)

### Matrix Definition

Look up the intersection of **Raw Severity** (X-axis, from Discovery) and **Business Criticality** (Y-axis, from Scoping) to determine the **Base Adjusted Severity**.

|  | **Raw: critical** | **Raw: high** | **Raw: medium** | **Raw: low** | **Raw: info** |
|---|---|---|---|---|---|
| **Criticality: critical** | critical | critical | high | medium | info |
| **Criticality: high** | critical | high | high | medium | info |
| **Criticality: moderate** | high | high | medium | low | info |
| **Criticality: low** | high | medium | low | low | info |

### Matrix Design Rationale

The matrix follows these principles:

1. **info stays info**: Informational findings do not constitute exploitable risk regardless of business criticality. They are always excluded from adjustment and sorted to the bottom.

2. **Critical business assets escalate risk**: A `medium` Raw Severity on a `critical` asset becomes `high` Base — reflecting that the business impact of exploitation is severe even if the technical difficulty is moderate.

3. **Low-criticality assets de-escalate risk**: A `high` Raw Severity on a `low`-criticality asset becomes `medium` Base — the technical severity is real, but the business impact of exploitation is limited.

4. **Maximum is critical**: The matrix never exceeds `critical`. Multiple `critical` inputs (e.g., critical Raw + critical Criticality) still produce `critical`.

5. **Diagonal alignment**: Along the diagonal (matching severity and criticality levels), the output generally equals the Raw Severity — when technical and business assessments agree, no adjustment is needed.

### Severity Level Order

From lowest to highest: `info` < `low` < `medium` < `high` < `critical`

This ordering is used for all comparisons, adjustments, and sorting throughout the Prioritization phase.

### Terminology Clarification

- **Severity** levels: `info`, `low`, `medium`, `high`, `critical` — aligned with CVSS v3.1.
- **Business Criticality** levels: `low`, `moderate`, `high`, `critical` — aligned with FIPS 199 / Scoping.
- Note: Severity uses `medium` while Criticality uses `moderate`. These are conventions from different frameworks and are not interchangeable.

---

## Part 2 — Contextual Adjustment Rules

After computing the Base Adjusted Severity from the matrix, apply contextual adjustments based on two factors: Exploitability Context and Compensating Controls.

### Exploitability Context

Three qualitative levels, assessed per exposure:

| Level | Definition | Adjustment | Criteria |
|-------|-----------|------------|----------|
| `confirmed-in-wild` | Active exploitation observed in real-world attacks | **+1** | Listed in CISA Known Exploited Vulnerabilities (KEV) catalog; CVE description or vendor advisory confirms active exploitation; threat intelligence reports active campaigns |
| `poc-available` | Public proof-of-concept code exists, but no observed real-world attacks | **0** | Exploit-DB entry exists; GitHub PoC repository; Nuclei template marked as verified; Metasploit module available |
| `theoretical` | Only theoretical exploitability; no public PoC or known attacks | **−1** | CVE description states possibility only; no public exploit code available; exploitation requires specific or unlikely preconditions; complexity is high with no known bypass |

**Assessment guidelines:**
- For exposures **with CVE**: Classify based on publicly available CVE information (NVD, vendor advisories, CISA KEV).
- For exposures **without CVE** (misconfigurations, information-disclosures, outdated-software): Classify based on the nature of the exposure:
  - Default credentials or known-weak configurations → typically `poc-available` (tools readily exploit these)
  - Information disclosures (version banners, directory listings) → typically `theoretical` (information alone is not directly exploitable)
  - Outdated software without specific CVE → typically `theoretical` unless known exploit campaigns target that version range

**Important**: This classification is based on publicly available intelligence only. No technical validation (PoC execution, penetration testing) is performed here — that is Phase 4's responsibility.

### Compensating Controls Adjustment

The compensating controls adjustment is based on whether the exposure has relevant controls already in place that reduce exploitability.

**Rule**: If the exposure has **at least one relevant and confirmed** compensating control → **−1**; otherwise → **0**.

The control relevance is determined by the Control-to-Exposure Mapping (see Part 3).

### Net Adjustment Calculation

```
Net Adjustment = Exploitability Adj. + Controls Adj.
Net Adjustment = clamp(Net, -1, +1)          ← cap at ±1 level
Adjusted Severity = clamp(Base + Net, info, critical)   ← stay within range
```

**Exception**: If Raw Severity = `info` → Adjusted Severity = `info` (skip all adjustment).

**Worked examples:**

| Base | Exploitability | Controls | Raw Net | Clamped Net | Adjusted |
|------|---------------|----------|---------|-------------|----------|
| high | confirmed-in-wild (+1) | None (0) | +1 | +1 | critical |
| high | confirmed-in-wild (+1) | WAF (-1) | 0 | 0 | high |
| medium | poc-available (0) | ACL (-1) | -1 | -1 | low |
| medium | theoretical (-1) | Segmentation (-1) | -2 | **-1** (clamped) | low |
| critical | confirmed-in-wild (+1) | None (0) | +1 | +1 | **critical** (capped) |
| low | theoretical (-1) | EDR (-1) | -2 | **-1** (clamped) | **low** (floor applied) |

**Floor rule for `low` → `info` demotion**: When an adjustment would push a `low` Base Severity down to `info`, apply the following rule:
- If the exposure Type is `vulnerability` or `misconfiguration` → **floor at `low`** (these types have inherent exploitable risk that `info` would misrepresent). Record "floor applied — vulnerability/misconfiguration cannot be demoted to info" in the Rationale.
- If the exposure Type is `information-disclosure` or `outdated-software` → allow demotion to `info` (these types may genuinely be informational when adjusted for context). Record the demotion reason in the Rationale.

> **Cross-phase applicability**: This formula (including the floor rule above) applies to **all phases** that compute Adjusted Severity. This includes Phase 3 (Prioritization) primary assessment and Phase 4 (Validation) in-place registration of newly discovered exposures.

---

## Part 3 — Compensating Controls Reference

### Host-Level Controls Checklist

| # | Control ID | Control Category | Description | Applicable Service Types |
|---|-----------|-----------------|-------------|--------------------------|
| 1 | CC-01 | Network Segmentation | Host is in a segmented network zone with restricted access (VLAN, micro-segmentation, DMZ) | All |
| 2 | CC-02 | WAF (Web Application Firewall) | Filters malicious HTTP/HTTPS requests before they reach the application | Web services (HTTP/HTTPS) |
| 3 | CC-03 | IDS/IPS | Intrusion Detection/Prevention system monitoring and/or blocking malicious traffic to the host | All |
| 4 | CC-04 | Firewall Rules / ACL | Access restrictions for specific services — source IP whitelists, port restrictions, deny-by-default policies | Specific ports/services |
| 5 | CC-05 | MFA / Strong Authentication | Multi-factor authentication or strong authentication mechanisms required for service access | Authentication services (SSH, Admin Panels, RDP) |
| 6 | CC-06 | TLS / Encryption in Transit | Transport-layer encryption protecting data in transit | Services transmitting sensitive data |
| 7 | CC-07 | EDR / Endpoint Protection | Endpoint detection and response agent actively running on the host | Host-level (any) |

### Control-to-Exposure Mapping

Use this mapping to determine which confirmed controls are relevant to each exposure:

| Exposure Affected Service | Potentially Relevant Controls |
|--------------------------|-------------------------------|
| HTTP / HTTPS related | CC-02 (WAF), CC-04 (ACL) |
| HTTP / HTTPS — information-disclosure or misconfiguration (cleartext) | CC-06 (TLS) — only relevant when the exposure involves data interception or cleartext transmission, not for application-layer vulnerabilities like path traversal or XSS |
| SSH related | CC-04 (ACL), CC-05 (MFA) |
| Database services (MySQL, PostgreSQL, etc.) | CC-01 (Segmentation), CC-04 (ACL) |
| Other TCP/UDP services | CC-01 (Segmentation), CC-03 (IDS/IPS), CC-04 (ACL) |
| Host-level (any exposure) | CC-01 (Segmentation), CC-03 (IDS/IPS), CC-07 (EDR) |

**Mapping rules:**
1. Each exposure is mapped to controls based on its `Affected Service` field from Discovery.
2. An exposure may match multiple rows — union all applicable controls.
3. A control is "relevant" only if it appears in the mapping AND is confirmed as "Yes" in the host-level inventory.
4. If the automatic mapping is uncertain (e.g., non-standard service names), present the mapping to the user for confirmation.

---

## Part 4 — Priority Sorting Rules

After computing the final Adjusted Severity for all exposures, sort them into a prioritized list.

### Primary Sort: Adjusted Severity (descending)

`critical` → `high` → `medium` → `low` → `info`

### Secondary Sort (tie-breaking): Raw Severity (descending)

When two exposures share the same Adjusted Severity, the one with higher Raw Severity ranks first. This ensures that technically more severe exposures receive attention first within the same risk tier.

### Tertiary Sort (further ties): Exploitability (descending)

If Adjusted Severity and Raw Severity are both identical:
- `confirmed-in-wild` → `poc-available` → `theoretical`

### Final tie-breaker: Exposure ID (ascending)

For exposures that are identical across all three criteria, sort by Exposure ID in ascending order for deterministic ordering.

---

## References

1. FIRST. *Common Vulnerability Scoring System v3.1: Specification Document*. June 2019. https://www.first.org/cvss/v3.1/specification-document
2. NIST. *Guide for Conducting Risk Assessments*. SP 800-30 Rev. 1. September 2012. https://csrc.nist.gov/pubs/sp/800-30/r1/final
3. NIST. *Standards for Security Categorization of Federal Information and Information Systems*. FIPS PUB 199. February 2004. https://csrc.nist.gov/pubs/fips/199/final
4. Gartner, Inc. *Implement a Continuous Threat Exposure Management (CTEM) Program*. Research Note G00766755. July 2022.
5. CISA. *Known Exploited Vulnerabilities Catalog*. https://www.cisa.gov/known-exploited-vulnerabilities-catalog
