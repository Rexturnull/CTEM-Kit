# Remediation Playbooks

This reference provides per-exposure-type remediation templates and verification procedures. It is loaded on-demand by the Mobilization skill when a user requests detailed remediation steps for a specific action (Step 1 expansion) or during in-session quick-fix verification (Step 5).

## Usage

When the user asks to expand a specific action (e.g., "expand ACT-001"), locate the relevant section by the exposure's **Type** (vulnerability / misconfiguration / information-disclosure / outdated-software) and fill in target-specific details from upstream data (OS, service version, configuration).

---

## Part 1 — Vulnerability Remediation

Applies to exposures with Type = `vulnerability`. Action Type is typically `patch`.

### 1.1 Software Upgrade

The primary remediation for known CVEs is upgrading to a patched version.

**Package Manager Commands (by OS):**

| OS / Distro | Update Index | Upgrade Specific Package | Upgrade All Security Patches |
|-------------|-------------|--------------------------|------------------------------|
| Debian / Ubuntu | `sudo apt update` | `sudo apt install --only-upgrade <package>` | `sudo apt upgrade` |
| RHEL / CentOS / Fedora | `sudo yum check-update` | `sudo yum update <package>` | `sudo yum update --security` |
| Alpine | `sudo apk update` | `sudo apk upgrade <package>` | `sudo apk upgrade` |
| Arch | `sudo pacman -Sy` | `sudo pacman -S <package>` | `sudo pacman -Syu` |

**Common Service-Specific Upgrades:**

| Service | Typical Package Name | Post-Upgrade Action |
|---------|---------------------|---------------------|
| Apache HTTP Server | `apache2` (Debian) / `httpd` (RHEL) | `sudo systemctl restart apache2` or `httpd` |
| Nginx | `nginx` | `sudo systemctl restart nginx` |
| OpenSSH | `openssh-server` | `sudo systemctl restart sshd` |
| MySQL / MariaDB | `mysql-server` / `mariadb-server` | `sudo systemctl restart mysql` or `mariadb` |
| PostgreSQL | `postgresql` | `sudo systemctl restart postgresql` |
| PHP | `php` / `libapache2-mod-php` | `sudo systemctl restart apache2` |
| ProFTPD / vsftpd | `proftpd` / `vsftpd` | `sudo systemctl restart proftpd` or `vsftpd` |

### 1.2 Interim Mitigations

When immediate patching is not feasible (e.g., requires maintenance window), apply temporary controls:

| Mitigation Type | Example | Notes |
|----------------|---------|-------|
| **WAF Rule** | Add ModSecurity rule blocking the exploit pattern | Reduces attack surface but does not eliminate the root cause |
| **Firewall ACL** | Restrict access to vulnerable service by IP/network | `sudo ufw allow from <trusted_network> to any port <port>` |
| **Service Isolation** | Bind service to localhost or internal interface only | Modify service config: `Listen 127.0.0.1:<port>` |
| **Service Disable** | Temporarily stop non-essential vulnerable service | `sudo systemctl stop <service> && sudo systemctl disable <service>` |

### 1.3 Verification Steps (Vulnerability)

After applying a patch or upgrade:

1. **Version check**: Confirm the installed version is at or above the patched version.
   ```
   <package> --version
   # or
   dpkg -l | grep <package>
   # or
   rpm -qa | grep <package>
   ```

2. **PoC re-test**: Re-run the same validation command used in Phase 4 to confirm the vulnerability no longer responds.
   - Example (path traversal): `curl -s --path-as-is http://<host>/icons/.%2e/%2e%2e/etc/passwd` → expect 403 or connection refused.
   - Example (CVE-specific): Re-run the Nuclei template that originally detected the CVE.

3. **Service status**: Ensure the service restarted cleanly.
   ```
   sudo systemctl status <service>
   ```

---

## Part 2 — Misconfiguration Remediation

Applies to exposures with Type = `misconfiguration`. Action Type is typically `configure` or `harden`.

### 2.1 SSH Hardening

| Issue | Fix | Config File |
|-------|-----|-------------|
| Weak ciphers | Remove weak ciphers from `Ciphers` directive | `/etc/ssh/sshd_config` |
| Weak MACs | Remove weak MACs from `MACs` directive | `/etc/ssh/sshd_config` |
| Weak key exchange | Remove weak algorithms from `KexAlgorithms` directive | `/etc/ssh/sshd_config` |
| Root login enabled | Set `PermitRootLogin no` | `/etc/ssh/sshd_config` |
| Password authentication (when key-based preferred) | Set `PasswordAuthentication no` | `/etc/ssh/sshd_config` |

**Recommended SSH hardening configuration block:**

```
# /etc/ssh/sshd_config — hardened settings
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,umac-128-etm@openssh.com
PermitRootLogin no
```

**Post-change**: `sudo sshd -t && sudo systemctl restart sshd`
(Always test config syntax before restart to avoid lockout.)

### 2.2 FTP Security

| Issue | Fix | Config File |
|-------|-----|-------------|
| Anonymous login enabled | Disable anonymous access | vsftpd: `anonymous_enable=NO` in `/etc/vsftpd.conf`; ProFTPD: remove or comment `<Anonymous>` block in `/etc/proftpd/proftpd.conf` |
| No chroot jail | Enable chroot | vsftpd: `chroot_local_user=YES`; ProFTPD: `DefaultRoot ~` |
| No TLS/SSL | Enable FTPS | vsftpd: `ssl_enable=YES`, configure cert paths; ProFTPD: enable `mod_tls`, configure `TLSEngine on` |
| Writable by anonymous | Disable anonymous upload | vsftpd: `anon_upload_enable=NO` |

**Post-change**: `sudo systemctl restart vsftpd` or `proftpd`

**Migration note**: If FTP is not business-critical, consider replacing FTP with SFTP (SSH-based) entirely — Action Type = `replace`.

### 2.3 Web Server Hardening

| Issue | Fix | Apache | Nginx |
|-------|-----|--------|-------|
| Directory listing enabled | Disable directory browsing | `Options -Indexes` in `<Directory>` or `.htaccess` | `autoindex off;` in location block |
| Server version exposed | Hide version info | `ServerTokens Prod` + `ServerSignature Off` in `httpd.conf` | `server_tokens off;` in `nginx.conf` |
| Missing security headers | Add security headers | `Header set X-Content-Type-Options "nosniff"` etc. via `mod_headers` | `add_header X-Content-Type-Options "nosniff";` etc. |
| HTTP TRACE enabled | Disable TRACE method | `TraceEnable Off` | TRACE is off by default in Nginx |
| Default/sample pages present | Remove default content | Remove `/var/www/html/index.html` or default vhost | Remove default server block |

**Common security headers to add:**

```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 0
Referrer-Policy: strict-origin-when-cross-origin
Content-Security-Policy: default-src 'self'
Permissions-Policy: geolocation=(), camera=(), microphone=()
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

**Post-change**: Test config syntax, then restart:
- Apache: `sudo apachectl configtest && sudo systemctl restart apache2`
- Nginx: `sudo nginx -t && sudo systemctl restart nginx`

### 2.4 Verification Steps (Misconfiguration)

1. **SSH**: Attempt connection with a weak cipher to confirm rejection.
   ```
   ssh -o Ciphers=aes128-cbc -o MACs=hmac-sha1 <user>@<host>
   # Expected: connection rejected / no matching cipher
   ```

2. **FTP anonymous**: Attempt anonymous login.
   ```
   ftp <host>
   # Login as: anonymous / any password
   # Expected: 530 Login incorrect
   ```

3. **Web server headers**: Check response headers.
   ```
   curl -sI http://<host>/ | grep -iE "server:|x-content-type|x-frame"
   # Expected: no version in Server header; security headers present
   ```

4. **Directory listing**: Request a directory path without an index file.
   ```
   curl -s http://<host>/icons/
   # Expected: 403 Forbidden (not a directory listing)
   ```

---

## Part 3 — Information Disclosure Remediation

Applies to exposures with Type = `information-disclosure`. Action Type is typically `configure` or `harden`.

### 3.1 Sensitive File Removal

| File Type | Common Locations | Remediation |
|-----------|-----------------|-------------|
| Credential files | Web root, FTP root, home directories | Remove file and **rotate all exposed credentials immediately** |
| Backup files | `.bak`, `.old`, `.swp`, `~` files in web root | Remove from web-accessible paths; store backups outside web root |
| `.git` directory | Web root `/.git/` | Remove `.git` from web root or block access via web server config |
| `.env` files | Web root `/.env` | Remove from web root; use environment variables or config outside web root |
| phpinfo() pages | `/info.php`, `/phpinfo.php` | Remove the file; disable `phpinfo()` in production |

**Blocking access via web server (alternative to removal):**

Apache (`.htaccess` or vhost config):
```
<DirectoryMatch "^/.*/\.git">
    Require all denied
</DirectoryMatch>
<FilesMatch "\.(bak|old|swp|env)$">
    Require all denied
</FilesMatch>
```

Nginx:
```
location ~ /\.git {
    deny all;
    return 404;
}
location ~ \.(bak|old|swp|env)$ {
    deny all;
    return 404;
}
```

### 3.2 Banner / Version Hiding

| Service | How to Hide Version | Config Location |
|---------|---------------------|-----------------|
| Apache | `ServerTokens Prod` + `ServerSignature Off` | `/etc/apache2/conf-enabled/security.conf` or `httpd.conf` |
| Nginx | `server_tokens off;` | `/etc/nginx/nginx.conf` (http block) |
| OpenSSH | Modify `Banner` directive; version is in protocol header and cannot be fully hidden | `/etc/ssh/sshd_config` — `Banner none`; consider `DebianBanner no` on Debian |
| ProFTPD | `ServerIdent off` or `ServerIdent on "FTP Server"` | `/etc/proftpd/proftpd.conf` |
| vsftpd | `ftpd_banner=FTP Service` | `/etc/vsftpd.conf` |
| MySQL | `skip-show-database` in `[mysqld]` section | `/etc/mysql/my.cnf` |
| PHP | `expose_php = Off` | `/etc/php/<version>/apache2/php.ini` |

### 3.3 Error Page Customization

| Issue | Fix |
|-------|-----|
| Stack traces in error responses | Disable debug/development mode; configure production error handling |
| Detailed error messages | Use generic error pages (403, 404, 500) without internal details |

Apache custom error pages:
```
ErrorDocument 403 /error/403.html
ErrorDocument 404 /error/404.html
ErrorDocument 500 /error/500.html
```

Nginx custom error pages:
```
error_page 403 /error/403.html;
error_page 404 /error/404.html;
error_page 500 502 503 504 /error/50x.html;
```

### 3.4 Verification Steps (Information Disclosure)

1. **Sensitive file access**: Attempt to retrieve the previously exposed file.
   ```
   curl -s http://<host>/<path-to-file>
   # Expected: 403 Forbidden or 404 Not Found
   ```

2. **Git directory**: Attempt to access `.git/HEAD`.
   ```
   curl -s http://<host>/.git/HEAD
   # Expected: 403 or 404 (not the git ref content)
   ```

3. **Banner check**: Connect to the service and inspect the banner.
   ```
   nmap -sV -p <port> <host>
   # Check: version info should be minimal or generic
   ```

4. **Error page**: Trigger an error and inspect the response body.
   ```
   curl -s http://<host>/nonexistent-page-12345
   # Expected: generic error page without stack traces or internal paths
   ```

---

## Part 4 — Outdated Software Remediation

Applies to exposures with Type = `outdated-software`. Action Type is typically `patch` (minor upgrade) or `replace` (major version / EOL).

### 4.1 Software Upgrade Paths

| Scenario | Approach | Action Type |
|----------|----------|-------------|
| Minor/patch version behind | Upgrade via package manager (see Part 1.1) | `patch` |
| Major version behind but still supported | Plan major version upgrade with backup + testing | `patch` |
| End-of-Life (EOL) software | Migrate to supported alternative | `replace` |

### 4.2 Common EOL Alternatives

| EOL Software | Recommended Alternative |
|-------------|------------------------|
| Python 2.x | Python 3.x |
| PHP 7.x | PHP 8.x |
| CentOS 8 | Rocky Linux / AlmaLinux / RHEL |
| MySQL 5.x | MySQL 8.x or MariaDB 10.x+ |
| Apache 2.2 | Apache 2.4 |
| OpenSSL 1.0.x / 1.1.x | OpenSSL 3.x |
| Node.js (non-LTS / EOL versions) | Current LTS version |
| Ubuntu 18.04 / 20.04 (post-ESM) | Ubuntu 22.04 LTS or 24.04 LTS |

### 4.3 Interim Mitigations (When Upgrade Is Not Immediate)

| Mitigation | Description |
|-----------|-------------|
| Network isolation | Restrict access to the outdated service to trusted IPs only |
| WAF / reverse proxy | Place a hardened reverse proxy in front of the outdated service |
| Disable unused features | Turn off modules, plugins, or features that increase attack surface |
| Enhanced monitoring | Increase logging and alerting for the affected service |

### 4.4 Verification Steps (Outdated Software)

1. **Version check**: Confirm the new version is installed and running.
   ```
   <service> --version
   # or check via package manager
   dpkg -l | grep <package>
   rpm -qa | grep <package>
   ```

2. **Service health**: Confirm the service is running correctly after upgrade.
   ```
   sudo systemctl status <service>
   curl -s http://<host>:<port>/ | head -5
   ```

3. **CVE re-check**: Scan with nmap or Nuclei to confirm previously detected CVEs no longer trigger.
   ```
   nmap --script vuln -p <port> <host>
   nuclei -u http://<host> -t <template-that-detected-the-issue>
   ```

---

## Part 5 — Investigation Actions (Inconclusive Exposures)

Applies to exposures with Validation Status = `inconclusive`. Action Type is always `investigate`.

### 5.1 Common Investigation Approaches

| Inconclusive Reason | Recommended Investigation |
|---------------------|--------------------------|
| Connection timeout during PoC | Re-run PoC with increased timeout; try from a different network position |
| Inconsistent results | Run PoC multiple times (3–5 iterations) and analyze variance |
| Partial response (ambiguous) | Try alternative PoC methods targeting the same vulnerability |
| Scanner detection but PoC failed | Compare scanner evidence with manual testing; check for virtual patching or IPS interference |
| Network-level filtering suspected | Test from different source IPs or over VPN; check if IPS/WAF is blocking specific payloads |

### 5.2 Investigation Commands

| Method | Command Template |
|--------|-----------------|
| Re-scan with nmap | `nmap -sV --script vuln -p <port> <host>` |
| Re-run Nuclei template | `nuclei -u http://<host> -t <template-id> -timeout 30` |
| Manual HTTP probe | `curl -v -m 30 <url-with-payload>` |
| Alternative network position | Execute from a different host / VPN / network segment |

### 5.3 Investigation Outcomes

After investigation, the exposure should be reclassified:
- **Confirmed exploitable** → Re-enter remediation queue as `confirmed`; generate full remediation plan
- **Confirmed not exploitable** → Reclassify as `false-positive`; document investigation evidence
- **Still inconclusive** → Carry forward as `inconclusive` with additional investigation notes for the next CTEM session

---

## Playbook Selection Guide

Use this table to quickly identify which section to reference:

| Exposure Type | Primary Section | Action Types |
|---------------|----------------|-------------|
| `vulnerability` | Part 1 | `patch`, `harden` (interim) |
| `misconfiguration` | Part 2 | `configure`, `harden` |
| `information-disclosure` | Part 3 | `configure`, `harden` |
| `outdated-software` | Part 4 | `patch`, `replace` |
| (inconclusive) | Part 5 | `investigate` |
