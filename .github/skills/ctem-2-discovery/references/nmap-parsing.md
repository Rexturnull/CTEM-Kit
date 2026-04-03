# nmap Output Parsing Guide — Discovery Phase

Rules for extracting service and vulnerability information from nmap output.
This file is loaded on demand when the user provides nmap scan results.

---

## Table of Contents

1. [Normal Text Output (-oN)](#normal-text-output--on)
2. [NSE Vulnerability Script Output](#nse-vulnerability-script-output)
3. [XML Output (-oX)](#xml-output--ox)
4. [Extraction Rules Summary](#extraction-rules-summary)

---

## Normal Text Output (-oN)

### Port Table Format

nmap's standard output presents open ports in a table like this:

```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http     Apache httpd 2.4.52 ((Ubuntu))
443/tcp  open  ssl/http Apache httpd 2.4.52 ((Ubuntu))
3306/tcp open  mysql    MySQL 8.0.30
```

### Extraction Rules

For each line in the port table:

| Field | How to Extract |
|-------|---------------|
| **Port** | Number before the `/` (e.g., `22`) |
| **Protocol** | After the `/` before whitespace (e.g., `tcp`) |
| **State** | Second column — only process lines where state is `open` |
| **Service** | Third column (e.g., `ssh`, `http`, `ssl/http`) |
| **Version** | Everything after the service column (e.g., `OpenSSH 8.9p1 Ubuntu 3ubuntu0.6`) |

**Special cases:**
- `ssl/http` → service is `http` (HTTPS), note the SSL wrapper
- `tcpwrapped` → service is wrapped/filtered, record as-is with a note
- `unknown` → record as "unknown" — may warrant manual investigation

### Service Version Cleaning

Extract a clean version string for the Discovery Summary:

| Raw Version String | Clean Version |
|-------------------|---------------|
| `OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)` | `OpenSSH 8.9` |
| `Apache httpd 2.4.52 ((Ubuntu))` | `Apache 2.4.52` |
| `MySQL 8.0.30` | `MySQL 8.0.30` |
| `nginx 1.18.0` | `nginx 1.18.0` |

Rule: keep the product name + major.minor.patch version. Drop OS-specific suffixes and parenthetical details.

### Host Status

Look for the host status line at the top:

```
Nmap scan report for <hostname> (<ip>)
Host is up (0.0015s latency).
```

Extract hostname and IP to cross-reference with Scoping Summary.

---

## NSE Vulnerability Script Output

### Format Recognition

NSE script results appear after the port table, grouped by port. They follow this pattern:

```
PORT   STATE SERVICE
80/tcp open  http
| http-vuln-cve2021-41773:
|   VULNERABLE:
|   Path Traversal in Apache HTTP Server 2.4.49/2.4.50
|     State: VULNERABLE
|     IDs:  CVE:CVE-2021-41773
|       A flaw was found in a change made to path normalization...
|     Disclosure date: 2021-10-05
|     References:
|       https://nvd.nist.gov/vuln/detail/CVE-2021-41773
|_      https://httpd.apache.org/security/vulnerabilities_24.html
```

### Extraction Rules for Vulnerability Findings

| Field | How to Extract |
|-------|---------------|
| **Affected Port** | The port header line above the script output (e.g., `80/tcp`) |
| **Script Name** | Line starting with `\|` followed by script name and colon (e.g., `http-vuln-cve2021-41773`) |
| **Vulnerable Status** | Look for `State: VULNERABLE` — only extract if state is VULNERABLE |
| **CVE** | Line containing `CVE:` followed by the identifier (e.g., `CVE-2021-41773`) |
| **Title** | The descriptive line after `VULNERABLE:` (e.g., "Path Traversal in Apache HTTP Server 2.4.49/2.4.50") |
| **Disclosure Date** | Line starting with `Disclosure date:` |
| **References** | URLs listed under `References:` |

### Vulnerability State Mapping

| nmap State | Action |
|-----------|--------|
| `VULNERABLE` | Record as exposure — type: `vulnerability` |
| `LIKELY VULNERABLE` | Record as exposure — add note "likely vulnerable, not confirmed" |
| `NOT VULNERABLE` | Skip — do not record |
| `ERROR` | Skip — note in parsing log if relevant |

### Common NSE Vuln Scripts and Their Exposure Types

| Script Pattern | Exposure Type |
|---------------|---------------|
| `*-vuln-*` | `vulnerability` |
| `ssl-*` (weak ciphers, expired cert) | `misconfiguration` |
| `http-server-header` | `information-disclosure` |
| `http-title` | `information-disclosure` (if reveals internal info) |
| `ssh-auth-methods` | `information-disclosure` |
| `*-brute` | Skip — these are active attack scripts, not findings |

### Non-Vulnerability NSE Script Output

Some default scripts (`-sC`) produce informational output that may indicate exposures:

```
| ssl-cert: Subject: commonName=example.com
| Not valid after:  2024-01-15T12:00:00
|_ssl-date: TLS randomness does not represent time

| ssh-hostkey:
|   256 SHA256:xxxx (ECDSA)
|_  256 SHA256:yyyy (ED25519)
```

**Extraction rules for informational scripts:**
- `ssl-cert` with expired date → type: `misconfiguration`, title: "Expired SSL Certificate"
- `ssl-enum-ciphers` showing weak ciphers (DES, RC4, export) → type: `misconfiguration`, title: "Weak SSL/TLS Ciphers"
- `http-server-header` revealing server version → type: `information-disclosure`, title: "Server Version Disclosure"

---

## XML Output (-oX)

### When to Use

Process XML when the user provides a `.xml` file path. XML is more reliable for structured parsing than text output.

### Key XML Elements

```xml
<host>
  <address addr="10.0.0.5" addrtype="ipv4"/>
  <hostnames>
    <hostname name="web-prod-01" type="PTR"/>
  </hostnames>
  <ports>
    <port protocol="tcp" portid="80">
      <state state="open"/>
      <service name="http" product="Apache httpd" version="2.4.52"/>
      <script id="http-vuln-cve2021-41773" output="VULNERABLE:...">
        <table key="CVE-2021-41773">
          <elem key="state">VULNERABLE</elem>
          <elem key="title">Path Traversal in Apache HTTP Server</elem>
        </table>
      </script>
    </port>
  </ports>
</host>
```

### XML Extraction Mapping

| Discovery Field | XPath / Element |
|----------------|----------------|
| IP Address | `host/address[@addrtype='ipv4']/@addr` |
| Hostname | `host/hostnames/hostname/@name` |
| Port | `host/ports/port/@portid` |
| Protocol | `host/ports/port/@protocol` |
| Service | `host/ports/port/service/@name` |
| Product | `host/ports/port/service/@product` |
| Version | `host/ports/port/service/@version` |
| Script ID | `host/ports/port/script/@id` |
| Script Output | `host/ports/port/script/@output` |
| CVE | `host/ports/port/script/table/@key` (when starts with "CVE-") |
| Vuln State | `host/ports/port/script/table/elem[@key='state']` |

---

## Extraction Rules Summary

### Service → Discovery Summary Mapping

| nmap Field | Discovery Summary Field |
|-----------|------------------------|
| portid | Port |
| protocol | Protocol |
| service name | Service |
| product + version | Version |
| (compare with Scoping) | In-Scope |

### Vulnerability → Exposure Record Mapping

| nmap Field | Exposure Field |
|-----------|---------------|
| Script title / description | Title |
| (see type mapping table) | Type |
| (not provided by nmap — default to `medium` unless CVE gives CVSS) | Raw Severity |
| CVE identifier | CVE |
| Port/Service | Affected Service |
| "nmap-script" | Source Tool |

**Important:** nmap does not provide CVSS scores. When a CVE is identified:
- Use the CVSS score from your knowledge base if available (most well-known CVEs have documented CVSS scores).
- If the CVSS score is not known, assign Raw Severity as `unconfirmed` and note: "CVSS score not available — cross-reference with NVD (https://nvd.nist.gov/) required." Present the finding to the user and ask them to provide the CVSS score or accept `medium` as a provisional rating.
