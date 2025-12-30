---
layout: post
title: "Cryptography-John-The-Ripper"
date: 2025-12-29
author: Wandipa Marema
---

Today I completed the Cryptography section of the cyber security 101 module. This sequence of the module, was a glimpse into the world of hacking. For two days now I've been learning the fundamentals
of the popular hash cracking tool, known as John the Ripper.

The tool gives one the ability to crack a hash to its original form. Just a side note, for the rest of this post I'll constantly refer to the tool as "john" :).
Before using the tool I had to understand the basic terms one needs to know in order to fully grasp the entire concept of the tool. The tool operates on hashes, which is a process of taking data of an arbitrary length, and converts it to another fixed length form. Though the hash is the output, and the data of random length is the input, there has to be a process to get from one side to the other, and that's done by the assistance of a hash algorithm. I discovered an overwhelmingly huge amount of hashing algorithms out there, but the ones I worked with often during the section were, MD5, and SHA1, which are deprecated for password storage and algorithms such as bcrypt and Argon2 are recommended. In the world of cryptography, utilising hashes is a good way of providing confidentiality, and integrity of private information. So when a bank for instance stores your data as a hash (Passwords, Banking details), this ensures your data isn't exposed despite your banks data breach.
The integrity aspect falls under verification. When you hash a confidential file, it's a fingerprint. When rehashed and the output(the hash) is the same, this notifies us, the file hasn't been tampered with. If not, the file has been modified. A small modification to the file will cause the hash to change dramatically.

Essentially I cracked a basic hash. The task firstly required us to figure out the hash type. This introduced us to hash identifiers, which are tools that deduce the possible hash type for a hash. This could be easily done with a free to use website such as Hash.com, or a famously used python tool called "Hash-Identifier". The target hash for the task used an MD5 hash type. So once I've identified the hash type I can use johns format specific cracking by telling john to use the specified hashing algorithm to use with the "--format" flag. The syntax for this was "john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash01.txt". Many standard hash formats require the "raw" prefix in john (like raw-md5), but formatted hashes like bcrypt don't need it.
I previously thought that "The tool gives one the ability to reverse a hash to its original form..." but that is wrong! This was a realisation for me... My "aha" moment! John isn't reversing hashes to original forms,but it's hashing thousands of words a second and comparing with the target hash! So when I cracked the given hash within the task, the hash represented the word "biscuit". This was something cool, but made me realise why enterprises require passwords to contain special characters and numbers, to diminish the possibility of the hash being cracked. This simple word in the task was found in a wordlist utilised by john, hashed and compared. All this was deduced in a matter of milliseconds. Enterprises demand passwords to have some ambiguity in order for the hashes to be uncrackable. Something I found impulsively frustrating, turns out to be for the security of your very own personal information.

Though that frustration can also be exploited for the benefit of the hacker, as people tend to predictably add numbers and special characters needed to the end of a password, and a capital letter at the beginning. For instance a website I want to make an account with,
demands me to create a password, and I add my name, "wandipa". I know, this is an easily crackable password but bare with me for the example. When submitting this password as is, due to the website's enforced password complexity, I would receive a prompt demanding my password contains the following. One uppercase letter minimum, one lowercase letter minimum, number(123...), and a special character(!@#$%...).

Me not wanting to give, as I believe an insignificant thing the time or effort to create an uncrackable password, worthy of being praised! I just do the minimum, by Capitalising the first letter of my password,
and adding the s1ngle number and $pecial character to the end, "Wandipa7#". Password complexity is essential, but hackers can exploit my predictability, of where I put my Capital letter, number, and special characters. This is known as "Password Complexity predictability". Hackers can make custom rules for john to follow, to enable it to hash words in certain patterns.

Though despite avoiding all this, adding a sufficinet password that isn't crackable. If it happens that two users have the same password which is definitely possible. They would have the same hash, this is why modern systems incorporate salting, adding random data before hashing, so even identical passwords will have different hashes. This makes rainbow tables, which is a database containing precomputed hashes, useless!  

John has the ability to crack zip files and rar files, which are files managed by the winrar folder manager. This was all a lot of fun and I'm excited to move on to the next topic. Exploitation Basics!


