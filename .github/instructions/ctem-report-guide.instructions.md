---
description: "CTEM report generation guide. Use when: all five CTEM phases are complete, user requests a session report, or archiving a session. Read on demand — do NOT preload."
---

# CTEM Report Generation Guide

This document is read on demand by `/ctem-flow` when a session report is needed.

## Session Report

1. Copy `reports/sessions/TEMPLATE.md` to `reports/sessions/YYYY-MM-DD-<session-id>.md`
2. Fill in all sections from the session data in `ctem-state.md`
3. **Previous Session**: read `reports/assets/<asset>.md` → `Last Assessed` to find the previous Session ID. If the asset was only assessed in the current session (First Seen = Last Assessed), set Previous Session to `N/A`.
4. Confirm report content with the user before writing

## Asset Profile Updates

Asset profiles are **created** during Scoping (Step 2) with basic identity fields. The **Exposure Registry** (Raw Severity, exposure status) is written and maintained by Discovery (Phase 2). This section updates the **Risk Trend Log and Current Risk Summary** fields after all phases complete.

See the **Field Ownership Table** in `reports/README.md` for the definitive mapping of which phase writes which field.

For each in-scope asset after report generation:

1. If `reports/assets/<asset>.md` does NOT exist, create it from `reports/assets/TEMPLATE.md`
2. Update the following sections:
   - **Risk Trend Log**: append a new row for this session
   - **Current Risk Summary**: reflect latest assessment (Overall Risk Level, Open Exposures count, Highest Severity, Trend)
   - **Adjusted Severity** column in Exposure Registry: update with Prioritization's final severity (if not already set by Phase 3)
3. Confirm with the user before writing each asset file

## Post-Report

After report and asset profiles are confirmed:

- Reset `ctem-state.md` to the blank template (all phases `not_started`, logs cleared)
- Notify the user that the session is archived and a new session can begin
