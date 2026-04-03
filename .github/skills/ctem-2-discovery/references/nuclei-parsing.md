# Nuclei Output Parsing Guide — Discovery Phase

Rules for extracting vulnerability information from Nuclei scan output.
This file is loaded on demand when the user provides Nuclei scan results.

---

## Table of Contents

1. [Plain Text Output](#plain-text-output)
2. [JSON Lines Output (-jsonl)](#json-lines-output--jsonl)
3. [Severity Mapping](#severity-mapping)
4. [Extraction Rules Summary](#extraction-rules-summary)

---

## Plain Text Output

### Format Recognition

Nuclei's default terminal output follows this pattern:

```
[2026-04-03T10:15:30] [apache-path-traversal] [http] [critical] http://10.0.0.5:80/cgi-bin/.%2e/%2e%2e/etc/passwd
[2026-04-03T10:15:31] [cve-2021-41773] [http] [high] http://10.0.0.5:80/icons/.%2e/%2e%2e/etc/passwd
[2026-04-03T10:15:32] [ssh-weak-ciphers] [network] [medium] 10.0.0.5:22
[2026-04-03T10:15:33] [http-missing-security-headers:x-frame-options] [http] [info] http://10.0.0.5:80
[2026-04-03T10:15:34] [tech-detect:apache] [http] [info] http://10.0.0.5:80
[2026-04-03T10:15:35] [ssl-dns-names] [ssl] [info] https://10.0.0.5:443
```

### Line Format

Each line follows this structure:
```
[timestamp] [template-id] [protocol] [severity] matched-url-or-host
```

### Extraction Rules

| Field | Position | How to Extract |
|-------|----------|---------------|
| **Timestamp** | 1st bracket | ISO 8601 timestamp of the finding |
| **Template ID** | 2nd bracket | Nuclei template identifier (e.g., `cve-2021-41773`, `ssh-weak-ciphers`) |
| **Protocol** | 3rd bracket | Protocol category (e.g., `http`, `network`, `ssl`, `dns`) |
| **Severity** | 4th bracket | One of: `critical`, `high`, `medium`, `low`, `info` |
| **Matched At** | After brackets | URL or host:port where the finding was detected |

### Template ID to CVE Mapping

Many Nuclei templates encode CVE information in the template ID:

| Template ID Pattern | CVE | Action |
|--------------------|-----|--------|
| `cve-YYYY-NNNNN` | Extract directly: `CVE-YYYY-NNNNN` | Record the CVE |
| `CVE-YYYY-NNNNN` | Same — uppercase variant | Record the CVE |
| Other (e.g., `ssh-weak-ciphers`) | No CVE | Record CVE as "—" |

### Template ID to Exposure Type Mapping

| Template ID Pattern | Exposure Type |
|--------------------|---------------|
| `cve-*` or `CVE-*` | `vulnerability` |
| `*-weak-*`, `*-misconfig*`, `*-default-*` | `misconfiguration` |
| `*-detect*`, `tech-detect*`, `*-disclosure*` | `information-disclosure` |
| `*-version*`, `outdated-*` | `outdated-software` |
| `http-missing-security-headers*` | `misconfiguration` |
| `ssl-*` (weak ciphers, expired) | `misconfiguration` |
| `exposed-*`, `*-panel*` | `information-disclosure` |

When a template ID doesn't clearly match a pattern, infer from the severity and protocol:
- `critical` or `high` severity → likely `vulnerability`
- `info` severity → likely `information-disclosure`
- For ambiguous cases, present to the user for classification

### Extracting Affected Service from Matched URL

| Matched At Format | Port | Service |
|------------------|------|---------|
| `http://host:80/...` | 80 | HTTP |
| `https://host:443/...` | 443 | HTTPS |
| `http://host:8080/...` | 8080 | HTTP |
| `host:22` | 22 | SSH |
| `host:3306` | 3306 | MySQL |
| `http://host/...` (no port) | 80 (default) | HTTP |
| `https://host/...` (no port) | 443 (default) | HTTPS |

### Sub-Template Findings

Some templates produce multiple findings delimited by `:` in the template ID:

```
[http-missing-security-headers:x-frame-options] [http] [info] http://10.0.0.5
[http-missing-security-headers:x-content-type-options] [http] [info] http://10.0.0.5
```

By default, record each sub-finding as a **separate exposure** — they represent distinct misconfigurations. However, during Step 3a (Deduplication), the user may choose to merge same-category sub-findings into a single exposure (see SKILL.md § 3a). Use the full template ID (including sub-component) as part of the title:
- Title: "Missing Security Header: X-Frame-Options"
- Title: "Missing Security Header: X-Content-Type-Options"

---

## JSON Lines Output (-jsonl)

### Format Recognition

When the user provides JSONL output (from `nuclei -jsonl`), each line is a JSON object:

```json
{
  "template-id": "cve-2021-41773",
  "template-path": "/path/to/templates/cves/2021/CVE-2021-41773.yaml",
  "info": {
    "name": "Apache HTTP Server 2.4.49 - Path Traversal",
    "author": ["pdteam"],
    "tags": ["cve", "cve2021", "lfi", "apache", "misconfig"],
    "description": "A flaw was found in a change made to path normalization in Apache HTTP Server 2.4.49...",
    "severity": "high",
    "classification": {
      "cvss-metrics": "CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N",
      "cvss-score": 7.5,
      "cve-id": ["CVE-2021-41773"],
      "cwe-id": ["CWE-22"]
    },
    "reference": [
      "https://nvd.nist.gov/vuln/detail/CVE-2021-41773",
      "https://httpd.apache.org/security/vulnerabilities_24.html"
    ]
  },
  "type": "http",
  "host": "http://10.0.0.5:80",
  "matched-at": "http://10.0.0.5:80/cgi-bin/.%2e/%2e%2e/etc/passwd",
  "timestamp": "2026-04-03T10:15:31+08:00",
  "matcher-status": true
}
```

### JSON Field Extraction

| Discovery Field | JSON Path | Notes |
|----------------|-----------|-------|
| **Template ID** | `template-id` | Used for dedup and type inference |
| **Title** | `info.name` | Use as-is — Nuclei template names are descriptive |
| **Raw Severity** | `info.severity` | Maps directly to our severity scale |
| **CVSS Score** | `info.classification.cvss-score` | If available, use for severity mapping (overrides label) |
| **CVE** | `info.classification.cve-id[0]` | First CVE in the array; record "—" if null/empty |
| **CWE** | `info.classification.cwe-id[0]` | Optional — useful context for Prioritization |
| **Description** | `info.description` | Useful for exposure details |
| **Protocol** | `type` | `http`, `network`, `ssl`, etc. |
| **Matched At** | `matched-at` | URL or host:port — extract port/service from this |
| **Host** | `host` | Target host — cross-reference with Scoping |
| **Tags** | `info.tags` | Help determine Exposure Type |
| **References** | `info.reference` | External links for more info |
| **Matcher Status** | `matcher-status` | Must be `true` — skip if `false` |

### JSON Tags to Exposure Type Mapping

Use the `info.tags` array for more accurate type classification:

| Tag | Exposure Type |
|-----|---------------|
| `cve`, `cve20XX` | `vulnerability` |
| `misconfig`, `misconfiguration` | `misconfiguration` |
| `exposure`, `disclosure` | `information-disclosure` |
| `tech`, `detect` | `information-disclosure` |
| `outdated` | `outdated-software` |
| `default-login`, `default-credential` | `misconfiguration` |
| `lfi`, `rfi`, `sqli`, `xss`, `rce`, `ssrf` | `vulnerability` |
| `panel`, `login` | `information-disclosure` |

Priority: if tags contain `cve` → `vulnerability` (overrides other tags).

---

## Severity Mapping

### Nuclei Severity to Raw Severity

Nuclei provides severity labels that map directly to our scale:

| Nuclei Severity | Raw Severity |
|----------------|-------------|
| `critical` | `critical` |
| `high` | `high` |
| `medium` | `medium` |
| `low` | `low` |
| `info` | `info` |

### When CVSS Score Is Available (JSON output)

If `info.classification.cvss-score` is present — especially useful when the severity label seems inconsistent:

| CVSS Score | Raw Severity |
|-----------|-------------|
| 9.0 – 10.0 | `critical` |
| 7.0 – 8.9 | `high` |
| 4.0 – 6.9 | `medium` |
| 0.1 – 3.9 | `low` |
| 0.0 | `info` |

**Conflict resolution:** if the CVSS score maps to a different severity than the label, **prefer the CVSS score**. Note the discrepancy in the exposure record.

---

## Extraction Rules Summary

### Processing Flow

1. **Determine format**: plain text or JSONL based on input structure.
2. **For each finding** (each line in text, each JSON object in JSONL):
   a. Extract Template ID and Severity.
   b. Skip if `matcher-status` is `false` (JSON) or line format doesn't match expected pattern.
   c. Determine CVE (from template ID or JSON `cve-id` field).
   d. Determine Exposure Type (from template ID patterns, tags, or severity).
   e. Extract Affected Service from the matched URL/host.
   f. Compile into an Exposure record.
3. **Deduplicate**: same template-id + same host:port = same finding (keep one, note multiple matches if URLs differ).
4. **Present** the full table to user for confirmation.

### Nuclei Finding → Exposure Record Mapping

| Nuclei Field | Exposure Field |
|-------------|---------------|
| `info.name` or template-id | Title |
| (see type mapping tables) | Type |
| `info.severity` or CVSS score | Raw Severity |
| `info.classification.cve-id` or template-id pattern | CVE |
| Extracted from `matched-at` | Affected Service |
| "nuclei" | Source Tool |
