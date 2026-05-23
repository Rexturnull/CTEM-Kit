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
| **Objective** | Confirm exposures are exploitable, assess attack paths, filter false positives | Three-module architecture (VRM / VGM / VPM) adapted from PentestGPT (Deng et al., 2024): validates each prioritized exposure via PoC-driven testing, classifies as `confirmed` / `false-positive` / `inconclusive`, discovers scanner blind-spot exposures via exploratory tasks, and chains confirmed exposures into attack paths |
| **Methods** | Breach and attack simulation, penetration testing, red team exercises | AI-guided validation using a Validation Testing Tree (VTT) — adapted from PentestGPT's PTT. VGM generates targeted validation commands (curl, nmap NSE, Nuclei PoC mode, gobuster, hydra, sqlmap); VPM parses results and provides validation judgments; VRM maintains the VTT and orchestrates the loop |
| **Output** | Validated exposure list with confirmed/dismissed classifications | Validation Summary in `ctem-state.md` (validation results, newly discovered exposures, attack paths, final VTT state) + updated Exposure Registry `Current Status` (`confirmed` / `false-positive`) and recalculated `Current Risk Summary` in `reports/assets/` |

### Phase 5 — Mobilization

| Aspect | Gartner Definition | This Implementation |
|--------|--------------------|---------------------|
| **Objective** | Ensure findings are acted upon through cross-team remediation workflows | Translates validated exposures into actionable remediation plans: Quick Fix + Action Type (`patch` / `configure` / `harden` / `replace` / `investigate`) + Effort estimate per exposure, with SLA-based timelines and attack path chain-breaker analysis |
| **Remediation Coverage** | Address all confirmed findings | Confirmed exposures → full Remediation Action; Inconclusive exposures → Investigation Action (re-validation guidance); False-positives → excluded |
| **Attack Path Strategy** | Understand attack chains to prioritize remediation effectively | Chain-Breaker Analysis: for each attack path, identifies the minimum-cost node whose remediation breaks the entire chain. Chain-breaker identity serves as a priority boost factor alongside Adjusted Severity |
| **Risk Acceptance** | Organizations may accept certain risks with proper governance | Structured risk acceptance workflow: requires Justification, Approver, and Review Date. High-severity acceptances trigger proactive warnings. Status transition: `confirmed → accepted`. Accepted exposures remain in risk calculation |
| **Timeline Management** | Remediation must have deadlines aligned with risk severity | Severity-based SLA defaults (critical=24h, high=7d, medium=30d, low=90d) with global and per-item override. Regulatory Context (PCI-DSS, GDPR, ISO 27001) noted for compliance but does not alter defaults |
| **Key Differentiator** | Not just creating tickets — ensuring organizational alignment and remediation follow-through | Hybrid interaction: auto-generates plans for user confirmation. Optional in-session quick-fix verification (user-triggered, `planned` as default). Remediation History tracking in Asset Profile enables cross-session longitudinal remediation tracking |
| **Output** | Actionable remediation plans with assigned owners and deadlines | Mobilization Summary in `ctem-state.md` (Remediation Actions, Attack Path Remediation, Risk Acceptance Log, Verification Results, Statistics) + updated Asset Profile (Remediation History, Current Status transitions, recalculated Current Risk Summary) |

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

## Adaptation Limitations

| Limitation | Description | Mitigation |
|-----------|-------------|------------|
| **Single-agent role-switching** | PentestGPT (Deng et al., 2024) uses three independent LLM sessions (ReasoningSession, GenerationSession, ParsingSession) to prevent context pollution between modules. This toolkit operates within a single VS Code Copilot agent, simulating the three modules via role-switching prompts within one context window. | The VTT provides a structured, persistent task tree that compensates for the lack of session isolation. Each module's prompt is clearly delineated, and the VTT serves as an explicit shared state that reduces implicit context dependency. |
| **No autonomous tool execution** | PentestGPT can invoke tools programmatically. This toolkit is prompt-driven — the user executes all commands manually and pastes results back. | The human-in-the-loop model aligns with Gartner's CTEM emphasis on organizational involvement and provides an additional safety layer for validation operations. |
| **Single-host scope** | Gartner CTEM encompasses multi-asset, enterprise-wide exposure management. This toolkit assesses one host per session. | Per-asset longitudinal tracking via Exposure Registry enables cross-session trend analysis. Multi-host extension is documented as a future enhancement. |

---

## Inter-Phase Data Flow

The following diagram shows the data dependencies between phases — what each phase reads and writes.

```
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                              ctem-state.md (Phase Summaries)                                 │
│  ┌──────────────┐  ┌──────────────────┐  ┌────────────────────┐  ┌───────────┐  ┌─────────┐ │
│  │   Scoping    │  │    Discovery     │  │  Prioritization    │  │Validation │  │Mobiliz. │ │
│  │   Summary    │  │    Summary       │  │  Summary           │  │ Summary   │  │ Summary │ │
│  └──────┬───────┘  └───────┬──────────┘  └─────────┬──────────┘  └─────┬─────┘  └────┬────┘ │
└─────────┼──────────────────┼───────────────────────┼───────────────────┼──────────────┼─────┘
          │                  │                       │                   │              │
  Phase 1 │ writes    Phase 2│ reads↑ writes    Phase 3│ reads↑ writes   Phase 4│ reads↑ writes  Phase 5│ reads↑ writes
  Scoping │           Discovery│                Prioritization│          Validation│        Mobilization│
          │                  │                       │                   │              │
┌─────────┼──────────────────┼───────────────────────┼───────────────────┼──────────────┼─────┐
│         ▼                  ▼                       ▼                   ▼              ▼     │
│              reports/assets/<host>.md (Asset Profile)                                      │
│  ┌────────────────┐  ┌───────────────────┐  ┌──────────────────┐  ┌────────────────────┐   │
│  │ Asset Identity │  │ Exposure Registry │  │Current Risk Sum. │  │Remediation History │   │
│  │  (Phase 1)     │  │  (Phase 2, 3, 4)  │  │  (Phase 3, 4, 5) │  │  (Phase 5)         │   │
│  └────────────────┘  └───────────────────┘  └──────────────────┘  └────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────────────────────┘

Data Flow:
  Phase 1 (Scoping)        ──writes──→  Scoping Summary + Asset Identity
  Phase 2 (Discovery)      ──reads───→  Scoping Summary
                           ──writes──→  Discovery Summary + Exposure Registry (Raw Severity, Status)
  Phase 3 (Prioritization) ──reads───→  Scoping Summary + Discovery Summary + Exposure Registry
                           ──writes──→  Prioritization Summary + Adjusted Severity + Current Risk Summary
  Phase 4 (Validation)     ──reads───→  All upstream summaries + Exposure Registry
                           ──writes──→  Validation Summary + Current Status (confirmed/false-positive)
                                        + Current Risk Summary (recalculated) + New Exposures
  Phase 5 (Mobilization)   ──reads───→  All upstream summaries + Exposure Registry + Remediation History
                           ──writes──→  Mobilization Summary + Remediation History
                                        + Current Status (mitigated/accepted) + Current Risk Summary (recalculated)
  Report Generation        ──reads───→  All summaries
                           ──writes──→  Session Report + Risk Trend Log
```

---

## References

1. Gartner, Inc. *Implement a Continuous Threat Exposure Management (CTEM) Program*. Research Note G00766755. July 2022.
2. NIST. *Standards for Security Categorization of Federal Information and Information Systems*. FIPS PUB 199. February 2004. https://csrc.nist.gov/pubs/fips/199/final
3. NIST. *Guide for Conducting Risk Assessments*. SP 800-30 Rev. 1. September 2012. https://csrc.nist.gov/pubs/sp/800-30/r1/final
4. NIST. *Guide for Mapping Types of Information and Information Systems to Security Categories*. SP 800-60 Vol. 1 Rev. 1. August 2008. https://csrc.nist.gov/pubs/sp/800-60/v1/r1/final
5. FIRST. *Common Vulnerability Scoring System v3.1: Specification Document*. June 2019. https://www.first.org/cvss/v3.1/specification-document
6. Deng, G., Liu, Y., Mayoral-Vilches, V., Liu, P., Li, Y., Xu, Y., Zhang, T., Liu, Y., Pinzger, M., & Rass, S. (2024). PentestGPT: An LLM-empowered Automatic Penetration Testing Tool. *arXiv preprint arXiv:2308.06782*. https://arxiv.org/abs/2308.06782
