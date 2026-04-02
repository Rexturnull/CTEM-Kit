# Tool Commands Reference — Scoping Phase

Commands the user can run to gather host information during Scoping.
This file is loaded on demand by the Scoping skill when the user needs guidance.

## Host Discovery

Verify the target is reachable:

```bash
nmap -sn <target>
```

- `-sn` — ping scan only, no port scan
- Confirms host is alive before proceeding

## OS Detection

Identify the operating system:

```bash
nmap -O <target>
```

- Requires root/sudo privileges
- Returns OS family, version, and confidence percentage
- If accuracy is low, add `--osscan-guess` for best-effort detection

## Service Enumeration

List open ports and identify running services:

```bash
nmap -sV -p- <target>
```

- `-sV` — probe open ports to determine service/version
- `-p-` — scan all 65535 ports (slower but comprehensive)
- For a faster initial scan, use `-p 1-1024` or `--top-ports 1000`

## Hostname Resolution

Resolve IP to hostname (or vice versa):

```bash
# IP → hostname
nslookup <ip>
dig -x <ip>

# hostname → IP
nslookup <hostname>
dig <hostname>
```

## Quick Combined Scan

If the user wants a single command that covers OS + services:

```bash
sudo nmap -O -sV --top-ports 1000 <target>
```

This provides OS detection and service versions for the most common 1000 ports in one pass.
