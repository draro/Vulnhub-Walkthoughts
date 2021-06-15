# Walkthrough VULNHUB --> SYSTEM FAILURE

### Thanks to my friend 0xJin for this funny machine

First step is to scan our Netwok to find the machine IP, in my case the host has ip 192.168.1.53.

We use nmap to scan the target machine with the command:

```bash
nmap -sC -sV -p- 192.168.1.53
```

```bash
└─# nmap -sC -sV -p- 192.168.1.53
Starting Nmap 7.91 ( https://nmap.org ) at 2021-06-15 07:03 EDT
Stats: 0:01:35 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 87.40% done; ETC: 07:04 (0:00:14 remaining)
Nmap scan report for Clover.homenet.telecomitalia.it (192.168.1.53)
Host is up (0.00059s latency).
Not shown: 65527 filtered ports
PORT     STATE  SERVICE    VERSION
20/tcp   closed ftp-data
21/tcp   open   ftp        vsftpd 3.0.2
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 ftp      ftp          4096 Mar 26 16:49 maintenance
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:192.168.1.200
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.2 - secure, fast, stable
|_End of status
22/tcp   open   ssh        OpenSSH 6.7p1 Debian 5+deb8u8 (protocol 2.0)
| ssh-hostkey:
|   1024 bc:a7:bf:7f:23:83:55:08:f7:d1:9a:92:46:c6:ad:2d (DSA)
|   2048 96:bd:c2:57:1c:91:7b:0a:b9:49:5e:7f:d1:37:a6:65 (RSA)
|   256 b9:d9:9d:58:b8:5c:61:f2:36:d9:b2:14:e8:00:3c:05 (ECDSA)
|_  256 24:29:65:28:6e:fa:07:6a:f1:6b:fa:07:a0:13:1b:b6 (ED25519)
80/tcp   open   http       Apache httpd 2.4.10 ((Debian))
| http-robots.txt: 3 disallowed entries
|_/admin /root /webmaster
|_http-server-header: Apache/2.4.10 (Debian)
|_http-title: Site doesn't have a title (text/html).
110/tcp  closed pop3
443/tcp  closed https
5781/tcp closed 3par-evts
8080/tcp closed http-proxy
MAC Address: 08:00:27:30:93:F3 (Oracle VirtualBox virtual NIC)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

Opening the website on the port 80:
![80](./clover/80.PNG?raw=true "port 80")

Let's run a scan with godirb:

```bash
┌──(animale㉿kali)-[~]
└─$ gobuster dir -t 50 --url http://192.168.1.53 --wordlist /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x php,js,txt,html                       130 ⨯
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.1.53
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              js,txt,html,php
[+] Timeout:                 10s
===============================================================
2021/06/15 07:07:38 Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 3]
/status               (Status: 200) [Size: 10]
/javascript           (Status: 301) [Size: 317] [--> http://192.168.1.53/javascript/]
/website              (Status: 301) [Size: 314] [--> http://192.168.1.53/website/]
/robots.txt           (Status: 200) [Size: 105]
/phpmyadmin           (Status: 301) [Size: 317] [--> http://192.168.1.53/phpmyadmin/]
/server-status        (Status: 403) [Size: 277]

===============================================================
2021/06/15 07:12:09 Finished
===============================================================

```

Let's see everything:

- /index.html, /status and /javascript don't give us anything.

- /website load a template and using the inspector we don't find anything interesting

- /robots.txt return as follow:

```
User-agent: *
Allow: /status
Allow: /status-admin

Disallow: /admin
Disallow: /root
Disallow: /webmaster
```

- /phpmyadmin return the phpMyAdmin login

Nothing really usefull there!!!

Let's try to connect to ftp as anonymous

```bash
┌──(root💀kali)-[~]
└─# mkdir /tmp/clover

┌──(root💀kali)-[~]
└─# cd /tmp/clover

┌──(root💀kali)-[/tmp/clover]
└─# ftp 192.168.1.53
Connected to 192.168.1.53.
220 (vsFTPd 3.0.2)
Name (192.168.1.53:animale): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

We can get in the ftp server, now we have to eplore it...

```bash
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Mar 26 16:49 maintenance
226 Directory send OK.
ftp> pwd
257 "/"
ftp> cd maintenance
250 Directory successfully changed.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp            13 Mar 26 16:48 locale.txt
-rw-r--r--    1 ftp      ftp             3 Mar 26 16:47 test.txt
-rw-r--r--    1 ftp      ftp            54 Mar 26 16:49 test2.txt
226 Directory send OK.
```

Let's download those files and see what is inside

```bash
ftp> mget *
mget locale.txt? y
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for locale.txt (13 bytes).
226 Transfer complete.
13 bytes received in 0.01 secs (1.3813 kB/s)
mget test.txt? y
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for test.txt (3 bytes).
226 Transfer complete.
3 bytes received in 0.05 secs (0.0649 kB/s)
mget test2.txt? y
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for test2.txt (54 bytes).
226 Transfer complete.
54 bytes received in 0.04 secs (1.1902 kB/s)
```

On our local machine, we navigate into the folder were data have been downloaded and open them.

```bash

┌──(root💀kali)-[/tmp/clover]
└─# ll
total 12
-rw-r--r-- 1 root root 13 Jun 15 07:33 locale.txt
-rw-r--r-- 1 root root 54 Jun 15 07:33 test2.txt
-rw-r--r-- 1 root root  3 Jun 15 07:33 test.txt

┌──(root💀kali)-[/tmp/clover]
└─# cat locale.txt
cGluZyBwb25n

┌──(root💀kali)-[/tmp/clover]
└─# cat test2.txt
We are under test.
Plese delete FTP anonymous login.

┌──(root💀kali)-[/tmp/clover]
└─# cat test.txt
OK

┌──(root💀kali)-[/tmp/clover]
└─# cat locale.txt| base64 -d
ping pong
```

It doesn't seems really helpful... Maybe we need a more advanced scan...

I tried to re-run gobuster using a different wordlist

```bash
┌──(root💀kali)-[/tmp/clover]
└─# gobuster dir --url http://192.168.1.53 --wordlist /usr/share/seclists/Discovery/Web-Content/dirsearch.txt

===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.1.53
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/dirsearch.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/06/15 07:58:14 Starting gobuster in directory enumeration mode
===============================================================
/.                    (Status: 200) [Size: 3]
/.htaccess-marco      (Status: 403) [Size: 277]
/.htaccess            (Status: 403) [Size: 277]
/.htaccess-dev        (Status: 403) [Size: 277]
/.htaccess-local      (Status: 403) [Size: 277]
/.htaccess.save       (Status: 403) [Size: 277]
/.htaccess.inc        (Status: 403) [Size: 277]
/.htaccess.sample     (Status: 403) [Size: 277]
/.htaccess.old        (Status: 403) [Size: 277]
/.htaccess/           (Status: 403) [Size: 277]
/.htaccess.bak        (Status: 403) [Size: 277]
/.html                (Status: 403) [Size: 277]
/.htaccessOLD         (Status: 403) [Size: 277]
/.htaccess.bak1       (Status: 403) [Size: 277]
/.htm                 (Status: 403) [Size: 277]
/.htaccessBAK         (Status: 403) [Size: 277]
/.htaccess.orig       (Status: 403) [Size: 277]
/.htpasswd.inc        (Status: 403) [Size: 277]
/.htaccess.txt        (Status: 403) [Size: 277]
/.htpasswd-old        (Status: 403) [Size: 277]
/.htpasswd/           (Status: 403) [Size: 277]
/.htaccessOLD2        (Status: 403) [Size: 277]
/.httr-oauth          (Status: 403) [Size: 277]
/.htpasswd.bak        (Status: 403) [Size: 277]
/.php                 (Status: 403) [Size: 277]
/.php3                (Status: 403) [Size: 277]
//                    (Status: 200) [Size: 3]
/CFIDE/               (Status: 200) [Size: 947]
/icons/               (Status: 403) [Size: 277]
/index.html           (Status: 200) [Size: 3]
/javascript/          (Status: 403) [Size: 277]
/javascript           (Status: 301) [Size: 317] [--> http://192.168.1.53/javascript/]
/phpmyadmin/          (Status: 200) [Size: 9166]
/robots.txt           (Status: 200) [Size: 105]
/status               (Status: 200) [Size: 10]
/website              (Status: 301) [Size: 314] [--> http://192.168.1.53/website/]
/website/             (Status: 200) [Size: 10013]
===============================================================
2021/06/15 07:58:16 Finished
===============================================================
```

There is a /CFIDE/ folder... Let's see what is inside

![CFIDE](./clover/CFIDE.PNG?raw=true "CFIDE")
