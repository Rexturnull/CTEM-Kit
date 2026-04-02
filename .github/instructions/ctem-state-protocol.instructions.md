---
description: "CTEM state management protocol. Use when: reading or writing ctem-state.md, tracking phase progress, performing backtrack checks, managing phase transitions."
applyTo: "ctem-state.md"
---

# CTEM State Protocol

This protocol governs how all CTEM skills and prompts interact with `ctem-state.md`.

## Valid Phase Status Values

| Status | Meaning |
|--------|--------|
| `not_started` | Phase has not begun |
| `in_progress` | Phase is currently being executed |
| `completed` | Phase finished with all outputs recorded |

No other status values are permitted.

## Prerequisites

A phase's prerequisite is met when all preceding phases have status `completed`. Scoping has no prerequisite.

## Before Starting Any Phase

1. Read `ctem-state.md` from the project root
2. Confirm the current phase and verify that prerequisites are met (see above)
3. If prerequisites are NOT met, STOP and report what is missing
4. Set the phase's status to `in_progress` in the Phase Status table

## After Completing Any Phase

1. Update the Phase Status Table in `ctem-state.md`:
   - Set the completed phase status to `completed`
   - Record timestamp and key findings summary
2. Write (or update) the phase's Summary section under `## Phase Summaries` (see below)
3. Append an entry to the Transition Log
4. Backtrack checks and report generation are managed by `/ctem-flow` — this protocol does not perform them

## Phase Summary Sections

Each phase writes a structured summary into the `## Phase Summaries` area of `ctem-state.md` upon completion. This is the primary mechanism for inter-phase data handoff — downstream phases read these summaries instead of accessing multiple files.

Summary sections must be nested under `## Phase Summaries` and named `### Scoping Summary`, `### Discovery Summary`, `### Prioritization Summary`, `### Validation Summary`, `### Mobilization Summary`.

Each phase skill defines its own summary format. The summary must be concise, structured (use tables where appropriate), and contain only actionable data needed by downstream phases.

When a phase is re-run due to backtracking, its summary section must be **replaced** (not appended) with the updated results.

## State File Format

`ctem-state.md` follows this exact structure (see the template in `ctem-state.md`).
