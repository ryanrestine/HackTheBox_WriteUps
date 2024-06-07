# HTB - OpenAdmin

#### Ip: 10.10.10.171
#### Name: OpenAdmin
#### Rating: Easy

----------------------------------------------------------------------

![OpenAdmin.png](../assets/openadmin_assets/OpenAdmin.png)

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/OpenAdmin]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV  10.10.10.171   
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-07 11:38 CDT
Nmap scan report for 10.10.10.171
Host is up (0.066s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 4b98df85d17ef03dda48cdbc9200b754 (RSA)
|   256 dceb3dc944d118b122b4cfdebd6c7a54 (ECDSA)
|_  256 dcadca3c11315b6fe6a489347c9be550 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.68 seconds
```

Looking at port 80 we find an Apache landing page:

openadmin_apache.png

Kicking off some directory fuzzing we find several pages:

openadmin_dirs.png

