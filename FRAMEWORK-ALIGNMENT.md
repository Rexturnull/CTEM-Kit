# CTEM Framework Alignment

This document maps Gartner's CTEM (Continuous Threat Exposure Management) framework definitions to this toolkit's implementation, providing traceability for each phase.

## Source

Gartner, Inc. *Implement a Continuous Threat Exposure Management (CTEM) Program*. Research Note G00766755. July 2022.

---

## Phase Alignment

### Phase 1 — Scoping

| Aspect | Gartner Definition | This Implementation |
|--------|--------------------|---------------------|
| **Objective** | Define the scope of the attack surface based on business risk and strategic priorities, not just IT asset inventories | Anchors scope to business outcomes via Business Function and CIA-triad Business Criticality Assessment (FIPS 199 / NIST SP 800-30) |
| **Scope Categories** | External attack surface, SaaS posture, digital identity, digital supply chain, cloud configuration | **Single-host infrastructure layer only** — SaaS posture, digital identity, digital supply chain, and cloud-native configuration are explicitly out of scope (see Scope Limitation in SKILL.md) |
| **Business Context** | Scoping must start from business outcomes, not technical inventories | Step 0 (Business Context) collects Business Function, Regulatory Context, and Threat Profile before any technical data |
| **Output** | Defined attack surface with business-context labels | Scoping Summary in `ctem-state.md` + Asset Profile in `reports/assets/` |

### Phase 2 — Discovery

| Aspect | Gartner Definition | This Implementation |
|--------|--------------------|---------------------|
| **Objective** | Actively discover assets, vulnerabilities, misconfigurations, and other exposures within the defined scope | Systematic identification of exposures using nmap, Nuclei, and manual entry within Scoping boundaries |
| **Discovery Targets** | Vulnerabilities, misconfigurations, asset inventory gaps, configuration drift | Four exposure types: `vulnerability`, `misconfiguration`, `information-disclosure`, `outdated-software` |
| **Continuous Nature** | Discovery should be ongoing, not a point-in-time exercise | Cross-session comparison (new / known / escalated / reopened / mitigated) enables longitudinal tracking via Exposure Registry |
| **Tool Independence** | Framework is tool-agnostic | Supports nmap + Nuclei natively; manual entry path accepts findings from any tool |
| **Output** | Comprehensive exposure inventory | Discovery Summary in `ctem-state.md` + Exposure Registry in `reports/assets/` |

### Phase 3 — Prioritization

| Aspect | Gartner Definition | This Implementation |
|--------|--------------------|---------------------|
| **Objective** | Rank exposures not just by CVSS severity, but by business impact, exploitability context, and compensating controls | Three-layer assessment: Risk Matrix (Raw Severity × Business Criticality) → Contextual Adjustment (Exploitability ±1, Compensating Controls ±1, net capped at ±1) → Adjusted Severity per exposure |
| **Key Differentiator** | Goes beyond traditional vulnerability severity to include business context, threat intelligence, and asset criticality | Separates Raw Severity (Discovery) from Adjusted Severity (Prioritization); exploitability classified as `confirmed-in-wild` / `poc-available` / `theoretical`; 7-item structured compensating controls checklist with automatic control-to-exposure mapping |
| **Output** | Prioritized exposure list with risk scores | Prioritized Exposure List with Adjusted Severity, Exploitability, Controls Applied, and Rationale written to Asset Profile (`Adjusted Severity` + `Current Risk Summary`) and Prioritization Summary in `ctem-state.md` |

### Phase 4 — Validation

| Aspect | Gartner Definition | This Implementation |
|--------|--------------------|---------------------|
| **Objective** | Confirm exposures are exploitable, assess attack paths, filter false positives | Planned — three-module design: Attack Path Reasoning, Exploit Validation, Result Analysis |
| **Methods** | Breach and attack simulation, penetration testing, red team exercises | Design supports AI-guided validation procedure generation and result analysis |
| **Output** | Validated exposure list with confirmed/dismissed classifications | Planned — will update Current Status in Exposure Registry |

### Phase 5 — Mobilization

| Aspect | Gartner Definition | This Implementation |
|--------|--------------------|---------------------|
| **Objective** | Ensure findings are acted upon through cross-team remediation workflows | Planned — remediation plan generation, action assignment, timeline tracking |
| **Key Differentiator** | Not just creating tickets — ensuring organizational alignment and remediation follow-through | Design includes Remediation History tracking in Asset Profile |
| **Output** | Actionable remediation plans with assigned owners and deadlines | Planned — will write to Session Report and Asset Profile |

---

## Scope Decisions and Rationale

| Decision | Rationale |
|----------|-----------|
| Single-host focus (not network/subnet) | Provides depth over breadth; enables detailed per-asset longitudinal tracking. Multi-host support is a future extension. |
| Infrastructure layer only (no SaaS, identity, supply chain) | Aligns with the most common CTEM entry point for organizations beginning their exposure management program. |
| CVSS v3.1 as primary severity basis | Broadest tool support (nmap, Nuclei) and NVD coverage as of 2026. CVSS v4.0 scores are recorded when available but v3.1 is used for mapping. See CVSS version note in Discovery SKILL.md. |
| 4-level Business Criticality (no Very Low) | A host warranting CTEM assessment implies at least Low impact. See justification in `business-criticality-matrix.md`. |
| Raw vs Adjusted Severity separation | Maintains clear phase boundaries: Discovery reports tool findings objectively; Prioritization adds business context. See Field Ownership Table in `reports/README.md`. |
| Per-asset Exposure IDs | Simplest scheme for single-host sessions. Composite key (`ASSET-ID/EXP-ID`) recommended for multi-host extension. |
| `outdated-software` requires user confirmation | AI training cutoff limits version-recency knowledge; human-in-the-loop validation ensures accuracy. |

---

## References

1. Gartner, Inc. *Implement a Continuous Threat Exposure Management (CTEM) Program*. Research Note G00766755. July 2022.
2. NIST. *Standards for Security Categorization of Federal Information and Information Systems*. FIPS PUB 199. February 2004. https://csrc.nist.gov/pubs/fips/199/final
3. NIST. *Guide for Conducting Risk Assessments*. SP 800-30 Rev. 1. September 2012. https://csrc.nist.gov/pubs/sp/800-30/r1/final
4. NIST. *Guide for Mapping Types of Information and Information Systems to Security Categories*. SP 800-60 Vol. 1 Rev. 1. August 2008. https://csrc.nist.gov/pubs/sp/800-60/v1/r1/final
5. FIRST. *Common Vulnerability Scoring System v3.1: Specification Document*. June 2019. https://www.first.org/cvss/v3.1/specification-document
