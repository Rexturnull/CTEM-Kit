# Tool Commands Reference — Discovery Phase

Commands the user can run to gather vulnerability and service information during Discovery.
This file is loaded on demand by the Discovery skill when recommending scans (Step 0).

---

## nmap — Service Detection + Vulnerability Scanning

### Basic Service Scan

Identify open ports and service versions within the scoped port range:

```bash
nmap -sV -sC -p <ports> <target>
```

- `-sV` — probe open ports for service/version info
- `-sC` — run default NSE scripts (includes basic enumeration)
- `-p <ports>` — specify ports from Scoping's In-Scope Services (e.g., `-p 22,80,443`)
- For comprehensive discovery, use `-p-` (all 65535 ports) but warn user it takes longer

### Vulnerability Scan

Run NSE vulnerability scripts against known open ports:

```bash
nmap -sV --script vuln -p <ports> <target>
```

- `--script vuln` — runs all scripts in the "vuln" category
- Detects known CVEs, misconfigurations, and common weaknesses
- Requires open port list — run the basic service scan first if needed

### Combined Service + Vuln Scan (Recommended)

Single command covering both service detection and vulnerability scanning:

```bash
sudo nmap -sV -sC --script vuln -p <ports> -oN nmap-discovery.txt <target>
```

- `-oN nmap-discovery.txt` — save normal output to file (easy to paste or read)
- Requires sudo for certain script operations
- This is the **recommended default command** for Discovery

### Full Port + Vuln Scan

When Scoping boundary is "host-level only" and all ports should be checked:

```bash
sudo nmap -sV -sC --script vuln -p- -oN nmap-full.txt <target>
```

- `-p-` — scan all 65535 ports
- Significantly slower; use only when a full port sweep is needed

### UDP Service Scan

When In-Scope Services include UDP ports (e.g., DNS/53, SNMP/161, NTP/123):

```bash
sudo nmap -sU -sV --top-ports 50 -oN nmap-udp.txt <target>
```

- `-sU` — UDP scan (requires root/sudo)
- `--top-ports 50` — scan the 50 most common UDP ports (balance between coverage and speed)
- For specific UDP ports: `sudo nmap -sU -sV -p 53,161,123 -oN nmap-udp.txt <target>`
- UDP scanning is significantly slower than TCP — warn the user to expect longer scan times
- Can be combined with TCP: `sudo nmap -sS -sU -sV -p T:22,80,443,U:53,161 <target>`

### Output Formats

| Flag | Format | Best For |
|------|--------|----------|
| `-oN <file>` | Normal text | Pasting into chat, human reading |
| `-oX <file>` | XML | Structured parsing, large outputs |
| `-oG <file>` | Grepable | Quick filtering with grep/awk |
| `-oA <base>` | All three | Comprehensive archival |

Recommend `-oN` for most users (easy to read and paste). For large scans, suggest `-oX` for structured parsing.

---

## Nuclei — Template-Based Vulnerability Scanning

### Basic Scan

Run all default templates against the target:

```bash
nuclei -u <target> -o nuclei-results.txt
```

- `-u <target>` — single target URL or IP
- `-o <file>` — save results to file
- Runs all enabled templates by default

### Scan with Severity Filter

Focus on findings above a certain severity threshold:

```bash
nuclei -u <target> -severity critical,high,medium -o nuclei-results.txt
```

- `-severity` — filter by severity level (comma-separated)
- Reduces noise from informational findings during initial assessment

### Web-Specific Scan

When the target hosts web services (HTTP/HTTPS):

```bash
nuclei -u http://<target> -t http/ -o nuclei-web.txt
nuclei -u https://<target> -t http/ -o nuclei-web-ssl.txt
```

- `-t http/` — use only HTTP-related templates
- Run against both HTTP and HTTPS if both are in scope
- Covers: misconfigurations, exposed panels, default credentials, web CVEs

### JSON Output (Recommended for Parsing)

```bash
nuclei -u <target> -jsonl -o nuclei-results.jsonl
```

- `-jsonl` — one JSON object per line (JSON Lines format)
- Easier for structured parsing than plain text
- Each line contains: template-id, severity, matched-at, CVE info, description

### CVE-Specific Scan

If targeting known CVEs:

```bash
nuclei -u <target> -t cves/ -o nuclei-cves.txt
```

- `-t cves/` — run only CVE detection templates
- Good for verifying specific known vulnerabilities

### Update Templates

Ensure templates are up to date before scanning:

```bash
nuclei -update-templates
```

- Always recommend running this before a scan if the user hasn't recently updated

---

## Recommended Scan Sequence

For a typical Discovery session, recommend this sequence:

### Step 1: nmap Service + Vuln Scan
```bash
sudo nmap -sV -sC --script vuln -p <scoped-ports> -oN nmap-discovery.txt <target>
```

### Step 2: Nuclei Full Scan
```bash
nuclei -update-templates && nuclei -u <target> -jsonl -o nuclei-results.jsonl
```

### Step 3 (if web services in scope): Nuclei Web Scan
```bash
nuclei -u http://<target> -t http/ -jsonl -o nuclei-web.jsonl
```

Tell the user:
> *"Share the results when done — paste the output directly or tell me the file path. If you only have access to one of these tools, that's fine — share what you have and I'll work with it."*
