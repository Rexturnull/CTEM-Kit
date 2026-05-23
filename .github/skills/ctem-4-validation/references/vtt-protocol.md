# VTT Protocol — Validation Testing Tree Format & Rules

This reference defines the format, states, initialization logic, update rules, and attack path conventions for the Validation Testing Tree (VTT) used in CTEM Phase 4.

## Origin

The VTT is adapted from PentestGPT's PTT (Penetration Testing Tree). PTT uses a layered task structure (`1, 1.1, 1.1.1`) with status tracking to guide penetration testing. VTT applies the same structural concept but starts from a prioritized exposure list rather than blank-target reconnaissance.

---

## VTT Format

### Tree Structure

```
VTT — Validation Testing Tree (Session <ID>, Target: <host>)
1. Validate EXP-XXX: <Title> (Adjusted: <severity>) — [status]
   1.1 <validation sub-task> — [status]
   1.2 <validation sub-task> — [status]
       1.2.1 <deeper sub-task> — [status]
2. Validate EXP-YYY: <Title> (Adjusted: <severity>) — [status]
   2.1 <validation sub-task> — [status]
3. Explore <Service/Port>: <exploration description> — [status]
   3.1 <exploratory sub-task> — [status]
   3.2 <exploratory sub-task> — [status]
N. Attack Path Analysis — [status]
   N.1 AP-001: <chain description> — [status]
   N.2 AP-002: <chain description> — [status]
```

### Naming Conventions

| Element | Format | Example |
|---------|--------|---------|
| Validation root task | `Validate EXP-XXX: <Title> (Adjusted: <severity>)` | `Validate EXP-001: Apache Path Traversal (Adjusted: critical)` |
| Exploratory root task | `Explore <Service/Port>: <description>` | `Explore HTTP/80: Web Application Deep Scan` |
| Attack path root task | `Attack Path Analysis` | Always the last root-level item |
| Attack path entry | `AP-NNN: <chain description>` | `AP-001: FTP anon → cred file → admin login` |

### Task Numbering

- Root-level tasks: sequential integers starting from 1.
- Sub-tasks: dot notation following the parent (`1.1`, `1.2`, `1.2.1`).
- Numbering is positional — if a root task is removed or completed, numbers are not reassigned.
- New root tasks (from newly discovered exposures) are appended before the Attack Path Analysis section.

---

## Task States

| State | Symbol in VTT | Definition |
|-------|--------------|------------|
| `to-do` | `[to-do]` | Not yet started. Available for execution. |
| `in-progress` | `[in-progress]` | Tester is currently executing this task. |
| `completed` | `[completed]` | Results received and analyzed. Task is done. |
| `not-applicable` | `[not-applicable]` | Skipped because preconditions are not met, or superseded by other results. |

### State Transitions

```
to-do → in-progress    (user begins execution)
in-progress → completed (results received and analyzed by VPM/VRM)
to-do → not-applicable  (precondition failed or task no longer relevant)
in-progress → not-applicable (partial execution revealed task is irrelevant)
```

### Root Task Completion

A root-level validation task (e.g., `Validate EXP-001`) is marked `[completed]` when:
- All its sub-tasks are either `completed` or `not-applicable`, AND
- A final validation judgment has been assigned: `confirmed`, `false-positive`, or `inconclusive`.

Append the judgment to the root task:
```
1. Validate EXP-001: Apache Path Traversal (Adjusted: critical) — [completed → confirmed]
```

---

## VTT Initialization

### Input

The Prioritized Exposure List from Phase 3, containing for each exposure:
- Exposure ID, Title, Adjusted Severity, Exploitability classification, Affected Service

The Open Services Detected table from Phase 2, listing all services.

### Initialization Rules

1. **Create validation tasks** — one root task per open exposure, ordered by Adjusted Severity descending (critical first, then high, medium, low). Ties are broken by Raw Severity.

2. **Generate validation sub-tasks per exposure** based on the exposure type:

   | Exposure Type | Typical Sub-tasks |
   |--------------|-------------------|
   | `vulnerability` (with CVE) | Confirm service version, execute specific CVE PoC, test related variants |
   | `misconfiguration` | Verify the misconfigured setting, test if it can be exploited, check for default states |
   | `information-disclosure` | Confirm the information is accessible, assess what is disclosed, check if it enables further attacks |
   | `outdated-software` | Confirm version, search for version-specific exploits, test known PoCs |

3. **Generate exploratory tasks** — one root-level "Explore" task per unique Affected Service (deduplicated from the exposure list + Open Services list). Exploratory tasks cover common attack vectors that automated scanners often miss:

   | Service Type | Exploratory Sub-tasks |
   |-------------|----------------------|
   | **HTTP/HTTPS** | Directory/file enumeration (gobuster/dirb), login page discovery, parameter fuzzing (SQLi, LFI/RFI, XSS), default credential testing on discovered admin panels, robots.txt and sitemap analysis |
   | **FTP** | Anonymous login test, directory listing, sensitive file search (credentials, configs, backups), write permission test |
   | **SSH** | Banner grab, weak/default password testing, key-based auth enumeration |
   | **SMB/CIFS** | Share enumeration, NULL session test, sensitive file search, writable share test |
   | **Database (MySQL/MSSQL/PostgreSQL)** | Default credential test, unauthorized access test, version-specific vulnerabilities |
   | **SMTP/POP3/IMAP** | Open relay test, user enumeration (VRFY/EXPN/RCPT), version-specific vulnerabilities |
   | **DNS** | Zone transfer test, subdomain enumeration |
   | **SNMP** | Default community string test, information enumeration |
   | **Other/Unknown** | Banner grab, version identification, known vulnerability search |

   Do not duplicate: if an exposure already covers a service's primary vulnerability, the exploratory task should focus on additional vectors.

4. **Attack Path Analysis** — always the last root-level item. Initially contains only the header:
   ```
   N. Attack Path Analysis — [to-do]
      (populated dynamically as exposures are confirmed)
   ```

### Initialization Example

```
VTT — Validation Testing Tree (Session S-002, Target: 10.10.10.5)
1. Validate EXP-001: Apache Path Traversal CVE-2021-41773 (Adjusted: critical) — [to-do]
   1.1 Confirm Apache version via banner/headers — [to-do]
   1.2 Execute path traversal PoC (curl) — [to-do]
   1.3 Test RCE variant CVE-2021-42013 — [to-do]
   1.4 Test with URL encoding variations — [to-do]
2. Validate EXP-002: FTP Anonymous Login (Adjusted: high) — [to-do]
   2.1 Test anonymous login — [to-do]
   2.2 Enumerate accessible directories — [to-do]
   2.3 Search for sensitive files (credentials, configs) — [to-do]
   2.4 Test write permissions — [to-do]
3. Validate EXP-003: SSH Weak Ciphers (Adjusted: low) — [to-do]
   3.1 Enumerate supported ciphers/algorithms — [to-do]
   3.2 Attempt connection using weak cipher — [to-do]
4. Explore HTTP/80: Web Application Deep Scan — [to-do]
   4.1 Directory enumeration (gobuster) — [to-do]
   4.2 Discover and test login pages — [to-do]
   4.3 Test for SQL injection on discovered parameters — [to-do]
   4.4 Test for LFI/RFI on discovered parameters — [to-do]
   4.5 Check robots.txt and sitemap — [to-do]
5. Explore FTP/21: Extended File System Analysis — [to-do]
   5.1 Deep directory traversal — [to-do]
   5.2 Check for backup files and archives — [to-do]
6. Attack Path Analysis — [to-do]
   (populated dynamically as exposures are confirmed)
```

---

## VTT Update Rules

### After Each Validation Result

1. **Mark the specific sub-task** as `[completed]`.
2. **If the result reveals new sub-tasks** (e.g., a directory listing reveals new paths to test):
   - Add new sub-tasks under the appropriate parent.
   - Number them sequentially after existing siblings.
3. **If the result makes other tasks irrelevant** (e.g., service is confirmed patched):
   - Mark those tasks as `[not-applicable]`.
4. **If a NEW FINDING is discovered**:
   - For minor: create a new root-level validation task (before Attack Path Analysis) with appropriate sub-tasks. Mark it `[to-do]`.
   - For major: add a note in the VTT and alert the user.
5. **If an exposure is confirmed**: check the Attack Path Analysis section for potential chains. If a chain becomes possible, add it.

### VTT Pruning

To keep the VTT readable:
- Collapse fully completed root tasks to a single line with their judgment:
  ```
  1. Validate EXP-001: Apache Path Traversal (Adjusted: critical) — [completed → confirmed]
  ```
- Only expand active (to-do / in-progress) tasks when displaying the full VTT.
- When displaying incremental updates, show only: the completed task, any changes, and the next task.

---

## Attack Path Format

### Path Entry

```
AP-NNN: <short chain description>
  Chain: EXP-XXX → EXP-YYY → EXP-ZZZ
  Steps:
    1. <first step description> (EXP-XXX: <status>)
    2. <second step description> (EXP-YYY: <status>)
    3. <third step description> (EXP-ZZZ: <status>)
  Combined Impact: <severity level>
  Status: confirmed / partial / theoretical
```

### Path Status Definitions

| Status | Definition |
|--------|-----------|
| `confirmed` | Every link in the chain has been validated (all referenced exposures are `confirmed`). |
| `partial` | Some links validated, others remain `to-do` or `inconclusive`. |
| `theoretical` | Chain logic is sound based on confirmed exposures, but the connecting steps have not been tested. |

### Path ID Assignment

- Format: `AP-NNN` (zero-padded three digits).
- Sequential within the session, starting at `AP-001`.
- Once assigned, an AP-ID is not reused even if the path is later invalidated.

### Combined Impact Assessment

The combined impact of an attack chain is assessed as follows:
- **Base**: the highest Adjusted Severity among all exposures in the chain.
- **Escalation**: if the chain achieves access beyond what any individual exposure provides (e.g., individual exposures are `high` but the chain achieves full admin access), the combined impact may be escalated up to one level above the base.
- **Cap**: combined impact cannot exceed `critical`.

---

## Dependency Tracking

### Parallel vs. Sequential Tasks

VRM should classify each to-do task as:

| Type | Criteria | User Guidance |
|------|----------|---------------|
| **Parallelizable** | No dependency on the outcome of other to-do tasks. Different services or independent tests. | List all at once; user picks execution order. |
| **Sequential** | Depends on the result of another task. E.g., "test credentials found in FTP" depends on FTP enumeration. | Present only after the prerequisite is completed. |

### Dependency Notation (internal)

When a task depends on another, VRM notes it internally:
```
4.3 Test for SQLi on /search page — [to-do] (depends: 4.1 directory enum)
```

The dependency annotation is for VRM's internal tracking. When presenting to the user, simply order tasks appropriately and explain dependencies in natural language:
> *"Task 4.3 (SQL injection testing) depends on discovering target pages in task 4.1 (directory enumeration). Please complete 4.1 first."*
