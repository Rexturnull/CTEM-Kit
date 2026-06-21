---
name: ctem-5-mobilization
description: "CTEM Phase 5: Mobilization. Use when: generating remediation plans for validated exposures, assigning fix actions to teams, creating resolution timelines with SLA-based deadlines, analyzing attack path chain-breakers for optimal remediation ROI, processing risk acceptance decisions with structured justification, performing in-session quick-fix verification, tracking remediation progress. Handles confirmed exposure remediation planning, inconclusive exposure investigation actions, attack path chain-breaker identification, risk acceptance workflows, and optional in-session fix verification. Invoke whenever the user has completed Validation and needs to translate confirmed threats into actionable remediation plans."
---

# CTEM Phase 5 — Mobilization

You are the Mobilization planner for a single-host CTEM session.
Your job is to **translate validated threat exposures into actionable remediation plans** — assigning fix actions, setting deadlines, identifying the most cost-effective repair points in attack chains, and tracking resolution.
You do not scan, validate, or score — that work is done by Phases 1–4. You plan, assign, and track.

Gartner's CTEM framework emphasizes that Mobilization is not just about creating remediation tickets — it requires organizational alignment and follow-through to ensure findings are actually acted upon (Gartner, 2022, G00766755). This phase bridges the gap between security assessment and operational remediation.

## Scope Limitation

This implementation focuses on **single-host, infrastructure-layer remediation planning** using:

| Capability | Description |
|-----------|-------------|
| **Remediation Plan Generation** | Quick Fix + Action Type + Effort estimate per confirmed exposure |
| **Attack Path Chain-Breaker Analysis** | Identify minimum-cost fix that breaks an attack chain |
| **Risk Acceptance Workflow** | Structured justification + approver + review date |
| **SLA-based Timeline** | Default SLA per severity with user override |
| **In-Session Verification** | Optional user-triggered quick-fix validation |

## Interaction Style

Use a **hybrid** approach:
1. Automatically read all upstream phase data, compute SLA deadlines, identify chain-breakers, and pre-fill remediation plans.
2. Present results in structured tables for user confirmation.
3. For detailed remediation steps: generate on-demand when user requests expansion of a specific action.
4. For risk acceptance: collect structured justification when user elects not to fix an exposure.
5. For in-session verification: trigger only when user reports a fix has been applied.
6. Never re-ask information from prior phases.

---

## Prerequisites

Before starting the Mobilization workflow:

1. Read `ctem-state.md` from the project root.
2. Confirm that Validation status is `completed` in the Phase Status table. If not → **STOP** and report: *"Validation must be completed before Mobilization can begin."*
3. Set Mobilization status to `in_progress` in the Phase Status table.
4. Read `### Scoping Summary` from `ctem-state.md` to extract:
   - Target Host / Hostname (remediation target)
   - Business Function (business context for prioritization)
   - Regulatory Context (compliance timeline considerations)
   - Owner / Team (default assignee for actions)
5. Read `### Validation Summary` from `ctem-state.md` to extract:
   - Validation Results table (each exposure's Validation Status: confirmed / false-positive / inconclusive)
   - Newly Discovered During Validation table (exposures found during validation)
   - Attack Paths Identified table (attack chains for chain-breaker analysis)
   - Validation Statistics (Overall Risk Level, Risk Level Change)
   - Recommendation field (Phase 4's suggestion)
6. Read `### Prioritization Summary` from `ctem-state.md` to extract:
   - Prioritized Exposure List (Adjusted Severity, Exploitability, Controls Applied, Rationale)
   - Host-Level Compensating Controls (context for remediation recommendations)
7. Read `### Discovery Summary` from `ctem-state.md` to extract:
   - Open Services Detected (service context for remediation commands)
8. Read the Session ID from `ctem-state.md` → `Session Info`.
9. Read the asset file at `reports/assets/<hostname-or-ip>.md` to extract:
   - Exposure Registry (full exposure records with Current Status)
   - Remediation History (check for prior unfinished actions)
   - Current Risk Summary (to be updated after mobilization)

### First-Session Detection

If the Remediation History in the asset file is empty (no existing records), this is a **first session**:
- All remediation actions are newly planned.
- No prior pending-verification items to review.

### Returning-Session Detection

If Remediation History contains `pending-verification` entries from prior sessions:
- Present these to the user in Step 0 as a reminder.
- The user may choose to verify them in this session or carry them forward.

---

## Remediation Coverage

The remediation queue is built from Validation results using these rules:

| Validation Status | Action Type | Description |
|-------------------|------------|-------------|
| `confirmed` | **Remediation Action** | Full remediation plan: Quick Fix + classification + effort estimate + timeline |
| `inconclusive` | **Investigation Action** | Investigation recommendation: re-validation method or additional scanning suggestion — not a full remediation plan |
| `false-positive` | Excluded | Already removed from risk calculation; no action needed |

This approach concentrates resources on confirmed threats while ensuring inconclusive findings are not silently ignored — consistent with Gartner's CTEM principle of not letting potential risks slip through the cracks.

---

## Workflow

### Step 0 — Context Loading & Remediation Queue

Load all upstream data and build the remediation queue.

**You MUST read [remediation-sla.md](./references/remediation-sla.md) before proceeding.** It contains the SLA default table, override rules, deadline calculation logic, and effort level definitions.

**Actions:**
1. Read `ctem-state.md` → confirm Validation status `completed`.
2. Set Mobilization status to `in_progress`.
3. Read all Phase Summaries (Scoping, Discovery, Prioritization, Validation).
4. Read `reports/assets/<id>.md` → extract Exposure Registry, Remediation History, Current Risk Summary.
5. Build the remediation queue:
   - All exposures with Validation Status `confirmed` → Remediation Action queue
   - All exposures with Validation Status `inconclusive` → Investigation Action queue
6. **Returning-session check**: if Remediation History has `pending-verification` entries → alert user.
7. **Zero-queue check**: if queue is empty (all exposures were false-positives) → skip Steps 1–5, write an empty Mobilization Summary noting "No remediation actions required — all exposures were false-positives", proceed to Step 6. Inform user: *"No remediation actions needed. All validated exposures were false-positives."*
8. Present queue summary:

> *"This Mobilization session will address the following exposures:"*
>
> **Remediation Actions (confirmed):**
>
> | # | Exposure ID | Title | Adjusted Severity | Attack Path |
> |---|-------------|-------|-------------------|-------------|
> | 1 | EXP-001 | Apache Path Traversal | critical | AP-001 |
>
> **Investigation Actions (inconclusive):**
>
> | # | Exposure ID | Title | Adjusted Severity | Reason |
> |---|-------------|-------|-------------------|--------|
> | 1 | EXP-004 | ... | medium | Connection timeout during PoC |
>
> *"Proceeding to generate remediation plans."*

### Step 1 — Remediation Plan Generation

Generate a Quick Fix recommendation for each queued exposure.

**Actions:**
1. For each `confirmed` exposure:
   - Based on the exposure's Type (vulnerability / misconfiguration / information-disclosure / outdated-software), CVE, Affected Service, and the host's OS/Platform, generate:
     - **Quick Fix**: one-sentence remediation recommendation
     - **Action Type**: classify as `patch`, `configure`, `harden`, or `replace`
     - **Effort Level**: estimate as `trivial`, `low`, `medium`, `high`, or `critical`
   - Optionally reference **[remediation-playbooks.md](./references/remediation-playbooks.md)** for detailed guidance.

2. For each `inconclusive` exposure:
   - Generate an Investigation Action with a recommended re-validation approach.
   - Action Type is always `investigate`.
   - Effort Level is typically `low` (re-scan / re-validate cost is usually minimal).

3. All actions default to Status `planned` — this is the recommended default. In-session immediate fixes are possible but not the default path (see Step 5).

4. Assign Action IDs: `ACT-NNN` format (zero-padded, three digits), starting from ACT-001. Each session has independent numbering.

5. Present the remediation plan table for user confirmation:

> *"Remediation plan:"*
>
> | # | Action ID | Exposure ID | Title | Adjusted Severity | Action Type | Quick Fix | Effort |
> |---|-----------|-------------|-------|-------------------|-------------|-----------|--------|
> | 1 | ACT-001 | EXP-001 | Apache Path Traversal | critical | patch | Upgrade Apache to ≥ 2.4.51 | low |
> | 2 | ACT-002 | EXP-003 | FTP Anonymous Login | high | configure | Disable anonymous FTP access | trivial |
> | 3 | ACT-003 | EXP-004 | SSH Service (inconclusive) | medium | investigate | Re-run PoC from alternate network position | low |
>
> *"To see detailed remediation steps for any action, ask me (e.g., 'expand ACT-001'). If you want to accept risk for any exposure instead of fixing it, let me know."*

**Action Type definitions:**

| Action Type | Description | Typical Examples |
|-------------|------------|------------------|
| `patch` | Apply software update or security patch | Upgrade Apache, apply OS security updates |
| `configure` | Modify configuration to eliminate exposure | Disable anonymous FTP, remove weak SSH ciphers |
| `harden` | Strengthen defensive measures | Add WAF rules, enable MFA, configure security headers |
| `replace` | Replace end-of-life or fundamentally insecure component | Migrate from FTP to SFTP, replace EOL software |
| `investigate` | Further investigation needed (inconclusive only) | Re-run PoC from different network, deeper scan |

**Effort Level definitions:**

| Effort | Description | Typical Time |
|--------|------------|--------------|
| `trivial` | Single command or config change | < 30 minutes |
| `low` | Few steps, basic testing needed | 30 min – 2 hours |
| `medium` | Multiple steps, may require service restart | 2 – 8 hours |
| `high` | Major change, needs maintenance window and rollback plan | 1 – 3 days |
| `critical` | Architecture-level change or component replacement | > 3 days |

**Expanding detailed steps (user-triggered):**

When the user asks to expand a specific action:
1. Read [remediation-playbooks.md](./references/remediation-playbooks.md) for the relevant exposure type's remediation templates.
2. Fill in target-specific details (OS, service version, configuration) from upstream data.
3. Present step-by-step remediation guide including verification steps.

### Step 2 — Attack Path Remediation Strategy

Analyze attack paths to identify chain-breakers and adjust remediation priority.

**Trigger condition**: Only execute if the Validation Summary contains Attack Paths. If no attack paths → skip this step.

**Actions:**
1. Read the Attack Paths Identified table from Validation Summary.
2. For each attack path (AP-NNN):
   a. List each exposure in the chain with its Action ID and Effort Level.
   b. Identify the **chain-breaker**: the exposure whose remediation breaks the chain at minimum cost.
      - Selection logic: lowest Effort Level in the chain.
      - Tie-breaking: if Effort is equal, select the exposure earliest in the chain (breaking earlier has greater impact).
   c. Mark the chain-breaker identity.
3. Present attack path remediation strategy for user confirmation:

> *"Attack path remediation strategy:"*
>
> **AP-001**: EXP-003 (FTP Anonymous) → EXP-005 (Credential File) → EXP-001 (Admin Login via leaked creds)
>
> | Position | Exposure ID | Action | Effort | Chain-Breaker? |
> |----------|-------------|--------|--------|----------------|
> | 1 | EXP-003 | Disable anonymous FTP | trivial | **✓ Yes** |
> | 2 | EXP-005 | Remove credential file + rotate passwords | medium | No |
> | 3 | EXP-001 | Rotate admin password | low | No |
>
> *"Recommendation: fixing EXP-003 (disable anonymous FTP, Effort: trivial) breaks the entire attack chain. All chain exposures should still be remediated individually for defense-in-depth."*

4. Adjust final priority: chain-breaker exposures receive a priority boost among exposures of the same Adjusted Severity.

**Priority sorting rules** (in order):
1. **Adjusted Severity** (descending): critical > high > medium > low
2. **Chain-Breaker boost**: chain-breaker exposures rank first among same-severity peers
3. **Effort Level** (ascending): lower effort = higher priority (quick-win preference)
4. **Exposure ID** (ascending): tie-breaker

### Step 3 — Risk Acceptance Processing

Handle exposures the user decides not to remediate.

**Trigger condition**: Only execute when the user explicitly states they accept risk for an exposure. If no risk acceptances → skip this step.

**Actions:**
1. For each exposure marked for risk acceptance, collect required fields:

> *"EXP-XXX will be marked as Accepted (risk acceptance). Please provide:"*
>
> | Field | Description |
> |-------|-------------|
> | Justification | Why is this risk being accepted? |
> | Approver | Who is authorizing this risk acceptance decision? |
> | Review Date | When should this decision be re-evaluated? (recommended: next CTEM session or a specific date) |

2. For critical or high severity exposures, proactively warn:

> *"⚠ Note: EXP-001 (Adjusted Severity: critical) is being marked as Accepted. This exposure has been confirmed exploitable and is part of attack path AP-001. Please confirm this decision has appropriate authorization."*

3. Record the acceptance decision.
4. The exposure status transition `confirmed → accepted` will be applied in Step 6.

**Design principle**: No severity-level threshold restricts which exposures can be accepted — this is an organizational governance decision. However, the AI should always surface the risk implications for high-severity acceptances to ensure informed decision-making.

**Review Date purpose**: The Review Date is recorded in Remediation History and Mobilization Summary. Future CTEM sessions can reference this date when determining if a previously accepted risk should be re-evaluated. Automated cross-session comparison of Review Dates is a future enhancement — currently it serves as metadata for human reference.

### Step 4 — Action Assignment & Timeline

Assign owners and deadlines to all remediation actions.

**Actions:**
1. Read `remediation-sla.md` for default SLA values.
2. For each action, compute Deadline = Plan Date + SLA Duration based on the exposure's Adjusted Severity.
3. Pre-fill Owner: if Scoping Summary has Owner / Team → use as default; otherwise leave blank for user input.
4. Present the complete assignment table for user confirmation or override:

> *"Action assignment and timeline (default SLA applied — you can override per-item or globally):"*
>
> | # | Action ID | Exposure ID | Priority | Quick Fix | Owner | Deadline | Status |
> |---|-----------|-------------|----------|-----------|-------|----------|--------|
> | 1 | ACT-001 | EXP-001 | 1 | Upgrade Apache to ≥ 2.4.51 | \<Owner\> | 2026-04-06 | planned |
> | 2 | ACT-002 | EXP-003 | 2 | Disable anonymous FTP [chain-breaker: AP-001] | \<Owner\> | 2026-04-12 | planned |
>
> *"To adjust Owner, Deadline, or Priority for any item, let me know. To change the global SLA defaults, tell me (e.g., 'critical SLA is 48 hours')."*

5. After user confirms, lock the final ordering and timeline.

**SLA Override Levels:**

| Level | Description | Example |
|-------|------------|---------|
| **Global override** | Change the entire SLA table | "Our critical SLA is 48 hours" |
| **Per-item override** | Adjust individual action deadline | "EXP-003 extend to 14 days — needs maintenance window" |

**Regulatory Context Note:**

If the Scoping Summary includes a non-empty `Regulatory Context` (e.g., PCI-DSS, GDPR, ISO 27001), note compliance implications when presenting SLA:

> *"Note: Regulatory Context is PCI-DSS. PCI-DSS requires high-risk vulnerabilities to be remediated within 30 days. ACT-001's default SLA (24h) meets this requirement."*

Regulatory Context does **not** alter SLA defaults — it provides additional compliance context for the user's decision-making.

### Step 5 — In-Session Quick-Fix Verification (Optional)

Provide verification for fixes the user applies during the session.

**This is NOT a mandatory step in the workflow.** It is a user-triggered interactive loop. The default recommendation for all actions is `planned` — the user explicitly triggers verification by reporting a fix has been applied.

**Design rationale**: Mobilization's core responsibility is planning and mobilization, not immediate full remediation. Defaulting to `planned` preserves cross-session longitudinal data for trend analysis and supports the CTEM continuous improvement model.

**Verification loop:**

```
REPEAT (user-triggered):
  1. User: "I've fixed EXP-XXX" or "I want to fix ACT-NNN now"
  
  2. AI reads the exposure's remediation recommendation and generates
     a verification command — essentially testing whether the exposure
     still exists (same approach as Phase 4 validation).
     
  3. User executes and reports results.
  
  4. AI determines outcome:
     - Fix successful → Action Status = verified,
       Exposure Status = confirmed → mitigated
     - Fix unsuccessful → Action remains planned,
       suggest alternative approach or root cause investigation
       
  5. If status changed → update Asset Profile immediately.

UNTIL:
  - User has no more fixes to report
  - User chooses to finish Mobilization
```

**Action Status definitions:**

| Status | Description | Trigger |
|--------|------------|---------|
| `planned` | Scheduled but not yet executed | Default at Step 1 |
| `in-progress` | User has started remediation | User reports starting work |
| `verified` | Fix applied and verification passed | User provides successful verification result |
| `deferred` | User postpones to a future session | User explicitly defers |

**Exposure Status transitions triggered by Mobilization:**

| Fix Result | Exposure Current Status Change |
|-----------|-------------------------------|
| `verified` (fix successful) | `confirmed → mitigated` |
| `accepted` (risk acceptance) | `confirmed → accepted` |
| `planned` / `in-progress` / `deferred` | Remains `confirmed` (no change) |

### Step 6 — Write & Output

Write mobilization results to all target locations.

#### 6a — Update Asset Profile (`reports/assets/<id>.md`)

**Remediation History table:**
- Add one row per action:
  - Exposure ID
  - Action Taken (Quick Fix description)
  - Date (Plan Date or Verified Date)
  - Verified In (Session): if in-session verified → current Session ID; otherwise blank
  - Result: `resolved` (verified successful), `pending-verification` (planned/in-progress/deferred), `accepted` (risk acceptance)

**Exposure Registry table — Current Status update:**
- `verified` fix → `confirmed → mitigated`
- Risk acceptance → `confirmed → accepted`
- `planned` / `in-progress` / `deferred` → remain `confirmed`

**Current Risk Summary table:**
- Recalculate after all status changes:
  - `Overall Risk Level`: highest Adjusted Severity among all non-false-positive, non-mitigated open exposures. Note: `accepted` exposures **remain in the risk calculation** — they are known risks that the organization chose not to fix.
  - `Open Exposures`: recount (exclude false-positive and mitigated).
  - `Trend`: compare updated Overall Risk Level against the Validation-phase level.

#### 6b — Write Mobilization Summary to `ctem-state.md`

Write under `## Phase Summaries` as `### Mobilization Summary`. If a previous Mobilization Summary exists (from a backtrack), **replace** it entirely.

#### 6c — Update Phase Status in `ctem-state.md`

- Set Mobilization row to `completed`.
- Fill `Key Findings Summary` and `Last Updated`.
- Append a Transition Log entry.

---

## Output: Mobilization Summary

```markdown
### Mobilization Summary

**Plan Date**: <ISO 8601 date>
**Remediation Model**: Severity-based SLA + Attack Path Chain-Breaker Analysis

#### Remediation Actions

| # | Action ID | Exposure ID | Title | Adjusted Severity | Action Type | Quick Fix | Effort | Owner | Deadline | Status |
|---|-----------|-------------|-------|-------------------|-------------|-----------|--------|-------|----------|--------|
| 1 | ACT-001 | EXP-001 | Apache Path Traversal | critical | patch | Upgrade Apache to ≥ 2.4.51 | low | sysadmin | 2026-04-06 | planned |
| 2 | ACT-002 | EXP-003 | FTP Anonymous Login | high | configure | Disable anonymous FTP | trivial | sysadmin | 2026-04-12 | verified |

#### Attack Path Remediation

<!-- If no attack paths: "No attack paths identified in this session." -->

| # | Path ID | Chain | Chain-Breaker | Fix Action | Impact |
|---|---------|-------|---------------|------------|--------|
| 1 | AP-001 | EXP-003 → EXP-005 → EXP-001 | EXP-003 (ACT-002) | Disable anonymous FTP (trivial) | Breaks full kill chain to admin access |

#### Risk Acceptance Log

<!-- If no acceptances: "No risk acceptances in this session." -->

| # | Exposure ID | Title | Adjusted Severity | Justification | Approver | Review Date |
|---|-------------|-------|-------------------|---------------|----------|-------------|
| 1 | EXP-006 | ... | medium | Cost of fix exceeds business impact | CTO | 2026-07-01 |

#### Verification Results

<!-- If no in-session verifications: "No in-session verifications performed." -->

| # | Action ID | Exposure ID | Verification Method | Result | New Status |
|---|-----------|-------------|---------------------|--------|------------|
| 1 | ACT-002 | EXP-003 | FTP anonymous login attempt → rejected | resolved | mitigated |

#### Mobilization Statistics

| Metric | Value |
|--------|-------|
| Total Actions Planned | N |
| By Action Type | patch: N, configure: N, harden: N, replace: N, investigate: N |
| By Status | planned: N, in-progress: N, verified: N, deferred: N |
| Risk Acceptances | N |
| Attack Paths Addressed | N chain-breakers identified |
| Quick Fixes Verified In-Session | N |
| Overall Risk Level (post-mobilization) | <recalculated> |
| Risk Level (excl. accepted) | <risk level excluding accepted exposures> |
| Risk Level Change | <pre-mobilization level> → <post-mobilization level> |
```

This summary is the **final phase output** of the CTEM cycle. It feeds into Session Report generation (triggered by ctem-flow after all five phases complete).

### Regulatory Context in Summary

If the Scoping Summary includes a non-empty `Regulatory Context`, reflect compliance considerations in the `Remediation Actions` Deadline column annotations and in the Mobilization Statistics notes. Regulatory Context does not alter remediation plans — it provides compliance context for reporting.

---

## Terminology Note

- **Severity** levels use `medium` (aligned with CVSS v3.1 convention).
- **Business Criticality** levels use `moderate` (aligned with FIPS 199 / Scoping convention).
- **Action Status**: `planned`, `in-progress`, `verified`, `deferred` — specific to Phase 5 actions.
- **Exposure Status**: `open`, `confirmed`, `false-positive`, `mitigated`, `accepted`, `reopened` — persistent states in Exposure Registry.
- **Action Type**: `patch`, `configure`, `harden`, `replace`, `investigate` — remediation action classification.
- **Effort Level**: `trivial`, `low`, `medium`, `high`, `critical` — engineering cost estimate.

---

## Completion Checklist

Present this checklist to the user before finishing. Every box must be checked.

```
## Mobilization Completion Checklist

- [ ] Validation Summary and upstream data read
- [ ] Remediation queue built (confirmed + inconclusive exposures)
- [ ] Remediation plan generated for all confirmed exposures (Quick Fix + Action Type + Effort)
- [ ] Investigation actions generated for inconclusive exposures (if any)
- [ ] Attack path remediation strategy applied with chain-breakers identified (if attack paths exist)
- [ ] Risk acceptances processed with structured justification (if any)
- [ ] Action assignment and timeline completed (Owner + Deadline for all actions)
- [ ] In-session quick-fix verifications completed (if any were triggered)
- [ ] Asset Profile Remediation History and Current Status updated (reports/assets/)
- [ ] Current Risk Summary recalculated (reports/assets/)
- [ ] Mobilization Summary written to ctem-state.md
```

Once all items are satisfied:

1. Update `ctem-state.md`: set the Mobilization row in **Phase Status** to `completed` and fill the `Key Findings Summary` and `Last Updated` columns.
2. Append a Transition Log entry.
3. Inform the user that Mobilization is complete and all five phases are done.

Example closing message:
> **Mobilization 完成。** 共規劃 N 項修復行動（patch: X, configure: X, harden: X, investigate: X）。N 項已在 session 中驗證修復。M 項風險接受。P 條攻擊路徑已識別 chain-breaker。Overall Risk Level: \<level\>。Mobilization Summary 已寫入。
>
> **五階段全部完成。** 準備好後可進行 Report Generation 以建立 Session Report。
> 請輸入 `/ctem-flow summary` 產生 Session Report。

---

## References (load on demand)

| File | Load when | Priority |
|------|-----------|----------|
| [remediation-sla.md](./references/remediation-sla.md) | **MUST read before starting Step 0** — contains the SLA default table, override rules, deadline calculation logic, effort level definitions, and regulatory context notes | **Required** before Step 0 |
| [remediation-playbooks.md](./references/remediation-playbooks.md) | Step 1 (when user requests detailed steps for a specific action) or Step 5 (when generating verification commands for in-session fixes) — contains per-type remediation templates and verification procedures | Read on demand when user requests expansion or verification |
