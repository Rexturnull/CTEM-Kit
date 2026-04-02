# Business Criticality Assessment Matrix

This reference defines the framework for evaluating business criticality of a target host during CTEM Scoping (Step 3).

## Framework Basis

This matrix combines two NIST frameworks in a layered approach:

| Layer | Framework | Role |
|-------|-----------|------|
| **Input** — per-dimension impact rating | **FIPS 199** (NIST, 2004) §3 | Formal definitions of Low / Moderate / High impact for each CIA dimension |
| **Output** — overall criticality derivation | **NIST SP 800-30 Rev. 1** (NIST, 2012) Table H-3 | 5-level qualitative impact scale used to derive a single overall criticality level |

### Primary Sources

1. **FIPS 199** — *Standards for Security Categorization of Federal Information and Information Systems*
   - Publisher: NIST, February 2004
   - URL: https://csrc.nist.gov/pubs/fips/199/final
   - Cited sections: **§3** — definitions of potential impact for confidentiality, integrity, and availability

2. **NIST SP 800-30 Rev. 1** — *Guide for Conducting Risk Assessments*
   - Publisher: NIST, September 2012
   - URL: https://csrc.nist.gov/pubs/sp/800-30/r1/final
   - Cited sections: **Appendix H, Table H-3** — Assessment Scale: Level of Impact (5-level qualitative scale); **Appendix I, Table I-2** — examples of adverse impacts at each level

### Supporting Source

3. **NIST SP 800-60 Vol. 1 Rev. 1** — *Guide for Mapping Types of Information and Information Systems to Security Categories*
   - Publisher: NIST, August 2008
   - URL: https://csrc.nist.gov/pubs/sp/800-60/v1/r1/final
   - Role: Informs the operational examples in the CIA questionnaire (mapping information types to impact levels)

### CTEM Alignment

Gartner's CTEM framework requires scoping to start from **business outcomes and risk**, not from technical asset inventories (Gartner, 2022, G00766755). This matrix operationalizes that principle: every rating is anchored to business impact as formally defined by NIST.

---

## Step 1 — CIA Impact Rating (FIPS 199 §3)

Ask the user the following three questions in order. For each dimension, record the answer as **high**, **moderate**, or **low**.

The **FIPS 199 Definition** column contains the formal criterion from FIPS 199 §3 — this is the authoritative basis for each rating. The **Operational Examples** column provides practical guidance derived from NIST SP 800-60 Vol. 1 information-type mappings; they illustrate the definition but do not replace it.

### Question 1 — Confidentiality

> If this machine were compromised, what would be the impact of unauthorized disclosure of its information?

| Rating | FIPS 199 §3 Definition | Operational Examples |
|--------|----------------------|---------------------|
| **High** | Unauthorized disclosure could be expected to have a **severe or catastrophic adverse effect** on organizational operations, organizational assets, or individuals | Customer PII, financial records, trade secrets, authentication credentials, cryptographic keys |
| **Moderate** | Unauthorized disclosure could be expected to have a **serious adverse effect** on organizational operations, organizational assets, or individuals | Internal documents, operational data — no regulated or highly sensitive information |
| **Low** | Unauthorized disclosure could be expected to have a **limited adverse effect** on organizational operations, organizational assets, or individuals | Publicly available or non-sensitive information only |

### Question 2 — Integrity

> If the data on this machine were tampered with, what would the business impact be?

| Rating | FIPS 199 §3 Definition | Operational Examples |
|--------|----------------------|---------------------|
| **High** | Unauthorized modification or destruction could be expected to have a **severe or catastrophic adverse effect** on organizational operations, organizational assets, or individuals | Directly affects customer-facing services, financial accuracy, or safety-critical operations |
| **Moderate** | Unauthorized modification or destruction could be expected to have a **serious adverse effect** on organizational operations, organizational assets, or individuals | Affects internal decision-making or reporting; detectable and correctable after the fact |
| **Low** | Unauthorized modification or destruction could be expected to have a **limited adverse effect** on organizational operations, organizational assets, or individuals | Minimal impact; easy to detect and recover from |

### Question 3 — Availability

> If this machine went offline for 24 hours, what would the business impact be?

| Rating | FIPS 199 §3 Definition | Operational Examples |
|--------|----------------------|---------------------|
| **High** | Disruption of access to or use of information or the system could be expected to have a **severe or catastrophic adverse effect** on organizational operations, organizational assets, or individuals | Core business operations fully disrupted; revenue loss or SLA breach |
| **Moderate** | Disruption of access to or use of information or the system could be expected to have a **serious adverse effect** on organizational operations, organizational assets, or individuals | Some services degraded but workarounds exist; no immediate revenue impact |
| **Low** | Disruption of access to or use of information or the system could be expected to have a **limited adverse effect** on organizational operations, organizational assets, or individuals | Negligible effect on daily operations |

---

## Step 2 — Derive Overall Criticality (SP 800-30 Rev. 1, Table H-3)

Map the three CIA ratings from Step 1 to a single **Business Criticality** level using the qualitative impact scale from NIST SP 800-30 Rev. 1, Appendix H, Table H-3.

### Derivation Rules

| Rule | Condition | Criticality | SP 800-30 Table H-3 Basis |
|------|-----------|-------------|---------------------------|
| **R1** | ≥ 2 CIA dimensions rated **High** | **Critical** | Multiple severe/catastrophic adverse effects → **Very High** impact |
| **R1-Q** | Exactly 1 CIA dimension rated **High** AND any R1 Qualifier met (see below) | **Critical** | Primary-mission failure, major financial loss, or severe harm to individuals → **Very High** impact |
| **R2** | Exactly 1 CIA dimension rated **High**, no R1 Qualifier met | **High** | Severe or catastrophic adverse effect → **High** impact |
| **R3** | Highest CIA dimension = **Moderate** | **Moderate** | Serious adverse effect → **Moderate** impact |
| **R4** | Highest CIA dimension = **Low** | **Low** | Limited adverse effect → **Low** impact |

### R1 Qualifiers — When a Single "High" Escalates to "Critical"

When exactly one CIA dimension is rated High, apply these qualifiers to determine whether the impact reaches SP 800-30's **Very High** threshold. If **any** qualifier is true, apply R1-Q (Critical).

| # | Qualifier | SP 800-30 Table I-2 Basis |
|---|-----------|---------------------------|
| Q1 | Compromise renders the organization **unable to perform one or more of its primary mission/business functions** | "inability to perform current missions/business functions" |
| Q2 | Compromise results in **major financial loss** that threatens organizational viability | "major financial loss" |
| Q3 | Compromise causes **severe or catastrophic harm to individuals** (e.g., loss of life, serious physical injury) | "severe or catastrophic harm to individuals" |

> **Note**: If none of Q1–Q3 clearly applies, default to R2 (High). When in doubt, do not escalate — downstream CTEM phases (Prioritization, Validation) provide further risk calibration.

### Criticality Level Summary

| Level | SP 800-30 Impact Level | Definition | Typical Examples |
|-------|----------------------|------------|------------------|
| **Critical** | Very High | Compromise produces multiple severe/catastrophic adverse effects; organization unable to perform primary missions | Customer-facing production servers, AD Domain Controllers, core databases, payment gateways |
| **High** | High | Compromise produces a severe or catastrophic adverse effect; significant degradation of mission capability | Internal key services, backup infrastructure, CI/CD pipelines |
| **Moderate** | Moderate | Compromise produces a serious adverse effect; noticeable degradation but core missions unaffected | Development environments, monitoring systems, internal tools |
| **Low** | Low | Compromise produces a limited adverse effect; minimal operational impact | Test machines, sandboxes, decommissioned-but-online services |

## Recording the Result

Write the following into the asset profile (`reports/assets/<id>.md`):

- **Business Criticality** field → the final level (`critical` / `high` / `moderate` / `low`)
- **Owner / Team** field → if provided during the conversation

Record per-dimension ratings and the applied derivation rule as an HTML comment in the asset file's `## Notes` section for full traceability:

```
<!-- CIA Assessment: C=<rating>, I=<rating>, A=<rating> | Rule: <R1|R1-Q|R2|R3|R4> | Criticality: <level> (<qualifier or justification>) -->
```

Examples:
```
<!-- CIA Assessment: C=high, I=high, A=high | Rule: R1 (3 dims High) | Criticality: critical -->
<!-- CIA Assessment: C=high, I=moderate, A=low | Rule: R1-Q (Q1: primary mission failure) | Criticality: critical -->
<!-- CIA Assessment: C=high, I=moderate, A=moderate | Rule: R2 | Criticality: high -->
<!-- CIA Assessment: C=moderate, I=low, A=moderate | Rule: R3 | Criticality: moderate -->
<!-- CIA Assessment: C=low, I=low, A=low | Rule: R4 | Criticality: low -->
```

---

## References

1. National Institute of Standards and Technology. *Standards for Security Categorization of Federal Information and Information Systems*. FIPS PUB 199. February 2004. Available: https://csrc.nist.gov/pubs/fips/199/final

2. National Institute of Standards and Technology. *Guide for Conducting Risk Assessments*. NIST Special Publication 800-30 Revision 1. September 2012. Available: https://csrc.nist.gov/pubs/sp/800-30/r1/final

3. National Institute of Standards and Technology. *Guide for Mapping Types of Information and Information Systems to Security Categories*. NIST Special Publication 800-60 Volume 1 Revision 1. August 2008. Available: https://csrc.nist.gov/pubs/sp/800-60/v1/r1/final

4. Gartner, Inc. *Implement a Continuous Threat Exposure Management (CTEM) Program*. Research Note G00766755. July 2022.
