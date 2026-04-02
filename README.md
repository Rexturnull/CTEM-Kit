# CTEM Kit

A prompt-engineering-only toolkit that automates the **Gartner CTEM (Continuous Threat Exposure Management)** five-phase workflow using AI skills and prompts in GitHub Copilot.

**Zero code. Pure Markdown. Fully AI-driven.**

## What is CTEM?

CTEM (Continuous Threat Exposure Management) is a five-phase framework introduced by Gartner for proactively managing an organization's threat exposure:

1. **Scoping** — Define what's in scope: assets, business criticality, attack surface boundaries
2. **Discovery** — Find exposures: vulnerabilities, misconfigurations, attack surface gaps
3. **Prioritization** — Rank exposures by risk: exploitability × business impact × context
4. **Validation** — Verify exposures are real and exploitable, filter false positives
5. **Mobilization** — Generate remediation plans, assign actions, track resolution

## Architecture: Prompt + Skills + Data

| Layer | Component | Purpose |
|-------|-----------|---------|
| Global Rules | `copilot-instructions.md` | Activation gate + minimal CTEM-mode rules (auto-loaded every conversation) |
| Flow Control | `/ctem-flow` (prompt) | Session lifecycle, phase transitions, backtrack checks, report generation |
| Phase Execution | `/ctem-*` (skills) | Independent, phase-specific analysis logic |
| State Governance | `ctem-state-protocol.instructions.md` | Read/write format rules for `ctem-state.md` (auto-loaded when state file is accessed) |
| State & Data | `ctem-state.md` | Single source of truth for session progress |
| Reports | `reports/` | Session reports (`sessions/`) and asset profiles (`assets/`) |

## Quick Start

### Prerequisites

- [VS Code](https://code.visualstudio.com/) with [GitHub Copilot](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot-chat) extension
- Copilot Chat with agent mode enabled

### How to Use

#### Option A: Guided Workflow (Recommended)

1. Open this project in VS Code
2. Open Copilot Chat and type:
   ```
   /ctem-flow Start a new CTEM session for target 192.168.1.0/24
   ```
3. The flow controller will read `ctem-state.md`, guide you through all five phases, handle transitions, and manage backtracking

#### Option B: Run Individual Phases

You can run any phase independently as a slash command:

| Command | Phase | What It Does |
|---------|-------|-------------|
| `/ctem-1-scoping` | 1. Scoping | Define target scope and asset inventory |
| `/ctem-2-discovery` | 2. Discovery | Parse tool outputs, identify exposures |
| `/ctem-3-prioritization` | 3. Prioritization | Score and rank exposures by risk |
| `/ctem-4-validation` | 4. Validation | Verify exploitability, filter false positives |
| `/ctem-5-mobilization` | 5. Mobilization | Generate remediation plans and action items |

> **Note**: When running phases individually, you are responsible for managing the phase order and updating `ctem-state.md`. The `/ctem-flow` prompt handles this for you in Option A.

#### Option C: Resume a Session

If you stopped mid-session:

1. Open Copilot Chat
2. Type:
   ```
   /ctem-flow Resume
   ```
3. The flow controller reads `ctem-state.md` and picks up where you left off

#### Common Commands

| What You Want | What to Type |
|---------------|-------------|
| Start new session | `/ctem-flow Start new session for <target>` |
| Continue | `/ctem-flow Resume` |
| After a phase completes | `/ctem-flow Phase complete, next step?` |
| Redo a phase | `/ctem-flow Go back to Validation` |
| Override recommendation | `/ctem-flow Skip to Mobilization` |
| Session summary | `/ctem-flow Summary` |

## Project Structure

```
ctem-kit/
├── .github/
│   ├── copilot-instructions.md          # Activation gate + CTEM-mode global rules
│   ├── instructions/
│   │   └── ctem-state-protocol.instructions.md  # State file read/write format rules
│   ├── prompts/
│   │   └── ctem-flow.prompt.md          # Workflow controller (session lifecycle, backtrack, reports)
│   └── skills/
│       ├── ctem-1-scoping/SKILL.md        # Phase 1: Scoping
│       ├── ctem-2-discovery/SKILL.md      # Phase 2: Discovery
│       ├── ctem-3-prioritization/SKILL.md # Phase 3: Prioritization
│       ├── ctem-4-validation/SKILL.md     # Phase 4: Validation
│       └── ctem-5-mobilization/SKILL.md   # Phase 5: Mobilization
├── reports/
│   ├── README.md                        # Reports directory guide
│   ├── sessions/
│   │   └── TEMPLATE.md                  # Per-session report template
│   └── assets/
│       └── TEMPLATE.md                  # Per-machine asset profile template
├── ctem-state.md                        # Session state tracking (AI-managed)
└── README.md                            # This file
```

### File Roles Explained

| File | Role | Who Uses It |
|------|------|-------------|
| `copilot-instructions.md` | Activation gate: CTEM rules only apply when user enters CTEM context. Contains minimal global rules; delegates details to flow and protocol. | AI (auto-loaded) |
| `ctem-state-protocol.instructions.md` | State file format rules: how to read/write `ctem-state.md`, prerequisite checks. Does NOT contain backtrack logic or report timing. | AI (auto-loaded when `ctem-state.md` is accessed) |
| `ctem-flow.prompt.md` | Workflow controller: session lifecycle, phase transitions, backtrack checks, session start protection, and report generation. Single entry point. | User invokes via `/ctem-flow` |
| `ctem-1-scoping/SKILL.md` | Phase 1 logic. Defines scope, inventories assets, maps attack surface. | User invokes via `/ctem-1-scoping` |
| `ctem-2-discovery/SKILL.md` | Phase 2 logic. Parses scan outputs, identifies exposures. | User invokes via `/ctem-2-discovery` |
| `ctem-3-prioritization/SKILL.md` | Phase 3 logic. Scores and ranks exposures. | User invokes via `/ctem-3-prioritization` |
| `ctem-4-validation/SKILL.md` | Phase 4 logic. Verifies exploitability using three-module approach (reasoning / generation / parsing). | User invokes via `/ctem-4-validation` |
| `ctem-5-mobilization/SKILL.md` | Phase 5 logic. Generates fix plans and tracks remediation. | User invokes via `/ctem-5-mobilization` |
| `ctem-state.md` | Live session state. Tracks which phases are done, findings summaries, and backtrack history. Reset when starting a new session (with protection). | AI reads/writes; user can inspect |
| `reports/sessions/TEMPLATE.md` | Session report template. Copied and filled after each CTEM round completes. | AI generates; user can inspect |
| `reports/assets/TEMPLATE.md` | Asset profile template. One per machine, created/updated when all five phases complete. | AI updates; user can inspect |

## Backtracking

Unlike linear workflows, CTEM requires **non-linear phase transitions**. For example, validating an exposure may reveal new attack surfaces that require re-discovery.

### How It Works

- After each phase, `/ctem-flow` performs a **Backtrack Check**
- It compares new findings against previous phase outputs in `ctem-state.md`
- If backtracking is needed, it recommends which phase to return to and why
- The user can also manually request a backtrack at any time
- Maximum **3 backtracks per session** to prevent infinite loops

### Backtrack Flow

```
Scoping → Discovery → Prioritization → Validation ──→ Mobilization
                                            │
            ┌───────────────────────────────┘
            │ (new exposures found during validation)
            ▼
         Discovery → Prioritization → Validation → Mobilization
```

## Extending the Kit

### Adding Detail to a Phase Skill

Edit the corresponding `SKILL.md` file under `.github/skills/ctem-<phase>/`. Each skill marked with `<!-- TODO -->` is a placeholder ready for full prompt implementation.

### Adding Reference Materials

Create a `references/` folder inside any skill directory:

```
.github/skills/ctem-4-validation/
├── SKILL.md
└── references/
    ├── attack-path-reasoning.md
    ├── exploit-validation.md
    └── result-analysis.md
```

Reference them from `SKILL.md` using relative links: `[Attack Path Reasoning](./references/attack-path-reasoning.md)`

### Adding Assets / Templates

Create an `assets/` folder inside any skill directory for reusable templates:

```
.github/skills/ctem-5-mobilization/
├── SKILL.md
└── assets/
    └── remediation-report-template.md
```

## License

MIT

