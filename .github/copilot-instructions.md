# CTEM Kit — Global Project Instructions

## Project Identity

This workspace is a **CTEM (Continuous Threat Exposure Management) Kit** — a prompt-driven automation toolkit for the Gartner CTEM five-phase framework. It contains NO code. All logic is expressed through Markdown-based instructions, skills, and agent definitions.

## CTEM Five Phases (Strict Order)

| # | Phase | Skill | Purpose |
|---|-------|-------|---------|
| 1 | Scoping | `/ctem-scoping` | Define target scope, asset inventory, business criticality |
| 2 | Discovery | `/ctem-discovery` | Identify exposures: vulnerabilities, misconfigurations, attack surface gaps |
| 3 | Prioritization | `/ctem-prioritization` | Rank exposures by exploitability, business impact, and context |
| 4 | Validation | `/ctem-validation` | Verify exploitability, filter false positives, confirm attack paths |
| 5 | Mobilization | `/ctem-mobilization` | Generate remediation plans, assign actions, track resolution |

## Global Rules (Always Apply)

1. **Phase State Tracking**: All phase transitions MUST be recorded in `ctem-state.md` at the project root. Read it before starting any phase. Update it after completing any phase.
2. **Phase Independence**: Each skill handles ONLY its own phase. It does NOT manage workflow transitions or decide what comes next.
3. **Workflow Control**: Phase sequencing, backtracking, and transition decisions are handled EXCLUSIVELY by the `@ctem-coordinator` agent OR by the user manually.
4. **Backtrack Support**: The workflow supports non-linear phase transitions. Validation may trigger a return to Discovery. The coordinator manages this logic.
5. **Tool Integration**: Skills may instruct the user to run external security tools (nmap, nuclei, nessus, etc.) and will parse the pasted output. Skills do NOT execute tools directly.
6. **Language**: All prompt content is in English.
