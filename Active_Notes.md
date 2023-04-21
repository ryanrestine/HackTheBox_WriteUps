# HTB - Active

##### Ip: 10.10.10.100
##### Hostname: active.htb
##### Rating: Easy

------------------------------------------------

/home/ryan/HTB/Active/active_card.png

## Enumeration

As always, lets kick things off with an Nmap scan covering all TCP ports. Because I'm not worried about traffic or the 'noise' I'm making, I'm adding the `--min-rate 10000` attribute to speed things along:

```text
sudo nmap -p-  --min-rate 10000 10.10.10.100
```

As usual with Windows AD boxes, there's quite a few ports open here, with all the tell-tell signs of Windows box (All those RPC ports), and an AD Domain Controller specifically (ports 53,88,389, etc). Here, I am scanning the discovered open ports from our `-p-` scan, and tacking on the `-sC` and `-sV` flags to enumerate versions and run common scripts against the ports:

```text
┌──(ryan㉿kali)-[~/HTB/Active]
└─$ sudo nmap -sC -sV -T4 10.10.10.100 -p 53,88,135,139,389,445,464,593,636,3268,3269,5722,9389,47001,49152,49153,49154,49155,49157,49158,49165,49168,49174
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-21 13:34 CDT
Nmap scan report for 10.10.10.100
Host is up (0.072s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-04-21 18:34:13Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5722/tcp  open  msrpc         Microsoft Windows RPC
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
49165/tcp open  msrpc         Microsoft Windows RPC
49168/tcp open  msrpc         Microsoft Windows RPC
49174/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2023-04-21T18:35:09
|_  start_date: 2023-04-21T01:33:39
| smb2-security-mode: 
|   210: 
|_    Message signing enabled and required
|_clock-skew: -2s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 71.01 seconds
```
Because there doesn't appear to be any http or FTP ports open, let's begin by enumerating the SMB ports (139/445) as well as the LDAP port, to try and find something juicy.

But before we do that, let's go ahead and add active.htb to the /etc/hosts file:

`10.10.10.100  active.htb`

### SMB Enumeration

Lately, my favorite way to enumerate shares (and my ability to read them) is with CrackMapExec. CME is an extremely powerful tool with so many use cases, and is one of my go-to tools when working with active directory (Here is a great cheat sheet/wiki on the tool: https://wiki.porchetta.industries/other-gitbook. In this case I tried enumerating shares with user 'guest' and a blank password (this didn't work, the Guest user had been smartly disabled), as well as a blank user and blank pw, which worked!

/home/ryan/HTB/Active/active_card.png

Ok cool, so looks like I have read access to the Replication share; let's check that out. For this I'm reverting back to the old-school smbclient tool to see what we can find. (CME has some cool ways to further check shares using the spider function, but I find smbclient easier for cases like this)

Running: `smbclient //10.10.10.100/replication` and simply hitting enter in lieu of a password gets me into the share. After poking around a bit I found an interesting file called Groups.xml, and used `get Groups.xml` to pull the file back to my machine. If on the fly, you can always use `more Groups.xml` which essentially just lets you read the file stored in temporary memory in a vim-like setting, but I prefer to store files like this locally, in case I need to revisit them.

This file proves to be super interesting!

/home/ryan/HTB/Active/gpp_pw.png

Here we have what appear to be a username SVC_TGS as well as a password. These Groups.xml files can be a goldmine for penetration testers because they can store extremely sensitive data (like PWs) as a Group Policy Preference (GPP) file, which can be decrypted using various tools. 

### Decrypting the PW Hash

Armed with this knowledge we can use the built in Kali tool gpp-decrypt:

```text
gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
```
/home/ryan/HTB/Active/gpp_decrypt.png

Nice! We were very easily able to crack this password! Seems we may know have some credentials, 
`SVC_TGS:GPPstillStandingStrong2k18`, lets see what we can do with them. 

### Kerberoasting

Going with a hunch based on the username (SVC_TGS or ticket-granting service), I'm going to try Kerberoasting first, and more specifically, because this is a machine account (SVC) I'm going to take a look at SPNs.

Using Impacket's tool GetUserSPNs I can run the following:

`impacket-GetUserSPNs -dc-ip 10.10.10.100 -request active.htb/SVC_TGS`

for an easy win at kerberoasting! Furthermore, this appears to be the administrator hash; we hopefully we can use this cracked credential to login to the target with admin access. 

/home/ryan/HTB/Active/kerberoast.png

### Cracking the Hash With JTR

From here I will simply copy the hash to a text file named kerb_hash, and let John The Ripper do the work for me to crack it:

Most excellent! John was able to crack the hash quite quickly:

/home/ryan/HTB/Active/jtr_crack.png

### Logging In

So now let's try to login with our newly found admin credentials of administrator:Ticketmaster1968.

Once again, Impacket to the rescue; let's try using psexec to login:

```text
impacket-psexec active.htb/administrator:Ticketmaster1968@10.10.10.100
```

And since I am logging in as administrator I can grab both flags on the SVC_TGS and Administrator desktops respectively.

/home/ryan/HTB/Active/user_txt.png

/home/ryan/HTB/Active/root_txt.png

And that's that! This was a great beginner friendly Active Directory box illustrating some key concepts needed to successfully enumerate and attack enterprise environments. 

### Something Extra

I recently found out about this and found it super interesting, so thought I would share here as well- Recall back to screen shot after logging in as administrator, where I ran the `whoami` command and got a response back as `nt authority\system`.

Look what happens when I login using the very same credentials using a tool like wmiexec:

/home/ryan/HTB/Active/wmiexec_login.png

Running `whoami` with this tool only lists me as active/administrator, whereas psexec will auto-elevate privileges to that of nt/system. For all intents and purposes my privileges are the same here, but for those of us that want to document the highest level of privileges (say for an examination, or to include in a report), psexec offers an easy win in that regard. Likely superficial, but interesting none-the-less. 

### Key Takeaways

- SMB can often contain a plethora of 'low hanging fruit' like we found here today. I was able to go from anonymous access to an SMB share to fully compromising the machine (or domain in this AD example) without anything too tricky, technically speaking. Sensitive information like credentials being stored somewhere like an open share like this is obviously not a good idea.

- Kerberoasting is an invaluable part of any attacker's arsenal. Learning about a few of the biggest attacks (golden/ silver tickets, SPNs, AS-REPs, etc) will pay dividends later on. 

- Similarly, Impacket and CrackMapExec are about as comprehensive and valuable tools as anyone could hope for when it comes to attacking Active Directory. These are huge tools that offer so much, and I am still only beginning to scratch there surface of what they can offer. Getting to know your tools is important in any line of work, but especially in pentesting; these are two worth getting to know well, in my opinion. 

Thanks for following along!

-Ryan

------------------------------------------------------------------------