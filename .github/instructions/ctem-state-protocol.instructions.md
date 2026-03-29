---
description: "CTEM state management protocol. Use when: reading or writing ctem-state.md, tracking phase progress, performing backtrack checks, managing phase transitions."
applyTo: "ctem-state.md"
---

# CTEM State Protocol

This protocol governs how all CTEM skills and agents interact with `ctem-state.md`.

## Before Starting Any Phase

1. Read `ctem-state.md` from the project root
2. Confirm the current phase and verify that prerequisites are met
3. If prerequisites are NOT met, STOP and report what is missing

## After Completing Any Phase

1. Update the Phase Status Table in `ctem-state.md`:
   - Set the completed phase status to `completed`
   - Record timestamp and key findings summary
2. Append an entry to the Transition Log
3. Perform the **Backtrack Check** (see below)

## Backtrack Check

After every phase completion, compare current findings against previous phase outputs recorded in `ctem-state.md`:

| Condition | Action |
|-----------|--------|
| New assets discovered that were NOT in Scoping | Recommend backtrack to **Scoping** |
| New exposures found that were NOT in Discovery | Recommend backtrack to **Discovery** |
| Risk profile significantly changed (severity upgrade) | Recommend backtrack to **Prioritization** |
| Validation findings are INCONCLUSIVE | Recommend re-running **Validation** (max 2 retries) |
| No new findings and results are conclusive | Recommend proceeding to the next phase |

## Backtrack Rules

- Maximum **3 total backtracks** per session to prevent infinite loops
- Each backtrack MUST state: `FROM [phase] → TO [phase] | REASON: [explanation]`
- After backtrack, only re-run affected phases — do NOT restart the entire workflow
- Increment the Backtrack Count in `ctem-state.md`

## State File Format

`ctem-state.md` follows this exact structure (see the template in `ctem-state.md`).
