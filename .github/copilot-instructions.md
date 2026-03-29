# CTEM Kit — Global Project Instructions

## Project Identity

This workspace is a **CTEM (Continuous Threat Exposure Management) Kit** — a prompt-driven automation toolkit for the Gartner CTEM five-phase framework. It contains NO code. All logic is expressed through Markdown-based instructions, skills, and prompt definitions.

## Activation Rule

The CTEM rules below apply **only when the user explicitly enters a CTEM context**:

- Invoking `/ctem-flow` or any `/ctem-*` skill
- Explicitly mentioning a CTEM session (e.g., "start new session", "resume", "phase complete")

**Non-CTEM requests are NOT subject to the rules below.** Respond with normal Copilot behavior.

## Global Rules (CTEM Mode)

1. **Workflow Ownership**: Phase sequencing, backtracking, and transition decisions are managed by `/ctem-flow`. Five-phase definitions, backtrack logic, and state update details live in `ctem-flow.prompt.md` — not repeated here.
2. **State File Governance**: Read/write rules for `ctem-state.md` are defined in `ctem-state-protocol.instructions.md` — not repeated here.
3. **Tool Integration**: Skills instruct the user to run external tools (nmap, nuclei, nessus, etc.) and parse pasted output. Skills do NOT execute tools directly.
4. **Language**: All prompt content is in English.
