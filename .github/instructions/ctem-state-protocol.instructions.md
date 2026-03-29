---
description: "CTEM state management protocol. Use when: reading or writing ctem-state.md, tracking phase progress, performing backtrack checks, managing phase transitions."
applyTo: "ctem-state.md"
---

# CTEM State Protocol

This protocol governs how all CTEM skills and prompts interact with `ctem-state.md`.

## Prerequisites

A phase's prerequisite is met when all preceding phases have status `completed`. Scoping has no prerequisite.

## Before Starting Any Phase

1. Read `ctem-state.md` from the project root
2. Confirm the current phase and verify that prerequisites are met (see above)
3. If prerequisites are NOT met, STOP and report what is missing

## After Completing Any Phase

1. Update the Phase Status Table in `ctem-state.md`:
   - Set the completed phase status to `completed`
   - Record timestamp and key findings summary
2. Append an entry to the Transition Log
3. Backtrack checks and report generation are managed by `/ctem-flow` — this protocol does not perform them

## State File Format

`ctem-state.md` follows this exact structure (see the template in `ctem-state.md`).
