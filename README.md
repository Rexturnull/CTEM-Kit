# CTEM Kit

A prompt-engineering-only toolkit that automates the **Gartner CTEM (Continuous Threat Exposure Management)** five-phase workflow using AI agent skills in GitHub Copilot.

**Zero code. Pure Markdown. Fully AI-driven.**

## What is CTEM?

CTEM (Continuous Threat Exposure Management) is a five-phase framework introduced by Gartner for proactively managing an organization's threat exposure:

1. **Scoping** — Define what's in scope: assets, business criticality, attack surface boundaries
2. **Discovery** — Find exposures: vulnerabilities, misconfigurations, attack surface gaps
3. **Prioritization** — Rank exposures by risk: exploitability × business impact × context
4. **Validation** — Verify exposures are real and exploitable, filter false positives
5. **Mobilization** — Generate remediation plans, assign actions, track resolution

## Quick Start

### Prerequisites

- [VS Code](https://code.visualstudio.com/) with [GitHub Copilot](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot-chat) extension
- Copilot Chat with agent mode enabled

### How to Use

#### Option A: Full Automated Workflow (Recommended)

1. Open this project in VS Code
2. Open Copilot Chat and select the **`@ctem-coordinator`** agent
3. Tell it your target:
   ```
   @ctem-coordinator Start a new CTEM session for target 192.168.1.0/24
   ```
4. The coordinator will guide you through all five phases, handle transitions, and manage backtracking automatically

#### Option B: Run Individual Phases

You can run any phase independently as a slash command:

| Command | Phase | What It Does |
|---------|-------|-------------|
| `/ctem-scoping` | 1. Scoping | Define target scope and asset inventory |
| `/ctem-discovery` | 2. Discovery | Parse tool outputs, identify exposures |
| `/ctem-prioritization` | 3. Prioritization | Score and rank exposures by risk |
| `/ctem-validation` | 4. Validation | Verify exploitability, filter false positives |
| `/ctem-mobilization` | 5. Mobilization | Generate remediation plans and action items |

> **Note**: When running phases individually, you are responsible for managing the phase order and updating `ctem-state.md`. The coordinator handles this for you in Option A.

#### Option C: Resume a Session

If you stopped mid-session:

1. Open Copilot Chat → select **`@ctem-coordinator`**
2. Say:
   ```
   @ctem-coordinator Resume the current session
   ```
3. The coordinator reads `ctem-state.md` and picks up where you left off

## Project Structure

```
ctem-kit/
├── .github/
│   ├── copilot-instructions.md          # Global rules for all CTEM interactions
│   ├── instructions/
│   │   └── ctem-state-protocol.instructions.md  # State file read/write rules
│   ├── agents/
│   │   └── ctem-coordinator.agent.md    # Workflow coordinator (manages phases)
│   └── skills/
│       ├── ctem-scoping/SKILL.md        # Phase 1: Scoping
│       ├── ctem-discovery/SKILL.md      # Phase 2: Discovery
│       ├── ctem-prioritization/SKILL.md # Phase 3: Prioritization
│       ├── ctem-validation/SKILL.md     # Phase 4: Validation
│       └── ctem-mobilization/SKILL.md   # Phase 5: Mobilization
├── ctem-state.md                        # Session state tracking (AI-managed)
└── README.md                            # This file
```

### File Roles Explained

| File | Role | Who Uses It |
|------|------|-------------|
| `copilot-instructions.md` | Defines CTEM context and global rules. Automatically loaded in every conversation. | AI (auto-loaded) |
| `ctem-state-protocol.instructions.md` | Rules for reading/writing `ctem-state.md`. Loaded when state file is accessed. | AI (auto-loaded when relevant) |
| `ctem-coordinator.agent.md` | The workflow manager. Decides phase order, handles backtracking, tracks progress. Does NOT do analysis. | User invokes via `@ctem-coordinator` |
| `ctem-scoping/SKILL.md` | Phase 1 logic. Defines scope, inventories assets, maps attack surface. | User invokes via `/ctem-scoping` or coordinator delegates |
| `ctem-discovery/SKILL.md` | Phase 2 logic. Parses scan outputs, identifies exposures. | User invokes via `/ctem-discovery` or coordinator delegates |
| `ctem-prioritization/SKILL.md` | Phase 3 logic. Scores and ranks exposures. | User invokes via `/ctem-prioritization` or coordinator delegates |
| `ctem-validation/SKILL.md` | Phase 4 logic. Verifies exploitability using three-module approach (reasoning / generation / parsing). | User invokes via `/ctem-validation` or coordinator delegates |
| `ctem-mobilization/SKILL.md` | Phase 5 logic. Generates fix plans and tracks remediation. | User invokes via `/ctem-mobilization` or coordinator delegates |
| `ctem-state.md` | Live session state. Tracks which phases are done, findings summaries, and backtrack history. | AI reads/writes; user can inspect |

## Backtracking

Unlike linear workflows, CTEM requires **non-linear phase transitions**. For example, validating an exposure may reveal new attack surfaces that require re-discovery.

### How It Works

- After each phase, the coordinator performs a **Backtrack Check**
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
.github/skills/ctem-validation/
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
.github/skills/ctem-mobilization/
├── SKILL.md
└── assets/
    └── remediation-report-template.md
```

